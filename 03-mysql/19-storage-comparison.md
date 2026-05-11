# 存储选型对比与未来趋势

> MySQL 是核心 OLTP 存储，但不是所有数据问题都该塞进 MySQL。面试里能讲清“边界”和“选型”，比只会分库分表更有架构感。

## 〇、多概念对比：6 大数据库选型（D 模板）

### 一句话定位

| 数据库 | 一句话定位 |
| --- | --- |
| **MySQL** | **OLTP 王者**（行存 + B+ 树 + InnoDB），互联网核心存储，**生态最广**，单机 TB 级 |
| **PostgreSQL** | **功能最强的开源关系库**（更强 SQL + JSON + GIS + 物化视图），**金融 / 政府首选** |
| **TiDB** | **分布式 NewSQL**（MySQL 协议 + Raft + TiKV），**水平扩展强一致** |
| **MongoDB** | **文档数据库**（BSON / 灵活 schema），**JSON 半结构化数据**首选，弱事务（4.0+ 多文档事务）|
| **ClickHouse** | **列存 OLAP**（MergeTree + 向量化），**亿级数据秒级聚合**，互联网分析标配 |
| **Elasticsearch** | **倒排索引搜索引擎**（Lucene），**全文搜索 + 日志分析 + 多条件检索** |

### 多维度对比（18 维度，必背）

| 维度 | MySQL | PostgreSQL | TiDB | MongoDB | ClickHouse | Elasticsearch |
| --- | --- | --- | --- | --- | --- | --- |
| **类型** | 关系型 OLTP | 关系型 OLTP | 分布式 NewSQL | 文档型 NoSQL | 列存 OLAP | 搜索引擎 |
| **存储引擎** | InnoDB（B+ 树）| Heap + B-tree | TiKV（**LSM Tree**）| WiredTiger（B 树）| **MergeTree（列存）** | Lucene（倒排索引）|
| **数据模型** | 行存 + 关系 | 行存 + 关系 + JSONB | 行存 + 关系 | 文档（BSON）| **列存** | 倒排索引文档 |
| **协议** | MySQL | PostgreSQL | **MySQL 兼容** | MongoDB wire | HTTP / TCP | HTTP REST |
| **SQL 支持** | ✅ 标准 | ✅ **更强**（窗口函数 / CTE / 物化视图 / 数组）| ✅ 兼容 MySQL | 弱（聚合管道）| ✅ SQL 子集 | ❌（DSL）|
| **事务** | ✅ ACID | ✅ ACID + **更严** | ✅ 分布式 ACID（Percolator）| 4.0+ 多文档 | 弱（最终一致）| ❌ |
| **MVCC** | undo log 链 | **多版本元组**（VACUUM 清理）| MVCC + Raft | 文档级 | - | - |
| **水平扩展** | 分库分表（中间件）| 分库分表 | ✅ **原生**（Raft）| ✅ Sharding | ✅ 分布式 MergeTree | ✅ Shard |
| **单机性能（写）** | 5w QPS | 3w QPS | 1-2w QPS（分布式开销）| 5-10w QPS | 100w 行/s（批写）| 1w docs/s |
| **单机性能（读）** | 10w QPS（点查）| 5-10w QPS | 5w QPS | 10w QPS | **亿级行秒级聚合** | 1w QPS |
| **单机容量** | TB 级（推荐 < 5TB）| TB 级 | **PB 级**（分布式）| TB 级 | PB 级 | TB-PB 级 |
| **典型延迟（点查 P99）** | 1-5 ms | 1-5 ms | 5-20 ms | 1-5 ms | N/A（聚合）| 10-50 ms |
| **典型场景** | 电商 / 订单 / 用户 | 金融 / 政府 / 复杂查询 | 大规模 OLTP / 弹性 | 内容管理 / IoT / 实时数据 | 日志分析 / BI / 用户行为 | 日志检索 / 全文搜索 |
| **运维复杂度** | 低（主从）| 低（VACUUM 需调）| 高（多组件 PD/TiKV/TiDB）| 中 | 中（数据建模重要）| 高（集群规划）|
| **国内活跃度** | 极高 | 中（金融）| 高（增长快）| 中 | 极高 | 极高 |
| **代表用户** | 大部分互联网 | 苹果 / Yandex / 银行 | 美团 / 小米 / 知乎 | 头条 / 滴滴（部分）| 字节 / 阿里 / 腾讯 | ELK 用户 |
| **开源协议** | GPL v2 | PostgreSQL（BSD-like）| Apache 2.0 | SSPL（不友好）| Apache 2.0 | Elastic License v2（不再 Apache）|

