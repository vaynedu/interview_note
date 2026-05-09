# 消息队列

> 后端核心中间件。本目录按"架构 + 可靠 + 顺序 + 副本 + 消费 + 选型 + 场景"组织。Kafka 视角为主，对比 RocketMQ/RabbitMQ/Pulsar。

## ★ 总览地图

> **新人/复习者优先看** [00-mq-map.md](00-mq-map.md)：知识树 / 8 层能力 / 题型分级 / 选型决策 / 线上排查地图 / 答题模板。

## 目录

| # | 文件 | 涵盖 |
| --- | --- | --- |
| 01 | [架构与核心原理](01-architecture.md) | Kafka 架构 / 为什么快 / 零拷贝 / 顺序写 / PageCache / 批处理 |
| 02 | [消息可靠性](02-reliability.md) | 不丢消息全链路 / 生产 ack / broker 副本 / 消费者 offset |
| 03 | [消息顺序与重复](03-order-and-dedup.md) | 分区顺序 / 幂等 / Exactly Once 三种语义 |
| 04 | [高可用与副本](04-ha-replication.md) | 副本机制 / ISR / Leader 选举 / 故障转移 |
| 05 | [消费者与 Rebalance](05-consumer-rebalance.md) | 消费者组 / offset / Rebalance / 消费模式 |
| 06 | [选型对比](06-comparison.md) | Kafka vs RocketMQ vs RabbitMQ vs Pulsar |
| 07 | [场景实战](07-scenarios.md) | 削峰 / 解耦 / 事务消息 / 延时 / 死信 / 重试 / 踩坑 |
| 08 | [线上案例](08-production-cases.md) | 消息积压 / 重复消费 / 消息丢失 / 顺序错乱 / 死信 / Rebalance |
| 09 | [事务消息与 Outbox](09-transaction-message-outbox.md) | 本地事务 + MQ / Outbox / RocketMQ 事务消息 / 对账补偿 |

## 跨章高频题

- **Kafka 为什么这么快？**（→ 01）
- **怎么保证消息不丢？**（→ 02）
- **怎么保证消息不重复？怎么实现 Exactly Once？**（→ 03）
- **怎么保证消息顺序？**（→ 03）
- ISR 是什么？为什么不用 Quorum？（→ 04）
- 副本如何选举 Leader？（→ 04）
- Rebalance 触发条件？怎么减少影响？（→ 05）
- offset 提交时机有几种？各自坑？（→ 05）
- Kafka 和 RocketMQ 怎么选？（→ 06）
- 消息积压了怎么办？（→ 07）
- 怎么实现延时消息？（→ 07）
- 死信队列怎么用？（→ 07）
- 消息积压如何排查？为什么扩容消费者不一定有效？（→ 08）
- 重复消费和消息丢失分别怎么治理？（→ 08）
- 本地事务提交和 MQ 发送如何保证最终一致？（→ 09）
- Outbox 和 RocketMQ 事务消息怎么选？（→ 09）

## 设计原则

- **图文并茂**，关键流程用 Mermaid
- **每篇独立可读**
- **场景驱动**：原理服务于"为什么 / 怎么用 / 坑"
