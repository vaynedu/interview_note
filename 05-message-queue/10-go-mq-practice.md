# Go MQ 工程实战（Kafka / RocketMQ）

> 05-message-queue 的 Go 落地篇：Kafka / RocketMQ 的 Go 客户端 + 生产者 / 消费者封装 + 超时 + 幂等 + 埋点 + 反模式。
>
> 与 [04-redis/20-go-redis-best-practices.md](../04-redis/20-go-redis-best-practices.md) / [03-mysql/16-go-mysql-practice.md](../03-mysql/16-go-mysql-practice.md) 对偶。
>
> 面试被问"你们 Go 里 Kafka 怎么用"时，这篇能直接给答案。

---

## 一、为什么要这一篇

```
典型面试追问链:
  "用什么 MQ？"
  → "Go 客户端哪个？sarama 还是 kafka-go？"
  → "生产者怎么保证不丢？"
  → "消费者怎么处理 Rebalance？"
  → "Lag 高怎么办？"
  → "消息堆积怎么排查？"
  → "失败重试怎么设计？"

只会 Producer.Send 的人答到第 2 问就卡住。
资深的答案应该是一套 SDK 级别的工程封装。
```

---

## 二、Go Kafka 客户端对比

### 2.1 三大客户端

| 维度 | Sarama | kafka-go (segmentio) | confluent-kafka-go |
| --- | --- | --- | --- |
| 维护者 | IBM (原 Shopify) | segmentio | Confluent 官方 |
| 纯 Go | ✓ | ✓ | ❌（CGO / librdkafka）|
| 性能 | 中 | 中 | **最高** |
| 易用性 | 复杂 | **简洁** | 中 |
| 功能完整 | ✓ | 基础够用 | **最全** |
| 文档 | 一般 | 好 | 好 |
| Issue 响应 | 慢 | 快 | 快 |
| 生产使用 | 很多 | 越来越多 | 大厂多 |
| 依赖 | 纯 Go | 纯 Go | C 库 |

### 2.2 选型建议

```
Sarama:
  ✓ 历史项目 / 社区 case 多
  ✗ API 复杂 / 升级麻烦
  → 如果已经在用就用，新项目慎选

kafka-go (推荐首选):
  ✓ API 简洁 Go idiomatic
  ✓ 活跃维护
  ✓ 纯 Go 无依赖
  ✗ 高级功能略弱
  → 大部分场景首选

confluent-kafka-go:
  ✓ 性能最好（C 库底层）
  ✓ 功能最全（Kafka Streams / Schema Registry）
  ✗ CGO 部署复杂（动态链接 / 交叉编译坑）
  → 对性能极致要求 + 运维能搞定 CGO
```

### 2.3 一般推荐 kafka-go

本篇代码以 `github.com/segmentio/kafka-go` 为主。

---

## 三、Kafka Producer 生产级封装

### 3.1 基础配置

```go
import "github.com/segmentio/kafka-go"

writer := &kafka.Writer{
    Addr:     kafka.TCP("broker1:9092", "broker2:9092", "broker3:9092"),
    Topic:    "order_events",
    Balancer: &kafka.Hash{},  // 按 Key Hash 分区（保顺序）

    // 可靠性
    RequiredAcks:           kafka.RequireAll,  // acks=-1（所有 ISR 确认）
    Async:                  false,             // 同步写（面试常说）

    // 性能
    BatchSize:              100,               // 批量 100 条
    BatchTimeout:           10 * time.Millisecond,
    BatchBytes:             1048576,           // 1MB
    Compression:            kafka.Snappy,      // 压缩

    // 超时
    WriteTimeout:           10 * time.Second,
    ReadTimeout:            10 * time.Second,

    // 重试
    MaxAttempts:            5,                 // 最多重试 5 次

    // 幂等（Kafka 0.11+）
    AllowAutoTopicCreation: false,
    Transport: &kafka.Transport{
        DialTimeout: 5 * time.Second,
        IdleTimeout: 30 * time.Second,
    },
}
defer writer.Close()
```

