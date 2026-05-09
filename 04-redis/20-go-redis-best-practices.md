# Go Redis 最佳实践

> 04-redis 的最后一块拼图：把原理沉到 **Go SDK 封装 + 生产实战**。
>
> 本篇基于 `github.com/redis/go-redis/v9`（原 `go-redis/v8`，v9 后合入官方组织）。
>
> 面试被问"你们 Redis 怎么用的"、"连接池怎么调"、"热 key 怎么防"时，这篇能直接给答案。

---

## 一、为什么需要这一篇

```
典型面试追问链:
  "说下 Redis 缓存"
  → "你们用什么客户端？"
  → "PoolSize 调多少？"
  → "超时怎么设？"
  → "pipeline 你用过吗？"
  → "热 key 在代码层怎么防？"
  → "Redis 挂了业务怎么办？"

只会 KEYS / HGETALL 的人答到第 3 个就卡住。
资深的答案应该是一套 SDK 级别的工程封装。
```

---

## 二、连接池参数调优（重中之重）

### 2.1 go-redis 连接池参数全览

```go
import "github.com/redis/go-redis/v9"

rdb := redis.NewClient(&redis.Options{
    Addr:         "127.0.0.1:6379",
    Password:     "",
    DB:           0,

    // === 连接池 ===
    PoolSize:        100,              // 最大连接数（默认 10 * NumCPU）
    MinIdleConns:    10,               // 最小空闲连接（预热用）
    MaxIdleConns:    50,               // 最大空闲连接（v9 新增）
    ConnMaxIdleTime: 30 * time.Minute, // 空闲多久后关闭（防僵尸）
    ConnMaxLifetime: 1 * time.Hour,    // 连接最大生命周期（防 FD 老化）
    PoolTimeout:     4 * time.Second,  // 取连接超时（池满等待）

    // === 超时 ===
    DialTimeout:  500 * time.Millisecond,  // 建连超时
    ReadTimeout:  200 * time.Millisecond,  // 读超时
    WriteTimeout: 200 * time.Millisecond,  // 写超时

    // === 重试 ===
    MaxRetries:      3,
    MinRetryBackoff: 8 * time.Millisecond,
    MaxRetryBackoff: 512 * time.Millisecond,
})
```

### 2.2 PoolSize 怎么定

```
核心公式:
  PoolSize ≈ QPS × 平均 RT(s) × 冗余系数

示例:
  单实例 QPS = 5000
  平均 RT   = 2ms = 0.002s
  冗余     = 2x
  → PoolSize = 5000 × 0.002 × 2 = 20

但 Go 服务通常远大于这个值（goroutine 波动 + 突发），实际:
  - 小服务:   PoolSize = 50-100
  - 中等服务: PoolSize = 200-500
  - 大流量:   PoolSize = 500-1000 + 多实例 Redis

经验:
  - 看 redis.PoolStats().TotalConns 长期 < PoolSize 的 60% → 够用
  - 看 WaitCount > 0 频繁 → 池太小
  - 看 StaleConns 多 → 空闲超时太长 / Redis 端主动断
```

### 2.3 MinIdleConns 为什么要设

```go
// ❌ 不设 → 冷启动第一波请求都要现建连
// DialTimeout=500ms × 100 并发 = 雪崩

// ✅ 设 MinIdleConns=10
// 启动时预热 10 条连接，第一波请求直接复用
```

### 2.4 ConnMaxLifetime 的坑

```
为什么要设:
  云环境 LB / NAT / Redis 端 timeout 都可能静默断连
  不设 → 某天突然大面积 "connection reset by peer"

推荐:
  ConnMaxLifetime = 30min - 1h
  ConnMaxIdleTime = 10min - 30min

双保险: 超过寿命 OR 超过空闲都关
```

### 2.5 监控连接池

```go
// 定时采集 PoolStats
func exportPoolStats(rdb *redis.Client) {
    stats := rdb.PoolStats()
    metrics.Gauge("redis.pool.total", float64(stats.TotalConns))
    metrics.Gauge("redis.pool.idle", float64(stats.IdleConns))
    metrics.Gauge("redis.pool.stale", float64(stats.StaleConns))
    metrics.Counter("redis.pool.hits").Add(float64(stats.Hits))
    metrics.Counter("redis.pool.misses").Add(float64(stats.Misses))
    metrics.Counter("redis.pool.timeouts").Add(float64(stats.Timeouts))
}
```

