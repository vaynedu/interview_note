# 消息队列 · 架构与核心原理

> Kafka 整体架构 / Topic-Partition-Segment / 为什么这么快（零拷贝 + 顺序写 + PageCache + 批处理 + 分区并行 + 压缩）

## 〇、核心提炼（5 段式）

### 核心机制（6 条必背 - Kafka 为什么快）

1. **顺序写磁盘** - HDD 顺序写 600MB/s（接近内存），随机写 100KB/s，**差 6000 倍**
2. **PageCache + 零拷贝** - 数据走 OS PageCache，`sendfile` 系统调用直接 kernel→socket，**避免 4 次拷贝**
3. **批量 + 压缩** - Producer 攒批（BatchSize / linger.ms）+ Snappy/LZ4/zstd 压缩，**网络 + 磁盘双收益**
4. **分区并行** - Topic 多分区，Producer 并发写不同分区，Consumer 组并行消费
5. **二进制协议 + 紧凑布局** - RESP-like 自定义协议，零冗余字段
6. **6.0+ IO 多线程** - 网络读写多线程化（不是命令执行），CPU 多核利用率提升

### 核心本质（必懂）

> Kafka 高吞吐的本质是 **"系统设计每一层都对硬件友好"**：
>
> - **不是某个 trick，是组合拳**：顺序写 + PageCache + 零拷贝 + 批量 + 压缩 + 分区
> - **顺序写是基石**：因为顺序写比随机写快 6000 倍，所以 Kafka 只 append、不 update
> - **PageCache 让"磁盘速度 ≈ 内存"**：OS 自动管理缓存，热数据全在 RAM
> - **零拷贝（sendfile）让消费近 0 拷贝开销**：直接从 PageCache → socket buffer
>
> **关键事实**：
> - **Kafka 单 Broker 吞吐 500MB/s+**（小消息 100w+ QPS）
> - **瓶颈通常是网络**，不是磁盘（顺序写跑满网卡都没问题）
> - **不适合所有场景**：业务消息 / 事务消息 / 延迟消息 → RocketMQ 更合适
>
> **CAP 视角**：
> - Kafka 默认 AP（acks=1）+ 业务幂等
> - 强一致（acks=all + min.insync=2）= CP，但 TPS 损失 30-50%

### 完整流程（面试必背）

```
Producer 写入完整流程:

1. Producer 端:
   - 序列化（Avro / JSON / Protobuf）
   - 按 Key 算分区（Hash 路由）
   - 加入 RecordAccumulator（内存 buffer，按 partition 分桶）
   - 达到 batch.size 或 linger.ms 触发发送
   - Sender 线程网络发送到 Broker

2. Broker 端（Leader 处理）:
   - 接收 RecordBatch
   - append 到对应 Partition 的当前 segment（顺序写）
   - 数据先到 OS PageCache
   - Followers 拉取（异步）
   - 满足 acks 条件后返回 Producer ACK
   - 周期性 fsync（log.flush.interval.ms）

3. Consumer 读取（零拷贝）:
   - poll() 拉取消息
   - Broker 通过 sendfile():
     kernel PageCache → socket buffer → 网卡
     完全不走用户态
   - Consumer 处理 → 提交 offset 到 __consumer_offsets

Topic-Partition-Segment 三层结构:
  Topic（逻辑分类）
    ├── Partition 0（物理分区，有序）
    │   ├── Segment 1 (00000000.log + .index + .timeindex)
    │   ├── Segment 2 (00000000.log ...)
    │   └── ...
    ├── Partition 1
    └── ...

  - Partition 在不同 Broker 上分布 → 水平扩展
  - 每个 Partition 内严格有序
  - Segment 切分：按大小（默认 1GB）或时间（默认 7 天）
  - 老 Segment 直接删除（保留时间到期）
```

### 6 大快原因 - 逐点讲透

#### 1. 顺序写磁盘（最关键）

```
对比:
  HDD 顺序写: 600 MB/s（接近内存）
  HDD 随机写: 100 KB/s（差 6000 倍）
  SSD 顺序写: 3 GB/s+
  SSD 随机写: 300 MB/s+

Kafka 设计:
  消息只 append 到 Segment 末尾
  不 update（已写入不可改）
  老 Segment 整体删除（不是逐条删）
  → 全是顺序写

对比 MySQL（B+ 树）:
  插入可能引起页分裂 → 随机 IO
  → 写性能差 Kafka 1-2 个数量级
```

