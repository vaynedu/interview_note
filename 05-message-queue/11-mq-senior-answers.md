# MQ 资深面试答题模板

> 把高频问题拆成 **"一句话 → 分层展开 → 边界 → 代价"** 四段式答题法。
>
> 与 [04-redis/21-senior-interview-answers.md](../04-redis/21-senior-interview-answers.md) / [03-mysql/20-mysql-senior-answers.md](../03-mysql/20-mysql-senior-answers.md) 对偶。

---

## 一、四段答题法

```
① 一句话：核心结论
② 分层展开：2-5 层
③ 边界 / 代价：什么情况失效
④ 实战佐证：真实案例 + 量化
```

---

## 二、Kafka 为什么这么快（最高频）

### 2.1 一句话

> "**顺序写盘 + PageCache + 零拷贝 + 批量 + 压缩 + 分区并行**，6 大原因。"

### 2.2 6 大原因

```
1. 顺序写磁盘
   - HDD 顺序写 600MB/s（接近内存）
   - 随机写 100KB/s
   - 6000 倍差距
   - Kafka 只 append，永远顺序

2. PageCache + 零拷贝
   - OS Page Cache 缓存写入
   - sendfile 直接 kernel → socket
   - 避免 4 次拷贝（kernel ↔ user）

3. 批量
   - Producer 攒批（BatchSize / linger.ms）
   - Broker 批量写
   - Consumer 批量拉

4. 压缩
   - Snappy / LZ4 / zstd
   - 网络 + 磁盘双重收益
   - CPU 换网络 / 磁盘

5. 分区并行
   - Topic 多分区
   - Producer 并发写不同分区
   - Consumer 组并行消费

6. 二进制协议
   - 高效编解码
   - 零冗余
```

### 2.3 边界

```
慢的场景:
  - 单分区瓶颈（无法横向扩展）
  - 大消息（破坏批量优势）
  - 频繁小事务（破坏顺序写）
  - 跨地域（网络 RTT）

→ 设计时按吞吐 / 延迟权衡
```

### 2.4 量化

```
单 Broker 吞吐:
  - 100 万 QPS（小消息）
  - 600MB/s（顺序写极限）
  - P99 < 50ms

集群吞吐（10 broker）:
  - 1000 万 QPS
  - 6GB/s
```

---

## 三、Kafka 怎么保证不丢消息

### 3.1 一句话

> "**生产端 acks=all + 重试，Broker 副本 + ISR，消费端手动 commit + 幂等**。三端配合。"

### 3.2 三端方案

```
生产端:
  - acks = all（所有 ISR 收到）
  - retries = 5+
  - max.in.flight.requests = 1（保顺序）
  - enable.idempotence = true（防重发）

Broker 端:
  - replication.factor = 3
  - min.insync.replicas = 2（至少 2 副本同步）
  - unclean.leader.election = false（防数据丢失）

消费端:
  - 手动 commit（业务成功后才提交）
  - 业务幂等
  - 幂等手段: DB 唯一索引 / Redis SETNX / 状态机
```

### 3.3 边界

```
即使三端都做:
  - 极端故障（多副本同时挂）仍可能丢
  - acks=all 性能损失
  - 业务必须有兜底（对账 / DLQ）

实战不能依赖 100%:
  - 加监控（Lag / 错误率）
  - 加对账（实时 + T+1）
  - 加 DLQ（人工兜底）
```

---

## 四、Kafka 怎么保证消息顺序

### 4.1 一句话

> "**同一 Key 路由到同一分区 + 单消费者单线程消费**。全局有序需要单分区（牺牲性能）。"

### 4.2 三层

```
全局有序（单分区）:
  - 整个 Topic 一个分区
  - 单消费者
  - 性能差（无并行）
  - 极少用

业务有序（推荐）:
  - 按业务 Key Hash 分区（如 order_id）
  - 同一订单消息在同一分区
  - 同分区严格顺序
  - 跨分区不保证

边界:
  - 同 Key 但 Producer 重试（max.in.flight > 1）可能乱序
  - → 设 max.in.flight.requests = 1（严格）
  - → 或开启幂等 Producer（保证）
```