**关键指标**：
- `Timeouts` 持续增长 → 池不够 / Redis 慢
- `StaleConns` 多 → 长连接被动断
- `Misses` 高 → 连接建得多，可能是 `MinIdleConns` 太小

---

## 三、超时分层设计

### 3.1 三层超时

```
客户端请求
  ↓
[HTTP 超时 3s]
  ↓ 业务逻辑
  ↓
[Context 超时 500ms]   ← 调 Redis 的上下文
  ↓
[Redis Read/Write 200ms] ← 单次 Redis 调用
  ↓
[Dial 500ms]             ← 建连（池内复用不触发）
```

### 3.2 推荐配置

| 超时 | 推荐值 | 说明 |
| --- | --- | --- |
| DialTimeout | 500ms | 建连（应对云网络抖动）|
| ReadTimeout | 100-500ms | 单命令读（大 key 适当放大）|
| WriteTimeout | 100-500ms | 单命令写 |
| PoolTimeout | ReadTimeout + 1s | 取连接等待（池满场景）|
| Context Timeout | 500ms-2s | 业务整体预算 |

### 3.3 为什么 PoolTimeout 要 > ReadTimeout

```
PoolTimeout < ReadTimeout 的坑:
  请求高峰 → 池满 → 新请求等 4s → PoolTimeout 报错
  但 Read 只给了 200ms → 就算取到连接也立刻超时

推荐:
  PoolTimeout = ReadTimeout + 1s
  给池留一点排队余地，但总体仍受业务超时约束
```

---

## 四、Context 与取消

### 4.1 go-redis v8+ 全部支持 context

```go
ctx, cancel := context.WithTimeout(ctx, 200*time.Millisecond)
defer cancel()

val, err := rdb.Get(ctx, "key").Result()
if err != nil {
    if errors.Is(err, context.DeadlineExceeded) {
        // 超时：连接已被 go-redis 丢弃（不会污染池）
    } else if errors.Is(err, context.Canceled) {
        // 上游取消
    } else if errors.Is(err, redis.Nil) {
        // key 不存在
    }
}
```

### 4.2 Context 取消真的能切断 Redis 吗

```
关键行为（go-redis v9）:
  - 命令发出前 ctx 已 Done → 直接返回错误，不发送
  - 命令发送中 ctx Done → 不会中断 TCP 写，但读响应时立刻返回
  - 连接被标记 bad，归还池时会被丢弃（避免污染下一个调用者）

结论:
  Context 超时 ≠ Redis 端也取消执行。
  Redis 照样会跑完那条命令（比如 KEYS *）。
  所以大命令 + 短超时 = 浪费 Redis 算力。
```

### 4.3 业务代码正确姿势

```go
// ❌ 错误：用背景 ctx 调 Redis
rdb.Get(context.Background(), key)  // 永不超时，泄露 goroutine 风险

// ❌ 错误：超时设在函数外层但没传
func getUser(id int64) (*User, error) {
    return rdb.Get(ctx, key).Result()  // 用了哪个 ctx？
}

// ✅ 正确：显式传 ctx + 业务级超时
func getUser(ctx context.Context, id int64) (*User, error) {
    ctx, cancel := context.WithTimeout(ctx, 200*time.Millisecond)
    defer cancel()
    return rdb.Get(ctx, key).Result()
}
```

---

## 五、Pipeline 封装

### 5.1 原生用法

```go
// Pipeline（非事务，性能 10x+ 提升的关键）
pipe := rdb.Pipeline()
incr := pipe.Incr(ctx, "counter")
expire := pipe.Expire(ctx, "counter", time.Hour)
_, err := pipe.Exec(ctx)  // 一次 RTT

incr.Val()    // 取结果
expire.Val()
```

### 5.2 批量 Pipeline 封装

```go
// 通用批量 Get
func MGetViaPipeline(ctx context.Context, rdb redis.Cmdable, keys []string) (map[string]string, error) {
    if len(keys) == 0 {
        return nil, nil
    }
    pipe := rdb.Pipeline()
    cmds := make([]*redis.StringCmd, len(keys))
    for i, k := range keys {
        cmds[i] = pipe.Get(ctx, k)
    }
    _, err := pipe.Exec(ctx)
    // Pipeline 整体可能报错，但单个命令结果仍可用
    if err != nil && !errors.Is(err, redis.Nil) {
        return nil, err
    }

    result := make(map[string]string, len(keys))
    for i, cmd := range cmds {
        v, err := cmd.Result()
        if errors.Is(err, redis.Nil) {
            continue
        }
        if err != nil {
            return nil, fmt.Errorf("get %s: %w", keys[i], err)
        }
        result[keys[i]] = v
    }
    return result, nil
}
```

