# sync 包

> Go 标准库并发原语：Mutex / RWMutex / WaitGroup / Once / Pool / Map / Cond / atomic

## 一、核心原理

### 1.1 sync.Mutex

```go
type Mutex struct {
    state int32   // bit 0=locked, 1=woken, 2=starving, 高位=等待者数
    sema  uint32  // 信号量, runtime_Semacquire/Semrelease
}
```

**两种模式**：
- **正常模式**：自旋几次抢锁，抢不到入队 FIFO 等待。新来的 g 优先（吞吐高，但可能饿死队尾）
- **饥饿模式**：等待 > 1ms 触发，Unlock 直接交给队首 g，新来的 g 排队尾

切换条件：等待时间 > 1ms 进饥饿，等到的 g 拿锁后等待时间 < 1ms 退出饥饿。

**自旋条件**（仅多核）：
- 锁是 locked 状态
- 没人在饥饿
- 自旋次数 < 4
- P 数量 > 1 且当前 g 不是 P 上唯一可运行 g

不可重入；禁止值拷贝（带 mu 的 struct 用指针传）。

### 1.2 sync.RWMutex

```go
type RWMutex struct {
    w           Mutex   // 写锁互斥
    writerSem   uint32  // 写者等待信号量
    readerSem   uint32  // 读者等待信号量
    readerCount int32   // 当前读者数(负值表示有写者等待)
    readerWait  int32   // 写者等待结束的读者数
}
```

- **RLock**：原子加 readerCount，<0 说明有写者等待，挂起
- **Lock**：先抢 w（写者互斥），再让 readerCount = -maxReaders，等所有当前读者退出
- **写者优先**：Lock 后续的 RLock 排队等待，避免写者饿死

### 1.3 sync.WaitGroup

```go
type WaitGroup struct {
    state atomic.Uint64  // 高 32 位 counter, 低 32 位 waiter count
    sema  uint32
}
```

- `Add(n)`：counter += n
- `Done()`：counter -= 1，归零时唤醒所有 waiter
- `Wait()`：counter > 0 时挂起

**规则**：`Add` 必须在 `Wait` 之前，且不能在 g 内 Add（应在 g 启动前）。Add 进负数 panic。

### 1.4 sync.Once

```go
type Once struct {
    done atomic.Uint32
    m    Mutex
}

func (o *Once) Do(f func()) {
    if o.done.Load() == 0 {
        o.doSlow(f)
    }
}

func (o *Once) doSlow(f func()) {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done.Load() == 0 {
        defer o.done.Store(1)
        f()
    }
}
```

**双重检查锁**经典实现。注意 `done.Store(1)` 用 defer 保证 f panic 时也设置（其实是 store 之前 panic 会让下次再尝试，看版本）。

### 1.5 sync.Pool

```go
type Pool struct {
    local     unsafe.Pointer  // 每 P 一个 poolLocal
    localSize uintptr
    victim    unsafe.Pointer  // 上一轮 GC 的 local (二级缓存)
    New       func() any
}

type poolLocal struct {
    private any        // 当前 P 独占, 无锁
    shared  poolChain  // 双向链表, 可被其他 P 偷
}
```

**Get 顺序**：private → shared 队首 → 偷其他 P 的 shared 队尾 → victim → New

**Put**：放 private（满了放 shared 队首）

**GC 行为**：每轮 GC 把 local → victim，victim → 丢弃。所以对象**最多存活两轮 GC**。不是持久缓存。

### 1.6 sync.Map

```go
type Map struct {
    mu     Mutex
    read   atomic.Pointer[readOnly]  // 无锁读
    dirty  map[any]*entry            // 写需要 mu
    misses int                       // read miss 次数
}
```

**两层**：read（atomic 读，几乎无锁）+ dirty（写）

**Load**：先查 read（无锁），不命中查 dirty（加锁），miss 多了把 dirty 提升到 read。

**适用场景**（官方明确）：
1. 一旦写入就只读（如配置）
2. 不同 g 操作不相交的 key 集合（如分片缓存）

写多场景**比 RWMutex+map 慢**。

### 1.7 sync.Cond

条件变量，用于"等待某条件成立"。