### 3.2 可靠性配置（不丢消息）

```
acks 三档:
  0:   不等 ACK（最快 / 可能丢）
  1:   leader 收到就行（中）
  all: 所有 ISR 收到（最安全）← 生产推荐

配合参数:
  Broker 端:
    min.insync.replicas = 2       至少 2 副本在 ISR
    unclean.leader.election = false 不选 OSR 为新 leader

  Producer 端:
    RequiredAcks = -1 (kafka.RequireAll)
    MaxAttempts ≥ 3
    enable.idempotence = true (幂等)
    max.in.flight.requests.per.connection ≤ 5
```

### 3.3 幂等 / 事务 Producer

```go
// 幂等: 防止网络重试造成重复
// Kafka 0.11+ 支持
// 默认: enable.idempotence = true

// kafka-go 写法:
writer := &kafka.Writer{
    ...
    Transport: &kafka.Transport{...},
}

// 单分区幂等（PID + Seq）
// 自动去重（Broker 侧）

// 跨分区事务（Kafka Streams / EOS）
// kafka-go 目前支持较弱，如需要用 confluent-kafka-go
```

### 3.4 同步 vs 异步写

```go
// 同步（阻塞等 ACK）
err := writer.WriteMessages(ctx, kafka.Message{
    Key:   []byte("user:123"),
    Value: []byte(`{"event":"order_created"}`),
})
if err != nil {
    // 处理失败（重试 / 落 DB 兜底）
}

// 异步（回调）
writer.Async = true
writer.Completion = func(messages []kafka.Message, err error) {
    if err != nil {
        log.Error("kafka async write fail", "err", err, "count", len(messages))
        metrics.Counter("mq.async_fail").Add(float64(len(messages)))
    }
}
writer.WriteMessages(ctx, msgs...)  // 立刻返回
```

**同步 vs 异步选择**：

```
同步:
  ✓ 可靠（立刻知道成败）
  ✗ 慢（等 ACK 往返）
  → 业务关键路径（订单 / 支付）

异步:
  ✓ 快
  ✗ 批量 fail 怎么兜底？
  → 日志 / 监控 / 非关键业务
```

### 3.5 消息 Key 选择（影响顺序）

```
Key 决定分区:
  Hash(key) % partitions = partition

设计原则:
  同一业务 ID → 同分区 → 保顺序

例子:
  订单消息:
    Key = order_id → 同订单顺序
  
  用户消息:
    Key = user_id → 同用户顺序
  
  无顺序需求:
    Key = nil → 随机分区（负载均衡）
```

### 3.6 失败兜底：Outbox 模式

```go
// 业务写 DB + 发消息 要原子

func CreateOrder(ctx context.Context, order *Order) error {
    return db.Transaction(func(tx *sql.Tx) error {
        // 1. 写业务表
        if _, err := tx.ExecContext(ctx, "INSERT INTO orders ..."); err != nil {
            return err
        }
        
        // 2. 写 outbox 表（同事务）
        event := buildEvent(order)
        if _, err := tx.ExecContext(ctx, 
            "INSERT INTO outbox (event_type, payload, created_at) VALUES (?, ?, ?)",
            "order_created", event, time.Now(),
        ); err != nil {
            return err
        }
        return nil
    })
}

// 后台 Worker 扫 outbox → 发 Kafka → 标记已发
func OutboxWorker(ctx context.Context) {
    ticker := time.NewTicker(1 * time.Second)
    for range ticker.C {
        events := db.Query("SELECT * FROM outbox WHERE sent = 0 LIMIT 100")
        for _, e := range events {
            if err := kafka.Send(ctx, e); err != nil {
                continue  // 下次重试
            }
            db.Exec("UPDATE outbox SET sent = 1 WHERE id = ?", e.ID)
        }
    }
}
```

### 3.7 Producer 通用封装