### 5.3 Pipeline vs MGet

```
MGet:
  原生命令，一次 RTT，服务端原子
  只能 GET

Pipeline:
  客户端组合，一次 RTT，服务端非原子
  任意命令组合

结论:
  纯批量读 → MGet
  混合命令（Get + Set + Expire）→ Pipeline
```

### 5.4 Pipeline 的坑

```
❌ Pipeline 里别跑大 key：单个 HGETALL 卡住会阻塞整个 pipeline 结果
❌ Pipeline 命令数太多（>1000）会占大量客户端内存
❌ Cluster 模式下跨 slot 的 pipeline 会被拆成多个 → 用 Hash Tag {user:123}
```

---

## 六、Transaction（MULTI/EXEC/WATCH）封装

### 6.1 原生 TxPipeline

```go
// MULTI/EXEC 但没 WATCH（服务端原子，但无乐观锁）
pipe := rdb.TxPipeline()
pipe.Incr(ctx, "counter")
pipe.Expire(ctx, "counter", time.Hour)
_, err := pipe.Exec(ctx)
```

### 6.2 WATCH + CAS 封装

```go
// 乐观锁扣减库存
func DecrStock(ctx context.Context, rdb *redis.Client, key string) error {
    return rdb.Watch(ctx, func(tx *redis.Tx) error {
        n, err := tx.Get(ctx, key).Int()
        if err != nil && !errors.Is(err, redis.Nil) {
            return err
        }
        if n <= 0 {
            return ErrOutOfStock
        }
        _, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
            pipe.Decr(ctx, key)
            return nil
        })
        return err
    }, key)
}

// WATCH 失败会返回 redis.TxFailedErr，业务层决定重试几次
```

### 6.3 为什么大多数场景不用 Transaction

```
Redis 事务的弱点:
  - 不支持回滚（中间命令失败，后续照常执行）
  - WATCH 在高并发下失败率高（写冲突多）
  - 不如 Lua 脚本原子 + 性能好

现实:
  需要原子 → 直接 Lua
  Transaction 只在：
    - 多命令顺序执行（不需要 Lua 复杂逻辑）
    - 需要 CAS 语义（WATCH）
```

---

## 七、Lua 脚本封装

### 7.1 Script 预加载 + EvalSha

```go
// 定义一次，复用
var decrStockScript = redis.NewScript(`
local v = redis.call('GET', KEYS[1])
if not v then return -1 end
local n = tonumber(v)
if n <= 0 then return 0 end
redis.call('DECR', KEYS[1])
return 1
`)

func DecrStock(ctx context.Context, rdb *redis.Client, key string) (int64, error) {
    // go-redis Run 自动处理 EvalSha → NOSCRIPT → Eval
    return decrStockScript.Run(ctx, rdb, []string{key}).Int64()
}
```

### 7.2 Lua 脚本的约束

```
✓ Cluster 模式下 KEYS 必须在同一 slot
  → 用 {tag} 包裹：{user:123}:stock / {user:123}:lock

✓ 脚本必须幂等（可能重试）

✓ 禁止阻塞命令（BLPOP / SUBSCRIBE）

✓ 脚本执行是单线程阻塞的，避免跑大循环（>100ms 的脚本会整体拖慢 Redis）

✓ 用 redis.pcall 替代 redis.call 可捕获错误
```

### 7.3 常用 Lua 脚本模板

```go
// 令牌桶限流
var tokenBucket = redis.NewScript(`
local key      = KEYS[1]
local capacity = tonumber(ARGV[1])
local rate     = tonumber(ARGV[2])
local now      = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local bucket = redis.call('HMGET', key, 'tokens', 'ts')
local tokens = tonumber(bucket[1]) or capacity
local ts     = tonumber(bucket[2]) or now

local delta = math.max(0, now - ts) * rate
tokens = math.min(capacity, tokens + delta)

if tokens < requested then
    return 0
end
tokens = tokens - requested
redis.call('HMSET', key, 'tokens', tokens, 'ts', now)
redis.call('EXPIRE', key, 60)
return 1
`)

// 分布式锁释放（只释放自己加的）
var unlockScript = redis.NewScript(`
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
end
return 0
`)
```