```go
type Cond struct {
    L Locker  // 通常是 *Mutex
    // ...
}

func (c *Cond) Wait()    // 等待: 释放 L, 阻塞, 被唤醒后重新拿 L
func (c *Cond) Signal()  // 唤醒一个
func (c *Cond) Broadcast() // 唤醒所有
```

实战中**很少用**，channel 通常更清晰。仅在条件复杂或需要广播大量等待者时考虑。

### 1.8 sync/atomic

无锁原子操作：Load/Store/Add/Swap/CompareAndSwap。

Go 1.19 引入类型化原子（`atomic.Int64`/`atomic.Pointer[T]`），强制对齐 + 不可值拷贝，比裸函数安全。

## 二、八股速记

- **Mutex** 不可重入、禁止值拷贝、有正常/饥饿两模式
- **RWMutex** 写者优先，读多写少时比 Mutex 快
- **WaitGroup** Add 在 Wait 之前调；不能在 g 里 Add
- **Once** 双重检查锁，`Do` 只执行一次（panic 也算一次？看版本）
- **Pool** 每 P 缓存 + GC 清理，**只用作临时对象复用**，不是持久缓存
- **Map** 适合"写一次读多次"或"key 不相交"，写多用 RWMutex+map
- **atomic.Int64** 等类型化原子（Go 1.19+）比裸函数安全
- 一切 sync 类型**禁止值拷贝**，go vet 会报警

## 三、面试真题

**Q1：Mutex 是公平锁吗？**
**默认非公平**（正常模式）：新来的 g 可以"插队"抢锁，吞吐高但队尾可能饿死。当某个 g 等待 > 1ms，进入**饥饿模式**变成 FIFO 公平，等队尾追上来再退出饥饿。设计平衡了吞吐和延迟尾部。

**Q2：RWMutex 什么时候比 Mutex 慢？**
读临界区**极短**时（如读一个 int），RWMutex 自身的原子操作开销超过保护的工作。简单读用 atomic 或 Mutex；复杂读路径才用 RWMutex。

**Q3：sync.Once 会重复执行吗？panic 怎么办？**
正常情况只执行一次。如果 f 中 panic：
- Go 1.x（旧）：done 未置位，下次 Do 会重试
- 现在（设 done 在 panic 前）：标记完成，下次直接跳过
**实际生产代码**：Once.Do 里的 f 应该自己 recover 处理 panic，不要依赖外层重试。

**Q4：sync.Pool 为什么对象会丢？**
两个原因：
1. **每轮 GC 清理**：local → victim，victim 丢弃
2. **每 P 独立**，Put 到 P1 的对象，Get 时若在 P2 可能拿到 New 的新对象

实践：放进 Pool 前**重置状态**，Get 后**不要假设 Put 过的内容**。

**Q5：sync.Map 比 map+RWMutex 强在哪？**
- read 字段是 `atomic.Pointer`，**Load 无锁**
- 适合读远多于写的场景（如全局配置缓存）
- 适合 key 集合很少变化的场景

弱在哪：
- 写需要 mu，**写多比 RWMutex+map 慢**
- 接口是 any，**类型断言开销 + 装箱**
- API 不如普通 map 灵活（没有 len、不支持 range 中删除）

**Q6：WaitGroup Add 在 g 里调用为什么不行？**

```go
var wg sync.WaitGroup
for _, t := range tasks {
    go func(t Task) {
        wg.Add(1)  // 错: 主 g 可能已经 wg.Wait() 了
        defer wg.Done()
        t.Run()
    }(t)
}
wg.Wait()  // 可能在 Add 之前就发现 counter==0 直接返回
```

正确：

```go
for _, t := range tasks {
    wg.Add(1)
    go func(t Task) {
        defer wg.Done()
        t.Run()
    }(t)
}
wg.Wait()
```

**Q7：atomic 操作能保证什么？**
- 操作的**原子性**：Load/Store/Add 不会被部分写入打断
- **顺序一致性**（Go 内存模型保证）：atomic 之间形成 happens-before
不能保证：复合操作的整体原子性（如"读+判断+写"要 CAS）。

**Q8：怎么实现一个并发安全的单例？**

```go
var (
    instance *Singleton
    once     sync.Once
)

func GetInstance() *Singleton {
    once.Do(func() {
        instance = &Singleton{...}
    })
    return instance
}
```

或更直接：包级 init 函数。