### 4.3 消费端

```
单分区单消费者（同 Group）保顺序:
  - 多 consumer 同 group 自动分配分区
  - 单 partition 只被一个 consumer 消费

业务侧:
  - 不要业务侧再分发到 worker pool（破坏顺序）
  - 单 goroutine 处理
```

---

## 五、Kafka 怎么保证消息不重复

### 5.1 一句话

> "**Kafka 至少一次（at-least-once）默认 + 业务幂等兜底**。Exactly Once 复杂且有限制。"

### 5.2 三层

```
1. Kafka 默认: at-least-once
   - 消费成功 + commit 失败 → 下次重消费
   - 必然有重复

2. Producer 幂等（防重发）
   - enable.idempotence = true
   - PID + 序列号去重
   - 单分区 / 单会话内有效

3. Exactly Once Semantics (EOS)
   - Producer 幂等 + 事务
   - 跨分区 / 跨服务事务
   - Consumer read_committed
   - 性能损失 + 配置复杂

4. 业务幂等（永远兜底）
   - DB 唯一索引（业务 ID）
   - Redis SETNX
   - 状态机 + 前置条件
```

### 5.3 实战

```
99% 场景:
  - Kafka at-least-once + 业务幂等
  - 简单 + 高效

1% 场景（金融对账等）:
  - Kafka EOS + 业务幂等双保险
  - 性能可接受
```

---

## 六、Kafka 选哪个 Go 客户端

### 6.1 一句话

> "**主流 kafka-go（segmentio）**，简洁纯 Go；高性能选 confluent-kafka-go（CGO）；老项目用 sarama。"

### 6.2 详见

[10-go-mq-practice.md 第二节](10-go-mq-practice.md)

---

## 七、Kafka vs RocketMQ vs RabbitMQ vs Pulsar

### 7.1 一句话

> "**Kafka 大数据 / 日志，RocketMQ 业务消息（事务+延迟），RabbitMQ 复杂路由，Pulsar 多租户跨地域**。"

### 7.2 选型表

| 维度 | Kafka | RocketMQ | RabbitMQ | Pulsar |
| --- | --- | --- | --- | --- |
| 吞吐 | 极高（百万）| 高（10w）| 中（万级）| 高 |
| 延迟 | ms | ms | μs | ms |
| 事务消息 | 跨分区事务 | **强** | 弱 | 支持 |
| 延迟消息 | ❌ | **18 级 / 任意** | 插件 | 原生 |
| 顺序 | 分区级 | 分区级 | 队列级 | 分区级 |
| 持久化 | 强 | 强 | 中 | 强 |
| 多租户 | 弱 | 中 | 中 | **强** |
| 跨地域 | MirrorMaker | DLedger | Federation | **原生** |
| 生态 | 大数据强 | 阿里业务强 | 老牌 | 后起 |
| 国内活跃 | 中 | **极高** | 中 | 增长中 |

### 7.3 选型决策

```
日志 / 数据流 / 大数据 → Kafka
业务消息 + 事务 / 延迟 → RocketMQ
复杂路由 / 老业务 → RabbitMQ
新项目 + 多租户 + 跨地域 → Pulsar
轻量内部 → Redis Streams
```

---

## 八、Kafka Rebalance 是什么

### 8.1 一句话

> "**消费者组内分区重新分配**。触发时所有 consumer 暂停，影响吞吐。"

### 8.2 触发条件

```
1. Consumer 加入 / 退出
2. Heartbeat 超时（session.timeout）
3. Topic 分区数变化
4. Consumer 处理太慢（长时间不 poll）
```

### 8.3 影响

```
- 暂停消费几秒到几十秒
- 可能重复消费（老 consumer 没 commit）
- 业务延迟 / 堆积
```

### 8.4 优化

```
1. Static Membership（2.3+）
   - Consumer 带 group.instance.id
   - 重启不触发 Rebalance
   - 适合滚动升级

2. 增量 Rebalance（2.4+）
   - CooperativeStickyAssignor
   - 不"Stop the World"
   - 只迁移变化的分区

3. 业务侧
   - 处理时间 < session.timeout
   - 大任务异步化
   - 加大 max.poll.interval

4. 监控
   - Rebalance 频率告警
   - 异常 Rebalance 排查根因
```