### 协作时序对比（同一业务在不同存储的查询路径）

```mermaid
sequenceDiagram
    participant App as 应用
    participant My as MySQL
    participant CH as ClickHouse
    participant ES as ES
    participant TiDB as TiDB

    Note over App,TiDB: 点查（按订单号查）
    App->>My: SELECT * WHERE order_id=123<br/>1-5ms（B+ 树）
    App->>CH: SELECT * WHERE order_id=123<br/>~50ms（列存不擅长点查）
    App->>ES: GET /orders/_doc/123<br/>10-50ms
    App->>TiDB: SELECT * WHERE order_id=123<br/>5-20ms（分布式开销）

    Note over App,TiDB: 聚合（昨日订单总数 + GROUP BY）
    App->>My: SELECT COUNT(*), city GROUP BY city<br/>10-100s（行存全表扫）
    App->>CH: 同 SQL<br/>~1s（列存 + 向量化）
    App->>ES: 聚合 DSL<br/>2-10s
    App->>TiDB: 同 SQL<br/>10-30s（TiKV 行存）

    Note over App,TiDB: 全文搜索（"宝马 SUV"）
    App->>My: WHERE title LIKE '%宝马%'<br/>不可用（全表扫）
    App->>ES: { "match": {"title": "宝马 SUV"} }<br/>50-200ms（倒排索引）
```

### 存储引擎深度对比

```
MySQL InnoDB（行存 + B+ 树）:
  ┌───────────────────────┐
  │ Buffer Pool（内存）    │
  │  ├ Free List          │
  │  ├ LRU                │
  │  └ Flush List         │
  └───────────────────────┘
            ↓
  ┌───────────────────────┐
  │ 表空间（磁盘）         │
  │  ├ 聚簇索引（数据）    │
  │  ├ 二级索引            │
  │  └ undo / redo log    │
  └───────────────────────┘
  特点: 点查 / 范围 / 事务 OLTP 强

ClickHouse MergeTree（列存 + LSM 思想）:
  ┌─────────────────────────┐
  │ Part（数据分片，按时间）│
  │  ├ 列文件 col1.bin       │
  │  ├ 列文件 col2.bin       │  ← 各列独立存储
  │  ├ 列文件 col3.bin       │
  │  ├ 主键索引 primary.idx  │  ← 稀疏索引
  │  └ 标记文件              │
  └─────────────────────────┘
  后台 Merge: 合并 Parts 提升查询性能

  特点: 
    - 列存 + 压缩（10:1 - 100:1）
    - 向量化执行（SIMD）
    - 不擅长点查 / 更新（只追加 + 批量 Merge）

TiDB TiKV（分布式 LSM Tree）:
  ┌──────────────────────────┐
  │ TiDB（SQL 层，无状态）    │
  └──────────────────────────┘
            ↓
  ┌──────────────────────────┐
  │ PD（元数据 + 调度）       │
  └──────────────────────────┘
            ↓
  ┌──────────────────────────┐
  │ TiKV（KV 存储，Raft 副本）│
  │  ├ Region 1（k1-k1000）   │
  │  ├ Region 2（k1001-k2000）│
  │  └ ...                    │
  └──────────────────────────┘
  特点:
    - 水平扩展 + 强一致
    - 分布式事务（Percolator）
    - 写性能受 Raft 限制

Elasticsearch（倒排索引）:
  原文档:
    Doc 1: "宝马 SUV X5"
    Doc 2: "宝马 3 系"
    Doc 3: "奔驰 SUV"
  
  倒排索引:
    "宝马" → [Doc1, Doc2]
    "SUV"  → [Doc1, Doc3]
    "X5"   → [Doc1]
  
  查询 "宝马 SUV": 取交集 → Doc1
  特点: 搜索极快 / 不擅长事务和更新
```

