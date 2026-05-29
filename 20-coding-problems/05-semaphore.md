# 信号量(Semaphore)

> **题目**:实现一个计数信号量(Counting Semaphore),允许最多 N 个 goroutine 同时持有,超出的阻塞等待。支持 `Acquire / Release / TryAcquire / AcquireTimeout`。
>
> 考查:**Mutex+Cond 还是 channel、超时实现、和互斥锁的本质区别、weighted 信号量**。

信号量是协程池 / 限流器 / 连接池的底层原语,Go 标准库 `golang.org/x/sync/semaphore` 提供官方版。

---

## 一、考点拆解

| 考点 | 不及格 | 合格 | 优秀 |
| --- | --- | --- | --- |
| 概念 | 等于 Mutex | 计数信号量(允许 N 个并发) | 二元 vs 计数 vs Weighted |
| 实现选型 | 只会一种 | channel 实现 | **Cond 和 channel 各自适用场景** |
| 超时 | 不支持 | `AcquireTimeout` | 用 `context.Context` |
| 公平性 | 没考虑 | FIFO 排队 | 解释为什么 channel 天然 FIFO |
| Weighted | 不知道 | 知道概念 | 写出 acquire(n) 实现 |
| 用途 | 只知道限并发 | 限并发 + 资源池 + 顺序协作 | 知道在协程池 / `errgroup.WithContext` 里的用法 |

---

## 二、信号量 vs 互斥锁(必懂)

| | Mutex | 信号量 |
| --- | --- | --- |
| **并发数** | **只能 1** 个 | N 个(配置) |
| **所有权** | **谁加锁谁释放** | **任意 goroutine 都可 Release** |
| **本质** | 互斥(临界区保护) | 计数(资源数控制) |
| **用途** | 保护数据结构 | 限流 / 连接池 / 协程并发数 |

> **关键差异**:互斥锁是"我占用,别人等";信号量是"我们最多 N 个一起进,多了排队"。
>
> 二元信号量(N=1)看起来像 Mutex,但**所有权语义不同**——A 可以 Acquire,B 可以 Release,Mutex 不行。

---

## 三、实现一:基于 channel(最简洁)

### 3.1 思路

```text
带缓冲 channel 容量 = N,塞进去一个"令牌"代表占用一个名额。
Acquire = 往 channel 写(满了阻塞)
Release = 从 channel 读
```

### 3.2 完整实现

```go
package sem

import (
    "context"
    "errors"
    "time"
)

type ChanSemaphore struct {
    ch chan struct{}
}

func NewChan(n int) *ChanSemaphore {
    if n <= 0 {
        panic("semaphore size must > 0")
    }
    return &ChanSemaphore{ch: make(chan struct{}, n)}
}

// Acquire 阻塞获取
func (s *ChanSemaphore) Acquire() {
    s.ch <- struct{}{}
}

// TryAcquire 非阻塞,获取不到立即返回 false
func (s *ChanSemaphore) TryAcquire() bool {
    select {
    case s.ch <- struct{}{}:
        return true
    default:
        return false
    }
}

// AcquireTimeout 带超时
func (s *ChanSemaphore) AcquireTimeout(d time.Duration) bool {
    select {
    case s.ch <- struct{}{}:
        return true
    case <-time.After(d):
        return false
    }
}

// AcquireCtx 带 context(更现代)
var ErrAcquireCanceled = errors.New("acquire canceled")

func (s *ChanSemaphore) AcquireCtx(ctx context.Context) error {
    select {
    case s.ch <- struct{}{}:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

// Release 释放
func (s *ChanSemaphore) Release() {
    select {
    case <-s.ch:
    default:
        panic("release without acquire")
    }
}
```

**资深点**:

| 设计 | 解释 |
| --- | --- |
| `chan struct{}` 而不是 `chan bool` | `struct{}` 零内存,语义就是"信号" |
| `Release` 用 `select default` | 防止 release 多于 acquire 时永久阻塞(改 panic 暴露 bug) |
| `time.After` 有内存泄漏 | 长跑系统改用 `time.NewTimer` + 显式 `Stop()`,本文简化用 After |
| **天然 FIFO** | Go channel 调度按 G 排队,**先来的先拿到**,公平 |