---

## 九、Kafka Lag 高怎么办

### 9.1 一句话

> "**短期扩消费实例 / 加分区 / 跳过非关键，长期优化消费速度 / 拆 Topic**。"

### 9.2 短期止血

```
方案 1: 扩消费实例
  - 加 consumer pod
  - 但不超过分区数（多余 consumer 空闲）

方案 2: 加分区
  - kafka-topics.sh --alter --partitions 32
  - 注意: 加分区不会触发数据 rebalance
  - 老 Key 仍在老分区

方案 3: 临时新 Topic
  - 创建大分区数新 Topic
  - 写一个搬运 Consumer + Producer
  - 后续 Consumer 消费新 Topic

方案 4: 跳过非关键
  - 重置 offset 到最新（reset-offsets --to-latest）
  - 接受丢消息
  - 紧急情况下用
```

### 9.3 长期优化

```
1. 优化消费速度
   - 业务代码优化（DB / 网络 / 锁）
   - 异步处理
   - 批量处理

2. 加分区
   - 评估业务并发度
   - 一次到位（避免后续再加）

3. 拆 Topic
   - 大流量 vs 小流量分开
   - 不同优先级分开

4. 限流上游
   - Producer 限流
   - 削峰
```

---

## 十、Kafka 消息堆积 1 亿条怎么处理

### 10.1 一句话

> "**临时新 Topic 多分区 + 搬运消费者 + 跳过非关键**。同时找根因（消费慢 / 下游慢 / 突发流量）。"

### 10.2 完整答法

```
T+0 应急止血:
  1. 评估影响（哪些业务延迟）
  2. 跳过非关键消息（reset offset）
  3. 关闭非核心消费

T+0 + 1h 临时方案:
  1. 加消费实例（不超过分区数）
  2. 临时增加分区
  3. 新建临时 Topic 多分区
     - 一个搬运 consumer 从老 Topic 拉
     - Producer 到新 Topic（多分区）
     - 多 Consumer 并发消费新 Topic

T+0 + 4h 根因:
  1. 看消费速度变化点
  2. 看下游 RT 变化（DB / RPC）
  3. 看流量变化（突发？）
  4. 看代码（最近发版？）

T+1d 长期改进:
  - 优化消费业务（DB 索引 / 异步）
  - 加监控 / 告警（Lag）
  - 加自动扩容
  - 拆 Topic
```

---

## 十一、Producer 同步 vs 异步

### 11.1 一句话

> "**关键业务同步等 ACK 不丢，非关键 / 高吞吐异步配回调**。"

### 11.2 对比

```
同步:
  ✓ 立刻知道成败
  ✓ 业务可重试
  ✗ 慢（ACK 往返 ms 级）

异步:
  ✓ 快（不阻塞）
  ✗ 失败需要回调处理
  ✗ 程序崩溃可能丢

实战:
  - 订单 / 支付等关键 → 同步
  - 日志 / 埋点 → 异步
```

---

## 十二、消费者怎么实现幂等

### 12.1 一句话

> "**业务 ID + DB 唯一索引 / Redis SETNX / 状态机** 三选一。所有消费都要有，不能依赖 MQ Exactly Once。"

### 12.2 三种方法

```
方法 1: DB 唯一索引（最强）
  INSERT INTO order_events (event_id, ...) 
  → 重复 → 1062 错误 → 业务忽略

方法 2: Redis SETNX
  SET processed:{event_id} 1 NX EX 86400
  ok → 第一次处理
  fail → 已处理

方法 3: 状态机
  UPDATE orders SET status = 'paid' 
  WHERE order_no = ? AND status = 'pending'
  affected = 0 → 已处理
```

详见 [10-go-mq-practice.md 第 4.3 节](10-go-mq-practice.md)。

---

## 十三、事务消息 vs Outbox

### 13.1 一句话