### 缺一不可分析

| 假设 | 后果 |
| --- | --- |
| **没 MySQL** | OLTP 互联网核心存储失去事实标准 |
| **没 PostgreSQL** | 复杂 SQL / 金融 / GIS 场景失去最强开源方案 |
| **没 TiDB** | 大规模 OLTP（PB 级）失去 MySQL 兼容的水平扩展方案 |
| **没 MongoDB** | 半结构化数据（schema 灵活）失去原生方案 |
| **没 ClickHouse** | 海量数据 OLAP 分析（用户行为 / 日志）退化到 SQL on Hadoop（慢且复杂）|
| **没 ES** | 全文搜索 / 日志检索退化为 SQL LIKE（性能崩盘）|

### MySQL vs PostgreSQL 深度对比（高频考点）

```
MySQL 优势:
  - 主从复制简单
  - InnoDB 性能稳定
  - 互联网生态最完善（Canal / DTS / Navicat）
  - 学习曲线低

PostgreSQL 优势:
  - SQL 标准支持更全（窗口函数 / CTE 递归 / 物化视图）
  - 类型更丰富（数组 / JSONB / GIS / UUID 原生）
  - MVCC 实现更优（不用 undo log，独立元组版本）
  - 扩展能力强（FDW / 自定义类型 / 索引）
  - DDL 事务支持（ALTER TABLE 可回滚！）
  - 复杂查询优化器更强

国内现状:
  - 互联网公司 90% MySQL
  - 银行 / 政府 50%+ PostgreSQL
  - 新项目趋势: PostgreSQL（功能更强 + License 友好）
```

### 行存 vs 列存深度对比

```
行存（MySQL / PG）:
  存储: [row1: id1,name1,age1] [row2: id2,name2,age2] ...
  
  适合: 
    SELECT * WHERE id=1   ← 整行取出快
    INSERT / UPDATE       ← 整行写入
  
  不适合:
    SELECT AVG(age) FROM table  ← 要读全部列（包括不需要的）

列存（ClickHouse）:
  存储: 
    id 列文件:   [id1, id2, id3, ...]
    name 列文件: [name1, name2, name3, ...]
    age 列文件:  [age1, age2, age3, ...]
  
  适合:
    SELECT AVG(age)         ← 只读 age 列文件，IO 极少
    GROUP BY city           ← 列存压缩 + 向量化
    分析 / 聚合 / BI
  
  不适合:
    SELECT * WHERE id=1     ← 要从多个列文件取数据再拼
    UPDATE / DELETE 单行    ← 多个列文件同步改

性能差距:
  100 亿行 SELECT AVG(age):
    MySQL: 不可行（OOM 或 1 小时+）
    ClickHouse: 5-30 秒

  100 亿行 INSERT 单行:
    MySQL: 1ms
    ClickHouse: 不推荐（应批量写）

业内做法:
  OLTP（在线交易）→ MySQL / PG
  OLAP（分析）   → ClickHouse
  数据同步: Canal/Debezium → Kafka → ClickHouse
```

### 怎么选（决策树）

```mermaid
flowchart TD
    Q1{业务类型?}

    Q1 -->|OLTP 在线交易| Q2
    Q1 -->|OLAP 分析| Q3
    Q1 -->|搜索 / 全文检索| ES
    Q1 -->|半结构化 / 灵活 schema| Mongo[MongoDB]
    Q1 -->|时序数据| TS[InfluxDB / TDengine]
    Q1 -->|图数据| Graph[Neo4j / Nebula]

    Q2{数据规模 + 功能需求?}
    Q2 -->|TB 级 + 简单| My[MySQL]
    Q2 -->|TB 级 + 复杂 SQL / GIS| PG[PostgreSQL]
    Q2 -->|PB 级 + 水平扩展| Ti[TiDB]

    Q3{规模?}
    Q3 -->|亿级 + 互联网| CH[ClickHouse]
    Q3 -->|PB 级 + 离线| Hadoop[Hive / Spark / Presto]

    style My fill:#9f9
    style CH fill:#9ff
    style Ti fill:#ff9
```

