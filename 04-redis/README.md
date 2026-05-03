# Redis

> 后端面试核心中间件之一。本目录按"架构原理 + 数据结构 + 持久化 + 复制 + 缓存 + 锁 + 调优 + 场景"组织，不按命令切。

## 目录

| # | 文件 | 涵盖 |
| --- | --- | --- |
| 01 | [架构与原理](01-architecture.md) | 单线程模型 / 事件循环 / 为什么快 / IO 多路复用 / 6.0 多线程 IO |
| 02 | [数据结构](02-data-structures.md) | String/List/Hash/Set/ZSet + Stream/HyperLogLog/Geo/Bitmap / 底层编码 / 场景 |
| 03 | [持久化](03-persistence.md) | RDB / AOF / 混合 / fork 与 COW / 重写 |
| 04 | [复制与集群](04-replication-cluster.md) | 主从复制 / 哨兵 / Cluster 分片(16384 slot) / 脑裂 |
| 05 | [缓存模式](05-cache-patterns.md) | Cache-Aside / Write-Through / 一致性 / 击穿穿透雪崩 / 双删 |
| 06 | [分布式锁](06-distributed-lock.md) | SETNX / Redlock / 续约 / 公平锁 / 红锁争议 |
| 07 | [踩坑与调优](07-pitfalls-tuning.md) | 大 key / 热 key / 内存淘汰 8 策略 / 慢查询 / 过期机制 |
| 08 | [场景实战](08-scenarios.md) | 排行榜 / 计数器 / 限流 / 队列 / 布隆 / 地理 / Pub-Sub / 会话 |
| 09 | [线上案例](09-production-cases.md) | 热 key / 大 key / 内存打满 / 慢查询 / 阻塞命令 / fork COW |
| 10 | [缓存一致性](10-cache-consistency-design.md) | Cache Aside / 延迟双删 / binlog 订阅 / 强一致读 / 删除失败重试 |

## 高频面试题（跨章索引）

- Redis 为什么单线程还这么快？（→ 01）
- 5 种数据结构底层编码？跳表为什么不用红黑树？（→ 02）
- RDB 和 AOF 怎么选？混合持久化是什么？（→ 03）
- 主从同步流程？为什么有全量+增量两种？（→ 04）
- Cluster 16384 slot 怎么定的？为什么不是 65536？（→ 04）
- 缓存击穿/穿透/雪崩怎么防？双删为什么不彻底？（→ 05）
- Redis 分布式锁怎么实现？Redlock 争议？（→ 06）
- 内存淘汰 8 种策略？过期 key 是怎么删除的？（→ 07）
- 怎么用 Redis 做限流？滑动窗口怎么实现？（→ 08）
- Pipeline 和事务区别？Lua 脚本场景？（→ 跨章）
- 热 key 和大 key 分别怎么排查和治理？（→ 09）
- Redis 内存打满、慢查询、fork 抖动怎么处理？（→ 09）
- 缓存一致性为什么常用先写 DB 再删缓存？（→ 10）
- 删除缓存失败怎么办？延迟双删和 binlog 订阅怎么选？（→ 10）

## 设计原则

- **每篇独立可读**，不强制跨文件跳转
- **图文并茂**，关键流程用 Mermaid
- **场景驱动**，原理服务于"什么时候用、怎么用、坑在哪"