```go
type Producer struct {
    writer *kafka.Writer
    breaker *gobreaker.CircuitBreaker
    fallback Fallback  // 如失败落 outbox
}

func (p *Producer) Send(ctx context.Context, key, value []byte) error {
    _, err := p.breaker.Execute(func() (interface{}, error) {
        return nil, p.writer.WriteMessages(ctx, kafka.Message{
            Key:   key,
            Value: value,
        })
    })
    if err != nil {
        metrics.Counter("mq.send_fail").Inc()
        // 失败走兜底
        if p.fallback != nil {
            return p.fallback.Save(ctx, key, value)
        }
        return err
    }
    metrics.Counter("mq.send_success").Inc()
    return nil
}
```

---

## 四、Kafka Consumer 生产级封装

### 4.1 基础 Consumer

```go
reader := kafka.NewReader(kafka.ReaderConfig{
    Brokers:  []string{"broker1:9092", "broker2:9092"},
    Topic:    "order_events",
    GroupID:  "order-consumer-group",

    // 消费参数
    MinBytes:       10 * 1024,         // 10KB
    MaxBytes:       10 * 1024 * 1024,  // 10MB
    MaxWait:        1 * time.Second,    // 最长等待
    ReadBatchTimeout: 10 * time.Second,
    
    // 偏移提交
    CommitInterval: 0,  // 0 = 手动提交（推荐）
    StartOffset:    kafka.FirstOffset,  // 新 group 从最早开始

    // Rebalance
    HeartbeatInterval: 3 * time.Second,
    SessionTimeout:    10 * time.Second,
    RebalanceTimeout:  30 * time.Second,
})
defer reader.Close()

for {
    msg, err := reader.FetchMessage(ctx)
    if err != nil {
        if err == ctx.Err() {
            return
        }
        log.Error("fetch fail", "err", err)
        continue
    }

    // 业务处理（必须幂等）
    if err := handleMessage(ctx, msg); err != nil {
        // 不 commit，下次会重试
        log.Error("handle fail", "err", err)
        continue
    }

    // 手动 commit
    if err := reader.CommitMessages(ctx, msg); err != nil {
        log.Error("commit fail", "err", err)
    }
}
```

### 4.2 手动 vs 自动 commit

```
自动 commit（reader.CommitInterval > 0）:
  ✓ 简单
  ✗ 可能丢消息：
    - 拉到消息 → 自动 commit → 业务还没处理 → crash
    - → 下次启动跳过这些消息 → 业务丢

手动 commit（推荐）:
  业务处理成功 → 再 commit
  ✓ 不丢（但可能重复，业务幂等兜底）
  ✗ 需要业务封装

生产强烈推荐手动
```

### 4.3 幂等消费（必备）

```go
// 消息可能重复：
// - Producer 重试
// - Consumer commit 失败重拉
// - Rebalance 重分配

// 业务必须幂等

// 方法 1: DB 唯一索引
func handleMessage(ctx context.Context, msg kafka.Message) error {
    event := parseEvent(msg)
    
    _, err := db.ExecContext(ctx,
        "INSERT INTO order_events (event_id, ...) VALUES (?, ...)",
        event.EventID)
    
    if mysqlErr, ok := err.(*mysql.MySQLError); ok && mysqlErr.Number == 1062 {
        // Duplicate，已处理过
        return nil
    }
    return err
}

// 方法 2: Redis SETNX
func handleMessage(ctx context.Context, msg kafka.Message) error {
    event := parseEvent(msg)
    
    key := fmt.Sprintf("processed:%s", event.EventID)
    ok, err := redis.SetNX(ctx, key, 1, 24*time.Hour).Result()
    if err != nil {
        return err
    }
    if !ok {
        return nil  // 已处理
    }
    
    return doBusiness(ctx, event)
}

// 方法 3: 状态机
func handleMessage(ctx context.Context, msg kafka.Message) error {
    event := parseEvent(msg)
    
    // 只有状态符合预期才推进
    result, err := db.ExecContext(ctx,
        "UPDATE orders SET status = 'paid' WHERE order_no = ? AND status = 'pending'",
        event.OrderNo)
    
    if affected, _ := result.RowsAffected(); affected == 0 {
        return nil  // 状态已变，已处理过
    }
    return err
}
```

