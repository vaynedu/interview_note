# Redis Cluster 运维治理（Operability）

> 目标：把 Redis（主从/哨兵/Cluster）从“会用”提升到“可稳定运维”。
>
> 你应该能回答：
>
> - 线上怎么扩容/缩容/迁移 slot，如何做到 **低风险、可回滚**？
> - 如何避免 **脑裂、写丢、全量同步风暴、fork 抖动、热点单点**？
> - 监控和告警怎么做？哪些指标一眼能看出问题？
>
> 建议前置阅读：
>
> - 基础机制：[`04-replication-cluster.md`](04-replication-cluster.md)
> - 故障案例：[`15-cluster-failover-cases.md`](15-cluster-failover-cases.md)
> - 调优与坑：[`07-pitfalls-tuning.md`](07-pitfalls-tuning.md)

---

## 一、一句话定位

Redis 运维治理的核心是：

- **把数据与角色（master/slave/slot）变化变成“可控变更”**
- **把故障模式（脑裂、写丢、同步风暴、热点、fork 抖动）变成“可观测+可止血+可演练”**

---

## 二、部署拓扑：同城多 AZ vs 跨机房

### 2.1 重要结论

- **Redis 的主从/Cluster 更适合同城多 AZ**（低延迟、稳定网络）
- 跨城市/跨地域的强一致高可用，成本和复杂度非常高，通常：
  - 异地做灾备（冷备/半同步/定时切）
  - 关键数据走 DB/日志系统保证一致性

### 2.2 为什么跨机房容易出事

- 复制延迟增大（RTT 增大） → 写丢窗口变大
- 网络分区概率更高 → 脑裂风险更高
- 全量同步更容易触发（backlog 不够/断连更频繁）

### 2.3 推荐拓扑（Cluster）

- 至少 3 master（避免单点 slot 集中）
- 每个 master 至少 1 slave
- master 与 slave **跨 AZ 部署**

---

## 三、容量规划：单实例大小与“fork/COW”上限

### 3.1 为什么要限制单实例内存

Redis 的 RDB/AOF rewrite 需要 fork。

- 实例越大 → fork 时间越长 → 主线程越容易抖动
- 写越多 → COW 复制页越多 → RSS 暴涨 → 容器更容易 OOMKilled

### 3.2 经验建议（可按业务修正）

- 单实例内存建议控制在：**4~16GB**（再大就要非常谨慎）
- 必须预留 COW 空间：
  - 极端情况下 RSS 可能接近 `used_memory * 2`（取决于写入量和 rewrite 持续时间）

### 3.3 实战建议

- 大缓存不要做“超大单实例”，而是做：
  - Cluster 水平扩展
  - 业务按域隔离多个 Redis 集群

---

## 四、关键配置：减少写丢与脑裂伤害

### 4.1 减少写丢：拒绝孤岛写

```conf
min-replicas-to-write 1
min-replicas-max-lag 10
```

解释：当主节点发现可用从不足或延迟过大时，**拒绝写**。

取舍：
- ✅ 降低“写成功但最终丢失”的概率
- ❌ 可能在从不可用时短暂不可写（可用性换一致性）

### 4.2 backlog：减少全量同步

```conf
repl-backlog-size 256mb
```

建议：根据写入流量估算：

- `backlog` 至少覆盖“可接受断连时长 × 写入速率”

（例如：写入 20MB/s，希望断连 10s 仍能增量同步，则 backlog 至少 200MB）

---

## 五、运维变更：扩容/缩容/reshard 的标准流程

> 目标：变更必须做到 **可预期、可观测、可回滚**。

### 5.1 变更前检查清单（强烈建议固化）

1. **低峰窗口**（明确开始/结束时间）
2. **禁止并发变更**：不要同时做 rewrite/bgsave、不要同时做大 key 清理
3. 监控面板就绪：
   - 延迟 P99
   - 复制延迟
   - 内存/RSS
   - `blocked_clients`
   - `evicted_keys`
4. 客户端能力确认：
   - 支持 MOVED/ASK
   - 超时、重试退避、幂等
5. 回滚预案：
   - 失败时停止迁移/回退 slot
   - 业务降级开关

### 5.2 扩容（增加节点）

流程：