#### 2. PageCache（让磁盘速度 ≈ 内存）

```
机制:
  - Kafka 不维护进程内大缓存（不像 Redis）
  - 直接依赖 OS PageCache
  - 写 → 先到 PageCache，OS 周期 fsync
  - 读 → PageCache 命中 → 不读磁盘

收益:
  - 热数据全在内存（业务上 99% 是最新数据）
  - 重启不丢缓存（PageCache 是 OS 级的）
  - 进程崩溃不丢缓存

为什么不自己实现缓存:
  - 引入 JVM/Go GC 压力（Kafka JVM 实现的）
  - 重复缓存（OS + 进程双层）
  - PageCache 已经足够好（OS 几十年优化）
```

#### 3. 零拷贝（sendfile）

```
传统读 + 发送（4 次拷贝 + 2 次系统调用）:
  1. read(): disk → kernel buffer  (DMA)
  2. read(): kernel buffer → user buffer  (CPU 拷贝)
  3. send(): user buffer → socket buffer  (CPU 拷贝)
  4. send(): socket buffer → 网卡  (DMA)
  
  → 2 次 CPU 拷贝 + 2 次系统调用 + 2 次上下文切换

sendfile 零拷贝（2 次 DMA + 0 次 CPU 拷贝）:
  1. sendfile(): disk → kernel PageCache  (DMA)
  2. sendfile(): kernel PageCache → 网卡  (DMA，需要网卡支持 SG-DMA)
  
  → 完全不进用户态
  → CPU 拷贝 0 次
  → 网卡吞吐打满

效果:
  Consumer 拉取消息几乎无 CPU 开销
  → Broker 可同时服务大量 Consumer
```

#### 4. 批量 + 压缩

```
Producer 批量:
  batch.size = 16KB（默认）
  linger.ms = 0（默认，攒到 batch.size 立即发）

  生产场景调优:
  batch.size = 64KB / linger.ms = 10
  → 攒更大批 → 网络包数减少 → 吞吐提升

压缩:
  支持 Snappy / LZ4 / zstd
  - Snappy: 速度优先（CPU 开销小）
  - LZ4: 速度 + 压缩比平衡
  - zstd: 压缩比最高（CPU 开销大）

  收益:
  - 网络带宽减少 5-10x
  - 磁盘占用减少 5-10x
  - CPU 开销换网络/磁盘

  典型: 文本日志压缩 10x，二进制压缩 2-3x
```

#### 5. 分区并行

```
Partition 是并行单位:
  - 不同 Partition 在不同 Broker 上
  - Producer 并发写不同 Partition
  - Consumer 组中每个 Consumer 消费不同 Partition

并行度:
  Topic 吞吐 = 单 Partition 吞吐 × Partition 数
  Consumer 并发 ≤ Partition 数

  限制:
  - 单 Topic Partition 数不宜过多（< 1000）
  - 单 Broker Partition 数 < 4000
  - 太多 → 元数据开销 + Controller 压力

Partition 分配:
  - 加分区不会触发数据 rebalance（已有数据留在老分区）
  - 同 Key 可能换分区（破坏顺序）
  - → 一次定到位
```

#### 6. 二进制协议 + 紧凑布局

```
Kafka 协议（自定义）:
  - 二进制（vs HTTP 文本）
  - 紧凑（每个字段定长 + 变长长度前缀）
  - 批量请求支持

消息格式:
  [Length] [Magic] [Attributes] [Timestamp] [Key] [Value]
  - 整批共享 Magic / Compression
  - 单条消息开销 ~10B

对比 HTTP:
  HTTP/1.1 文本 + 重复 header → 开销 100B+
  → 协议层就快 10x
```

### 一句话总结

