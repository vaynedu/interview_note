# goroutine

> Go 用户态协程：M:N 调度，栈从 2KB 起伸缩，创建/销毁极轻量，由 runtime 全权管理

## 一、核心原理

### 1.1 g 结构

```go
// runtime/runtime2.go
type g struct {
    stack       stack          // 栈范围 [stack.lo, stack.hi)
    stackguard0 uintptr        // 栈保护边界,触发栈扩容/抢占
    m           *m             // 当前绑定的 M(可能为 nil)
    sched       gobuf          // 调度上下文(PC/SP/BP/g 自身)
    atomicstatus uint32        // _Grunnable/_Grunning/_Gwaiting/...
    goid        int64          // 唯一 ID
    waitreason  waitReason     // 阻塞原因(channel send/recv/select/...)
    ...
}
```

`gobuf` 保存 g 切换出去时的寄存器现场，下次调度时恢复。

### 1.2 创建与销毁

`go f(x)` 编译为 `runtime.newproc`：
1. 从 P 的 gFree 链表（或全局 sched.gFree）取一个空 g，没有就 `malg` 新建（栈 2KB）
2. 拷贝参数到 g 的栈
3. 设置 PC 指向 f
4. 把 g 放到当前 P 的 runnext / runq

g 退出时调用 `goexit`，清理后放回 P 的 gFree 复用，**栈内存不立即释放**（控制在阈值内复用）。

### 1.3 栈管理

- 初始栈 **2KB**（早期是 4KB / 8KB）
- 函数序言里有 `stackguard0` 检查，发现栈不够就调用 `morestack`
- 栈扩容：分配 2 倍新栈，**复制旧栈所有内容**，调整栈上指针（precise stack copy）
- 栈最大 1GB（amd64），超出 panic("stack overflow")
- GC 时也可能**栈收缩**（使用率低于 1/4）

### 1.4 状态机

```
_Gidle → _Grunnable → _Grunning ⇄ _Gwaiting (chan/sleep/syscall/...)
                         ↓
                      _Gdead → 复用回 gFree
```

### 1.5 与线程对比

| | goroutine | OS thread |
| --- | --- | --- |
| 创建成本 | 微秒级，几 KB | 毫秒级，MB 级栈 |
| 切换成本 | 用户态，只换几寄存器 | 内核态系统调用 |
| 栈大小 | 2KB 起，动态伸缩 | 固定 1~8MB |
| 调度方 | Go runtime | OS 内核 |
| 数量 | 百万级可行 | 几千就吃不消 |

## 二、八股速记

- M:N 调度，**N 个 goroutine 复用 M 个 OS thread**
- 栈 **2KB 起，动态伸缩**，最大 1GB
- 栈扩容用**复制法**，所以 g 上指针都要被 GC/runtime 准确识别
- 创建即 `go f()`，**没有 join，没有返回值**，要拿结果用 channel 或 errgroup
- g 没有 ID 方法（runtime.Goid 故意不暴露），是为了防止开发者依赖 g 局部状态做坑爹设计
- panic 不会跨 g 传播，子 g 的 panic 会让整个进程 crash（除非自己 recover）
- 主 g 退出整个程序退出，不等其他 g
- runtime.Gosched() 主动让出，runtime.Goexit() 立即终止当前 g

## 三、面试真题

**Q1：goroutine 是协程还是线程？**
是 Go runtime 调度的**用户态协程**（M:N 调度模型）。多个 goroutine 复用少量 OS 线程，由 Go runtime 在用户态完成调度，避免大部分内核态切换开销。

**Q2：goroutine 栈为什么从 2KB 起？怎么扩容？**
小栈节省内存（百万级 g 才可行）。函数调用前编译器插入栈检查，不够时调用 `morestack`：分配 2 倍新栈、复制旧栈内容、调整栈上指针、继续执行。代价是每次扩容有拷贝，但通过 size doubling 摊销。

**Q3：goroutine 数量没有上限吗？**
理论上限：内存 / 栈大小。实际限制：调度开销、GC 压力、文件句柄等系统资源。生产环境用 worker pool / 信号量限制并发数，避免无限制 `go func()`。

**Q4：goroutine 间怎么传错误？**
- 单 g：channel 传 `(result, error)` 结构
- 多 g 任意失败即取消：`golang.org/x/sync/errgroup`
- 收集所有错误：`hashicorp/go-multierror` 或 channel 收集