### 4.4 并发消费（同分区顺序 + 多分区并行）

```go
// 单分区单 goroutine 保顺序
// 多分区可并行

func consumeTopic(ctx context.Context) {
    // 方式 1: 每个 partition 一个 goroutine（kafka-go 抽象）
    reader := kafka.NewReader(kafka.ReaderConfig{...})
    
    // FetchMessage 内部已经处理分区
    // 同分区消息严格顺序
    for {
        msg, _ := reader.FetchMessage(ctx)
        handleMessage(ctx, msg)
        reader.CommitMessages(ctx, msg)
    }
    
    // 方式 2: 业务侧分发（Hash 再分流）
    // 拿到 msg 后按 Key Hash 分到 goroutine pool
    // 保证同 Key 同 goroutine
}

// 分区内并发处理（小心顺序）
workers := make(chan kafka.Message, 100)
for i := 0; i < 10; i++ {
    go func() {
        for msg := range workers {
            handleMessage(ctx, msg)
        }
    }()
}

for {
    msg, _ := reader.FetchMessage(ctx)
    workers <- msg
}

// ⚠️ 这样丢了顺序。严格顺序 → 单 goroutine
```

### 4.5 Rebalance 处理

```
Rebalance 触发:
  - Consumer 加入 / 退出
  - 心跳超时
  - Topic 分区变化

影响:
  - 所有 consumer 暂停消费
  - 分区重新分配
  - 可能重复消费（老 consumer 没 commit 就被踢）

应对:
  - 缩短消息处理时间（< session.timeout.ms）
  - 增大 session.timeout.ms（但要配合 heartbeat）
  - Static Membership（2.3+，consumer 带唯一 ID，不因重启 rebalance）
  - 增量 Rebalance（Kafka 2.4+，CooperativeSticky）
```

```go
// kafka-go 设置
reader := kafka.NewReader(kafka.ReaderConfig{
    ...
    HeartbeatInterval: 3 * time.Second,
    SessionTimeout:    10 * time.Second,
    RebalanceTimeout:  60 * time.Second,
    // kafka-go 默认使用 Range assignor
    // 需要 CooperativeSticky 可以切换底层客户端
})
```

### 4.6 Consumer 通用封装

```go
type Consumer struct {
    reader  *kafka.Reader
    handler func(ctx context.Context, msg kafka.Message) error
}

func (c *Consumer) Run(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
        }

        msg, err := c.reader.FetchMessage(ctx)
        if err != nil {
            if errors.Is(err, context.Canceled) {
                return nil
            }
            log.Error("fetch fail", "err", err)
            time.Sleep(1 * time.Second)
            continue
        }

        // 埋点
        start := time.Now()
        metrics.Counter("mq.consume", "topic", msg.Topic).Inc()
        
        // 业务处理 + 重试
        err = c.handleWithRetry(ctx, msg, 3)
        metrics.Histogram("mq.consume.duration").Observe(time.Since(start).Seconds())
        
        if err != nil {
            // 达到重试上限 → 进死信
            c.sendToDLQ(ctx, msg, err)
        }

        // 无论成功还是进 DLQ，都要 commit（避免阻塞）
        if err := c.reader.CommitMessages(ctx, msg); err != nil {
            log.Error("commit fail", "err", err)
        }
    }
}

func (c *Consumer) handleWithRetry(ctx context.Context, msg kafka.Message, maxRetries int) error {
    for i := 0; i < maxRetries; i++ {
        if err := c.handler(ctx, msg); err == nil {
            return nil
        } else if !isRetryable(err) {
            return err
        }
        time.Sleep(time.Duration(1<<i) * time.Second)  // 指数退避
    }
    return fmt.Errorf("max retries exceeded")
}
```

### 4.7 死信队列（DLQ）