> "**事务消息 MQ 内置，Outbox 业务自实现**。前者依赖 RocketMQ，后者通用（Kafka 也能用）。"

### 13.2 对比

```
事务消息（RocketMQ）:
  Half message → 本地事务 → COMMIT/ROLLBACK → 回查
  
  ✓ 简单（MQ 框架内置）
  ✓ 强一致
  ✗ 与 RocketMQ 耦合
  ✗ 回查接口要业务实现

Outbox（业务表 + 后台扫描）:
  写业务表 + 写 outbox 表（同事务）
  → 后台 Worker 扫 outbox → 发 MQ
  
  ✓ 与 MQ 解耦（Kafka / RocketMQ 都行）
  ✓ 完全 DB 控制
  ✗ 后台 Worker 复杂度
  ✗ 延迟（取决于扫描间隔）

CDC（更高级）:
  binlog → Canal / Debezium → MQ
  
  ✓ 业务零侵入
  ✓ 严格顺序
  ✗ 运维 binlog 系统
```

### 13.3 选型

```
RocketMQ 用户 → 事务消息
通用 / Kafka → Outbox
极致解耦 → CDC
```

---

## 十四、为什么不用 Redis Pub/Sub 当队列

### 14.1 一句话

> "**Redis Pub/Sub 不持久化，订阅者没连就丢**。只能用于"可丢"场景（如缓存失效）。"

### 14.2 详见

[04-redis/19-redis-design-tradeoffs.md 第 4.2 节](../04-redis/19-redis-design-tradeoffs.md)

```
关键消息走 Kafka / RocketMQ
轻量场景考虑 Redis Stream（5.0+）
缓存失效广播可用 Pub/Sub
```

---

## 十五、Kafka 消费组怎么工作

### 15.1 一句话

> "**Group 内 Consumer 自动分配分区，每分区只被组内一个 Consumer 消费**。多 Group 互不影响。"

### 15.2 关键点

```
分配规则:
  - Range（默认）：按分区号排序分给 consumer
  - Round Robin：轮询
  - Sticky：尽量保留原分配
  - CooperativeSticky（2.4+）：增量

Consumer 数量 vs 分区数:
  Consumer < 分区数 → 一个 Consumer 多个分区
  Consumer = 分区数 → 一对一（最佳）
  Consumer > 分区数 → 多余 Consumer 闲置

Offset 存储:
  - Kafka 内部 topic __consumer_offsets
  - 按 group.id + topic + partition

广播消费（多副本）:
  - 不同 group 各自消费一份
  - 每 group 都收到全部
```

---

## 十六、Kafka 分区数怎么定

### 16.1 一句话

> "**按吞吐 + 消费并发度算 + 留 20% 冗余**。单 Topic < 1000 分区。"

### 16.2 算法

```
所需分区数 = max(
    吞吐需求 / 单分区吞吐,    # 比如 1GB/s / 50MB/s = 20
    消费并发度,               # 比如 16 个 consumer
)
× 1.2  # 冗余

例子:
  吞吐: 100MB/s
  单分区: 50MB/s（保守）
  消费 consumer: 8 个
  → 分区 = max(2, 8) × 1.2 = 10
```

### 16.3 注意

```
单 Topic 分区不要太多:
  - < 1000（Kafka 元数据开销）
  - 单 Broker < 4000

加分区代价:
  - 不会自动 rebalance 老数据
  - 同 Key 可能换分区（破坏顺序）
  - 一次到位最好
```

---

## 十七、ISR / HW / LEO 是什么

### 17.1 一句话

> "**ISR (In-Sync Replicas)** 是同步副本集合，**HW (High Watermark)** 是已被所有 ISR 同步的最大 offset，**LEO (Log End Offset)** 是分区最新 offset。"

### 17.2 详细

