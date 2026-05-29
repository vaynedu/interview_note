# 限流器(Rate Limiter)

> **题目**:实现一个限流器,允许每秒最多 N 个请求,超过的拒绝或等待。要求线程安全,支持突发流量。
>
> 考查:**4 种主流算法 + 时间精度 + atomic 无锁优化 + 分布式扩展**。

这是手写题里**算法分支最多**的一道,大厂常用"先讲算法对比,再选一种手写"的考法。

---

## 一、考点拆解

| 考点 | 不及格 | 合格 | 优秀 |
| --- | --- | --- | --- |
| 算法选型 | 只会计数器 | 知道 4 种 | **能讲清每种取舍** |
| 时间精度 | 秒级(int64 秒)| 毫秒(time.Now().UnixMilli)| 纳秒(time.Now().UnixNano)|
| 突发流量 | 严格平均(漏桶) | **允许突发**(令牌桶) | 配置突发上限 |
| 并发 | `sync.Mutex` 全锁 | 关键路径 atomic | **CAS 无锁** |
| 边界 | 时钟回拨没处理 | 处理回拨 | NTP 同步 + 单调时钟 |
| 分布式 | 单机 | Redis + Lua | **令牌桶 Lua 脚本(原子)** |

---

## 二、4 种算法对比(必背)

| 算法 | 原理 | 突发 | 精度 | 内存 | 经典场景 |
| --- | --- | --- | --- | --- | --- |
| **固定窗口计数器** | 每秒重置计数 | ❌ **临界双倍突发** | 秒级 | O(1) | 简单场景 |
| **滑动窗口** | 环形数组分桶,加权计算 | 接近真实 | 毫秒级 | O(桶数)| Sentinel / 网关 |
| **漏桶 Leaky Bucket** | 队列 + 恒定速率出 | ❌ 不允许 | 毫秒级 | O(队列)| **流量整形**(出口)|
| **令牌桶 Token Bucket** | 定速放令牌,请求消耗 | ✅ **桶上限** | 毫秒级 | O(1) | **入口限流**(主流)|

### 2.1 固定窗口的"双倍突发"问题

```text
QPS 限 100:
   [0.0s ~ 1.0s] 第 0.9s 涌入 100 个 → 通过
   [1.0s ~ 2.0s] 第 1.0s 涌入 100 个 → 通过(新窗口)
   实际 0.9s ~ 1.0s 100ms 内通过了 200 个 = QPS 2000!
```

这是固定窗口最大的坑,**滑动窗口和令牌桶都没有这个问题**。

### 2.2 漏桶 vs 令牌桶(高频追问)

```text
漏桶:      _____ ─→  请求队列     ─→  恒定出水(均匀)
令牌桶:    _____ ←─  定时加令牌(可累积突发)
                                  ─→ 请求拿令牌通过
```

| | 漏桶 | 令牌桶 |
| --- | --- | --- |
| 出口速率 | **恒定** | 平均恒定,**瞬时可突发** |
| 突发吸收 | ❌(超额排队/丢) | ✅(桶里有存量) |
| 适合 | **出口/下游保护**(下游处理能力固定) | **入口限流**(允许短时尖峰)|
| 实现 | 队列 + 定时器 | 计数 + 时间戳 |
| 业界主流 | Nginx limit_req | Guava RateLimiter / Go x/time/rate |

> **资深答**:"令牌桶允许突发,适合保护**入口**;漏桶强制平均,适合保护**下游**。两个不是替代关系。"

---

## 三、令牌桶(主推手写实现)

### 3.1 思路

```text
桶容量 cap,定速 r tokens/sec
每次请求来:
  1. 算"距上次取令牌过了多久"
  2. 按速率补充令牌(不超过 cap)
  3. 如果剩余令牌 >= 1,允许 + 消耗 1
  4. 否则拒绝(或等待)
```

**关键 trick**:不需要定时器后台加令牌,**懒加载**——每次请求来时按时间差**惰性算**当前应有多少令牌。

### 3.2 完整实现(单机,毫秒精度)