---

## 八、批量读写

### 8.1 MGet / MSet

```go
// 最快的批量读（前提：同 slot）
vals, err := rdb.MGet(ctx, "k1", "k2", "k3").Result()
// vals 是 []interface{}，nil 表示不存在
```

### 8.2 Cluster 模式下的批量：分 slot 并发

```go
// 跨 slot 的 MGet 会报 CROSSSLOT 错误
// 解决方案：按 slot 分组 + 并发 Pipeline

func ClusterMGet(ctx context.Context, rdb *redis.ClusterClient, keys []string) (map[string]string, error) {
    // 按 slot 分桶
    slotKeys := make(map[int][]string)
    for _, k := range keys {
        slot := rdb.ClusterKeySlot(ctx, k).Val()
        slotKeys[int(slot)] = append(slotKeys[int(slot)], k)
    }

    // 每个 slot 并发
    result := make(map[string]string, len(keys))
    var mu sync.Mutex
    g, ctx := errgroup.WithContext(ctx)

    for _, ks := range slotKeys {
        ks := ks
        g.Go(func() error {
            vals, err := rdb.MGet(ctx, ks...).Result()
            if err != nil {
                return err
            }
            mu.Lock()
            defer mu.Unlock()
            for i, v := range vals {
                if s, ok := v.(string); ok {
                    result[ks[i]] = s
                }
            }
            return nil
        })
    }
    return result, g.Wait()
}
```

### 8.3 批量写：Pipeline 分批

```go
const batchSize = 500

func BulkSet(ctx context.Context, rdb redis.Cmdable, kvs map[string]string, ttl time.Duration) error {
    keys := make([]string, 0, len(kvs))
    for k := range kvs {
        keys = append(keys, k)
    }
    for i := 0; i < len(keys); i += batchSize {
        end := i + batchSize
        if end > len(keys) {
            end = len(keys)
        }
        pipe := rdb.Pipeline()
        for _, k := range keys[i:end] {
            pipe.Set(ctx, k, kvs[k], ttl)
        }
        if _, err := pipe.Exec(ctx); err != nil {
            return err
        }
    }
    return nil
}
```

---

## 九、埋点指标

### 9.1 go-redis Hook 接口（v9）

```go
type MetricsHook struct{}

func (h *MetricsHook) DialHook(next redis.DialHook) redis.DialHook {
    return func(ctx context.Context, network, addr string) (net.Conn, error) {
        start := time.Now()
        conn, err := next(ctx, network, addr)
        metrics.Histogram("redis.dial.duration").Observe(time.Since(start).Seconds())
        if err != nil {
            metrics.Counter("redis.dial.error").Inc()
        }
        return conn, err
    }
}

func (h *MetricsHook) ProcessHook(next redis.ProcessHook) redis.ProcessHook {
    return func(ctx context.Context, cmd redis.Cmder) error {
        start := time.Now()
        err := next(ctx, cmd)
        dur := time.Since(start)

        labels := []string{"cmd", cmd.Name()}
        metrics.HistogramWithLabels("redis.cmd.duration", labels).Observe(dur.Seconds())
        if err != nil && !errors.Is(err, redis.Nil) {
            metrics.CounterWithLabels("redis.cmd.error", labels).Inc()
        }
        if dur > 100*time.Millisecond {
            log.Warn("slow redis", "cmd", cmd.Name(), "dur", dur, "args", cmd.Args())
        }
        return err
    }
}

func (h *MetricsHook) ProcessPipelineHook(next redis.ProcessPipelineHook) redis.ProcessPipelineHook {
    return func(ctx context.Context, cmds []redis.Cmder) error {
        start := time.Now()
        err := next(ctx, cmds)
        metrics.HistogramWithLabels("redis.pipe.duration",
            []string{"size", strconv.Itoa(len(cmds))}).Observe(time.Since(start).Seconds())
        return err
    }
}

rdb.AddHook(&MetricsHook{})
```

### 9.2 必打的指标清单

| 指标 | 类型 | 说明 |
| --- | --- | --- |
| `redis.cmd.duration{cmd}` | Histogram | 各命令 RT 分布 |
| `redis.cmd.error{cmd}` | Counter | 错误计数（排除 redis.Nil）|
| `redis.pool.timeouts` | Counter | 取连接超时次数 |
| `redis.pool.total/idle/stale` | Gauge | 池状态 |
| `redis.pipe.size` | Histogram | Pipeline 命令数分布 |
| `redis.slowlog` | Counter | 慢命令（自定义阈值）|
| `redis.hit_rate` | Gauge | 缓存命中率（业务层统计）|