**实战推荐表**：

| 业务 | 推荐 | 备注 |
| --- | --- | --- |
| 电商 / 订单 / 用户中心 | **MySQL** | OLTP 标配 |
| 金融账户 / 银行 | **PostgreSQL** | ACID 严格 + 复杂报表 |
| 大规模交易（如美团）| **TiDB / OceanBase** | 水平扩展 |
| 内容管理 / IoT 数据 | **MongoDB** | 文档模型 |
| 用户行为分析 / 漏斗 | **ClickHouse** | 亿级聚合 |
| 全文搜索 / 日志检索 | **Elasticsearch** | 倒排索引 |
| 监控指标 / 时序 | **InfluxDB / Prometheus** | 时序专用 |
| 配置中心 | **etcd / Nacos** | 强一致 KV |
| 缓存 | **Redis** | 不算"数据库" |
| 图数据（社交关系）| **Neo4j / Nebula Graph** | 图遍历 |

### 反模式

```
❌ 把日志写 MySQL → 性能崩盘（应用 ClickHouse / ES）
❌ 把搜索写 MySQL LIKE '%xxx%' → 全表扫（应用 ES）
❌ 把分析跑在主库 → 影响 OLTP 性能（应抽到 ClickHouse / 从库）
❌ MongoDB 当事务库用 → 4.0 之前没多文档事务
❌ ClickHouse 当 OLTP 用 → 不擅长点查 + 不支持更新
❌ TiDB 部署 < 3 节点 → 失去高可用意义
❌ 所有数据塞一个库 → 该拆 OLAP / 搜索 / 缓存 / 主库
```

### 一句话总结（D 模板专属）

> 6 大数据库选型核心是 **"按业务类型分层 + 各司其职"**：
> **OLTP**（MySQL / PG / TiDB）+ **OLAP**（ClickHouse）+ **搜索**（ES）+ **文档**（MongoDB）+ **缓存**（Redis）。
> **缺一不可**：MySQL 替代不了 ClickHouse 的亿级聚合 / ES 替代不了 MySQL 的事务 / TiDB 替代不了 ClickHouse 的列存。
> **业内现状**：互联网公司常用 **MySQL（OLTP）+ ClickHouse（OLAP）+ ES（搜索）+ Redis（缓存）四件套**。
> **关键事实**：行存 vs 列存差距极大（亿级聚合 1 小时 vs 5 秒）；TiDB 兼容 MySQL 协议但不是"性能更好的 MySQL"（点查反而更慢）。

---

## 一、核心判断

先按问题类型选存储：

```mermaid
flowchart TB
    Data["数据需求"] --> Tx{"强事务 / 强一致?"}
    Tx -- 是 --> MySQL["MySQL / PostgreSQL<br/>OLTP"]
    Tx -- 水平扩展强 --> TiDB["TiDB<br/>分布式 SQL"]
    Data --> Search{"搜索 / 多条件检索?"}
    Search -- 是 --> ES["Elasticsearch / OpenSearch"]
    Data --> OLAP{"大规模聚合分析?"}
    OLAP -- 实时 --> CK["ClickHouse<br/>实时 OLAP"]
    OLAP -- 离线 --> Hive["Hive / Spark / Lakehouse<br/>离线数仓"]
    Data --> TS{"时间序列指标?"}
    TS -- 是 --> TSDB["时序数据库<br/>Prometheus / InfluxDB / VictoriaMetrics"]
    Data --> Hot{"热点低延迟?"}
    Hot -- 是 --> Redis["Redis<br/>缓存 / 计数 / 热点"]
```

一句话：

> MySQL 适合交易主链路；分析、搜索、时序、缓存、离线计算应该交给更合适的存储系统。

## 二、常见存储对比