```go
package limiter

import (
    "sync"
    "time"
)

type TokenBucket struct {
    capacity   float64   // 桶容量(允许的最大突发)
    rate       float64   // 每秒生成 token 数
    tokens     float64   // 当前 token 数
    lastRefill time.Time // 上次补充时间
    mu         sync.Mutex
}

func NewTokenBucket(rate, capacity float64) *TokenBucket {
    return &TokenBucket{
        capacity:   capacity,
        rate:       rate,
        tokens:     capacity, // 初始装满
        lastRefill: time.Now(),
    }
}

// Allow 非阻塞,够 1 个令牌就放过
func (tb *TokenBucket) Allow() bool {
    return tb.AllowN(1)
}

func (tb *TokenBucket) AllowN(n float64) bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()

    tb.refill()
    if tb.tokens >= n {
        tb.tokens -= n
        return true
    }
    return false
}

// refill 惰性补充令牌
func (tb *TokenBucket) refill() {
    now := time.Now()
    elapsed := now.Sub(tb.lastRefill).Seconds()
    tb.tokens += elapsed * tb.rate
    if tb.tokens > tb.capacity {
        tb.tokens = tb.capacity
    }
    tb.lastRefill = now
}

// Wait 阻塞版本:等到有令牌为止
func (tb *TokenBucket) Wait(n float64) time.Duration {
    tb.mu.Lock()
    tb.refill()
    if tb.tokens >= n {
        tb.tokens -= n
        tb.mu.Unlock()
        return 0
    }
    // 计算还需多久能凑够
    need := n - tb.tokens
    wait := time.Duration(need / tb.rate * float64(time.Second))
    tb.tokens -= n // 预扣(可能为负,下次 refill 会补)
    tb.lastRefill = time.Now()
    tb.mu.Unlock()

    time.Sleep(wait)
    return wait
}
```

**讲解时主动点出的细节**:

| 设计 | 资深点 |
| --- | --- |
| `tokens float64` | **不要用 int**,定速 r=3 时每 333ms 加一个,int 会丢精度 |
| 惰性 refill | 省了一个定时器 goroutine,代价是有锁,适合中低 QPS |
| `Wait` 里"预扣 tokens" | 防止多个 Wait 同时被骗到"够用"——**先占位再 Sleep** |
| `time.Now()` | 用单调时钟(Go 的 `time.Now` 自带单调部分,Sub 自动忽略回拨) |

### 3.3 测试

```go
func main() {
    tb := NewTokenBucket(10, 10) // 10 QPS,桶 10

    for i := 0; i < 20; i++ {
        ok := tb.Allow()
        fmt.Printf("[%v] req %d → %v\n", time.Now().Format("15:04:05.000"), i, ok)
        time.Sleep(50 * time.Millisecond)
    }
}
// 现象:前 10 个秒杀通过(突发),之后每 100ms 才能再过一个
```

---

## 四、滑动窗口(辅推实现)

### 4.1 思路:把 1 秒切成 N 个小桶,每桶记数

```text
QPS 限 100,切成 10 个 100ms 小桶:
[10][12][9][8][11][15][9][11][8][7]   ← 当前秒
                    ^ 现在
求和 = 100 → 拒绝下一个
```

每个新请求:**滚动丢弃过期的桶 + 累加当前窗口**。

### 4.2 完整实现

```go
type SlidingWindow struct {
    capacity int           // 窗口内允许的总数
    window   time.Duration // 窗口大小,如 1s
    buckets  int           // 切多少格
    counts   []int64       // 每格计数
    times    []int64       // 每格的纳秒时间戳
    mu       sync.Mutex
}

func NewSlidingWindow(capacity int, window time.Duration, buckets int) *SlidingWindow {
    return &SlidingWindow{
        capacity: capacity,
        window:   window,
        buckets:  buckets,
        counts:   make([]int64, buckets),
        times:    make([]int64, buckets),
    }
}

func (s *SlidingWindow) Allow() bool {
    s.mu.Lock()
    defer s.mu.Unlock()

    now := time.Now().UnixNano()
    bucketNs := int64(s.window) / int64(s.buckets)
    idx := int((now / bucketNs) % int64(s.buckets))

    // 当前桶过期(老数据)→ 清零
    if now-s.times[idx] >= int64(s.window) {
        s.counts[idx] = 0
        s.times[idx] = now
    }

    // 统计窗口内总数(剔除老桶)
    var total int64
    cutoff := now - int64(s.window)
    for i := 0; i < s.buckets; i++ {
        if s.times[i] >= cutoff {
            total += s.counts[i]
        }
    }

    if total >= int64(s.capacity) {
        return false
    }
    s.counts[idx]++
    s.times[idx] = now
    return true
}
```

