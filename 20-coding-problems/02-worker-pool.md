# 协程池(Worker Pool)

> **题目**:用 Go 实现一个协程池,支持提交任务、控制最大并发数、优雅关闭、panic 不影响其他 worker。
>
> 考查:**channel 当队列 + WaitGroup 收尾 + defer recover 兜底 + Close 语义**。

这是阻塞队列([01-blocking-queue.md](01-blocking-queue.md))的"应用题"——把队列接到 N 个消费者上,就是协程池。

---

## 一、考点拆解

| 考点 | 不及格 | 合格 | 优秀 |
| --- | --- | --- | --- |
| 并发控制 | `go f()` 无上限 | 固定 N 个 worker | 动态扩缩容 |
| 任务队列 | 全局变量 + 锁 | buffered channel | 支持背压策略 |
| 提交策略 | 阻塞 / 直接丢 | 可选阻塞/非阻塞/超时 | 拒绝策略可插拔 |
| 关闭 | 直接 `close(taskCh)` | Stop + WaitGroup | Drain(等任务跑完)vs Cancel(立即停)|
| panic 安全 | 一个 panic 整个崩 | 每个 worker `defer recover` | 上报监控 + 可选重启 worker |
| 结果获取 | 没有 | Result channel | Future / 泛型 |

> **资深表达**:协程池的本质是"**一个有界任务队列 + 一组消费者 goroutine**",难点不在功能,在**关闭语义**和**panic 隔离**。

---

## 二、思路推导

### 2.1 为什么不直接 `go f()`

```go
for _, task := range tasks {
    go task() // ❌
}
```

问题:
- **goroutine 数量不可控**(百万任务 → 百万 goroutine → 调度 / 栈内存爆炸)
- **背压无处施加**(下游慢了上游不知道)
- **panic 直接打爆进程**

协程池 = **用固定 N 个 worker 消费同一个 task channel**,把"提交"和"执行"解耦。

### 2.2 队列怎么选

| 选项 | 评价 |
| --- | --- |
| 无缓冲 channel | submit 必须等到有 worker 空,**同步阻塞**,但天然背压 |
| 有缓冲 channel | submit 入队即返回,**异步**,缓冲满了再阻塞 |
| 自己写的 BlockingQueue | 没必要,channel 已经是阻塞队列 |

实战:**有缓冲 channel + 缓冲长度 = N(worker 数)左右**,既能吸收抖动又不堆积太久。

### 2.3 关闭语义(最容易翻车)

| 语义 | 行为 | 适用 |
| --- | --- | --- |
| **Stop / Drain** | 不再接新任务,**等队里的跑完** | 业务正常下线 |
| **StopNow / Cancel** | 不再接 + **取消队里未跑的**,正在跑的可以选择中断 | 紧急故障 |
| **Shutdown(timeout)** | Drain + 超时强杀 | k8s 优雅关闭 |

资深点:**两种语义要 API 上分开**(Java `ExecutorService.shutdown` vs `shutdownNow`),不能糊在一起。

---

## 三、完整实现(标准版:固定 worker + Drain 关闭)