### 9.3 OpenTelemetry 集成

```go
import "github.com/redis/go-redis/extra/redisotel/v9"

// Trace
redisotel.InstrumentTracing(rdb)
// Metrics
redisotel.InstrumentMetrics(rdb)
```

官方包已经覆盖了大部分埋点，自定义 Hook 主要用来加业务维度（如租户 ID）。

---

## 十、熔断 / 限流 / Fallback SDK 包装

### 10.1 Redis 降级设计的核心原则

```
Redis 不是业务数据源，是性能优化。
→ Redis 挂了业务必须能跑（可能慢 / 降级）。
→ SDK 层要封装熔断 + fallback。
```

### 10.2 带熔断的 Client 封装

```go
import "github.com/sony/gobreaker"

type Client struct {
    rdb     *redis.Client
    breaker *gobreaker.CircuitBreaker
    fb      Fallback  // DB 回源 / 默认值
}

type Fallback interface {
    Get(ctx context.Context, key string) (string, error)
}

func New(rdb *redis.Client, fb Fallback) *Client {
    cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
        Name:        "redis",
        MaxRequests: 3,                    // 半开时允许的探测请求
        Interval:    60 * time.Second,     // 计数窗口
        Timeout:     30 * time.Second,     // 打开 → 半开等待
        ReadyToTrip: func(c gobreaker.Counts) bool {
            return c.Requests >= 20 &&
                float64(c.TotalFailures)/float64(c.Requests) > 0.5
        },
        OnStateChange: func(name string, from, to gobreaker.State) {
            log.Warn("redis breaker state change", "from", from, "to", to)
            metrics.Counter("redis.breaker.change").Inc()
        },
    })
    return &Client{rdb: rdb, breaker: cb, fb: fb}
}

func (c *Client) Get(ctx context.Context, key string) (string, error) {
    result, err := c.breaker.Execute(func() (interface{}, error) {
        val, err := c.rdb.Get(ctx, key).Result()
        if errors.Is(err, redis.Nil) {
            // key 不存在不算 Redis 故障（不触发熔断）
            return "", err
        }
        return val, err
    })
    if err == nil {
        return result.(string), nil
    }
    if errors.Is(err, redis.Nil) {
        return "", err
    }
    // Redis 故障或熔断打开 → fallback
    metrics.Counter("redis.fallback").Inc()
    if c.fb != nil {
        return c.fb.Get(ctx, key)
    }
    return "", err
}
```

**关键点**：`redis.Nil` 不触发熔断（正常业务），只有连接 / 超时 / 服务端异常才算故障。

### 10.3 限流（客户端保护 Redis）

```go
// golang.org/x/time/rate 客户端令牌桶
type Client struct {
    rdb     *redis.Client
    limiter *rate.Limiter  // 防止 Redis 被自己打挂
}

func (c *Client) Get(ctx context.Context, key string) (string, error) {
    if !c.limiter.Allow() {
        metrics.Counter("redis.rate_limited").Inc()
        return "", ErrRateLimited
    }
    return c.rdb.Get(ctx, key).Result()
}
```

### 10.4 Fallback 策略

```
Fallback 三档:

1. 降级读 DB（最常见）
   - 加一层本地限流，防止 DB 被打挂

2. 默认值
   - 如首页推荐：Redis 挂 → 返回兜底推荐列表

3. 直接失败
   - 强一致场景：宁可失败也不返回脏数据
```

---

## 十一、热 key 保护

### 11.1 什么是"热 key 在代码层的保护"

```
Redis 端热 key 治理:
  - 拆分 key
  - 本地缓存镜像
  - 读写分离

Go 服务层（本篇重点）:
  - singleflight 合并回源
  - 热 key 自动检测 + 本地短 TTL 缓存
  - 熔断触发时的保护
```

### 11.2 singleflight：合并并发回源