```go
// 业务失败超过重试 → 发到死信 topic
func (c *Consumer) sendToDLQ(ctx context.Context, msg kafka.Message, err error) {
    dlqMsg := kafka.Message{
        Topic: msg.Topic + ".DLQ",
        Key:   msg.Key,
        Value: msg.Value,
        Headers: []kafka.Header{
            {Key: "original-topic", Value: []byte(msg.Topic)},
            {Key: "error", Value: []byte(err.Error())},
            {Key: "timestamp", Value: []byte(time.Now().Format(time.RFC3339))},
        },
    }
    dlqWriter.WriteMessages(ctx, dlqMsg)
    metrics.Counter("mq.dlq").Inc()
    alert.Fire("message sent to DLQ", dlqMsg)
}

// DLQ 消费者（人工 / 工具修复后重发）
```

### 4.8 消费堆积（Lag）监控

```go
// 监控指标
func monitorLag() {
    ticker := time.NewTicker(30 * time.Second)
    for range ticker.C {
        stats := reader.Stats()
        metrics.Gauge("mq.lag", "topic", stats.Topic).Set(float64(stats.Lag))
        if stats.Lag > 10000 {
            alert.Fire("consumer lag too high", stats.Lag)
        }
    }
}

// Lag 暴涨应急:
// 1. 扩消费实例（但不超过分区数）
// 2. 扩分区（kafka-topics.sh --alter）
// 3. 新建临时 topic 多分区 + 搬运
// 4. 跳过非关键消息
```

---

## 五、RocketMQ Go 实战

### 5.1 官方 Go Client

```go
import "github.com/apache/rocketmq-client-go/v2"
import "github.com/apache/rocketmq-client-go/v2/producer"
import "github.com/apache/rocketmq-client-go/v2/primitive"

// Producer
p, _ := rocketmq.NewProducer(
    producer.WithNameServer([]string{"namesrv:9876"}),
    producer.WithRetry(3),
    producer.WithGroupName("order-producer"),
    producer.WithSendMsgTimeout(3*time.Second),
)
p.Start()
defer p.Shutdown()

msg := &primitive.Message{
    Topic: "order_events",
    Body:  []byte(`{"order_no":"123"}`),
}
msg.WithTag("order_created")
msg.WithKeys([]string{"order_no_123"})
msg.WithShardingKey("user_123")  // 顺序消息分区

result, err := p.SendSync(ctx, msg)
```

### 5.2 事务消息（RocketMQ 独有强项）

```go
// 事务监听器
type OrderTxListener struct{}

func (t *OrderTxListener) ExecuteLocalTransaction(msg *primitive.MessageExt) primitive.LocalTransactionState {
    // 执行本地事务（DB 写订单）
    err := saveOrder(msg)
    if err != nil {
        return primitive.RollbackMessageState
    }
    return primitive.CommitMessageState
}

func (t *OrderTxListener) CheckLocalTransaction(msg *primitive.MessageExt) primitive.LocalTransactionState {
    // Broker 回查：订单是否存在
    if orderExists(msg.GetKeys()) {
        return primitive.CommitMessageState
    }
    return primitive.RollbackMessageState
}

// 事务 Producer
txProducer, _ := rocketmq.NewTransactionProducer(
    &OrderTxListener{},
    producer.WithNameServer([]string{"namesrv:9876"}),
)
txProducer.Start()

msg := primitive.NewMessage("order_events", []byte(`{...}`))
result, err := txProducer.SendMessageInTransaction(ctx, msg)
```

### 5.3 延迟消息

```go
// RocketMQ 18 级延迟
msg := &primitive.Message{
    Topic: "delay_queue",
    Body:  []byte(`{"order_no":"123"}`),
}
msg.WithDelayTimeLevel(3)  // 10s (level 3 = 10s)

// 级别:
// 1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h

// 5.x 任意精确延迟
msg.WithProperty("__STARTDELIVERTIME", "1700000000000")  // 毫秒时间戳
```

### 5.4 顺序消息

```go
// 同 ShardingKey → 同 Queue → 顺序
msg.WithShardingKey("order_123")

result, err := p.SendSync(ctx, msg)

// 消费端
consumer.SetConsumeMode(primitive.ConsumeOrderly)
```

### 5.5 Kafka vs RocketMQ 的 Go 实战对比