**资深点**:
- 桶数越多越精确,但 CPU 越高(每次 Allow 要扫所有桶)
- 实战桶数取 **10~60**(Sentinel 用 60)
- 比令牌桶**多一倍内存**,但**对突发更敏感**(令牌桶可以瞬间放掉 cap 个)

---

## 五、漏桶(知道即可)

```go
type LeakyBucket struct {
    capacity int           // 桶容量
    rate     time.Duration // 漏速,每 rate 漏一个
    water    int
    lastLeak time.Time
    mu       sync.Mutex
}

func (lb *LeakyBucket) Allow() bool {
    lb.mu.Lock()
    defer lb.mu.Unlock()

    // 先漏水
    now := time.Now()
    leaked := int(now.Sub(lb.lastLeak) / lb.rate)
    if leaked > 0 {
        lb.water -= leaked
        if lb.water < 0 {
            lb.water = 0
        }
        lb.lastLeak = now
    }

    if lb.water < lb.capacity {
        lb.water++
        return true
    }
    return false
}
```

> 漏桶和令牌桶的代码几乎对称,**令牌桶是"加",漏桶是"减"**。实际业务里漏桶用得少,优先掌握令牌桶。

---

## 六、进阶变体

### 6.1 atomic 无锁令牌桶(高 QPS 优化)

```go
type LockFreeTokenBucket struct {
    rate       float64
    capacity   float64
    state      atomic.Uint64 // 高 32 位:lastRefill ms,低 32 位:tokens × 1000
}

func (tb *LockFreeTokenBucket) Allow() bool {
    for {
        old := tb.state.Load()
        // 解包 lastRefill / tokens
        last := int64(old >> 32)
        tokens := float64(uint32(old)) / 1000.0

        now := time.Now().UnixMilli()
        tokens += float64(now-last) / 1000.0 * tb.rate
        if tokens > tb.capacity {
            tokens = tb.capacity
        }
        if tokens < 1 {
            return false
        }
        tokens -= 1

        newState := (uint64(now) << 32) | uint64(tokens*1000)
        if tb.state.CompareAndSwap(old, newState) {
            return true
        }
        // CAS 失败,重试
    }
}
```

资深点:
- CAS 失败会重试,**极高 QPS 下 CAS 风暴**反而比 Mutex 还慢
- 用 atomic 的前提是**单变量能塞下所有状态**(这里用 uint64 拼了两个字段)

### 6.2 分布式限流(Redis + Lua)

单机限流扛不住分布式部署,得在 Redis 里做。

```lua
-- 令牌桶 Lua 脚本(原子)
-- KEYS[1] = 桶 key
-- ARGV[1] = rate, ARGV[2] = capacity, ARGV[3] = now_ms, ARGV[4] = n
local key = KEYS[1]
local rate = tonumber(ARGV[1])
local cap = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local need = tonumber(ARGV[4])

local data = redis.call("HMGET", key, "tokens", "last")
local tokens = tonumber(data[1]) or cap
local last = tonumber(data[2]) or now

-- 补充令牌
tokens = tokens + (now - last) / 1000 * rate
if tokens > cap then tokens = cap end

local ok = 0
if tokens >= need then
    tokens = tokens - need
    ok = 1
end

redis.call("HMSET", key, "tokens", tokens, "last", now)
redis.call("PEXPIRE", key, 60000)
return ok
```