```go
import "golang.org/x/sync/singleflight"

type Cache struct {
    rdb *redis.Client
    sf  singleflight.Group
}

func (c *Cache) Get(ctx context.Context, key string, loader func() (string, error)) (string, error) {
    // 先查 Redis
    val, err := c.rdb.Get(ctx, key).Result()
    if err == nil {
        return val, nil
    }
    if !errors.Is(err, redis.Nil) {
        return "", err
    }

    // Redis miss → 回源（同 key 并发合并为一次）
    v, err, _ := c.sf.Do(key, func() (interface{}, error) {
        // 双检查（singleflight 过程中可能其他协程已写入）
        if val, err := c.rdb.Get(ctx, key).Result(); err == nil {
            return val, nil
        }
        // 真正回源
        val, err := loader()
        if err != nil {
            return "", err
        }
        c.rdb.Set(ctx, key, val, 5*time.Minute)
        return val, nil
    })
    if err != nil {
        return "", err
    }
    return v.(string), nil
}
```

**效果**：1000 并发同 key miss，只有 1 次回源 DB。

### 11.3 热 key 自动识别 + 本地短 TTL

```go
// 热点统计：滑动窗口计数
type HotKeyDetector struct {
    threshold int           // 阈值：多少 QPS 算热
    window    time.Duration // 窗口
    counter   sync.Map      // key → *atomicCounter
}

func (d *HotKeyDetector) Hit(key string) bool /* isHot */ {
    v, _ := d.counter.LoadOrStore(key, newAtomicCounter(d.window))
    c := v.(*atomicCounter)
    qps := c.Incr()
    return qps >= d.threshold
}

// 结合本地缓存
type LocalRedisCache struct {
    rdb      *redis.Client
    local    *ristretto.Cache
    detector *HotKeyDetector
}

func (c *LocalRedisCache) Get(ctx context.Context, key string) (string, error) {
    // L1: 本地
    if v, ok := c.local.Get(key); ok {
        return v.(string), nil
    }

    // L2: Redis
    val, err := c.rdb.Get(ctx, key).Result()
    if err != nil {
        return "", err
    }

    // 热点识别 → 自动写本地（短 TTL 防脏）
    if c.detector.Hit(key) {
        c.local.SetWithTTL(key, val, 1, 30*time.Second)
    }
    return val, nil
}
```

### 11.4 大 key 保护

```go
// 拒绝直接 HGETALL 大 key
func SafeHGetAll(ctx context.Context, rdb *redis.Client, key string, maxFields int) (map[string]string, error) {
    size, err := rdb.HLen(ctx, key).Result()
    if err != nil {
        return nil, err
    }
    if size > int64(maxFields) {
        return nil, fmt.Errorf("hash too large: %d fields, use HSCAN", size)
    }
    return rdb.HGetAll(ctx, key).Result()
}

// 改用 HSCAN 游标分批
func HScanAll(ctx context.Context, rdb *redis.Client, key string) (map[string]string, error) {
    result := make(map[string]string)
    var cursor uint64
    for {
        items, next, err := rdb.HScan(ctx, key, cursor, "", 100).Result()
        if err != nil {
            return nil, err
        }
        for i := 0; i < len(items); i += 2 {
            result[items[i]] = items[i+1]
        }
        if next == 0 {
            break
        }
        cursor = next
    }
    return result, nil
}
```

---

## 十二、多级缓存通用组件

### 12.1 组件接口设计

```go
// 目标：业务代码只写 cache.Get(ctx, key, loader)，其他全自动

type MultiCache interface {
    Get(ctx context.Context, key string, dst any, loader Loader) error
    Set(ctx context.Context, key string, val any, ttl time.Duration) error
    Del(ctx context.Context, key string) error
}

type Loader func(ctx context.Context) (any, error)
```

### 12.2 完整实现