| 维度 | Kafka | RocketMQ |
| --- | --- | --- |
| Go 客户端成熟度 | 高（kafka-go）| 中（官方 Go）|
| 事务消息 | 弱（自实现 Outbox）| **原生强** |
| 延迟消息 | 无（插件实现）| **原生 18 级 / 任意** |
| 顺序消息 | 同 Key 同分区 | 同 ShardingKey |
| 回溯能力 | **强**（Offset）| 有限 |
| 吞吐 | **高** | 高 |
| 文档 | 丰富 | 中英文都有 |
| 生态 | Kafka Streams / Connect | RocketMQ Streams |

---

## 六、消息可靠性三端保证

### 6.1 生产端不丢

```go
// 3 招
// 1. acks = all + 重试
writer.RequiredAcks = kafka.RequireAll
writer.MaxAttempts = 5

// 2. 幂等 Producer（Kafka 0.11+）
// enable.idempotence = true（默认开启）

// 3. Outbox 兜底
// 业务写 DB + outbox 事务 → 后台发 MQ
```

### 6.2 Broker 不丢

```
副本 + ISR:
  RF (Replication Factor) = 3
  min.insync.replicas = 2
  unclean.leader.election.enable = false

刷盘:
  - log.flush.interval.ms = 1000（每秒）
  - 或交给 OS PageCache + 副本冗余
```

### 6.3 消费端不丢

```go
// 手动 commit + 处理成功才提交
msg, _ := reader.FetchMessage(ctx)
if err := handle(msg); err != nil {
    // 不 commit，下次重试
    continue
}
reader.CommitMessages(ctx, msg)

// + 业务幂等（应对重复）
```

---

## 七、埋点 + 监控

### 7.1 必打指标

| 指标 | 说明 |
| --- | --- |
| `mq.send.duration` | 发送耗时 |
| `mq.send.success` | 发送成功数 |
| `mq.send.fail` | 发送失败数 |
| `mq.consume.duration` | 消费耗时 |
| `mq.consume.lag` | 消费堆积 |
| `mq.consume.retry` | 重试次数 |
| `mq.dlq` | 死信数量 |
| `mq.rebalance` | Rebalance 次数 |

### 7.2 OpenTelemetry 集成

```go
// 生产侧注入 trace
func (p *Producer) Send(ctx context.Context, msg kafka.Message) error {
    ctx, span := tracer.Start(ctx, "kafka.produce")
    defer span.End()
    
    // 把 trace 注入 header
    propagator.Inject(ctx, kafkaCarrier{&msg})
    
    return p.writer.WriteMessages(ctx, msg)
}

// 消费侧提取 trace
func handleMessage(ctx context.Context, msg kafka.Message) error {
    ctx = propagator.Extract(ctx, kafkaCarrier{&msg})
    ctx, span := tracer.Start(ctx, "kafka.consume")
    defer span.End()
    
    // 业务逻辑在同一条 trace 下
    return doBusiness(ctx, msg)
}
```

### 7.3 Lag 告警

```yaml
# Prometheus rule
- alert: KafkaConsumerLagHigh
  expr: kafka_consumer_lag > 100000
  for: 5m
  annotations:
    summary: "Consumer lag > 100k for 5min"
```

---

## 八、反模式清单

### 反模式 1：自动 commit + 手动处理

```go
// ❌ 可能丢消息
CommitInterval: 5 * time.Second  // 自动提交

// ✅ 手动 commit
CommitInterval: 0
reader.CommitMessages(ctx, msg)
```

### 反模式 2：业务不幂等

```go
// ❌ 消息重发 → 重复处理
func handle(msg) error {
    return processOrder(msg)  // 没去重
}

// ✅ 加幂等
func handle(msg) error {
    if alreadyProcessed(msg.EventID) {
        return nil
    }
    return processOrder(msg)
}
```

### 反模式 3：消息处理太慢导致 Rebalance

```go
// ❌ 单条消息处理 30 秒
// session.timeout = 10s → Rebalance → 重复消费

// ✅ 
// 1. 优化处理时间 < 1s
// 2. 大任务异步化（扔到 worker pool）
// 3. 调大 session.timeout
```

