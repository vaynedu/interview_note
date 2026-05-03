# MySQL

> 面向 Go 后端面试的 MySQL 专题笔记：按主题聚焦，而不是堆题目。

## 分类导航

### 核心原理

| 文件 | 内容 |
| --- | --- |
| [01-architecture.md](01-architecture.md) | MySQL 整体架构、Server 层、InnoDB、SQL 执行流程 |
| [02-index.md](02-index.md) | B+Tree、聚簇索引、联合索引、覆盖索引、索引失效、索引设计 |
| [03-transaction-lock-mvcc.md](03-transaction-lock-mvcc.md) | ACID、隔离级别、MVCC、锁、死锁、事务边界 |
| [04-log.md](04-log.md) | redo log、undo log、binlog、WAL、两阶段提交、崩溃恢复 |
| [13-innodb-buffer-pool.md](13-innodb-buffer-pool.md) | InnoDB Buffer Pool：页、脏页、LRU、刷盘、数据库抖动 |

### 性能优化与线上变更

| 文件 | 内容 |
| --- | --- |
| [06-slow-sql-optimization.md](06-slow-sql-optimization.md) | 慢 SQL、EXPLAIN、深分页、count、大表治理 |
| [14-explain-optimizer.md](14-explain-optimizer.md) | EXPLAIN 与优化器：执行计划、成本估算、索引选择、filesort |
| [15-online-ddl-mdl.md](15-online-ddl-mdl.md) | Online DDL 与 MDL：元数据锁、大表变更、gh-ost/pt-osc |

### 复制、高可用与分布式一致性

| 文件 | 内容 |
| --- | --- |
| [05-replication-ha.md](05-replication-ha.md) | 主从复制、异步/半同步、主从延迟、读写分离、高可用 |
| [09-distributed-transaction.md](09-distributed-transaction.md) | 分布式事务：XA、TCC、Saga、本地消息表、事务消息 |
| [17-consistency-reconciliation.md](17-consistency-reconciliation.md) | 数据一致性与对账：最终一致、补偿任务、状态修复、人工兜底 |

### 表设计与架构设计

| 文件 | 内容 |
| --- | --- |
| [07-schema-design.md](07-schema-design.md) | 表设计、索引设计、分库分表、线上 DDL、误删恢复 |
| [12-sharding.md](12-sharding.md) | 分库分表：为什么分、何时分表、何时分库、分片键、扩容迁移 |
| [18-field-types-charset-time.md](18-field-types-charset-time.md) | 字段类型、字符集、NULL、金额、时间、时区坑 |
| [11-order-system-design.md](11-order-system-design.md) | 订单系统设计：每日 2000 万订单、分库分表、冷热归档、查询治理 |
| [19-storage-comparison.md](19-storage-comparison.md) | 存储选型对比与未来趋势：MySQL、TiDB、ClickHouse、Hive、时序库、LSM、冷热分层 |

### 面试表达与应用实战

| 文件 | 内容 |
| --- | --- |
| [08-comparison.md](08-comparison.md) | 高频概念对比：锁、日志、索引、复制、分表、缓存 |
| [10-production-cases.md](10-production-cases.md) | 线上案例：慢 SQL、主从延迟、死锁、DDL、误删、连接池 |
| [16-go-mysql-practice.md](16-go-mysql-practice.md) | Go 使用 MySQL 实战：database/sql、连接池、事务、超时、Rows 关闭 |

## 高频题速览

### 架构

- MySQL 一条 SQL 是如何执行的？
- Server 层和存储引擎层分别负责什么？
- InnoDB 和 MyISAM 有哪些差异？
- 为什么 MySQL 8.0 移除了查询缓存？

### 索引

- 为什么 MySQL 使用 B+Tree 作为主要索引结构？
- 聚簇索引和二级索引有什么区别？
- 什么是回表？什么是覆盖索引？
- 联合索引的最左前缀原则怎么理解？
- 哪些场景会导致索引失效？
- 如何为一个列表查询设计索引？

### 事务、锁、MVCC

- ACID 分别由什么机制保证？
- MySQL 有哪些隔离级别？默认隔离级别是什么？
- MVCC 是什么？Read View 怎么理解？
- 快照读和当前读有什么区别？
- 间隙锁、临键锁解决什么问题？
- 死锁如何产生？怎么排查和避免？

### 日志

- redo log、undo log、binlog 分别解决什么问题？
- WAL 机制是什么？
- redo log 和 binlog 为什么需要两阶段提交？
- binlog 有哪些格式？生产一般怎么选？
- MySQL 崩溃恢复大致如何进行？

### 主从复制与高可用

- MySQL 主从复制原理是什么？
- 异步复制、半同步复制、全同步有什么区别？
- 主从延迟如何产生？怎么优化？
- 写后读从库读不到数据怎么办？
- MySQL 高可用切换要注意什么？

### 慢 SQL 与优化

- 慢 SQL 排查流程是什么？
- EXPLAIN 重点看哪些字段？
- 为什么深分页慢？怎么优化？
- `count(*)` 为什么可能很慢？
- 大表如何治理？

### 表设计与架构

- 字段类型如何选择？
- 唯一约束和幂等如何设计？
- 什么时候需要分库分表？
- 分片键怎么选？
- 线上 DDL 有什么风险？
- 误删数据如何恢复？