> Kafka 高吞吐的核心是：**顺序写 + PageCache + 零拷贝 + 批量 + 压缩 + 分区并行 + 二进制协议**，
> 本质是**系统设计每一层都对硬件友好**：顺序写让磁盘 ≈ 内存（差 6000 倍 vs 随机写），
> PageCache + sendfile 让消费近 0 拷贝，分区并行让水平扩展线性。
> **单 Broker 吞吐 500MB/s+ / 百万 QPS**，瓶颈在网络不在磁盘。
> 但 **不适合所有场景**（业务事务消息 / 延迟消息 → RocketMQ），Kafka 是"大数据 / 日志流"的最优解。

---

## 一、整体架构

```mermaid
flowchart TB
    subgraph Producer["生产者"]
        P1[Producer 1]
        P2[Producer 2]
    end

    subgraph Cluster["Kafka 集群"]
        subgraph B1["Broker 1"]
            T1P0[Topic-A P0 Leader]
            T1P1[Topic-A P1 Follower]
        end
        subgraph B2["Broker 2"]
            T2P0[Topic-A P0 Follower]
            T2P1[Topic-A P1 Leader]
        end
        subgraph B3["Broker 3"]
            T3P0[Topic-A P0 Follower]
            T3P1[Topic-A P1 Follower]
        end
    end

    subgraph Consumer["消费者组"]
        C1[Consumer 1]
        C2[Consumer 2]
    end

    ZK[(ZooKeeper / KRaft<br/>元数据 + 选主)]

    P1 -->|push| T1P0
    P2 -->|push| T2P1

    C1 -->|pull| T1P0
    C2 -->|pull| T2P1

    B1 <-.元数据.-> ZK
    B2 <-.元数据.-> ZK
    B3 <-.元数据.-> ZK

    style T1P0 fill:#9f9
    style T2P1 fill:#9f9
```

### 1.1 核心概念

| 概念 | 含义 |
| --- | --- |
| **Producer** | 生产者，发送消息 |
| **Consumer** | 消费者，拉取消息 |
| **Consumer Group** | 消费者组，组内分摊分区，组间广播 |
| **Broker** | Kafka 服务节点 |
| **Topic** | 消息主题（逻辑分类） |
| **Partition** | 分区，Topic 的物理拆分单位（**并行度单位**） |
| **Replica** | 副本，每个分区有 1 主 N 从 |
| **Leader / Follower** | 分区主从。读写都走 Leader，Follower 只复制 |
| **ISR** | In-Sync Replicas，与 Leader 数据同步的副本集合 |
| **Offset** | 消息在分区内的偏移量（位置） |
| **Segment** | 分区的物理文件单元（默认 1GB） |
| **Controller** | 集群中一个 Broker 担任，负责管理元数据和选主 |

### 1.2 Topic / Partition / Segment 三层

```mermaid
flowchart TB
    Topic[Topic: orders]
    Topic --> P0[Partition 0]
    Topic --> P1[Partition 1]
    Topic --> P2[Partition 2]

    P0 --> S00[Segment 0<br/>00000000.log<br/>00000000.index]
    P0 --> S01[Segment 1<br/>10000000.log<br/>10000000.index]
    P0 --> S02[Segment 2<br/>20000000.log]

    style Topic fill:#9f9
    style P0 fill:#9ff
    style S00 fill:#ff9
```

- **Topic** 是逻辑概念
- **Partition** 是物理拆分（一个 Topic 的消息分散到多个 partition）
- **Segment** 是 Partition 内的文件单元（按大小或时间滚动，默认 1GB）

每个 Segment 含：
- `.log`：消息数据（顺序追加）
- `.index`：稀疏索引（offset → 文件位置）
- `.timeindex`：时间索引

### 1.3 写入流程

```mermaid
sequenceDiagram
    participant P as Producer
    participant L as Leader Broker
    participant F1 as Follower 1
    participant F2 as Follower 2

    P->>P: 1. 选 partition (key hash 或 RR)
    P->>P: 2. 累积到 batch (满或超时)
    P->>L: 3. send (一次发一批, 压缩)

    L->>L: 4. 顺序追加到 log 文件 (PageCache)
    L->>F1: 5. 复制
    L->>F2: 复制
    F1-->>L: ack
    F2-->>L: ack

    L-->>P: 6. ack (按 acks 配置返回)

    Note over L,F2: log 异步刷盘到磁盘<br/>(默认 fsync 由 OS 控制)
```

### 1.4 读取流程