### 反模式 4：Pub/Sub 当可靠消息队列

```go
// ❌ Redis Pub/Sub
// 消费者没订阅 → 消息丢

// ✅ 用 Kafka / RocketMQ
// 持久化 + 消费者组 + ACK
```

### 反模式 5：所有消息用一个 topic

```go
// ❌
topic = "events"
// 订单 / 用户 / 商品 全塞

// ✅ 按业务拆
"order_events"
"user_events"
"product_events"
```

### 反模式 6：无死信队列

```go
// ❌ 失败消息直接丢
if err := handle(msg); err != nil {
    log.Error(err)
    reader.Commit(msg)  // 丢了
}

// ✅ 发死信
if err := handle(msg); err != nil {
    sendToDLQ(msg, err)
    reader.Commit(msg)
}
```

### 反模式 7：同步发大消息

```go
// ❌ 发 10MB 消息
msg.Value = giantPayload

// ✅ 大对象走 OSS，MQ 只存引用
msg.Value = `{"oss_url":"..."}` 
```

### 反模式 8：Lag 不监控

```
❌ 堆积几小时才发现
✅ 监控 lag + 告警 + 自动扩消费
```

### 反模式 9：Producer 全局单例开销大

```go
// ❌ 每次请求 new Writer
writer := &kafka.Writer{...}
defer writer.Close()

// ✅ 全局单例
var globalWriter *kafka.Writer
func init() {
    globalWriter = &kafka.Writer{...}
}
```

### 反模式 10：用 Kafka 做延迟消息

```go
// ❌ 跑定时任务扫 Kafka topic
// 低效 + 精度差

// ✅ 延迟消息走 RocketMQ / 定时队列
```

---

## 九、Kafka vs RocketMQ 面试讲法

```
"我们根据业务特点选型:

  核心业务（异步解耦）→ Kafka
    - 吞吐高（百万 QPS）
    - 回溯好（重放数据）
    - 生态广（Flink / Spark Streaming）

  业务事务消息 → RocketMQ
    - 原生事务消息（half + 回查）
    - 原生延迟消息
    - 业务场景友好

  两者组合用:
    - 数据流 / 日志 / 审计 → Kafka
    - 业务消息 / 订单通知 / 延迟关单 → RocketMQ

Go 客户端:
  - Kafka: kafka-go (推荐首选 / 简洁 / 活跃)
  - RocketMQ: 官方 Go (功能够用 / 文档 OK)

工程化:
  - 所有 Producer 全局单例
  - 所有 Consumer 手动 commit + 业务幂等
  - 失败进 DLQ + 人工 / 工具修复
  - 全链路 OTel 追踪
  - Lag 监控 + 告警"
```

---

## 十、面试加分点

- **kafka-go 选型理由** (简洁 / 纯 Go / 活跃)
- **acks=all + min.insync=2 + 幂等 Producer** 三件套不丢
- **手动 commit + 业务幂等** 消费端
- **Outbox 模式** 业务 + 消息原子
- **OpenTelemetry 跨 MQ 追踪**
- **死信队列 DLQ** 必备
- **Rebalance 应对** Static Membership / 增量
- **Kafka + RocketMQ 组合** 使用
- **10 大反模式**避坑
- **量化数据** 吞吐 / 延迟 / 堆积

---

## 十一、关联阅读

```
本目录:
- 00-mq-map.md                  知识地图
- 02-reliability.md             可靠性三端
- 03-order-and-dedup.md         顺序与去重
- 05-consumer-rebalance.md      Rebalance
- 06-comparison.md              MQ 对比
- 08-production-cases.md        生产案例
- 09-transaction-message-outbox.md 事务消息 / Outbox
- 11-mq-senior-answers.md       MQ 答题模板（待）

跨模块:
- 04-redis/20-go-redis-best-practices.md  Redis Go（对偶）
- 03-mysql/16-go-mysql-practice.md        MySQL Go（对偶）
- 06-distributed/03-transaction.md        分布式事务
```