### 3.3 典型用法:限并发爬虫

```go
sem := NewChan(10) // 最多 10 个并发

for _, url := range urls {
    sem.Acquire()
    go func(u string) {
        defer sem.Release()
        fetch(u)
    }(url)
}
```

---

## 四、实现二:基于 Mutex+Cond(无 channel 版)

### 4.1 为什么还要写一遍

面试官常追问"不让用 channel 怎么写"。Cond 版是经典并发原语写法,Java/C++ 的 Semaphore 都是这个思路。

### 4.2 完整实现

```go
type CondSemaphore struct {
    capacity int
    count    int // 当前已被占用数
    mu       sync.Mutex
    cond     *sync.Cond
}

func NewCond(n int) *CondSemaphore {
    s := &CondSemaphore{capacity: n}
    s.cond = sync.NewCond(&s.mu)
    return s
}

func (s *CondSemaphore) Acquire() {
    s.mu.Lock()
    defer s.mu.Unlock()
    for s.count == s.capacity {
        s.cond.Wait()
    }
    s.count++
}

func (s *CondSemaphore) TryAcquire() bool {
    s.mu.Lock()
    defer s.mu.Unlock()
    if s.count == s.capacity {
        return false
    }
    s.count++
    return true
}

func (s *CondSemaphore) Release() {
    s.mu.Lock()
    defer s.mu.Unlock()
    if s.count == 0 {
        panic("release without acquire")
    }
    s.count--
    s.cond.Signal() // 唤醒一个等待者
}
```

**Cond 版 vs Channel 版**:

| | Channel 版 | Cond 版 |
| --- | --- | --- |
| 代码量 | 短 | 略长 |
| 公平性 | **FIFO**(channel 保证) | **不保证**(Cond.Signal 唤醒任意一个) |
| 性能 | channel 调度有 G 切换 | Mutex 更轻 |
| 超时支持 | **天然**(select) | **难**(Cond 没 WaitTimeout) |
| 推荐 | **大多数场景** | 无 channel 限制 / 极致性能 |

> Cond 版要实现 `AcquireTimeout` 必须用 `time.AfterFunc + Broadcast` 模拟,**不优雅**,这是 channel 版的最大优势。

---

## 五、Weighted Semaphore(资深扩展)

标准库 `golang.org/x/sync/semaphore` 的核心特性:**一次可以 Acquire(n)**,适合"不同任务消耗不同资源量"。

```text
桶容量 100:
  大任务 Acquire(50)  → 占 50
  小任务 Acquire(10) × 5 → 占 50
  下一个大任务 Acquire(50) → 阻塞,等
```

### 5.1 思路

channel 版**做不了** weighted——一次只能写一个 struct{}。必须 Mutex+Cond + FIFO 队列。

### 5.2 简化实现

```go
type WeightedSemaphore struct {
    capacity int64
    cur      int64
    mu       sync.Mutex
    waiters  list.List // FIFO 队列
}

type waiter struct {
    n     int64
    ready chan struct{} // 该 waiter 被唤醒的信号
}

func (s *WeightedSemaphore) Acquire(ctx context.Context, n int64) error {
    s.mu.Lock()
    if s.cur+n <= s.capacity && s.waiters.Len() == 0 {
        // 没人排队 + 够用,直接获取
        s.cur += n
        s.mu.Unlock()
        return nil
    }
    // 入队等待
    w := &waiter{n: n, ready: make(chan struct{})}
    s.waiters.PushBack(w)
    s.mu.Unlock()

    select {
    case <-w.ready:
        return nil
    case <-ctx.Done():
        // 这里要把 w 从队列里删掉,省略
        return ctx.Err()
    }
}

func (s *WeightedSemaphore) Release(n int64) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.cur -= n
    // 尝试唤醒队首(只要它的 n 够分)
    for e := s.waiters.Front(); e != nil; {
        w := e.Value.(*waiter)
        if s.cur+w.n > s.capacity {
            break // 队首不够,后面也别试(保持 FIFO)
        }
        s.cur += w.n
        next := e.Next()
        s.waiters.Remove(e)
        close(w.ready)
        e = next
    }
}
```

