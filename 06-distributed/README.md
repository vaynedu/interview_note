# 分布式

> 后端架构核心。本目录按"理论 + 共识 + 事务 + 锁 + ID + 治理 + 注册 + 模式"组织，覆盖资深面试常考点。

## ★ 总览地图

> **新人/复习者优先看** [00-distributed-map.md](00-distributed-map.md)：知识树 / 8 层能力 / 题型分级 / 分布式事务 5 种方案 / 三方锁对比 / 答题模板。

## 目录

| # | 文件 | 涵盖 |
| --- | --- | --- |
| 01 | [理论基础](01-theory.md) | CAP / BASE / 一致性模型 / FLP / Quorum |
| 02 | [共识算法](02-consensus.md) | Paxos / Raft / ZAB / Gossip / 选举 |
| 03 | [分布式事务](03-transaction.md) | 2PC / 3PC / TCC / Saga / 本地消息表 / 最大努力通知 |
| 04 | [分布式锁(对比)](04-lock.md) | Redis vs ZK vs etcd / 单点多点 / 适用场景 |
| 05 | [分布式 ID](05-id-generation.md) | UUID / 雪花算法 / 号段 / Leaf / 时钟回拨 |
| 06 | [限流熔断降级](06-rate-limit-circuit.md) | 4 算法 / 单机 vs 分布式 / Sentinel / Hystrix / 半开 |
| 07 | [服务发现与负载均衡](07-service-discovery-lb.md) | CP/AP / 一致性 Hash / 算法 / 健康检查 |
| 08 | [分布式设计模式](08-design-patterns.md) | 幂等 / 重试 / 补偿 / 灰度 / 链路追踪 / 雪崩防护 |
| 09 | [协调服务对比](09-coordination-services.md) | Redis / ZooKeeper / etcd 对比 / 注册发现 / 配置中心 / 选主 |
| 10 | [分布式锁场景](10-distributed-lock-cases.md) | 分布式锁工程实践 / 幂等 / fencing token / 线上坑 |

## 跨章高频题

- CAP 是什么？P 一定满足吗？（→ 01）
- 强一致 / 最终一致 / 线性一致区别？（→ 01）
- Raft 怎么选主？日志怎么复制？（→ 02）
- Paxos 和 Raft 区别？为什么生产用 Raft？（→ 02）
- 2PC vs TCC vs Saga 怎么选？（→ 03）
- 本地消息表���么保证最终一致？（→ 03）
- Redis / ZK / etcd 分布式锁怎么选？（→ 04）
- 雪花算法时钟回拨怎么办？（→ 05）
- 滑动窗口和令牌桶区别？（→ 06）
- 熔断三态（关 / 开 / 半开）？（→ 06）
- 一致性 Hash 解决什么？（→ 07）
- ZK 和 Eureka/Nacos 注册中心怎么选？（→ 07）
- 怎么实现幂等？（→ 08）
- 重试为什么会放大雪崩？（→ 08）
- Redis / ZooKeeper / etcd 除了锁还有哪些协调场景？（→ 09）
- 配置中心和注册发现为什么需要 watch / lease？（→ 09）
- 防重复提交一定要用分布式锁吗？（→ 10）
- 什么是 fencing token？解决什么问题？（→ 10）

## 设计原则

- **图文并茂**，关键流程用 Mermaid
- **每篇独立可读**，不强制跨文件
- **场景驱动**：原理服务于"为什么 / 怎么用 / 坑在哪"