```mermaid
sequenceDiagram
    participant C as Consumer
    participant L as Leader Broker
    participant FS as 文件系统/PageCache

    C->>L: fetch (offset=N, max_bytes=1MB)
    L->>FS: 查 .index 找 offset=N 的位置
    L->>FS: 读 .log 文件 (PageCache 命中则极快)
    FS->>L: 数据
    L->>L: sendfile() 零拷贝直接发给 socket
    L-->>C: 数据 (压缩格式, 由 consumer 解压)

    Note over C,L: Consumer 处理后提交 offset
```

## 二、Kafka 为什么这么快（核心高频题）

```mermaid
mindmap
  root((Kafka 快))
    顺序写
      磁盘顺序 I/O 接近内存
      vs 随机 I/O 快 100x
    PageCache
      利用 OS 文件缓存
      不在 JVM 堆
    零拷贝 sendfile
      内核态直接发 socket
      减少 4 次拷贝到 2 次
    批量发送
      Producer batch
      减少 RTT
    压缩
      支持 gzip/snappy/lz4/zstd
      减少网络和磁盘
    分区并行
      多 partition 并发读写
      水平扩展
    异步刷盘
      写 PageCache 即返回
      OS 决定何时 fsync
```

### 2.1 顺序写（最关键）

**误区**：磁盘慢。
**真相**：**磁盘随机 I/O 慢，顺序 I/O 接近内存**。

```
机械硬盘:
- 随机 I/O:  ~100 IOPS, ~1 MB/s
- 顺序 I/O:  ~500 MB/s

SSD:
- 随机 I/O:  ~50K IOPS
- 顺序 I/O:  ~3 GB/s
```

Kafka 设计：**消息只追加到 log 末尾**（append-only），从不修改、不随机读。

```mermaid
flowchart LR
    subgraph Log["Partition 的 .log 文件"]
        M1[msg 1]
        M2[msg 2]
        M3[msg 3]
        M4[msg 4]
        M5[msg 5 ← 新消息追加]
    end

    Producer -->|append| M5

    style M5 fill:#9f9
```

带来的好处：
- 顺序写 → 接近内存速度
- 文件不删（按时间/大小过期整段删）→ 无碎片
- 索引简单（offset → 文件位置，二分查找）

### 2.2 PageCache（操作系统页缓存）

```mermaid
flowchart TB
    App[Kafka 进程] -->|write| PC[(OS PageCache<br/>内存)]
    PC -.异步刷盘.-> Disk[(磁盘)]

    Reader[Consumer 读取] --> PC
    PC -->|命中| Reader
    PC -->|未命中, 读磁盘| Disk

    style PC fill:#9f9
```

**关键设计**：
- 写入直接写 PageCache（OS 管理），**不立即刷盘**
- 读取优先 PageCache（命中则不读磁盘）
- 大量消息读写都在内存中完成

**为什么不用应用层缓存（如 JVM 堆）？**
- PageCache 是 OS 管理，**不占进程堆**（GC 友好）
- 进程重启 PageCache 仍在
- 多个进程可共享
- OS 已经高度优化（预读、回写策略）

### 2.3 零拷贝（Zero-Copy / sendfile）

#### 传统读发流程（4 次拷贝）

```mermaid
flowchart LR
    Disk[(磁盘)] -->|1 DMA| KB[内核 buffer]
    KB -->|2 CPU| UB[用户 buffer]
    UB -->|3 CPU| SB[Socket buffer]
    SB -->|4 DMA| NIC[网卡]

    style KB fill:#fcc
    style UB fill:#fcc
    style SB fill:#fcc
```

4 次数据拷贝 + 4 次上下文切换（用户态↔内核态）。

#### Kafka 零拷贝（2 次拷贝）

```mermaid
flowchart LR
    Disk[(磁盘)] -->|1 DMA| KB[内核 buffer / PageCache]
    KB -->|2 DMA| NIC[网卡]

    Note["sendfile 系统调用<br/>数据不进用户态"]

    style KB fill:#9f9
```

**`sendfile()` 系统调用**：
- 数据从 PageCache 直接 DMA 到网卡
- 不经过用户态（不用读到 Kafka 进程的内存）
- 减少 2 次拷贝 + 上下文切换