```go
package pool

import (
    "context"
    "errors"
    "sync"
)

var (
    ErrPoolClosed = errors.New("pool closed")
    ErrPoolFull   = errors.New("pool full")
)

type Task func()

type Pool struct {
    taskCh  chan Task
    workers int
    wg      sync.WaitGroup

    closeOnce sync.Once
    closed    chan struct{}
}

// New 创建协程池
//   workers: 并发数
//   queueSize: 任务队列容量(满时 Submit 阻塞)
func New(workers, queueSize int) *Pool {
    if workers <= 0 || queueSize < 0 {
        panic("invalid pool args")
    }
    p := &Pool{
        taskCh:  make(chan Task, queueSize),
        workers: workers,
        closed:  make(chan struct{}),
    }
    p.start()
    return p
}

func (p *Pool) start() {
    for i := 0; i < p.workers; i++ {
        p.wg.Add(1)
        go p.worker()
    }
}

func (p *Pool) worker() {
    defer p.wg.Done()
    for task := range p.taskCh { // close(taskCh) 后 range 自动退出
        p.runSafely(task)
    }
}

// runSafely 隔离 panic,一个任务挂掉不影响 worker 和其他任务
func (p *Pool) runSafely(task Task) {
    defer func() {
        if r := recover(); r != nil {
            // 实战:这里上报监控 / 打 log
            // log.Errorf("pool task panic: %v\n%s", r, debug.Stack())
        }
    }()
    task()
}

// Submit 提交任务,池满则阻塞
func (p *Pool) Submit(t Task) error {
    select {
    case <-p.closed:
        return ErrPoolClosed
    default:
    }
    select {
    case p.taskCh <- t:
        return nil
    case <-p.closed:
        return ErrPoolClosed
    }
}

// TrySubmit 非阻塞提交,池满直接返回错误
func (p *Pool) TrySubmit(t Task) error {
    select {
    case <-p.closed:
        return ErrPoolClosed
    case p.taskCh <- t:
        return nil
    default:
        return ErrPoolFull
    }
}

// SubmitWithContext 带 ctx 的提交,可超时/取消
func (p *Pool) SubmitWithContext(ctx context.Context, t Task) error {
    select {
    case <-p.closed:
        return ErrPoolClosed
    case <-ctx.Done():
        return ctx.Err()
    case p.taskCh <- t:
        return nil
    }
}

// Shutdown 优雅关闭:不再接新任务,等队里任务跑完
func (p *Pool) Shutdown() {
    p.closeOnce.Do(func() {
        close(p.closed)
        close(p.taskCh) // worker 的 range 退出
    })
    p.wg.Wait()
}
```

**讲解时主动点出的细节**:

| 设计 | 资深点 |
| --- | --- |
| `range p.taskCh` | channel close 后 range 自动退出,**不需要额外退出信号** |
| `defer recover()` 包裹任务 | 一个任务 panic **不能影响 worker 本身**,否则池会"越跑越少" |
| `closeOnce` | Shutdown 必须**幂等**,重复调用不能 panic |
| `closed` channel + `taskCh` 两个信号 | `closed` 拒新提交、`taskCh close` 让 worker 退出,**职责分开** |
| Submit 内**先 select closed 再 select taskCh** | 防止 close 后还能塞进去(虽然 channel close 后写会 panic,但提前判断更优雅)|

---

## 四、测试

```go
package main

import (
    "fmt"
    "sync/atomic"
    "time"
)

func main() {
    p := New(3, 5)

    var done int32
    for i := 1; i <= 10; i++ {
        i := i
        _ = p.Submit(func() {
            time.Sleep(100 * time.Millisecond)
            atomic.AddInt32(&done, 1)
            fmt.Printf("task %d done\n", i)
        })
    }

    p.Shutdown() // 等 10 个任务全部完成
    fmt.Printf("all done, count=%d\n", atomic.LoadInt32(&done))
}
```

输出:每批 3 个并发 → 100ms 一批 → 10 个总耗时约 4×100ms,符合并发数限制。

---

## 五、进阶变体

### 5.1 StopNow:不等队里任务,立即停

```go
func (p *Pool) StopNow() (pending int) {
    p.closeOnce.Do(func() {
        close(p.closed)
    })
    // 不 close(taskCh),改用 closed 信号让 worker 退出
    // (前面的 worker() 要改成 select { case <-p.closed: return; case t := <-p.taskCh: ... })
    // 统计未执行的任务数
    for {
        select {
        case <-p.taskCh:
            pending++
        default:
            p.wg.Wait()
            return
        }
    }
}
```

> 注意:`StopNow` 和 `Shutdown` 不能同时存在,因为 worker 主循环结构不同。**实战二选一**,或参数化。

### 5.2 带结果的 Submit(Future 模式)

```go
type Future[T any] struct {
    result chan T
    err    chan error
}

func (f *Future[T]) Wait() (T, error) {
    select {
    case v := <-f.result:
        return v, nil
    case e := <-f.err:
        var zero T
        return zero, e
    }
}

func SubmitFunc[T any](p *Pool, fn func() (T, error)) *Future[T] {
    f := &Future[T]{
        result: make(chan T, 1),
        err:    make(chan error, 1),
    }
    p.Submit(func() {
        v, e := fn()
        if e != nil {
            f.err <- e
        } else {
            f.result <- v
        }
    })
    return f
}
```

资深点:result chan 容量必须 = 1,**否则 task 写入会阻塞,worker 永远释放不出来**。

### 5.3 动态扩缩容(高级)