### 专题对比

- redo log 和 binlog 有什么区别？
- 快照读和当前读有什么区别？
- 普通索引和唯一索引有什么区别？
- 主从复制和读写分离是什么关系？
- 分区表和分库分表有什么区别？

### 分布式事务

- 单库事务和分布式事务的边界是什么？
- XA 两阶段提交有哪些阶段？
- TCC 的 Try、Confirm、Cancel 分别做什么？
- Saga 和 TCC 怎么选？
- 本地消息表和事务消息如何保证最终一致？

### 线上案例

- 你线上遇到过慢 SQL 吗？怎么解决？
- 从库读压力过大为什么可能反过来影响主库？
- 大事务为什么会导致主从延迟？
- 死锁频发怎么定位和治理？
- 线上 DDL、误删数据、连接池打满分别怎么处理？

### 订单系统设计

- 每日 2000 万订单，MySQL 如何设计？
- 订单表怎么拆？分片键怎么选？
- 用户查订单和商家查订单如何同时支持？
- 订单状态流转、支付回调、库存扣减如何保证一致性？
- 历史订单、报表、后台导出如何治理？

### 分库分表

- 为什么要分库分表？
- 何时只分表？何时必须分库？
- 垂直拆分和水平拆分有什么区别？
- 分片键怎么选？
- 分库分表后跨库查询、事务、分页、扩容怎么处理？

### InnoDB 内存与刷盘

- Buffer Pool 是什么？为什么能提升性能？
- 脏页是什么？什么时候刷盘？
- redo log 写满为什么会让 MySQL 抖动？
- MySQL 为什么会突然变慢？

### EXPLAIN 与优化器

- EXPLAIN 重点看哪些字段？
- 为什么有索引却没走？
- `Using filesort` 一定慢吗？
- 统计信息不准确会造成什么问题？

### Online DDL 与 MDL

- 什么是 MDL 元数据锁？
- 为什么简单加字段也可能阻塞线上请求？
- 大表加索引如何安全执行？
- gh-ost 和 pt-online-schema-change 大致原理是什么？

### Go 使用 MySQL

- `database/sql` 连接池参数如何设置？
- 为什么 `rows.Close()` 很重要？
- 事务里为什么不能调用外部服务？
- 超时、重试、幂等如何设计？

### 数据一致性与对账

- 最终一致是不是不保证一致？
- 本地事务 + MQ 后如何发现消息丢失或消费失败？
- 订单、支付、库存如何对账？
- 补偿任务如何设计，如何避免重复补偿？
- 什么场景需要人工兜底？

### 字段类型、字符集与时间坑

- 金额为什么不能用 `float` / `double`？
- `NULL` 会带来哪些查询和 Go Scan 问题？
- `varchar` 长度和索引长度有什么关系？
- `utf8` 和 `utf8mb4` 有什么区别？
- `datetime` 和 `timestamp` 怎么选？

### 存储选型与趋势

- MySQL 适合什么，不适合什么？
- TiDB、ClickHouse、Hive、时序数据库分别适合什么场景？
- 订单系统里哪些数据应该放 MySQL，哪些应该放 ES、ClickHouse、Hive？
- LSM Tree 和 B+Tree 有什么区别？
- 冷热分层、湖仓一体、云原生数据库的发展方向是什么？

## 复习路径

1. 先看 [01-architecture.md](01-architecture.md)，建立整体结构。
2. 再看 [02-index.md](02-index.md) 和 [03-transaction-lock-mvcc.md](03-transaction-lock-mvcc.md)，这是最高频部分。
3. 然后看 [04-log.md](04-log.md) 和 [05-replication-ha.md](05-replication-ha.md)，建立一致性、复制、高可用视角。
4. 再看 [06-slow-sql-optimization.md](06-slow-sql-optimization.md) 和 [07-schema-design.md](07-schema-design.md)，准备场景题和架构题。
5. 看 [13-innodb-buffer-pool.md](13-innodb-buffer-pool.md)、[14-explain-optimizer.md](14-explain-optimizer.md) 和 [15-online-ddl-mdl.md](15-online-ddl-mdl.md)，补齐性能抖动、执行计划和线上变更。
6. 看 [12-sharding.md](12-sharding.md)、[11-order-system-design.md](11-order-system-design.md)、[18-field-types-charset-time.md](18-field-types-charset-time.md) 和 [19-storage-comparison.md](19-storage-comparison.md)，补齐分库分表、系统设计、字段设计和存储选型。
7. 最后看 [08-comparison.md](08-comparison.md)、[09-distributed-transaction.md](09-distributed-transaction.md)、[17-consistency-reconciliation.md](17-consistency-reconciliation.md)、[10-production-cases.md](10-production-cases.md) 和 [16-go-mysql-practice.md](16-go-mysql-practice.md)，补齐横向对比、分布式一致性、线上案例和 Go 实战。

## 答题原则

- 原理题：先讲边界，再讲流程，再讲为什么这样设计。
- 场景题：先确认现象，再拆原因，再给方案和取舍。
- 架构题：先讲目标，再讲基础方案，再讲风险、监控和兜底。
- 不要只背结论，面试官通常会追问“为什么”和“线上怎么处理”。
