# Redis Cluster 故障与切换案例（Failover）

> 目标：把“主从切换、脑裂、哨兵误判、Cluster 迁移/reshard、客户端 MOVED/ASK、跨机房部署”等问题，整理成**可复用的线上排查 + 止血 + 长期治理**的案例库。
>
> 建议阅读路径：
>
> - 基础：[`04-replication-cluster.md`](04-replication-cluster.md)
> - 调优与风险：[`07-pitfalls-tuning.md`](07-pitfalls-tuning.md)
> - 线上案例入口：[`09-production-cases.md`](09-production-cases.md)
>
---

## 一、总览：面试与线上要回答什么

### 1.1 一句话定位

Redis 的“高可用”问题本质是：

- **在故障/网络分区下，写入会不会丢？读到的是不是最新？恢复后怎么收敛？**

### 1.2 高频追问清单

- 主从切换时，**哪些写会丢**？窗口在哪？
- 脑裂时写丢失怎么理解？怎么减少？
- 哨兵为什么会误判？网络分区时会发生什么？
- Cluster reshard / slot 迁移对业务有什么影响？
- 客户端遇到 **MOVED / ASK** 怎么处理？会不会造成抖动？
- 跨机房部署 Redis Cluster 有哪些坑？
- 为什么 Cluster 不适合多 key 事务？跨 slot 怎么办？
- PSYNC / repl backlog 深水区：为什么会退化为全量同步？
- AOF rewrite + 主从同步叠加时，为什么会出现延迟抖动/内存暴涨？

---

## 二、事故排查通用框架（Runbook 骨架）

> 线上故障优先级：**先止血（恢复服务） > 再定位（找根因） > 最后治理（防复发）**

### 2.1 先问 5 个问题

1. 影响面：单分片/单主节点，还是整集群？
2. 现象：读慢/写失败/大量超时/主从断连/集群状态 fail？
3. 时间线：是否刚发版、扩容、迁移、触发了 BGSAVE/AOF rewrite？
4. 数据一致性：是否允许读旧？是否出现写丢？
5. 环境：是否跨机房？是否有网络抖动？

### 2.2 常用命令清单

```bash
# 集群整体
redis-cli -c -h <any-node> CLUSTER INFO
redis-cli -c -h <any-node> CLUSTER NODES
redis-cli -c -h <any-node> CLUSTER SLOTS

# 主从复制
redis-cli -h <node> INFO replication
redis-cli -h <node> ROLE

# 持久化/重写/fork
redis-cli -h <node> INFO persistence
redis-cli -h <node> INFO memory
redis-cli -h <node> LATENCY LATEST

# 慢查询
redis-cli -h <node> SLOWLOG GET 20
redis-cli -h <node> INFO commandstats
```

### 2.3 关键指标（排查方向）

- **复制**：`master_link_status`、`master_last_io_seconds_ago`、`repl_backlog_size`、`repl_backlog_histlen`、`slave_repl_offset`、`master_repl_offset`
- **集群**：`cluster_state`、fail 节点数、`cluster_known_nodes`
- **延迟**：P99、`LATENCY LATEST`、slowlog
- **内存**：`used_memory`、`used_memory_rss`、碎片率、COW 放大

---

## 三、案例 1：主从切换导致数据丢失（写丢窗口）

### 现象

- 故障后自动切主（sentinel/cluster failover）
- 切换完成后，发现**最近几秒/几十秒的数据缺失**

### 根因（机制解释）

Redis 主从复制默认是**异步复制**：

- 写请求先在主节点执行成功返回给客户端
- 再异步把写传播给从节点

因此在以下窗口会丢数据：

```text
主节点已响应客户端成功
  -> 但写尚未复制到从节点
  -> 主节点故障
  -> 从节点提升为新主
  -> 新主没有这部分写
```

### 排查

- 在新主上看 `INFO replication`：`master_repl_offset` / `slave_repl_offset` 是否明显落后
- 关注故障发生前后复制延迟（网络/IO/fork）

### 止血

- 如果业务允许：接受丢失并做补偿（从 DB 重放、重新构建）
- 如果不允许：考虑把关键写从“Redis 作为最终存储”改成“DB 作为准入”，Redis 只做缓存/加速

### 长期治理（减小丢失概率/窗口）

1. **至少 1 个从节点确认后再写成功**（降低丢失概率）

```conf
min-replicas-to-write 1
min-replicas-max-lag 10
```

含义：当主节点检测到可用从节点不足或延迟过大时，**拒绝写入**（宁可失败也不返回“成功但会丢”）。

2. 关键链路采用：
   - “写 DB + 删缓存”
   - 或写入走 MQ/日志保证可重放

### 面试表达模板

> Redis 默认异步复制，所以主从切换存在写丢窗口：主节点返回成功但写尚未同步到从节点就宕机会丢。工程上可通过 min-replicas-to-write 降低丢失概率，或者把 Redis 降级为缓存/加速层，关键写落 DB 或可重放日志。

---

## 四、案例 2：脑裂（Network Partition）导致“写丢失”

### 现象

- 网络分区后出现“两个主”或写入分叉
- 网络恢复后发现一部分写被回滚/丢弃

### 根因（怎么理解写丢）

脑裂常见于：

- Sentinel 模式：分区导致哨兵判断主不可达，从而提升从为新主
- 原主其实还活着，并继续接受写（尤其在客户端仍可达原主的情况下）

网络恢复后会出现：

- 新主的数据被认为是权威
- 原主上的“分叉写”会被新主覆盖/丢弃（重新同步时丢）

