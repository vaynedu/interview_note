# Channel 模式与场景 Cookbook

> 可复制粘贴的 channel 设计模式与真实业务场景代码，强化理解与使用
>
> 配合 [channel.md](channel.md)（原理）+ [select.md](select.md)（多路复用）+ [context.md](context.md)（取消传播）

## 目录

### 设计模式
1. [Pipeline 流水线](#1-pipeline-流水线)
2. [Fan-out / Fan-in 扇出扇入](#2-fan-out--fan-in-扇出扇入)
3. [Worker Pool 工作池](#3-worker-pool-工作池)
4. [Semaphore 信号量（并发限制）](#4-semaphore-信号量)
5. [Rate Limiter 限流器](#5-rate-limiter-限流器)
6. [Heartbeat 心跳](#6-heartbeat-心跳)
7. [Broadcast 广播（一发多收）](#7-broadcast-广播)
8. [Pub/Sub 发布订阅](#8-pubsub-发布订阅)
9. [Future / Promise](#9-future--promise)
10. [Timeout 超时控制](#10-timeout-超时控制)
11. [Cancellation 取消传播](#11-cancellation-取消传播)
12. [Graceful Shutdown 优雅退出](#12-graceful-shutdown-优雅退出)
13. [Request Coalescing / Single Flight 请求合并](#13-request-coalescing)
14. [Priority Queue 优先级队列（嵌套 select）](#14-priority-queue)
15. [Bounded Parallelism 有界并发](#15-bounded-parallelism)

### 业务场景
16. [连接池](#16-连接池)
17. [爬虫调度器](#16-爬虫调度器)
18. [异步任务队列](#18-异步任务队列)
19. [实时日志聚合](#19-实时日志聚合)
20. [批量消费 + 定时 flush](#20-批量消费--定时-flush)

---

## 1. Pipeline 流水线

**场景**：数据处理链路（生产 → 变换 → 过滤 → 聚合），每阶段一个 goroutine，用 channel 串起。

```go
// 阶段 1：生产
func gen(ctx context.Context, nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            select {
            case <-ctx.Done():
                return
            case out <- n:
            }
        }
    }()
    return out
}

// 阶段 2：变换
func square(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case <-ctx.Done():
                return
            case out <- n * n:
            }
        }
    }()
    return out
}

// 阶段 3：过滤
func filter(ctx context.Context, in <-chan int, pred func(int) bool) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            if !pred(n) { continue }
            select {
            case <-ctx.Done():
                return
            case out <- n:
            }
        }
    }()
    return out
}

// 使用
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    nums := gen(ctx, 1, 2, 3, 4, 5)
    sqs  := square(ctx, nums)
    odds := filter(ctx, sqs, func(n int) bool { return n%2 == 1 })

    for v := range odds {
        fmt.Println(v)  // 1, 9, 25
    }
}
```

**要点**：
- 每阶段 defer close（生产端关闭）
- ctx 贯穿，任一阶段退出则上游停止
- 每阶段 goroutine 数 1（单线程处理本阶段）

---

## 2. Fan-out / Fan-in 扇出扇入

**场景**：单任务源 → 多 worker 并行处理 → 合并结果。

```go
// Fan-out：同一个 in 分发到多个 worker
func fanOut(ctx context.Context, in <-chan int, n int, work func(int) int) []<-chan int {
    outs := make([]<-chan int, n)
    for i := 0; i < n; i++ {
        ch := make(chan int)
        outs[i] = ch
        go func() {
            defer close(ch)
            for v := range in {
                select {
                case <-ctx.Done():
                    return
                case ch <- work(v):
                }
            }
        }()
    }
    return outs
}

// Fan-in：多个 chan 合并到一个
func fanIn[T any](ctx context.Context, chans ...<-chan T) <-chan T {
    out := make(chan T)
    var wg sync.WaitGroup
    wg.Add(len(chans))
    for _, c := range chans {
        go func(c <-chan T) {
            defer wg.Done()
            for v := range c {
                select {
                case <-ctx.Done():
                    return
                case out <- v:
                }
            }
        }(c)
    }
    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}

// 使用
func main() {
    ctx := context.Background()
    nums := gen(ctx, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

    // 4 个 worker 并行计算平方
    outs := fanOut(ctx, nums, 4, func(n int) int { return n * n })

    // 合并结果
    merged := fanIn(ctx, outs...)
    for v := range merged {
        fmt.Println(v)
    }
}
```

**要点**：
- fanOut 用多 goroutine 从同一个 in 读（goroutine 抢）
- fanIn 用 len(chans) 个 goroutine 把数据搬到 out
- WaitGroup 协调 close 时机

**性能**：
- worker 数 ≈ CPU 核数（CPU 密集）或更多（IO 密集）
- 超过饱和点再加 worker 无用

---

## 3. Worker Pool 工作池

**场景**：固定 worker 数处理任务队列，避免每任务启一个 goroutine。

```go
type Job struct {
    ID   int
    Data string
}

type Result struct {
    JobID int
    Value string
    Err   error
}

func workerPool(ctx context.Context, nWorkers int, jobs <-chan Job) <-chan Result {
    results := make(chan Result, nWorkers)
    var wg sync.WaitGroup
    wg.Add(nWorkers)

    for i := 0; i < nWorkers; i++ {
        go func(workerID int) {
            defer wg.Done()
            for job := range jobs {
                select {
                case <-ctx.Done():
                    return
                default:
                }
                res := process(job)
                select {
                case <-ctx.Done():
                    return
                case results <- res:
                }
            }
        }(i)
    }

    go func() {
        wg.Wait()
        close(results)
    }()
    return results
}

func process(j Job) Result {
    time.Sleep(100 * time.Millisecond)
    return Result{JobID: j.ID, Value: "done: " + j.Data}
}

// 使用
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    jobs := make(chan Job, 100)
    results := workerPool(ctx, 10, jobs)

    // 投递任务
    go func() {
        defer close(jobs)
        for i := 0; i < 50; i++ {
            jobs <- Job{ID: i, Data: fmt.Sprintf("task-%d", i)}
        }
    }()

    // 收结果
    for r := range results {
        fmt.Printf("job %d: %s\n", r.JobID, r.Value)
    }
}
```

**vs 每任务一 goroutine**：
- Worker Pool：固定资源，适合**长期高 QPS**
- Per-goroutine：无限制，适合**突发短任务**（Go goroutine 很轻）

**典型场景**：HTTP 下载器 / 图片处理 / RPC 调用 / 爬虫

---

## 4. Semaphore 信号量

**场景**：限制并发数，比 Worker Pool 更轻量。

```go
type Semaphore chan struct{}

func NewSemaphore(n int) Semaphore {
    return make(chan struct{}, n)
}

func (s Semaphore) Acquire() { s <- struct{}{} }
func (s Semaphore) Release() { <-s }

// 带超时
func (s Semaphore) AcquireWithTimeout(d time.Duration) bool {
    select {
    case s <- struct{}{}:
        return true
    case <-time.After(d):
        return false
    }
}

// 带 context
func (s Semaphore) AcquireCtx(ctx context.Context) error {
    select {
    case s <- struct{}{}:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

// 使用：限制并发 RPC 调用
func batchRPC(ctx context.Context, reqs []Request) []Response {
    sem := NewSemaphore(10)  // 最多 10 并发
    var wg sync.WaitGroup
    results := make([]Response, len(reqs))

    for i, req := range reqs {
        wg.Add(1)
        go func(idx int, r Request) {
            defer wg.Done()
            sem.Acquire()
            defer sem.Release()
            results[idx] = doRPC(ctx, r)
        }(i, req)
    }
    wg.Wait()
    return results
}
```

**vs Worker Pool**：
- Semaphore 更轻量（不常驻 goroutine）
- 适合一次性批量任务
- 每任务启一个 goroutine + sem 控并发数

**扩展**：`golang.org/x/sync/semaphore` 官方加权信号量。

---

## 5. Rate Limiter 限流器

**场景**：控制 QPS（每秒最多 N 次）。

### 5.1 简单版（time.Tick）

```go
func rateLimiter(ctx context.Context, rate time.Duration, work func()) {
    ticker := time.NewTicker(rate)
    defer ticker.Stop()
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            work()
        }
    }
}

// 5 QPS
rateLimiter(ctx, 200*time.Millisecond, doRequest)
```

### 5.2 令牌桶（带突发）

```go
type TokenBucket struct {
    tokens chan struct{}
    rate   time.Duration
}

func NewTokenBucket(capacity int, rate time.Duration) *TokenBucket {
    tb := &TokenBucket{
        tokens: make(chan struct{}, capacity),
        rate:   rate,
    }
    // 预填满
    for i := 0; i < capacity; i++ {
        tb.tokens <- struct{}{}
    }
    // 定时补充
    go func() {
        ticker := time.NewTicker(rate)
        defer ticker.Stop()
        for range ticker.C {
            select {
            case tb.tokens <- struct{}{}:
            default:  // 桶满，丢弃
            }
        }
    }()
    return tb
}

func (tb *TokenBucket) Allow() bool {
    select {
    case <-tb.tokens:
        return true
    default:
        return false
    }
}

func (tb *TokenBucket) Wait(ctx context.Context) error {
    select {
    case <-tb.tokens:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

**用法**：
- `Allow()` 非阻塞（限流拒绝）
- `Wait()` 阻塞（排队等）

**官方实现**：`golang.org/x/time/rate.Limiter` 性能更好，生产优先用。

---

## 6. Heartbeat 心跳

**场景**：定期发送"我还活着"信号，配合超时判活。

```go
func heartbeat(ctx context.Context, interval time.Duration) <-chan time.Time {
    beat := make(chan time.Time, 1)
    go func() {
        defer close(beat)
        ticker := time.NewTicker(interval)
        defer ticker.Stop()
        for {
            select {
            case <-ctx.Done():
                return
            case t := <-ticker.C:
                select {
                case beat <- t:  // 非阻塞发送（接收方慢也不卡）
                default:
                }
            }
        }
    }()
    return beat
}

// 监听方
func watch(ctx context.Context, beat <-chan time.Time, timeout time.Duration) {
    t := time.NewTimer(timeout)
    defer t.Stop()
    for {
        select {
        case <-ctx.Done():
            return
        case <-beat:
            if !t.Stop() { <-t.C }
            t.Reset(timeout)
        case <-t.C:
            log.Error("worker dead!")
            return
        }
    }
}
```

**典型应用**：
- Service Mesh Sidecar 健康检查
- 长连接心跳
- 流处理进度上报

---

## 7. Broadcast 广播

**场景**：一个事件通知所有监听者（关闭信号是典型代表）。

### 7.1 一次性广播：close(chan)

```go
type Broadcaster struct {
    done chan struct{}
    once sync.Once
}

func New() *Broadcaster {
    return &Broadcaster{done: make(chan struct{})}
}

// 触发广播（可重复调用，只生效一次）
func (b *Broadcaster) Signal() {
    b.once.Do(func() { close(b.done) })
}

// 监听
func (b *Broadcaster) Done() <-chan struct{} {
    return b.done
}

// 使用
b := New()
for i := 0; i < 100; i++ {
    go func() {
        select {
        case <-b.Done():  // 所有 goroutine 同时收到
            return
        case job := <-jobs:
            process(job)
        }
    }()
}
b.Signal()  // 广播，所有 goroutine 退出
```

**核心**：close 后所有 `<-done` 都立即返回零值。

### 7.2 反复广播：需要 Pub/Sub（见 #8）

---

## 8. Pub/Sub 发布订阅

**场景**：多订阅者，每个订阅者独立接收所有消息。

```go
type PubSub[T any] struct {
    mu      sync.RWMutex
    subs    map[int]chan T
    nextID  int
    closed  bool
}

func NewPubSub[T any]() *PubSub[T] {
    return &PubSub[T]{subs: make(map[int]chan T)}
}

func (p *PubSub[T]) Subscribe(buf int) (int, <-chan T) {
    p.mu.Lock()
    defer p.mu.Unlock()
    if p.closed {
        ch := make(chan T)
        close(ch)
        return 0, ch
    }
    id := p.nextID
    p.nextID++
    ch := make(chan T, buf)
    p.subs[id] = ch
    return id, ch
}

func (p *PubSub[T]) Unsubscribe(id int) {
    p.mu.Lock()
    defer p.mu.Unlock()
    if ch, ok := p.subs[id]; ok {
        close(ch)
        delete(p.subs, id)
    }
}

func (p *PubSub[T]) Publish(msg T) {
    p.mu.RLock()
    defer p.mu.RUnlock()
    if p.closed { return }
    for _, ch := range p.subs {
        select {
        case ch <- msg:
        default:  // 慢订阅者丢消息（可改为 block / 断开）
        }
    }
}

func (p *PubSub[T]) Close() {
    p.mu.Lock()
    defer p.mu.Unlock()
    if p.closed { return }
    p.closed = true
    for id, ch := range p.subs {
        close(ch)
        delete(p.subs, id)
    }
}

// 使用
ps := NewPubSub[string]()
id1, ch1 := ps.Subscribe(10)
id2, ch2 := ps.Subscribe(10)

go func() { for v := range ch1 { fmt.Println("sub1:", v) } }()
go func() { for v := range ch2 { fmt.Println("sub2:", v) } }()

ps.Publish("hello")  // 两个订阅者都收到
ps.Publish("world")

ps.Unsubscribe(id1)
ps.Close()
```

**策略**：
- 慢订阅者策略：丢消息 / 阻塞 / 断开（看业务）
- 生产级别看 nats.io、Redis Pub/Sub

---

## 9. Future / Promise

**场景**：异步获取结果，未来某时刻取出（类似 Java Future）。

```go
type Future[T any] struct {
    result chan T
    err    chan error
}

func Go[T any](fn func() (T, error)) *Future[T] {
    f := &Future[T]{
        result: make(chan T, 1),
        err:    make(chan error, 1),
    }
    go func() {
        v, err := fn()
        if err != nil {
            f.err <- err
        } else {
            f.result <- v
        }
    }()
    return f
}

func (f *Future[T]) Get(ctx context.Context) (T, error) {
    var zero T
    select {
    case v := <-f.result:
        return v, nil
    case err := <-f.err:
        return zero, err
    case <-ctx.Done():
        return zero, ctx.Err()
    }
}

// 使用
ctx := context.Background()
f := Go(func() (string, error) {
    time.Sleep(time.Second)
    return "hello", nil
})

// ... 做其他事
v, err := f.Get(ctx)
```

**典型场景**：并行 RPC 调用 / 异步加载。

---

## 10. Timeout 超时控制

```go
// 基础版
func doWithTimeout[T any](fn func() T, d time.Duration) (T, error) {
    var zero T
    ch := make(chan T, 1)
    go func() { ch <- fn() }()
    select {
    case v := <-ch:
        return v, nil
    case <-time.After(d):
        return zero, context.DeadlineExceeded
    }
}

// ⚠️ goroutine 可能泄漏（fn 还在跑）
// 如果 fn 支持 ctx，改用下面的版本：

func doWithCtx[T any](ctx context.Context, fn func(context.Context) (T, error)) (T, error) {
    // 推荐让 fn 自己接受 ctx，内部监听 cancel
    return fn(ctx)
}

// 使用
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()
result, err := doWithCtx(ctx, fetchData)
```

**陷阱**：`time.After` + goroutine 泄漏（详见 [select.md 坑 1](select.md)）。长循环里用 `time.NewTimer + Reset`。

---

## 11. Cancellation 取消传播

**场景**：父任务取消时所有子任务一起取消（context 标准用法的本质）。

```go
// 自己实现（原理演示）
type CancelChain struct {
    done chan struct{}
    once sync.Once
}

func (c *CancelChain) Cancel() {
    c.once.Do(func() { close(c.done) })
}

func (c *CancelChain) Done() <-chan struct{} { return c.done }

// 子任务继承
func WithParent(parent *CancelChain) *CancelChain {
    child := &CancelChain{done: make(chan struct{})}
    go func() {
        select {
        case <-parent.Done():
            child.Cancel()
        case <-child.Done():
        }
    }()
    return child
}
```

**实战直接用 context.Context**：`context.WithCancel` / `context.WithTimeout` / `context.WithDeadline` 已经把这套做好。

---

## 12. Graceful Shutdown 优雅退出

**场景**：HTTP 服务收到 SIGTERM 后不立刻退出，等在途请求处理完。

```go
func main() {
    srv := &http.Server{Addr: ":8080", Handler: handler}

    // 启动服务
    go func() {
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    // 等信号
    sig := make(chan os.Signal, 1)
    signal.Notify(sig, syscall.SIGTERM, syscall.SIGINT)
    <-sig

    // 优雅关闭
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    log.Println("shutting down...")

    // 1. 反注册（如有注册中心）
    registry.Deregister()

    // 2. sleep 等客户端缓存过期
    time.Sleep(5 * time.Second)

    // 3. 停止接受新请求 + 等在途请求
    if err := srv.Shutdown(ctx); err != nil {
        log.Error("shutdown err:", err)
    }

    // 4. flush trace / metrics
    tp.Shutdown(ctx)
    logger.Sync()

    // 5. 关资源
    db.Close()
    redis.Close()

    log.Println("bye")
}
```

详见 [07-microservice/02-registry-discovery.md](../../07-microservice/02-registry-discovery.md) 第 6 节 优雅下线。

---

## 13. Request Coalescing

**场景**：多个并发请求查同一 key，合并为 1 次后端调用（防缓存击穿）。

```go
// 手写版
type Group struct {
    mu sync.Mutex
    m  map[string]*call
}

type call struct {
    wg  sync.WaitGroup
    val interface{}
    err error
}

func (g *Group) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
    g.mu.Lock()
    if g.m == nil { g.m = make(map[string]*call) }
    if c, ok := g.m[key]; ok {
        g.mu.Unlock()
        c.wg.Wait()  // 等待进行中的调用
        return c.val, c.err
    }
    c := &call{}
    c.wg.Add(1)
    g.m[key] = c
    g.mu.Unlock()

    c.val, c.err = fn()
    c.wg.Done()

    g.mu.Lock()
    delete(g.m, key)
    g.mu.Unlock()

    return c.val, c.err
}

// 使用
var g Group
val, err := g.Do("user:123", func() (interface{}, error) {
    return db.Query("user:123")
})
```

**官方**：`golang.org/x/sync/singleflight.Group` 直接用，别自己写。

**典型**：防缓存击穿（热点 key 过期时，1000 个并发查 DB → 合并成 1 次）。

---

## 14. Priority Queue

**场景**：多路 channel 有优先级差异（如高优任务先处理）。

```go
// select 自身不支持优先级（伪随机），需嵌套 select
func worker(ctx context.Context, high, low <-chan Job) {
    for {
        // 第一层：优先 high
        select {
        case <-ctx.Done():
            return
        case j := <-high:
            processHigh(j)
            continue
        default:
        }

        // 第二层：high 没就绪才看 low
        select {
        case <-ctx.Done():
            return
        case j := <-high:
            processHigh(j)
        case j := <-low:
            processLow(j)
        }
    }
}
```

**坑**：如果 high 持续高速到来，**low 永远不被处理**（饥饿）。

**修复**：
```go
// 每处理 N 个 high 强制让出一次给 low
const yieldEvery = 10
count := 0
for {
    select {
    case j := <-high:
        if count++; count >= yieldEvery {
            // 强制处理一个 low
            select {
            case jj := <-low: processLow(jj)
            default:
            }
            count = 0
        }
        processHigh(j)
    case j := <-low:
        processLow(j)
    case <-ctx.Done(): return
    }
}
```

---

## 15. Bounded Parallelism

**场景**：处理 URL 列表，限制并发为 N，收集所有结果。

```go
func fetchAll(ctx context.Context, urls []string, n int) []Result {
    results := make([]Result, len(urls))
    sem := make(chan struct{}, n)
    var wg sync.WaitGroup

    for i, url := range urls {
        wg.Add(1)
        sem <- struct{}{}  // 阻塞获取令牌
        go func(idx int, u string) {
            defer wg.Done()
            defer func() { <-sem }()
            results[idx] = fetch(ctx, u)
        }(i, url)
    }
    wg.Wait()
    return results
}
```

**vs Worker Pool**：
- Bounded Parallelism：每任务一 goroutine + sem 限并发
- Worker Pool：固定 N 个 goroutine 从队列消费

**选型**：
- 任务数少（< 1 万）/ 每任务时间长 → Bounded Parallelism
- 任务数多 / 每任务短 → Worker Pool

---

## 业务场景

## 16. 连接池

**场景**：限制 DB / RPC 连接数，复用连接。

```go
type Pool struct {
    conns chan *Conn
    factory func() (*Conn, error)
}

func NewPool(size int, factory func() (*Conn, error)) *Pool {
    p := &Pool{
        conns: make(chan *Conn, size),
        factory: factory,
    }
    // 懒创建，也可预填
    return p
}

func (p *Pool) Get(ctx context.Context) (*Conn, error) {
    select {
    case c := <-p.conns:
        if c.IsAlive() { return c, nil }
        return p.factory()
    default:
        // 池空，新建（不超 size）
        select {
        case c := <-p.conns:
            return c, nil
        case <-ctx.Done():
            return nil, ctx.Err()
        default:
            return p.factory()
        }
    }
}

func (p *Pool) Put(c *Conn) {
    if !c.IsAlive() {
        c.Close()
        return
    }
    select {
    case p.conns <- c:  // 放回池
    default:
        c.Close()  // 池满，关闭
    }
}
```

**实战**：`database/sql` 内置连接池 / redis.Pool / grpc-go 内置。

---

## 17. 爬虫调度器

**场景**：爬虫限速 + 去重 + 并发抓取。

```go
type Crawler struct {
    visited  sync.Map
    limiter  <-chan time.Time
    results  chan string
    nWorkers int
}

func NewCrawler(qps int, nWorkers int) *Crawler {
    return &Crawler{
        limiter:  time.Tick(time.Second / time.Duration(qps)),
        results:  make(chan string, 100),
        nWorkers: nWorkers,
    }
}

func (c *Crawler) Run(ctx context.Context, seeds []string) <-chan string {
    urls := make(chan string, 1000)

    // 投 seed
    go func() {
        for _, u := range seeds { urls <- u }
    }()

    // N 个 worker
    var wg sync.WaitGroup
    for i := 0; i < c.nWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for url := range urls {
                if _, loaded := c.visited.LoadOrStore(url, true); loaded {
                    continue  // 已访问
                }
                select {
                case <-ctx.Done(): return
                case <-c.limiter:  // 限速
                }
                html, err := fetch(url)
                if err != nil { continue }

                // 解析新 URL 投入队列
                for _, next := range extractLinks(html) {
                    select {
                    case urls <- next:
                    default:  // 队列满，丢弃
                    }
                }

                select {
                case c.results <- url:
                case <-ctx.Done(): return
                }
            }
        }()
    }

    go func() {
        wg.Wait()
        close(c.results)
    }()
    return c.results
}
```

**整合**：限流（Rate Limiter）+ Worker Pool + 去重（sync.Map）。

---

## 18. 异步任务队列

**场景**：Web 请求异步处理（发邮件 / 生成报表）。

```go
type TaskQueue struct {
    tasks  chan Task
    workers int
    wg     sync.WaitGroup
    quit   chan struct{}
}

type Task func(ctx context.Context) error

func NewTaskQueue(bufferSize, workers int) *TaskQueue {
    q := &TaskQueue{
        tasks:   make(chan Task, bufferSize),
        workers: workers,
        quit:    make(chan struct{}),
    }
    q.start()
    return q
}

func (q *TaskQueue) start() {
    for i := 0; i < q.workers; i++ {
        q.wg.Add(1)
        go q.worker()
    }
}

func (q *TaskQueue) worker() {
    defer q.wg.Done()
    for task := range q.tasks {
        ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        if err := task(ctx); err != nil {
            log.Error(err)
        }
        cancel()
    }
}

// 投递任务
func (q *TaskQueue) Submit(t Task) error {
    select {
    case q.tasks <- t:
        return nil
    default:
        return errors.New("queue full")
    }
}

// 关闭
func (q *TaskQueue) Shutdown(ctx context.Context) error {
    close(q.tasks)  // 通知 worker 退出

    done := make(chan struct{})
    go func() { q.wg.Wait(); close(done) }()

    select {
    case <-done:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

// 使用
q := NewTaskQueue(1000, 10)
q.Submit(func(ctx context.Context) error {
    return sendEmail(ctx, "user@example.com")
})

// 关闭
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
q.Shutdown(ctx)
```

**生产级**：建议直接用 MQ（Kafka / RocketMQ），不自建。

---

## 19. 实时日志聚合

**场景**：多 goroutine 写日志，集中 flush 到文件 / ES。

```go
type LogAggregator struct {
    logs   chan string
    done   chan struct{}
}

func NewLogAggregator(bufSize int, flushInterval time.Duration, batchSize int) *LogAggregator {
    la := &LogAggregator{
        logs: make(chan string, bufSize),
        done: make(chan struct{}),
    }

    go func() {
        defer close(la.done)
        ticker := time.NewTicker(flushInterval)
        defer ticker.Stop()

        buffer := make([]string, 0, batchSize)
        flush := func() {
            if len(buffer) == 0 { return }
            // 批量写 ES / 文件
            writeBatch(buffer)
            buffer = buffer[:0]
        }

        for {
            select {
            case log, ok := <-la.logs:
                if !ok {
                    flush()
                    return
                }
                buffer = append(buffer, log)
                if len(buffer) >= batchSize {
                    flush()
                }
            case <-ticker.C:
                flush()
            }
        }
    }()
    return la
}

func (la *LogAggregator) Log(s string) {
    select {
    case la.logs <- s:
    default:  // 满了丢弃（低优日志）/ 或阻塞
    }
}

func (la *LogAggregator) Close() {
    close(la.logs)
    <-la.done  // 等 flush 完
}
```

**要点**：攒批（batchSize）+ 定时（flushInterval）双触发。

---

## 20. 批量消费 + 定时 flush

**场景**：Kafka 消费 / DB 批量写入 / Metrics 上报。

```go
type Batcher[T any] struct {
    in       chan T
    size     int
    maxWait  time.Duration
    flush    func([]T)
}

func (b *Batcher[T]) Run(ctx context.Context) {
    buf := make([]T, 0, b.size)
    ticker := time.NewTicker(b.maxWait)
    defer ticker.Stop()

    doFlush := func() {
        if len(buf) > 0 {
            b.flush(append([]T(nil), buf...))  // 拷贝防并发修改
            buf = buf[:0]
        }
    }

    for {
        select {
        case <-ctx.Done():
            doFlush()
            return
        case v := <-b.in:
            buf = append(buf, v)
            if len(buf) >= b.size {
                doFlush()
                ticker.Reset(b.maxWait)  // 重置定时器
            }
        case <-ticker.C:
            doFlush()
        }
    }
}
```

**典型参数**：
- size = 100-1000（取决于单项大小）
- maxWait = 100ms - 1s（延迟 vs 吞吐权衡）

**实战**：Metrics 上报 / Kafka Producer / DB 批量 INSERT。

---

## 速查选型表

| 场景 | 推荐模式 |
| --- | --- |
| 固定并发度 worker | Worker Pool（#3）|
| 一次性批量并发 | Semaphore（#4）/ Bounded Parallelism（#15）|
| 限速 | Rate Limiter（#5）/ 令牌桶 |
| 一次性广播（关闭信号） | close(chan)（#7）|
| 反复广播（事件订阅） | Pub/Sub（#8）|
| 异步结果 | Future（#9）|
| 取消传播 | context（#11）|
| 合并重复请求 | singleflight（#13）|
| 优先级 | 嵌套 select（#14）|
| 批量聚合 | Batcher（#20）|

---

## 性能 Tips

1. **buffer 合理**：
   - `make(chan T, 0)` 同步点，严格协调
   - `make(chan T, 1)` 信号量 / 解耦生产消费 1 个
   - `make(chan T, N)` 削峰，N ≈ 预期突发

2. **关闭协议**：
   - 1:1 / 1:N → 发送方关
   - N:1 / N:M → 不关 ch，用 stopCh + sync.Once

3. **nil chan 妙用**：select 动态屏蔽 case

4. **避免 time.After 泄漏**：循环里用 NewTimer + Reset

5. **极致性能**：百万 QPS 级别考虑自研无锁队列（Disruptor 风格）

6. **别用 channel 做同步**：atomic / Mutex 快 10-1000x

---

## 参考阅读

- [channel.md](channel.md) - 原理 / 源码 / 性能
- [select.md](select.md) - 多路复用
- [context.md](context.md) - 取消传播
- [sync-package.md](sync-package.md) - Mutex / WaitGroup / Cond / Once / Map / Pool
- [memory-model.md](memory-model.md) - happens-before
- Go 官方博客：Pipelines 和 Cancellation 系列
- 《Go Concurrency Patterns》by Rob Pike（Google I/O 2012）