资深点:
- **Lua 脚本保证原子性**,否则"读-改-写"中间会被插入别的请求
- 时间用**客户端传 now**,避免 Redis 节点间时钟不同步
- 过期时间设大点(60s),冷 key 自动回收
- 大规模流量下用 **Redis Cluster + 按业务维度切 key**(`limit:userId:1234`)避免单 slot 热点

### 6.3 多维度组合限流

实战不是"一个限流器走天下",而是**多层 + 多维**:

```text
全局限流(整个服务 100w QPS)
   └─ 用户维度(每用户 100 QPS)
       └─ 接口维度(每接口 + 每用户 30 QPS)
           └─ 资源维度(单条数据 + 用户 10 QPS)
```

任一层超限就拒绝。这是网关 / WAF 的标准做法。

---

## 七、典型坑

| 坑 | 现象 | 修复 |
| --- | --- | --- |
| 用 int 算 tokens | 低速率丢精度 | 改 float64 |
| 时钟回拨 | tokens 暴增或暴减 | 用 `time.Now()` 单调时钟,或夹 `if elapsed < 0 { elapsed = 0 }` |
| 固定窗口边界双倍突发 | 实际 QPS 翻倍 | 换滑动窗口或令牌桶 |
| Wait 不预扣 tokens | 多个 Wait 都被骗"够用",最后实际超限 | Sleep 前先把 tokens 扣掉 |
| Redis 限流没 Lua | "读-改-写"不原子,并发超限 | EVAL 脚本 |
| 全局共享一个桶 | 单点热点 | 按维度拆 key |
| 分布式限流忽略 Redis 故障 | Redis 挂了限流全失效或全失败 | 降级到本地令牌桶兜底 |
| 突发上限设太小 | 正常流量被误杀(JS 客户端瞬间打多个请求)| 桶 cap 留 1.5~2× QPS 余量 |

---

## 八、和 `golang.org/x/time/rate` 对比

标准库 `rate.Limiter` 就是**令牌桶**。手写题里 95% 面试官会接着问"那 x/time/rate 怎么实现的":

| 维度 | 标准库 | 本文实现 |
| --- | --- | --- |
| 算法 | 令牌桶 | 令牌桶 |
| 时间精度 | 纳秒 | 毫秒 |
| 并发优化 | Mutex + 惰性算 | 同 |
| 额外 API | `Reserve`(预约不立即扣)/ `SetLimit`(动态改)| 自己加 |
| 突发参数 | `burst` | `capacity` |

资深点:**手写实现要主动说"和标准库思路一样,我没造轮子重复发明"**,加分。

---

## 九、现场表达模板

> "限流主流有 4 种:固定窗口计数、滑动窗口、漏桶、令牌桶。
> **固定窗口**最简单但有临界突发问题——0.9s 和 1.0s 各放 100 个,实际 100ms 内过了 200 个。
> **滑动窗口**把窗口切多个小桶解决了边界问题,精度更高但内存大,Sentinel 用的就是这个。
> **漏桶**强制恒定出口,适合保护下游;**令牌桶**允许突发,适合保护入口,**Guava / Go x/time/rate 用的都是令牌桶**。
>
> 我手写一个令牌桶:核心是**懒加载补令牌**——不开后台 goroutine 定时加,而是每次请求来按 `(now - lastRefill) × rate` 算应有多少。
> 关键 trick 是 tokens 必须用 float64,低速率才不丢精度;
> 时钟回拨用 Go 的单调时钟自动忽略。
>
> 分布式场景换成 Redis + Lua 脚本,保证'读-改-写'原子。
> 实际业务往往是多维组合限流——全局 + 用户 + 接口 + 资源,任一超限就拒。"

---

## 十、一句话总结

> **限流 4 算法**:固定窗口(简单但有边界突发)、滑动窗口(精度高内存大)、漏桶(恒定出口保下游)、**令牌桶(允许突发保入口,主流)**。
>
> **令牌桶手写要点**:`tokens float64` + 懒加载 refill + Wait 预扣 tokens + 单调时钟防回拨。
>
> 分布式上 Redis,Lua 保原子。多维组合限流才是真实工程做法。