### 排查

- 对比两个节点上的 offset / role
- 检查当时网络、哨兵日志、failover 时间线

### 止血

- 立即在业务侧：
  - 限制写入口（只允许写到当前权威主）
  - 或临时降级为只读/返回旧值

### 长期治理

1. 配置 `min-replicas-to-write` 防止孤岛主继续写
2. 跨机房慎用 Sentinel/Cluster 的自动切主（更建议同城多 AZ，异地做灾备）
3. 对“强一致写”不要把 Redis 当最终存储

### 面试表达模板

> 脑裂本质是网络分区导致不同节点对“谁是主”达成不同结论，出现写入分叉。恢复后会以新主为准，旧主上的分叉写会丢。治理核心是避免孤岛主继续对外写（min-replicas-to-write）+ 关键数据落可靠存储。

---

## 五、案例 3：哨兵误判（Sentinel false positive）

### 现象

- 主节点并未真正宕机，但发生 failover
- 客户端出现短暂大量超时/连接重建

### 常见原因

- 网络抖动、丢包、延迟升高
- 主节点在 fork/AOF rewrite 时短暂卡顿（event loop 不能及时回应 PING）
- CPU 打满导致响应慢

### 排查

- 看哨兵日志：ODOWN/SDOWN 的触发原因
- 看 Redis `LATENCY LATEST` 是否有 fork/command 延迟尖刺
- 看当时 CPU/网络指标

### 治理思路

- 调整 sentinel 的超时参数（避免太敏感）
- 控制 fork：
  - 实例别太大
  - rewrite/BGSAVE 放低峰
- 保障主节点 CPU 余量

---

## 六、案例 4：Cluster reshard / slot 迁移导致抖动

### 现象

- 扩容/缩容/迁移期间：
  - 命令延迟升高
  - 客户端错误增多（MOVED/ASK）
  - 某些 key 突发失败或超时

### 原因（机制解释）

- reshard/迁移需要移动 slot 上的 key
- 迁移期间会出现：
  - key 在源节点和目标节点之间“过渡态”
  - 客户端需要重定向
  - 迁移本身占用 IO/CPU/网络

### 止血

- 迁移降速（降低并发迁移/限速）
- 业务侧限流 + 重试（带退避）
- 在低峰窗口迁移

### 长期治理

- 建立标准变更流程：压测、低峰、回滚预案
- 客户端必须支持重定向并做连接池隔离
- 迁移时避免同时做 AOF rewrite / BGSAVE

---

## 七、案例 5：MOVED / ASK 与客户端路由

### 现象

- `MOVED <slot> <host:port>`
- `ASK <slot> <host:port>`

### 标准解释

- **MOVED**：slot 的归属已经稳定改变，客户端应更新路由表并重试
- **ASK**：slot 正在迁移过程中的临时重定向，需要先 `ASKING` 再执行命令

### 追问点

- 为什么会产生 MOVED/ASK？（路由表过期/reshard）
- 客户端是否自动处理？（需要使用支持 Cluster 的 client）
- 业务应该如何重试？（短重试 + 退避 + 幂等）

---

## 八、案例 6：PSYNC / repl backlog 深水区（为什么会全量同步）

### 现象

- 从节点频繁触发全量同步（full resync），主节点 fork 频繁，延迟抖动

### 机制解释

- 增量同步依赖：
  - replication id
  - offset
  - **repl backlog buffer**（主节点环形缓冲区）

当从节点断开时间过长，或者 backlog 太小导致所需增量数据被覆盖，就只能退化到全量同步。

### 排查

- 看 `INFO replication`：
  - `repl_backlog_size`
  - `repl_backlog_histlen`
  - `master_repl_offset` vs `slave_repl_offset`

### 治理

- 合理设置 `repl-backlog-size`（结合写流量和可接受断连时长）
- 降低网络抖动、连接中断
- 从节点初始化/重启尽量错峰

---

## 九、案例 7：AOF rewrite + 主从同步叠加导致抖动/内存暴涨

### 现象

- 延迟尖刺
- RSS 暴涨（COW 放大）
- 主从复制延迟扩大

### 机制解释（关键点）

- AOF rewrite / RDB 需要 fork
- fork 会触发 COW：写越多，复制页越多，RSS 放大
- rewrite 期间主线程仍在处理写，AOF 缓冲、复制缓冲都可能变大

### 治理

- 控制实例大小（不要过大单实例）
- 低峰进行 rewrite
- 预留足够内存（避免被 OOMKilled）
- 评估持久化策略（缓存场景未必需要强持久化）

---

## 十、附录：Cluster 多 key 事务的边界

### 为什么不适合多 key 事务

Redis Cluster 要求多 key 操作必须在同一个 slot，否则会报错。

### 解决思路

- 使用 hash tag：`{user:1}:a`、`{user:1}:b` 强制落同 slot
- 或将事务逻辑上移到应用/DB（避免跨 slot 原子性诉求）

---

## 十一、面试 2 分钟总结

```text
Redis 高可用核心是：主从复制默认异步，所以 failover 有写丢窗口；网络分区会脑裂并导致分叉写在恢复时被回滚。
我会从复制 offset、cluster_state、slowlog、latency、fork/COW 等指标入手定位。
治理上通过 min-replicas-to-write 减少孤岛写、控制实例大小和 rewrite 时机减少误判与抖动，并在 Cluster 迁移时确保客户端正确处理 MOVED/ASK，变更低峰、限速、可回滚。
```
