# Redis

> 后端面试核心中间件之一。本目录按"架构原理 + 数据结构 + 持久化 + 复制 + 缓存 + 锁 + 调优 + 场景 + 多级缓存"组织，不按命令切。

## ★ 总览地图

> **新人/复习者优先看** [00-redis-map.md](00-redis-map.md)：知识树 / 题型分类（基础/中级/资深）/ 学习路径 / 系统设计中的角色 / 线上排查地图 / 答题模板

## 目录

| # | 文件 | 涵盖 |
| --- | --- | --- |
| **00** | **[Redis 知识地图 ★](00-redis-map.md)** | **总览：知识树 / 题型分类 / 学习路径 / 系统设计角色 / 排查地图 / 答题模板** |
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
| 11 | [多级缓存](11-multi-tier-cache.md) | 多级缓存 + Go 本地缓存对比（Ristretto/BigCache/freecache）+ 热点检测 |
| 15 | [集群故障与切换案例](15-cluster-failover-cases.md) | 写丢窗口 / 脑裂 / 哨兵误判 / reshard 风险 / MOVED/ASK / PSYNC/backlog / rewrite 抖动 |
| 16 | [集群运维治理](16-redis-cluster-operability.md) | 拓扑/容量规划 / 关键配置 / 扩缩容与迁移流程 / 客户端治理 / 监控告警 / 演练复盘 |
| 17 | [源码深水区：对象与编码](17-object-encoding-internals.md) | redisObject / SDS / 编码升级 / quicklist / skiplist / 渐进式 rehash |
| 18 | [源码深水区：事件循环与内存](18-eventloop-memory-internals.md) | 事件循环 / IO threads / BIO / 过期循环 / LFU 近似 / 碎片与 jemalloc |
| 20 | [Go Redis 最佳实践](20-go-redis-best-practices.md) | go-redis 连接池 / 超时分层 / Pipeline / Lua / Hook 埋点 / 熔断 fallback / 热 key SDK / 多级缓存组件 / 反模式 |

## 高频面试题（跨章索引）

- Redis 为什么单线程还这么快？（→ 01）
- 5 种数据结构底层编码？跳表为什么不用红黑树？（→ 02）
- RDB 和 AOF 怎么选？混合持久化是什么？（→ 03）
- 主从同步流程？为什么有全量+增量两种？（→ 04）
- Cluster 16384 slot 怎么定的？为什么不是 65536？（→ 04）
- 缓存击穿/穿透/雪崩怎么防？双删为什么不彻底？（→ 05）
- Redis 分布式锁怎么实现？Redlock 争议？（→ 06 + ../06-distributed/04）
- 内存淘汰 8 种策略？过期 key 是怎么删除的？（→ 07）
- 怎么用 Redis 做限流？滑动窗口怎么实现？（→ 08）
- Pipeline 和事务区别？Lua 脚本场景？（→ 跨章）
- 热 key 和大 key 分别怎么排查和治理？（→ 09）
- Redis 内存打满、慢查询、fork 抖动怎么处理？（→ 09）
- 缓存一致性为什么常用先写 DB 再删缓存？（→ 10）
- 删除缓存失败怎么办？延迟双删和 binlog 订阅怎么选？（→ 10）
- 多级缓存怎么设计？本地缓存 Ristretto/BigCache 怎么选？（→ 11）
- 主从切换时哪些写会丢？脑裂写丢失怎么理解？（→ 15）
- MOVED/ASK 是什么？reshard 会影响业务哪些指标？（→ 15）
- go-redis 连接池 PoolSize 怎么调？ConnMaxLifetime 为什么要设？（→ 20）
- Pipeline 在 Go 里怎么封装？Cluster 跨 slot 怎么办？（→ 20）
- Redis 挂了业务怎么办？熔断 + fallback 怎么包一层 SDK？（→ 20）
- 热 key 在 Go 服务里怎么实现保护？singleflight 用过吗？（→ 20）

## 学习顺序

1. 先看 [00-redis-map.md](00-redis-map.md) 建立总图
2. 按 01-04 学**机制**（架构 / 数据结构 / 持久化 / 集群）
3. 学 05-06 学**核心场景**（缓存 / 锁）
4. 学 07-08 学**调优 + 实战场景**
5. 学 09-11 学**资深题**（生产事故 / 一致性 / 多级缓存）
6. 学 15-18 学**资深追问：高可用 + 源码**（故障/运维治理 + 事件循环/内存/编码）
7. 学 20 落地**Go SDK 工程化**（连接池 / 超时 / 埋点 / 熔断 / 热 key）
8. 用 [99-meta/redis-20.md](../99-meta/redis-20.md) 速记面试题
9. 综合实战看 [10-system-design/16-high-concurrency-scenarios.md](../10-system-design/16-high-concurrency-scenarios.md)

## 设计原则

- **每篇独立可读**，不强制跨文件跳转
- **图文并茂**，关键流程用 Mermaid
- **场景驱动**，原理服务于"什么时候用、怎么用、坑在哪"

## 与其他模块关联

- 缓存专题汇总：[99-meta/01-cross-topic-index.md](../99-meta/01-cross-topic-index.md)
- 分布式锁三方对比：[06-distributed/04-lock.md](../06-distributed/04-lock.md)
- 速记题集：[99-meta/redis-20.md](../99-meta/redis-20.md)
- 综合实战：[10-system-design/16-high-concurrency-scenarios.md](../10-system-design/16-high-concurrency-scenarios.md)