**资深点**:
- **必须 FIFO**(不能跳过队首满足后面的小请求),否则大请求**永远饿死**
- 队首不够分时**直接 break**,不要继续看后面——保证公平
- 标准库实现还处理了 ctx 取消时从队列删除(防止内存泄漏),完整实现见 `x/sync/semaphore`

---

## 六、典型应用场景

### 6.1 限制并发数(最常用)

```go
sem := NewChan(20)
for _, item := range items {
    sem.Acquire()
    go func(i Item) {
        defer sem.Release()
        process(i)
    }(item)
}
```

### 6.2 资源池(数据库连接 / HTTP 客户端)

```go
type DBPool struct {
    sem  *ChanSemaphore
    pool []*sql.DB
    mu   sync.Mutex
}

func (p *DBPool) Get(ctx context.Context) (*sql.DB, error) {
    if err := p.sem.AcquireCtx(ctx); err != nil {
        return nil, err
    }
    p.mu.Lock()
    defer p.mu.Unlock()
    conn := p.pool[len(p.pool)-1]
    p.pool = p.pool[:len(p.pool)-1]
    return conn, nil
}
```

### 6.3 协程池的并发控制

协程池本质就是"信号量 + 任务队列"(详见 [02-worker-pool.md](02-worker-pool.md))。

### 6.4 `errgroup.WithContext` 的容量限制

`errgroup` 1.20 后支持 `g.SetLimit(n)`,内部就是个信号量。

---

## 七、典型坑

| 坑 | 现象 | 修复 |
| --- | --- | --- |
| Release 多于 Acquire | channel 版死锁 / Cond 版 panic | 在 Release 时校验,严格配对 |
| 协程 leak(Acquire 后忘 Release)| 长期持有,池子越来越小 | **`defer Release`** 永远配对 |
| 用 Mutex 当信号量 | 限不了多并发 | Mutex 只能 1,信号量任意 N |
| Cond 版不能超时 | 要超时只能拼 timer + Broadcast | **要超时直接用 channel 版** |
| Weighted 版不 FIFO | 大请求饿死 | 队首不够分直接 break |
| `time.After` 频繁调用 | 内存泄漏 | 改 `time.NewTimer` + Stop |
| panic 后没 Release | 名额永久占用 | `defer Release` 不要放业务代码里 |

---

## 八、Go 标准库对照

```go
import "golang.org/x/sync/semaphore"

sem := semaphore.NewWeighted(10)
ctx := context.Background()

// 占 3 个
if err := sem.Acquire(ctx, 3); err != nil {
    return err
}
defer sem.Release(3)

// 非阻塞尝试
if !sem.TryAcquire(5) {
    return errors.New("busy")
}
```

资深表达:"我手写的思路和 `x/sync/semaphore` 一致——**Mutex + FIFO 队列 + 每个 waiter 一个 ready chan**。"

---

## 九、现场表达模板

> "信号量本质是**计数器 + 排队等待**,和互斥锁的区别是**允许 N 个同时持有**,且**所有权不绑定 goroutine**。
>
> 两种实现:
> - **channel 版**最简洁:`make(chan struct{}, N)`,Acquire 写 / Release 读。**天然 FIFO + 天然支持超时**(select),实战首选。
> - **Cond 版**适合不能用 channel 的场景或追求性能,但**超时实现不优雅**(Cond 没 WaitTimeout)。
>
> 进阶是 **Weighted 信号量**——一次 Acquire(n) 个,标准库 `x/sync/semaphore` 就是这个。
> 实现要点:**必须 FIFO + 队首不够分时直接 break**,否则大请求会饿死。
>
> 典型用途:限并发数、资源池、连接池、`errgroup.SetLimit`。
> 协程池本质也是信号量 + 任务队列。"

---

## 十、一句话总结

> **信号量 = 计数版互斥锁**;
>
> - **channel 实现**最简,`chan struct{}` 容量=N,天然 FIFO + 天然超时,实战首选
> - **Cond 实现**性能略优,但超时难;能用 channel 就别用 Cond
> - **Weighted 信号量**(`x/sync/semaphore`)需要 **FIFO 队列 + 每 waiter 一个 ready chan**,防大请求饿死
> - 用法:限并发 / 资源池 / 协程池 / `errgroup.SetLimit`