1. 加节点到集群（不立即迁移 slot）
2. 观察稳定性（连接、CPU、内存）
3. 分批迁移 slot（限速）
4. 观察 MOVED/ASK、延迟抖动
5. 迁移完成后复盘

风险点：
- 迁移期间网络/CPU 升高
- 客户端重定向导致短期错误率升高
- 热点 slot 迁移会导致单节点短期压力

### 5.3 缩容（下线节点）

原则：
- 先迁走 slot，再下线节点

风险点：
- 下线前如果仍持有 slot，会造成访问失败

### 5.4 reshard 限速（核心）

- 不追求“越快越好”，追求“稳”
- 对生产系统，迁移速度建议以“不影响业务 P99”为准

---

## 六、客户端治理：超时、重试、连接池、MOVED/ASK

### 6.1 超时是第一生产力

必须配置：
- Dial timeout
- Read timeout
- Write timeout

避免：
- 下游 Redis 卡住 → goroutine 堆积 → 服务整体雪崩

### 6.2 重试策略（带边界）

- 只对可重试错误重试（网络抖动、MOVED/ASK）
- 必须：退避 + 上限

避免：
- 不加边界的重试会形成“重试风暴”

### 6.3 MOVED/ASK 的工程要求

- 使用支持 Cluster 的客户端（例如 go-redis cluster）
- 对 MOVED：更新路由并重试
- 对 ASK：ASKING 后重试

重点：
- 迁移期间错误率上升是常态，关键在于控制重试与超时

---

## 七、监控与告警：一眼看懂 Redis 健康

### 7.1 必备面板

1. 延迟：P50/P90/P99 + slowlog 数量
2. QPS：`instantaneous_ops_per_sec`
3. 命中率：`keyspace_hits/(hits+misses)`
4. 内存：`used_memory`、`used_memory_rss`、碎片率
5. 淘汰：`evicted_keys`
6. 阻塞：`blocked_clients`
7. 复制：`master_link_status`、复制延迟
8. fork：`latest_fork_usec`（持久化时）

### 7.2 告警建议（示例）

- P99 延迟持续 > 50ms（按业务调整）
- `blocked_clients > 0` 持续 1 分钟
- `mem_fragmentation_ratio > 1.5` 持续 10 分钟
- 主从断连（`master_link_status:down`）
- `evicted_keys` 激增（缓存被打爆/淘汰策略不合理）

---

## 八、常见“运维级”故障模式与治理

### 8.1 全量同步风暴

现象：
- 多个从节点同时 full resync
- 主节点频繁 fork，延迟抖动

治理：
- 增大 `repl-backlog-size`
- 从节点重启错峰
- 排查网络抖动/连接中断

### 8.2 热点 slot / 热 key

现象：
- 集群整体没满，但某个 master CPU 满

治理：
- key 副本分片
- 本地缓存（多级缓存）
- 热点拆分/隔离业务集群

### 8.3 fork 抖动 / COW OOM

现象：
- latency 尖刺
- RSS 暴涨

治理：
- 控制实例大小
- 低峰 rewrite
- 预留内存

---

## 九、演练与复盘：把事故变成能力

### 9.1 建议做的 6 个演练

1. 主节点宕机 → 自动 failover
2. 从节点宕机 → 复制恢复
3. 网络分区（模拟脑裂）
4. reshard 迁移 slot → 观察 MOVED/ASK
5. AOF rewrite / BGSAVE → 观察 fork 与延迟
6. 大 key 删除（UNLINK）→ 观察阻塞

### 9.2 复盘模板（推荐固化到仓库）

```text
背景：
现象：
影响：
排查路径：
根因：
止血：
长期治理：
可复用经验：
面试表达：
```

---

## 十、面试 2 分钟总结

```text
Redis 运维治理核心是把故障模式（写丢、脑裂、同步风暴、fork 抖动、热点单点）变成可观测、可止血、可演练。
生产上我会优先控制单实例大小与 fork/COW 风险，配置 min-replicas-to-write 减少孤岛写，合理设置 repl-backlog-size 减少全量同步。
变更上扩缩容/reshard 必须低峰限速可回滚，客户端要正确处理 MOVED/ASK，并配置超时和有边界重试。
最后用监控面板 + 演练 + 复盘把稳定性沉淀成体系。
```