适用：消息原样转发（生产者写入啥就发给消费者啥，不修改内容）。Kafka 原始消息体不动，符合 sendfile 模型。

> 实际操作系统还有 `splice` / `mmap+write` 等，sendfile 是 Kafka 主用。

### 2.4 批量发送（Batch）

**生产者**：

```
linger.ms=5         # 等 5ms 凑批次
batch.size=16384    # 16KB 满批就发
```

```mermaid
sequenceDiagram
    participant App as 应用
    participant Buf as Producer Buffer
    participant K as Kafka

    App->>Buf: send msg1
    App->>Buf: send msg2
    App->>Buf: send msg3
    Note over Buf: 累积到 16KB 或等 5ms
    Buf->>K: 一次发送 [msg1, msg2, msg3]
    K-->>Buf: ack
```

**好处**：
- 1 次网络 RTT 发 N 条消息
- 1 次 ack，吞吐 ×N
- 配合压缩（批内压缩比单条好）

**代价**：单条 RT 略增加（最多 linger.ms）。

**消费者**：

```
fetch.min.bytes=1
fetch.max.bytes=50MB
max.poll.records=500
```

每次 fetch 拉一批（默认 50MB / 500 条）。

### 2.5 压缩

```
compression.type=snappy   # 推荐 (lz4/zstd 也常用)
```

| 算法 | 压缩比 | CPU |
| --- | --- | --- |
| gzip | 高 | 高 |
| snappy | 中 | 低 |
| lz4 | 中 | 低 |
| zstd | **高** | **低** |

**zstd 现代首选**：压缩比和 lz4 接近，但比 snappy 高。

**Kafka 的压缩特点**：
- **批级压缩**（一批整体压缩，比单条压缩比高）
- **端到端**：Producer 压缩 → Broker 不解压保存 → Consumer 解压
- 减少网络 + 磁盘

### 2.6 分区并行

```mermaid
flowchart LR
    Topic[Topic A<br/>3 个分区]
    Topic --> P0[Partition 0<br/>Broker 1]
    Topic --> P1[Partition 1<br/>Broker 2]
    Topic --> P2[Partition 2<br/>Broker 3]

    Producer --> P0
    Producer --> P1
    Producer --> P2

    C1[Consumer 1] --> P0
    C2[Consumer 2] --> P1
    C3[Consumer 3] --> P2
```

- 一个 Topic 的写入分散到 N 个 broker（水平扩展）
- 一个消费者组内 N 个消费者并行消费（每个负责一个 partition）
- **吞吐 ≈ 单 partition 吞吐 × 分区数**

### 2.7 异步刷盘

```
log.flush.interval.messages=Long.MAX  # 不主动刷 (默认)
log.flush.interval.ms=Long.MAX
```

**默认依赖 OS 刷盘**：
- 写 PageCache 立即返回（极快）
- OS 根据脏页比例 / 时间 决定何时 fsync 到磁盘
- 一般数秒到几十秒

**代价**：机器突然断电（不是进程崩）可能丢未刷盘数据。**Kafka 靠副本保证可靠**，不靠单机 fsync。

详见 `02-reliability.md`。

### 2.8 总结：5 大核心 + 配套优化

```mermaid
mindmap
  root((Kafka 快))
    设计层面
      顺序写
      分区并行
      批量
      压缩
    OS 层面
      PageCache
      零拷贝 sendfile
      异步刷盘
    数据结构
      .index 稀疏索引
      二分查找
```

## 三、Kafka 选主与元数据管理

### 3.1 ZooKeeper 时代（2.x 及之前）

```mermaid
flowchart LR
    ZK[(ZooKeeper)] --> Meta[元数据<br/>topic/partition/leader]
    ZK --> Ctrl[Controller 选举]

    B1[Broker 1] -.注册.-> ZK
    B2[Broker 2] -.注册.-> ZK
    B3[Broker 3] -.注册.-> ZK

    style ZK fill:#9f9
```

**ZK 的职责**：
- Broker 注册（临时节点）
- Topic 元数据存储
- Controller 选举（一个 Broker 当 Controller，管 partition leader 选举）
- ISR 列表
- 配置

**痛点**：
- ZK 是外部依赖（运维复杂）
- ZK 性能瓶颈（大集群元数据多）
- 选主慢