## 四、手写实现

**1. 基于 atomic 的并发计数器：**

```go
type Counter struct {
    n atomic.Int64
}
func (c *Counter) Inc()     { c.n.Add(1) }
func (c *Counter) Get() int64 { return c.n.Load() }
```

**2. 自旋锁（演示，生产用 Mutex）：**

```go
type SpinLock struct {
    state atomic.Int32
}
func (s *SpinLock) Lock() {
    for !s.state.CompareAndSwap(0, 1) {
        runtime.Gosched()  // 让出 P,避免空转
    }
}
func (s *SpinLock) Unlock() { s.state.Store(0) }
```

**3. 限频器（基于 channel + Pool 思路）：**

```go
type Limiter struct {
    tokens chan struct{}
}
func NewLimiter(n int) *Limiter {
    l := &Limiter{tokens: make(chan struct{}, n)}
    for i := 0; i < n; i++ { l.tokens <- struct{}{} }
    return l
}
func (l *Limiter) Acquire() { <-l.tokens }
func (l *Limiter) Release() { l.tokens <- struct{}{} }
```

**4. sync.Once 的等价实现：**

```go
type MyOnce struct {
    done atomic.Uint32
    m    sync.Mutex
}
func (o *MyOnce) Do(f func()) {
    if o.done.Load() == 1 { return }
    o.m.Lock()
    defer o.m.Unlock()
    if o.done.Load() == 0 {
        defer o.done.Store(1)
        f()
    }
}
```

**5. 用 Pool 复用 buffer：**

```go
var bufPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}

func getBuf() *bytes.Buffer { return bufPool.Get().(*bytes.Buffer) }
func putBuf(b *bytes.Buffer) {
    b.Reset()
    bufPool.Put(b)
}
```

## 五、踩坑与最佳实践

### 坑 1：Mutex 值拷贝

```go
type S struct {
    mu sync.Mutex
}
func (s S) Foo() {  // s 是副本! mu 也是副本
    s.mu.Lock()
    // ...
}
```

**修复**：改 `*S` 接收者。`go vet` 直接报。

### 坑 2：Mutex 重入死锁

```go
func (s *S) Foo() {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.Bar()  // Bar 也 Lock → 死锁
}
func (s *S) Bar() {
    s.mu.Lock()
    defer s.mu.Unlock()
    // ...
}
```

Go 不支持可重入锁。**修复**：拆出无锁版本 `barLocked`，调用方自己 Lock。

### 坑 3：RWMutex 写者饿死（旧版本）

老 Go 版本 RLock 持续到来，Lock 永远抢不到。现在的 RWMutex 已经实现"写者到来后阻塞新 RLock"修复，但仍要避免长时间持读锁。

### 坑 4：Pool 放大对象阻止内存回收

```go
type bigBuf struct{ data [10*1024*1024]byte }
var pool = sync.Pool{New: func() any { return new(bigBuf) }}
```

每次 GC 把 local 转 victim，victim 仍占内存，下次 GC 才丢。大对象用 Pool 反而**抬升常驻内存**。建议 ≤ 几十 KB 才用 Pool。

### 坑 5：atomic 操作非对齐字段

32 位平台上 `atomic.AddInt64` 要求 8 字节对齐。结构体字段顺序错可能崩溃。
**修复**：用 Go 1.19+ 的 `atomic.Int64`（自动对齐），或把 64 位字段放结构体首位。

### 坑 6：sync.Map 当通用 map 用

```go
m := &sync.Map{}
for i := 0; i < 1e6; i++ {
    m.Store(i, val)  // 写多场景比 map+Mutex 慢
}
```

sync.Map 是**专用工具**，不是普通 map 的并发版本。

### 最佳实践

- **简单计数用 atomic**，超过单变量考虑 Mutex
- 临界区**尽量小**，IO 不在锁内
- 锁的粒度选择：粗（少且简单）vs 细（多但提高并发度）
- defer Unlock 是默认选择，除非热路径需要手动 Unlock 减少 defer 开销
- 用 `go vet` + `-race` 检测锁拷贝和数据竞争
- 高并发场景考虑分片：`type ShardedMap struct { shards [N]*sync.Mutex; data [N]map[...]... }`
- 优先 channel 表达"通信"，sync 表达"保护共享状态"