```go
type multiCache struct {
    local    LocalCache   // 如 ristretto
    remote   *redis.Client
    sf       singleflight.Group
    breaker  *gobreaker.CircuitBreaker
    detector *HotKeyDetector

    localTTL  time.Duration
    remoteTTL time.Duration
    codec     Codec  // JSON / msgpack / protobuf
}

func (c *multiCache) Get(ctx context.Context, key string, dst any, loader Loader) error {
    // === L1: 本地 ===
    if data, ok := c.local.Get(key); ok {
        return c.codec.Unmarshal(data.([]byte), dst)
    }

    // === L2: Redis（带熔断）===
    result, err := c.breaker.Execute(func() (interface{}, error) {
        return c.remote.Get(ctx, key).Bytes()
    })
    if err == nil {
        data := result.([]byte)
        // 热点识别 → 写本地
        if c.detector.Hit(key) {
            c.local.SetWithTTL(key, data, int64(len(data)), c.localTTL)
        }
        return c.codec.Unmarshal(data, dst)
    }
    if !errors.Is(err, redis.Nil) && !errors.Is(err, gobreaker.ErrOpenState) {
        log.Warn("redis get fail, fallback to loader", "err", err)
    }

    // === L3: 回源（singleflight 合并）===
    v, err, _ := c.sf.Do(key, func() (interface{}, error) {
        // 再查一次 Redis（可能已被别的协程写入）
        if result, err := c.remote.Get(ctx, key).Bytes(); err == nil {
            return result, nil
        }
        val, err := loader(ctx)
        if err != nil {
            return nil, err
        }
        data, err := c.codec.Marshal(val)
        if err != nil {
            return nil, err
        }
        // 异步写 Redis，不阻塞返回
        go func() {
            ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
            defer cancel()
            c.remote.Set(ctx, key, data, c.remoteTTL)
        }()
        return data, nil
    })
    if err != nil {
        return err
    }
    return c.codec.Unmarshal(v.([]byte), dst)
}

func (c *multiCache) Del(ctx context.Context, key string) error {
    c.local.Del(key)
    return c.remote.Del(ctx, key).Err()
}
```

### 12.3 业务使用

```go
var cache = NewMultiCache(local, rdb, Options{
    LocalTTL:  30 * time.Second,
    RemoteTTL: 10 * time.Minute,
})

func GetUser(ctx context.Context, id int64) (*User, error) {
    var u User
    err := cache.Get(ctx, fmt.Sprintf("user:%d", id), &u, func(ctx context.Context) (any, error) {
        return userRepo.FindByID(ctx, id)
    })
    if err != nil {
        return nil, err
    }
    return &u, nil
}
```

---

## 十三、反模式清单（面试避坑）

### 反模式 1：KEYS * 扫描

```go
// ❌ 生产禁用：阻塞 Redis
rdb.Keys(ctx, "user:*")

// ✅ 改用 SCAN 游标
iter := rdb.Scan(ctx, 0, "user:*", 100).Iterator()
for iter.Next(ctx) {
    key := iter.Val()
    // ...
}
```

### 反模式 2：每请求新建 Client

```go
// ❌ 每个请求建连接
func handler(w http.ResponseWriter, r *http.Request) {
    rdb := redis.NewClient(&redis.Options{...})  // 泄露！
    defer rdb.Close()
    ...
}

// ✅ 全局单例 + 池复用
var rdb = redis.NewClient(...)  // init 时创建
```

### 反模式 3：大 value 走 Redis

```go
// ❌ 塞 10MB 的 JSON
rdb.Set(ctx, "bigdata", giantJSON, 0)

// ✅ 大对象走对象存储，Redis 只存引用
rdb.Set(ctx, "bigdata:ref", "oss://bucket/path", 0)
```

### 反模式 4：Set 不带 TTL

```go
// ❌ 永不过期 → 内存涨爆
rdb.Set(ctx, key, val, 0)

// ✅ 永远带 TTL（或主动清理）
rdb.Set(ctx, key, val, 1*time.Hour)
```

### 反模式 5：用 Redis 当队列但不用 Stream

```go
// ⚠️ LPUSH + BRPOP 简单队列：丢消息、无 ACK
// ✅ Redis Stream：XADD + XREADGROUP + XACK
```

### 反模式 6：忘记 Pipeline 错误处理

```go
// ❌
pipe.Exec(ctx)  // 返回值丢了

// ✅
if _, err := pipe.Exec(ctx); err != nil && !errors.Is(err, redis.Nil) {
    return err
}
```

### 反模式 7：JSON 序列化大对象 + 高频读

```go
// ❌ 每次读都 Unmarshal 1MB JSON → CPU 爆
// ✅ 拆字段用 Hash：HGET user:1 name
//    或用 msgpack / protobuf
```

### 反模式 8：Hook 里做阻塞操作

```go
// ❌ ProcessHook 里打日志写远程
// → Redis 每个命令都被远程日志拖慢

// ✅ Hook 里只做本地采样 / 异步上报
```

### 反模式 9：忽视 Cluster 的 MOVED 重定向

```go
// ⚠️ 直连某个节点 + 手动路由
// → 扩缩容 / 迁移时大量 MOVED 错误

// ✅ 用 ClusterClient，自动处理路由
rdb := redis.NewClusterClient(&redis.ClusterOptions{
    Addrs: []string{...},
})
```