```go
func (p *Pool) Resize(n int) {
    p.mu.Lock()
    defer p.mu.Unlock()
    if n > p.workers {
        for i := p.workers; i < n; i++ {
            p.wg.Add(1)
            go p.worker()
        }
    } else {
        // 缩容:发 stopSignal 让多余 worker 退出
        for i := n; i < p.workers; i++ {
            p.stopSignal <- struct{}{}
        }
    }
    p.workers = n
}
```

worker 要改成同时 select `taskCh` 和 `stopSignal`。难点:**缩容时正在跑任务的 worker 不能强杀**,只能等它跑完再 self-exit。

### 5.4 拒绝策略(参考 Java ThreadPoolExecutor)

| 策略 | 行为 | 场景 |
| --- | --- | --- |
| **Abort** | 直接报错(默认) | 强一致,不能丢 |
| **CallerRuns** | 让调用方自己跑这个任务 | 自然背压,保护下游 |
| **DiscardOldest** | 丢弃队首最老的任务 | 实时数据,旧的无意义 |
| **Discard** | 静默丢弃新任务 | 日志类,允许丢 |

```go
type RejectPolicy func(t Task) error

func AbortPolicy(t Task) error { return ErrPoolFull }
func CallerRunsPolicy(t Task) error { t(); return nil }
```

---

## 六、典型坑

| 坑 | 现象 | 修复 |
| --- | --- | --- |
| worker 里没 recover | 一个 panic 把 worker 干掉,池容量越来越小 | 每个 worker 入口 `defer recover` |
| `close(taskCh)` 后还有人 `Submit` | panic: send on closed channel | Submit 内先判 `<-p.closed` |
| `close` 被调多次 | panic: close of closed channel | `sync.Once` |
| Future result chan 无 buffer | task 写不进去 → worker 卡住 | result chan 容量必须 ≥1 |
| Shutdown 不 WaitGroup | 主程序直接退出,任务没跑完 | `p.wg.Wait()` |
| 队列容量 0(无缓冲) | Submit 同步阻塞,看起来像死锁 | 文档说明 / 默认给个合理 buffer |
| 全局共享一个池 | 任务互相阻塞 | 按业务分池(IO 池 / CPU 池)|

---

## 七、和 channel 原生用法对比

很多人会问"既然 channel 已经能限流,为什么还要协程池"?

```go
// 方案 A:信号量(用 channel 当令牌池,N 个并发)
sem := make(chan struct{}, 10)
for _, task := range tasks {
    sem <- struct{}{}
    go func(t Task) {
        defer func() { <-sem }()
        t()
    }(task)
}
```

| 维度 | 信号量方案 | 协程池 |
| --- | --- | --- |
| 实现复杂度 | 极简 | 中等 |
| goroutine 复用 | ❌ 每次新建 | ✅ 复用 |
| panic 隔离 | 需自己加 recover | 池内置 |
| 关闭语义 | 无 | Drain / Cancel |
| 监控统计 | 难 | 容易(队列长度 / worker 数) |
| 适用 | **简单批量任务** | **长期服务的稳定后台** |

资深表达:**短任务批处理用信号量足够,长跑后台服务用协程池**。两者不是非此即彼。

---

## 八、现场表达模板

> "协程池本质是**有界任务队列 + N 个消费者 goroutine**。
> 队列我直接用 buffered channel,worker 在 `for range taskCh` 里消费,close 后 range 自动退出。
>
> 关键有几个点:
> 1. 每个 worker 必须 `defer recover`,否则一个任务 panic 把 worker 干掉,池容量越来越小。
> 2. Close 要用 `sync.Once` 保证幂等,而且要区分 Shutdown(等任务跑完)和 StopNow(立即停)。
> 3. Submit 要支持阻塞 / 非阻塞 / 带 ctx 三种 API,匹配不同业务场景。
>
> 进阶可以做:拒绝策略可插拔、动态扩缩容、Future 模式返回结果。
> 但如果只是短任务批处理,其实**信号量 channel 就够了**,不一定上协程池。"

---

## 九、一句话总结

> **协程池 = buffered channel 当队列 + N 个 worker `for range` 消费 + 每个 worker `defer recover` + Shutdown 用 `sync.Once` + WaitGroup 等齐**。
>
> 难点不在功能,在**关闭语义(Drain vs Cancel)、panic 隔离、拒绝策略**——这三个讲清楚就过了。
