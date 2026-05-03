# 高频面试题全局索引

> 这个文件用于面试前快速定位问题，不替代各专题正文。先看问题，再跳到对应专题复习。

## 一、Go 高频

| 问题 | 复习位置 |
| --- | --- |
| GMP 调度模型是什么？ | [01-go-language](../01-go-language/) |
| goroutine 和线程有什么区别？ | [02-os/01-process-thread.md](../02-os/01-process-thread.md) |
| channel 底层和使用坑？ | [99-meta/go-concurrency-100.md](go-concurrency-100.md) |
| context 取消传播怎么用？ | [99-meta/go-concurrency-100.md](go-concurrency-100.md) |
| Go 服务 CPU 高怎么排查？ | [02-os/15-cpu-memory-troubleshooting.md](../02-os/15-cpu-memory-troubleshooting.md) |
| pprof 怎么看 CPU/heap/goroutine？ | [02-os/14-performance-tools.md](../02-os/14-performance-tools.md) |

## 二、MySQL 高频

| 问题 | 复习位置 |
| --- | --- |
| 一条 SQL 怎么执行？ | [03-mysql/01-architecture.md](../03-mysql/01-architecture.md) |
| B+Tree、聚簇索引、回表、覆盖索引？ | [03-mysql/02-index.md](../03-mysql/02-index.md) |
| 事务、锁、MVCC 怎么理解？ | [03-mysql/03-transaction-lock-mvcc.md](../03-mysql/03-transaction-lock-mvcc.md) |
| redo/undo/binlog 区别？ | [03-mysql/04-log.md](../03-mysql/04-log.md) |
| 主从复制和主从延迟怎么处理？ | [03-mysql/05-replication-ha.md](../03-mysql/05-replication-ha.md) |
| 慢 SQL 怎么优化？ | [03-mysql/06-slow-sql-optimization.md](../03-mysql/06-slow-sql-optimization.md) |
| 分库分表什么时候做？ | [03-mysql/12-sharding.md](../03-mysql/12-sharding.md) |
| Go 使用 MySQL 有哪些坑？ | [03-mysql/16-go-mysql-practice.md](../03-mysql/16-go-mysql-practice.md) |

## 三、Redis 高频

| 问题 | 复习位置 |
| --- | --- |
| Redis 为什么快？ | [04-redis/01-architecture.md](../04-redis/01-architecture.md) |
| Redis 数据结构底层实现？ | [04-redis/02-data-structures.md](../04-redis/02-data-structures.md) |
| RDB/AOF 怎么选？ | [04-redis/03-persistence.md](../04-redis/03-persistence.md) |
| 主从、哨兵、Cluster 怎么理解？ | [04-redis/04-replication-cluster.md](../04-redis/04-replication-cluster.md) |
| 缓存击穿、穿透、雪崩怎么处理？ | [04-redis/05-cache-patterns.md](../04-redis/05-cache-patterns.md) |
| 热 key、大 key 怎么排查？ | [04-redis/09-production-cases.md](../04-redis/09-production-cases.md) |
| 缓存一致性怎么做？ | [04-redis/10-cache-consistency-design.md](../04-redis/10-cache-consistency-design.md) |
| Redis 分布式锁有什么坑？ | [04-redis/06-distributed-lock.md](../04-redis/06-distributed-lock.md) |

## 四、MQ 高频

| 问题 | 复习位置 |
| --- | --- |
| Kafka 为什么快？ | [05-message-queue/01-architecture.md](../05-message-queue/01-architecture.md) |
| 如何保证消息不丢？ | [05-message-queue/02-reliability.md](../05-message-queue/02-reliability.md) |
| 如何保证消息不重复？ | [05-message-queue/03-order-and-dedup.md](../05-message-queue/03-order-and-dedup.md) |
| 顺序消息怎么设计？ | [05-message-queue/03-order-and-dedup.md](../05-message-queue/03-order-and-dedup.md) |
| Rebalance 有什么影响？ | [05-message-queue/05-consumer-rebalance.md](../05-message-queue/05-consumer-rebalance.md) |
| 消息积压怎么排查？ | [05-message-queue/08-production-cases.md](../05-message-queue/08-production-cases.md) |
| 本地事务 + MQ 怎么保证一致？ | [05-message-queue/09-transaction-message-outbox.md](../05-message-queue/09-transaction-message-outbox.md) |

## 五、分布式 高频