### 反模式 10：Lua 脚本里用随机数 / 时间不传参

```lua
-- ❌ 主从不一致（Redis 7 前）
local now = redis.call('TIME')

-- ✅ 从客户端传入
-- ARGV[1] = now
```

---

## 十四、推荐配置模板

### 14.1 小型服务

```go
redis.NewClient(&redis.Options{
    Addr:            "127.0.0.1:6379",
    PoolSize:        50,
    MinIdleConns:    5,
    ConnMaxLifetime: 30 * time.Minute,
    DialTimeout:     500 * time.Millisecond,
    ReadTimeout:     200 * time.Millisecond,
    WriteTimeout:    200 * time.Millisecond,
    PoolTimeout:     1500 * time.Millisecond,
    MaxRetries:      2,
})
```

### 14.2 高并发服务（5w+ QPS）

```go
redis.NewClusterClient(&redis.ClusterOptions{
    Addrs:           []string{...},
    PoolSize:        300,                     // 每节点
    MinIdleConns:    30,
    MaxIdleConns:    150,
    ConnMaxLifetime: 1 * time.Hour,
    ConnMaxIdleTime: 20 * time.Minute,
    DialTimeout:     500 * time.Millisecond,
    ReadTimeout:     100 * time.Millisecond,  // 更严格
    WriteTimeout:    100 * time.Millisecond,
    PoolTimeout:     1 * time.Second,
    MaxRetries:      1,                       // 高并发下重试=雪崩放大器
    RouteByLatency:  true,                    // 就近读从
})
```

### 14.3 读多写少 + 强缓存

```go
// Client 侧读偏好
&redis.ClusterOptions{
    ReadOnly:       true,  // 从节点分担读
    RouteRandomly:  true,  // 随机路由到副本
}
```

---

## 十五、面试表达

```text
"你们 Redis 怎么封装的？"

我们在 SDK 层做了这几件事：

1. 连接池按 QPS × RT 估算，监控 PoolStats 调整
2. 超时分三层：Dial 500ms / Read 200ms / Context 业务预算
3. Pipeline 封装了批量 Get/Set + Cluster 跨 slot 自动分桶
4. Lua 脚本统一预加载、EvalSha + NOSCRIPT 自动回退
5. Hook 打了 cmd duration / error / 慢命令日志 / OTel tracing
6. 包了一层熔断（gobreaker）+ fallback 回源 DB
7. 热 key 用滑窗统计 + 本地短 TTL 镜像
8. 多级缓存组件：本地 + Redis + singleflight 合并回源
9. 反模式通过代码检查 + Review 拦截（KEYS / 无 TTL / 大 key）

本质是：Redis 是性能优化层，业务不能依赖它存在。
```

---

## 十六、面试加分点

- **PoolSize 按 `QPS × RT × 冗余` 算** 不是拍脑袋
- **超时分三层**（Dial / Read / Context），并能说出 PoolTimeout > ReadTimeout 的原因
- **ConnMaxLifetime** 防 NAT 静默断
- **Context 超时 ≠ Redis 端取消**（Redis 照样跑完）
- **Pipeline vs MGet 取舍**：纯读用 MGet，混合用 Pipeline
- **Lua 脚本用 NewScript + Run**（自动 EvalSha 回退）
- **Cluster 跨 slot 批量** 按 slot 分桶 + 并发
- **Hook 只做采样** 不做阻塞
- **熔断区分 redis.Nil 和真故障**
- **singleflight 防缓存击穿**
- **热 key 自动识别 + 本地短 TTL**
- **大 key 拒绝 HGETALL / SMEMBERS**，改 SCAN 系列
- **Set 必须带 TTL**
- **全局单例 Client** 而不是每次新建

---

## 十七、关联阅读

```
本目录:
- 05-cache-patterns.md     缓存模式
- 06-distributed-lock.md   分布式锁（配合 Lua）
- 07-pitfalls-tuning.md    大 key / 热 key 治理
- 09-production-cases.md   生产事故案例
- 10-cache-consistency-design.md 一致性方案
- 11-multi-tier-cache.md   多级缓存（本地缓存选型对比）

Go 相关:
- 01-go/06-goroutine-concurrency.md  并发原语
- 01-go/... singleflight / errgroup

跨模块:
- 06-distributed/04-lock.md      三方锁对比
- 06-distributed/06-rate-limit-circuit.md 限流熔断
- 13-engineering/04-observability-integration.md 可观测
```