### 3.2 KRaft 时代（2.8+ 实验，3.3+ 生产可用）

Kafka 自己用 Raft 替代 ZK：

```mermaid
flowchart LR
    subgraph K["KRaft 集群"]
        Q1[Quorum 1<br/>Active Controller]
        Q2[Quorum 2<br/>Standby]
        Q3[Quorum 3<br/>Standby]
    end

    B1[Broker 1] --> Q1
    B2[Broker 2] --> Q1
    B3[Broker 3] --> Q1

    Q1 <-.Raft 复制.-> Q2
    Q1 <-.Raft 复制.-> Q3

    style Q1 fill:#9f9
```

**优势**：
- **去 ZK 依赖**
- 元数据扩展性更好（百万级 partition 可行）
- 重启恢复更快
- 运维简化

**Kafka 4.0** 完全移除 ZK 支持。

## 四、Topic / Partition 设计

### 4.1 partition 数量怎么定？

**考虑因素**：
- **吞吐**：一个 partition 几十~几百 MB/s，按总吞吐 / 单 partition 算
- **消费者并发度**：一个消费者组里**消费者数 ≤ partition 数**（多余的消费者闲置）
- **副本开销**：partition × 副本数 = broker 上总分区数（太多影响性能）

**经验**：
- 小 topic：3~6 partition
- 中 topic：10~30 partition
- 大 topic：50~100 partition
- 单 broker 总 partition < 4000（性能拐点）

### 4.2 partition 选择策略

Producer 发消息时怎么选 partition？

```
1. 指定 partition: 直接用
2. 有 key: hash(key) % partitions (同 key 同 partition, 保证顺序)
3. 无 key:
   - Kafka 2.4 前: round-robin
   - Kafka 2.4+: sticky partitioner (一段时间发同一个, 攒批好压缩)
```

### 4.3 副本数

**通常 3**（1 leader + 2 follower）：
- 容忍 1 个副本挂
- 跨 rack 部署提高容灾
- 副本太多浪费存储和带宽

## 五、Segment 与索引

### 5.1 Segment 结构

每个 partition 是一个目录：

```
/kafka-logs/orders-0/
├── 00000000000000000000.log         # 起始 offset 0 的 segment
├── 00000000000000000000.index       # 稀疏索引
├── 00000000000000000000.timeindex
├── 00000000000000123456.log         # 起始 offset 123456
├── 00000000000000123456.index
└── ...
```

文件名 = 该 segment 的起始 offset。

### 5.2 滚动条件

```
log.segment.bytes=1073741824        # 1GB 满了滚
log.roll.ms=604800000                # 7 天到了滚
```

满任一条件 → 关闭当前 segment，开新 segment。

### 5.3 稀疏索引

```
.index 内容:
offset    file_position
0         0
4         4096
8         8192
```

每**几条消息**才记一次索引（不是每条），节省空间。

**查找过程**：
1. 二分查找 .index 找到最近的 offset
2. 从对应位置顺序扫描 .log 找精确消息

### 5.4 日志清理

两种策略：

#### 删除策略（默认）

```
log.retention.hours=168              # 保留 7 天
log.retention.bytes=-1               # 不限大小
log.cleanup.policy=delete
```

按时间或大小删过期 segment。

#### 压缩策略（log compaction）

```
log.cleanup.policy=compact
```

保留每个 key 的**最新值**，旧值删除。

适合：状态快照（如配置、用户最新位置）。

## 六、性能数据（参考）

| 配置 | 吞吐 |
| --- | --- |
| 单 broker 单 partition | 100 MB/s ~ 500 MB/s |
| 单 broker 多 partition | 1 GB/s+ |
| 单 partition QPS | 10w+ |
| 集群（10 broker, 100 partition） | 10 GB/s+ |

实际取决于：消息大小、副本数、压缩、磁盘类型、网卡。

## 七、高频面试题

**Q1：Kafka 为什么这么快？**

7 大原因（最高频题）：
1. **顺序写**：append-only，磁盘顺序 I/O 接近内存
2. **PageCache**：写读都走 OS 缓存，不进 JVM 堆
3. **零拷贝**：sendfile 减少数据拷贝（4 → 2 次）
4. **批量**：生产者攒批 + 消费者拉批
5. **压缩**：批内压缩（snappy/lz4/zstd）
6. **分区并行**：水平扩展，多 broker 多 partition
7. **异步刷盘**：写 PageCache 即返回，OS 决定刷盘