**Q5：goroutine 泄漏怎么排查？**
1. `pprof.Lookup("goroutine").WriteTo(os.Stdout, 1)` 看 g 数量和堆栈
2. 看 waitReason 集中在哪：`chan send`/`chan receive`/`select` 通常是泄漏点
3. 工具：`go tool pprof http://localhost:6060/debug/pprof/goroutine`
4. 治理：每个 `go func()` 都问"它什么时候退出"，加 ctx 控制

**Q6：goroutine 切换比线程切换快多少？**
线程切换约 1~10μs（含内核态、TLB flush 等）；goroutine 切换约 100~200ns，快 10~100 倍。但 goroutine 切换次数也更频繁（GC、抢占、chan 操作都触发）。

**Q7：goroutine 能被强制 kill 吗？**
**不能**。Go 没有 `g.Kill()`。退出方式只能是：(1) g 自己 return；(2) 通过 ctx/chan 通知它退出，它自己配合。这是设计选择——强制 kill 会让资源清理状态不可控。

## 四、手写实现

**1. WaitGroup 风格的并发执行：**

```go
func parallel(tasks []func() error) error {
    var wg sync.WaitGroup
    errCh := make(chan error, len(tasks))
    for _, t := range tasks {
        wg.Add(1)
        go func(t func() error) {
            defer wg.Done()
            if err := t(); err != nil {
                errCh <- err
            }
        }(t)
    }
    wg.Wait()
    close(errCh)
    for e := range errCh {
        if e != nil {
            return e
        }
    }
    return nil
}
```

**2. 任意失败即取消（手写 errgroup 简化版）：**

```go
type Group struct {
    wg     sync.WaitGroup
    errOnce sync.Once
    err    error
    cancel context.CancelFunc
}

func WithContext(parent context.Context) (*Group, context.Context) {
    ctx, cancel := context.WithCancel(parent)
    return &Group{cancel: cancel}, ctx
}

func (g *Group) Go(f func() error) {
    g.wg.Add(1)
    go func() {
        defer g.wg.Done()
        if err := f(); err != nil {
            g.errOnce.Do(func() {
                g.err = err
                if g.cancel != nil {
                    g.cancel()
                }
            })
        }
    }()
}

func (g *Group) Wait() error {
    g.wg.Wait()
    if g.cancel != nil {
        g.cancel()
    }
    return g.err
}
```

**3. Worker Pool（限制并发）：**

```go
func workerPool[T, R any](
    ctx context.Context,
    in <-chan T,
    n int,
    work func(T) R,
) <-chan R {
    out := make(chan R, n)
    var wg sync.WaitGroup
    wg.Add(n)
    for i := 0; i < n; i++ {
        go func() {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done():
                    return
                case v, ok := <-in:
                    if !ok { return }
                    select {
                    case out <- work(v):
                    case <-ctx.Done():
                        return
                    }
                }
            }
        }()
    }
    go func() { wg.Wait(); close(out) }()
    return out
}
```

## 五、踩坑与最佳实践

### 坑 1：goroutine 泄漏（最常见）

```go
func leak() {
    ch := make(chan int)
    go func() {
        v := <-ch  // 永远没人发送 → g 永久阻塞
        fmt.Println(v)
    }()
}
```

特征：`pprof goroutine` 看到一类 g 数量随时间线性增长。

### 坑 2：循环变量捕获（Go 1.22 前）

```go
for i := 0; i < 10; i++ {
    go func() {
        fmt.Println(i)  // 可能全打印 10
    }()
}
```

老版本：`go func(i int) { ... }(i)` 或 `i := i`。Go 1.22+ 默认每轮新变量。

### 坑 3：忘记 `wg.Add` 先于 `go`

```go
go func() {
    wg.Add(1)  // 错: 主 g 可能已经 wg.Wait() 了
    defer wg.Done()
    ...
}()
```

`wg.Add` 必须在启动 g **之前**调用。

### 坑 4：子 g panic 不被父 g 捕获

```go
go func() {
    panic("oops")  // 整个进程 crash, 父 g recover 救不到
}()
```

每个 g 都要在入口加 `defer recover`（或用统一 GoSafe 包装函数）。

### 坑 5：忽略 ctx 导致无法优雅停机

```go
go func() {
    for {
        doWork()  // 没看 ctx, 主程序退出时它还在跑
    }
}()
```

每个长生命周期 g 都应监听 ctx。

### 最佳实践

- **每个 `go func()` 都问三件事**：什么时候退出？怎么取消？panic 怎么办？
- 用 `errgroup` / `sync.WaitGroup` + `context` 组合控制
- 限制 worker 数量，不要 `for _,t:=range tasks { go t() }` 无界并发
- 封装 GoSafe 统一加 recover + 日志
- 排查泄漏：定期对比 `runtime.NumGoroutine()` 趋势 + pprof