| 存储 | 适合场景 | 不适合场景 | 关键词 |
| --- | --- | --- | --- |
| MySQL | 订单、支付、用户、库存、强事务 OLTP | 海量聚合分析、全文搜索、时序高写入 | 事务、索引、主从、分库分表 |
| TiDB | 分布式 SQL、MySQL 协议兼容、水平扩展 | 极致低延迟热点交易、复杂强热点写 | HTAP、Raft、分布式事务 |
| ClickHouse | 报表、宽表聚合、实时 OLAP、明细分析 | 高频事务更新、强一致交易 | 列式、向量化、压缩 |
| Hive | 离线数仓、T+1 报表、海量历史数据 | 实时查询、低延迟接口 | 离线、批处理、低成本 |
| 时序数据库 | 监控、IoT、指标、按时间聚合 | 复杂事务、关系 join | 时间线、压缩、降采样 |
| Elasticsearch | 搜索、多条件筛选、运营后台检索 | 资金账务、强一致交易 | 倒排索引、搜索、近实时 |
| Redis | 缓存、热点数据、计数、限流、分布式锁辅助 | 持久交易主存储、大规模复杂查询 | 内存、低延迟、缓存 |

## 三、MySQL 的边界

### 1. MySQL 适合什么

适合：

- 订单主表。
- 支付流水。
- 用户账户。
- 库存扣减。
- 状态机流转。
- 幂等记录。
- 强约束和唯一索引。

原因：

- 支持事务。
- 支持唯一约束。
- 支持二级索引。
- 生态成熟。
- 运维和排障经验丰富。

### 2. MySQL 不适合什么

不适合：

- 大规模明细聚合。
- 复杂多维报表。
- 海量全文搜索。
- 高基数时序指标写入。
- 大量历史冷数据低频查询。
- 多维 ad-hoc 分析。

原因：

- 行存更适合 OLTP，不适合大宽表聚合扫描。
- 分库分表后跨分片聚合复杂。
- 大查询会影响 Buffer Pool。
- 报表查询容易拖慢主从复制。

## 四、典型架构：订单系统多存储协同

```mermaid
flowchart LR
    Order["订单服务"] --> MySQL["MySQL<br/>订单主链路"]
    Order --> Redis["Redis<br/>热点订单 / 状态短缓存"]
    Order --> MQ["MQ<br/>订单事件"]
    MQ --> ES["ES<br/>搜索 / 运营查询"]
    MQ --> CK["ClickHouse<br/>实时报表"]
    MQ --> Hive["Hive / Lakehouse<br/>离线数仓"]
    MQ --> TSDB["时序库<br/>QPS / RT / 错误率"]
```

解释：

- **MySQL**：保存订单事实和交易状态。
- **Redis**：缓存热点订单详情、用户最近订单状态。
- **ES**：支持运营后台按手机号、订单号、商家、状态、地址等多条件检索。
- **ClickHouse**：支持实时 GMV、商家经营分析、转化漏斗。
- **Hive / Lakehouse**：保存历史明细，跑 T+1 报表和离线模型。
- **时序库**：保存系统指标，不保存业务交易事实。

面试表达：

```text
订单主链路我会放 MySQL，因为需要事务、唯一索引、状态机和可靠持久化。
但运营搜索不直接扫订单分片，而是同步到 ES。
实时经营分析进入 ClickHouse，历史离线分析进入 Hive 或湖仓。
监控指标进入时序数据库，热点状态可以用 Redis 短缓存。
这样 MySQL 只承担它擅长的 OLTP，不把搜索、报表、时序都压到一个库里。
```

## 五、TiDB：分布式 SQL

TiDB 可以理解为兼容 MySQL 协议的分布式 SQL 数据库，常见特点：

- SQL 接入体验接近 MySQL。
- 底层数据自动分片。
- 通过 Raft 做多副本一致性。
- 适合水平扩展和一定 HTAP 场景。

适合：

- 单机 MySQL 容量和扩展压力明显。
- 希望保留 SQL 能力。
- 不想在业务层做复杂分库分表。
- 查询维度比较多，需要分布式 SQL 能力。