**Q2：什么是零拷贝？**

传统流程：磁盘 → 内核 buf → 用户 buf → socket buf → 网卡（4 次拷贝）。

**Kafka 用 sendfile**：磁盘 → PageCache → 网卡（2 次 DMA 拷贝，无用户态）。

减少 CPU 拷贝 + 上下文切换 → 大幅提升吞吐。

**Q3：Kafka 用 PageCache 不用 JVM 堆缓存？**

- **不占 JVM 堆**：避免 GC 压力（Kafka 只用很小的堆，几 GB 即可）
- **进程重启缓存仍在**：PageCache 是 OS 的
- **多进程共享**
- **OS 已高度优化**：预读、回写、淘汰

> Kafka 几乎所有数据操作都在 PageCache 中完成。

**Q4：Topic / Partition / Replica 是什么关系？**

```
Topic (逻辑) → 分成 N 个 Partition (物理) → 每个 Partition 有 M 个 Replica (副本)
```

- Topic 是消息分类
- Partition 是并行单位（吞吐扩展）
- Replica 是高可用单位（容错）

**Q5：partition 数量怎么定？**

按以下计算：
- 目标吞吐 / 单 partition 吞吐
- 消费者数量（最多 = partition 数）
- 副本因子（partition × 副本数 = broker 上总分区）

经验：小 topic 3~6，中 10~30，大 50~100。单 broker 不超 4000 partition。

**Q6：Kafka 的消息存储结构？**

```
Topic
└── Partition
    └── Segment (按大小/时间滚动, 默认 1GB / 7 天)
        ├── .log         (消息数据)
        ├── .index       (稀疏索引: offset → 位置)
        └── .timeindex   (时间索引)
```

查找：先二分 .index 找最近 offset，再顺序扫 .log。

**Q7：顺序写为什么快？**

- 磁盘随机 I/O：~100 IOPS（HDD）/ ~50K（SSD）
- 磁盘顺序 I/O：~500 MB/s（HDD）/ ~3 GB/s（SSD）

差**几个数量级**。Kafka 全程顺序追加，从不随机写，速度接近内存。

**Q8：Kafka 不依赖 ZK 后用什么？**

**KRaft 模式**（2.8 实验，3.3 生产可用，4.0 完全移除 ZK）：
- Kafka 自己用 Raft 协议
- Controller 用 Quorum 选举
- 元数据存在 Kafka 自己的内部 topic

优势：去外部依赖、扩展性更好、运维简化。

**Q9：为什么 Kafka 写入不立即 fsync？**

为了性能。fsync 慢（ms 级），写 PageCache 是 ns 级。

**靠副本保证可靠**：即使单机断电丢数据，其他副本仍有。

`acks=all` 确保多数副本写入 PageCache 后才返回，即使所有副本同时断电（极小概率）才会丢。

**Q10：Kafka 和 Redis 的设计差异？**

| | Kafka | Redis |
| --- | --- | --- |
| 用途 | 消息队列 | 内存数据库 |
| 数据存储 | 磁盘（顺序写） | 内存 |
| 持久化 | 默认开 | 可选 |
| 顺序保证 | 分区内 | - |
| 消费模式 | 拉模式 | 推/拉 |
| 多消费者 | 消费者组（一消息一消费者）+ 多组广播 | List 单消费 / Pub-Sub 广播 |
| 数据保留 | 按时间/大小 | TTL / 淘汰 |

## 八、面试加分点

- 强调"顺序写 + PageCache + 零拷贝"是 Kafka 三大杀手锏
- 零拷贝具体是 `sendfile` 系统调用
- PageCache 不在 JVM 堆，不影响 GC
- 顺序写让磁盘性能接近内存
- 分区是并行度单位
- segment 文件名是起始 offset
- 索引是稀疏的（不是每条都记）
- KRaft 替代 ZK 是趋势
- 异步刷盘 + 副本 = 性能 + 可靠的平衡
- Kafka 设计哲学：**简单、批量、顺序**