```
ISR:
  - 与 Leader 保持同步的副本集合
  - replica.lag.time.max.ms（默认 30s）落后 → 踢出
  - min.insync.replicas 控制最少 ISR 数量

LEO (Log End Offset):
  - 每个副本独立维护
  - 写入下一条消息的位置
  - Leader LEO = 最新写入位置

HW (High Watermark):
  - Leader 维护（取所有 ISR 的最小 LEO）
  - 已被认为"committed"的最大 offset
  - Consumer 只能读到 HW（不能读未确认数据）

写入流程:
  1. Producer 写到 Leader
  2. Leader 写本地 + LEO++
  3. Followers 拉取 + 写本地 + LEO++
  4. Followers 上报 LEO
  5. Leader 计算 HW（取最小）
  6. HW 推进 → Producer ACK（如果 acks=all）
```

---

## 十八、消息回溯 / 重放

### 18.1 一句话

> "**Kafka 通过 reset offset 回溯，RocketMQ 也支持但能力较弱**。回溯前评估业务影响。"

### 18.2 Kafka 回溯

```bash
# 重置到最早
kafka-consumer-groups.sh --reset-offsets \
  --group <group> --topic <topic> --to-earliest --execute

# 重置到最新
--to-latest

# 重置到时间戳
--to-datetime 2026-01-01T00:00:00.000

# 重置到指定 offset
--to-offset 1000

# 前进 / 后退
--shift-by -100
```

### 18.3 业务考虑

```
副作用:
  - 重复消费 → 业务幂等保证
  - 大量重消费 → 性能影响
  - 跨分区不一致

实战:
  - 紧急修复（消费 bug 后回放）
  - 数据迁移
  - 测试 / 演练

避坑:
  - 业务幂等是前提
  - 评估下游影响
  - 灰度回放（先 1% 再全量）
```

---

## 十九、20 问速答索引

```
原理:
Q1: Kafka 为什么快        → 二
Q2: ISR / HW / LEO        → 十七
Q3: Producer 写入流程     → 十七 + 三
Q4: Consumer 工作原理     → 十五

可靠性:
Q5: 不丢消息              → 三
Q6: 不重复消费            → 五 + 十二
Q7: 顺序保证              → 四
Q8: Exactly Once          → 五

性能:
Q9: 同步 vs 异步          → 十一
Q10: 分区数怎么定         → 十六
Q11: 怎么提升吞吐         → 二

故障:
Q12: Lag 高怎么办         → 九
Q13: 堆积 1 亿条          → 十
Q14: Rebalance 影响       → 八

设计:
Q15: 选型对比             → 七
Q16: 事务消息 vs Outbox   → 十三
Q17: 不用 Pub/Sub         → 十四
Q18: 回溯重放             → 十八

Go 实战:
Q19: Go 客户端选型        → 六 + 10-go-mq-practice
Q20: 幂等消费实现         → 十二 + 10-go-mq-practice
```

---

## 二十、面试加分点

- **6 大快原因**（顺序 / PageCache / 零拷贝 / 批量 / 压缩 / 分区）背下来
- **不丢消息三端方案**（acks=all + RF=3 + 手动 commit + 幂等）
- **Static Membership + 增量 Rebalance** 减少抖动
- **WRITESET / Cooperative** 新特性
- **事务消息 vs Outbox 取舍**
- **业务幂等 是 EOS 兜底**
- **Lag 应对 4 招**（扩消费 / 加分区 / 搬运 / 跳过）
- **Pub/Sub 不持久化** 边界清晰
- **回溯重放 + 业务幂等** 组合
- **量化数据**（吞吐 / 延迟 / 分区数）

---

## 二十一、关联阅读

```
本目录:
- 00-mq-map.md             知识地图
- 01-architecture.md       架构原理
- 02-reliability.md        可靠性三端
- 03-order-and-dedup.md    顺序去重
- 04-ha-replication.md     HA 副本
- 05-consumer-rebalance.md Rebalance
- 06-comparison.md         MQ 对比
- 07-scenarios.md          场景实战
- 08-production-cases.md   生产案例
- 09-transaction-message-outbox.md 事务消息 / Outbox
- 10-go-mq-practice.md     Go 实战

跨模块:
- 04-redis/21-senior-interview-answers.md  Redis 答题（对偶）
- 03-mysql/20-mysql-senior-answers.md     MySQL 答题（对偶）
- 06-distributed/03-transaction.md        分布式事务
```