不适合盲目替代 MySQL 的场景：

- 极致低延迟核心交易。
- 强热点行频繁更新。
- 团队没有分布式数据库运维经验。
- 原 SQL 和事务模型不适合分布式执行。

面试表达：

> TiDB 能降低业务分库分表复杂度，但不是“无限扩展的 MySQL”。分布式事务、热点、跨 Region 延迟、执行计划和运维复杂度仍然要评估。

## 六、ClickHouse：实时 OLAP

ClickHouse 是列式 OLAP 数据库。

适合：

- 大宽表聚合。
- 实时报表。
- 明细数据分析。
- 按时间、商家、地区、类目聚合。

为什么快：

- 列式存储，只读需要的列。
- 压缩率高。
- 向量化执行。
- 适合批量写入和大范围扫描。

不适合：

- 高频单行更新。
- 强事务。
- 订单主链路。
- 高并发点查替代 MySQL。

订单场景：

```text
MySQL 保存订单状态。
订单变更通过 MQ 同步到 ClickHouse。
ClickHouse 做 GMV、订单量、商家经营、实时看板。
```

## 七、Hive / 湖仓：离线大数据

Hive 更偏离线数仓，常和 HDFS、Spark、Iceberg、Hudi、Delta Lake 等生态一起出现。

适合：

- 海量历史数据。
- T+1 报表。
- 离线模型训练。
- 明细长期留存。
- 低成本存储。

不适合：

- 低延迟接口查询。
- 高频更新。
- 强事务交易。

常见分层：

```text
ODS：原始明细
DWD：清洗后的明细
DWS：按主题汇总
ADS：应用层报表
```

面试表达：

> MySQL 热库只保留在线业务需要的数据，历史明细通过 CDC 或离线同步进入 Hive/湖仓，用于报表、审计和模型分析。

## 八、时序数据库

时序数据库适合带时间戳的指标数据。

典型数据：

```text
接口 QPS
接口 RT
错误率
CPU / 内存 / 磁盘
IoT 设备指标
业务指标曲线
```

特点：

- 按时间写入。
- 写入量大。
- 查询通常按时间范围聚合。
- 需要降采样和压缩。
- 数据有保留周期。

不适合：

- 复杂关系 join。
- 交易事务。
- 订单主数据。

常见能力：

- Retention：保留最近 N 天。
- Downsampling：高精度数据降采样为分钟、小时级。
- Tag 查询：按实例、接口、业务线过滤。

面试表达：

> 监控指标不适合放 MySQL。时序数据库更适合高频时间序列写入、按时间聚合、压缩和自动过期。

## 九、LSM Tree 是什么

LSM Tree，全称 Log-Structured Merge Tree，是很多现代存储系统的核心思想。

核心思路：

```text
写入先进入内存结构
  -> 追加写 WAL
  -> 内存表满后刷成磁盘有序文件
  -> 后台不断合并文件
```

```mermaid
flowchart TB
    Write["写入"] --> WAL["WAL 追加日志"]
    Write --> Mem["MemTable 内存有序表"]
    Mem --> Flush["刷盘生成 SSTable"]
    Flush --> L0["Level 0"]
    L0 --> Compact["Compaction 合并"]
    Compact --> L1["Level 1 / Level N"]
```

优势：

- 写入吞吐高。
- 顺序写友好。
- 适合大规模写入。

代价：

- 读可能需要查多个层级文件。
- 需要 Bloom Filter、索引、缓存优化。
- 后台 Compaction 会带来写放大和抖动。

常见应用：

- RocksDB。
- LevelDB。
- HBase。
- Cassandra。
- TiKV 底层也和 LSM/RocksDB 相关。

和 B+Tree 的粗略对比：

| 结构 | 写入 | 读取 | 典型系统 |
| --- | --- | --- | --- |
| B+Tree | 原地更新，随机写较多 | 点查和范围查询稳定 | MySQL InnoDB |
| LSM Tree | 顺序写，写吞吐高 | 读放大、Compaction 成本 | RocksDB、HBase、Cassandra |

## 十、冷热分层