| 问题 | 复习位置 |
| --- | --- |
| CAP、BASE、Quorum 怎么理解？ | [06-distributed/01-theory.md](../06-distributed/01-theory.md) |
| Raft 怎么选主和复制日志？ | [06-distributed/02-consensus.md](../06-distributed/02-consensus.md) |
| 2PC、TCC、Saga 怎么选？ | [06-distributed/03-transaction.md](../06-distributed/03-transaction.md) |
| Redis/ZK/etcd 分布式锁怎么选？ | [06-distributed/04-lock.md](../06-distributed/04-lock.md) |
| Redis/ZooKeeper/etcd 协调服务区别？ | [06-distributed/09-coordination-services.md](../06-distributed/09-coordination-services.md) |
| 分布式锁有哪些线上坑？ | [06-distributed/10-distributed-lock-cases.md](../06-distributed/10-distributed-lock-cases.md) |
| 限流、熔断、降级怎么做？ | [06-distributed/06-rate-limit-circuit.md](../06-distributed/06-rate-limit-circuit.md) |

## 六、线上排查 高频

| 问题 | 复习位置 |
| --- | --- |
| 高 CPU 怎么排查？ | [02-os/15-cpu-memory-troubleshooting.md](../02-os/15-cpu-memory-troubleshooting.md) |
| 高内存 / OOM 怎么排查？ | [02-os/11-memory-oom.md](../02-os/11-memory-oom.md) |
| 接口偶发超时怎么排查？ | [02-os/17-network-troubleshooting.md](../02-os/17-network-troubleshooting.md) |
| 磁盘满、inode 满、deleted 文件怎么查？ | [02-os/18-disk-io-troubleshooting.md](../02-os/18-disk-io-troubleshooting.md) |
| K8s OOMKilled / throttling 怎么查？ | [02-os/19-container-k8s-resource.md](../02-os/19-container-k8s-resource.md) |
| 大厂线上事故怎么复盘？ | [02-os/16-production-incident-cases.md](../02-os/16-production-incident-cases.md) |
| 跨系统问题怎么排查？ | [13-engineering/00-troubleshooting-runbook.md](../13-engineering/00-troubleshooting-runbook.md) |

## 七、系统设计 高频

| 问题 | 复习位置 |
| --- | --- |
| 系统设计题如何开场？ | [10-system-design/01-design-framework.md](../10-system-design/01-design-framework.md) |
| 短链系统怎么设计？ | [10-system-design/02-short-code-platform.md](../10-system-design/02-short-code-platform.md) |
| 秒杀系统怎么设计？ | [10-system-design/03-seckill-system.md](../10-system-design/03-seckill-system.md) |
| 直播/弹幕系统怎么设计？ | [10-system-design/04-realtime-barrage.md](../10-system-design/04-realtime-barrage.md)、[10-system-design/05-live-streaming.md](../10-system-design/05-live-streaming.md) |
| Feed / IM / 评论 / 点赞怎么设计？ | [10-system-design](../10-system-design/) |
| 支付、优惠券、库存怎么设计？ | [10-system-design/13-payment-system.md](../10-system-design/13-payment-system.md)、[10-system-design/14-coupon-system.md](../10-system-design/14-coupon-system.md)、[10-system-design/15-inventory-system.md](../10-system-design/15-inventory-system.md) |
| 网关、配置中心、监控、日志平台怎么设计？ | [10-system-design/17-monitoring-alerting.md](../10-system-design/17-monitoring-alerting.md)、[10-system-design/18-api-gateway.md](../10-system-design/18-api-gateway.md)、[10-system-design/19-config-center.md](../10-system-design/19-config-center.md)、[10-system-design/20-log-search-platform.md](../10-system-design/20-log-search-platform.md) |

## 八、项目复盘 高频

| 问题 | 复习位置 |
| --- | --- |
| 你项目里最有挑战的问题是什么？ | [14-projects/00-project-map.md](../14-projects/00-project-map.md) |
| 项目怎么讲出技术深度？ | [14-projects/01-project-story-framework.md](../14-projects/01-project-story-framework.md) |
| 性能优化项目怎么讲？ | [14-projects/02-performance-optimization-cases.md](../14-projects/02-performance-optimization-cases.md) |
| 线上事故怎么讲？ | [14-projects/03-production-incident-stories.md](../14-projects/03-production-incident-stories.md) |
| 简历项目怎么准备？ | [14-projects/04-project-template.md](../14-projects/04-project-template.md) |