冷热分层是未来存储设计里非常重要的方向。

```mermaid
flowchart LR
    Hot["热数据<br/>最近 3 个月<br/>MySQL / Redis"] --> Warm["温数据<br/>3~12 个月<br/>历史库 / ClickHouse"]
    Warm --> Cold["冷数据<br/>1 年以上<br/>Hive / 对象存储"]
```

核心思想：

- 热数据：在线业务高频访问，要求低延迟。
- 温数据：低频查询，允许稍慢。
- 冷数据：主要用于审计、归档、离线分析。

订单例子：

- 最近 3 个月订单在 MySQL 热库。
- 3 到 12 个月订单放历史库或 ClickHouse。
- 1 年以上明细进入 Hive 或对象存储。

收益：

- 控制 MySQL 热表规模。
- 降低索引膨胀。
- 降低备份恢复压力。
- 报表和历史查询不影响交易链路。

## 十一、未来存储发展方向

### 1. 云原生存储

趋势：

- 计算存储分离。
- 弹性扩缩容。
- Serverless 数据库。
- 自动备份、容灾、迁移。

价值：

- 降低运维成本。
- 按需扩容。
- 更强的多可用区容灾能力。

风险：

- 成本不可控。
- 云厂商绑定。
- 性能抖动和网络延迟要评估。

### 2. HTAP

HTAP 希望同时支持：

- OLTP：在线交易。
- OLAP：分析查询。

代表方向：

- TiDB。
- OceanBase。
- PolarDB-X 等分布式数据库。

面试表达：

> HTAP 的目标是减少交易库和分析库之间的数据同步成本，但在线交易和复杂分析天然资源诉求不同，仍然要看隔离能力、延迟和成本。

### 3. 湖仓一体

湖仓方向把数据湖的低成本和数仓的管理能力结合起来。

关键词：

- Iceberg。
- Hudi。
- Delta Lake。
- 对象存储。
- 表格式。
- ACID 元数据。

适合：

- 海量历史数据。
- 离线 + 近实时分析。
- 数据治理。

### 4. 多模数据库

一个系统同时支持多种模型：

- 文档。
- 图。
- 时序。
- KV。
- SQL。

优点：

- 降低系统数量。
- 接入统一。

风险：

- 每种模型都未必做到最好。
- 复杂场景仍要专用系统。

### 5. 存储智能化

趋势：

- 自动索引推荐。
- 自动冷热分层。
- 自动参数调优。
- 自动异常诊断。
- AI 辅助 SQL 优化。

但面试里要保持清醒：

> 自动化能降低运维成本，但核心链路仍然要理解数据模型、查询路径和一致性边界。

## 十二、面试答题模板

### 问为什么不用 MySQL 做所有事情

```text
MySQL 适合 OLTP，比如订单、支付、用户、库存这些需要事务和唯一约束的核心链路。
但它不适合大规模报表聚合、全文搜索、时序指标和海量历史离线分析。
所以我会让 MySQL 承担交易事实，搜索走 ES，实时分析走 ClickHouse，离线历史走 Hive 或湖仓，监控指标走时序数据库，热点数据走 Redis。
这样每个存储承担自己擅长的部分，避免把所有压力都压到 MySQL。
```

### 问 LSM Tree

```text
LSM Tree 的核心是写入先进入内存结构并追加 WAL，之后刷成磁盘有序文件，再由后台 compaction 合并。
它把随机写变成顺序写，适合高写入吞吐场景。
代价是读可能要查多个层级文件，且 compaction 会带来写放大和抖动。
MySQL InnoDB 更典型的是 B+Tree，点查和范围查询稳定；LSM 更常见于 RocksDB、HBase、Cassandra 这类系统。
```

### 问冷热分层

```text
冷热分层是按访问频率和数据生命周期选择存储。
热数据放 MySQL 或 Redis，保证低延迟；
温数据放历史库或 ClickHouse，支持低频查询和分析；
冷数据放 Hive、湖仓或对象存储，降低长期留存成本。
这样可以控制 MySQL 热表规模，减少索引膨胀和备份恢复压力。
```
