# goroutine

> Go 用户态协程：M:N 调度，栈从 2KB 起伸缩，创建/销毁极轻量，由 runtime 全权管理

## 一、一句话总结(背诵版)

> **goroutine 是 Go runtime 调度的轻量级用户态协程**——栈 2KB 起动态伸缩,创建/切换都在用户态,`go func()` 一行启动,**百万并发不是难事**(同等内存下 OS thread 撑不过几千)。

延伸(被追问时再展开):

> 它是 Go 并发模型的基石——**M:N 调度**把 N 个 goroutine 复用到 M 个 OS 线程上,**抢占 + 协作**结合调度,配合 channel 实现 CSP。"线程级开销、函数级心智",这才是 Go 高并发的根。

---

## 二、使用场景(场景化记忆)

> 凡是"独立任务 + 可并行 + 不强求顺序"的活儿,都该考虑开 goroutine。

| 场景 | 一句话 | 典型用法 |
| --- | --- | --- |
| **服务器每请求一 G** | net/http 默认每个请求开一个 goroutine | `http.Handler` 内直接同步写,runtime 帮你并发 |
| **并行处理(map-reduce 风格)** | N 个独立子任务同时跑,聚合结果 | `errgroup.Group{}.Go(...)` + `Wait()` |
| **异步执行(fire-and-forget)** | 发邮件 / 打日志 / 上报埋点,不阻塞主流程 | `go SafeLog(...)`(必须 recover) |
| **后台 worker / 定时任务** | 常驻 goroutine 消费 channel / ticker | `for { select { case <-ticker.C: ... case <-ctx.Done(): return } }` |
| **扇出扇入(fan-out / fan-in)** | 1 源 → N worker → 1 收集 | worker pool + 结果 channel |
| **超时 / 取消传播** | 用 ctx 控制一批 goroutine 同时退出 | `errgroup.WithContext` |

**判断要不要开 G 的标准**:**任务能并行 + 有明确退出条件 + 数量可控**——三个都满足才开,否则用 worker pool。

---

## 三、常见错误(高频踩坑)

| 错误 | 现象 | 根因 | 修复 |
| --- | --- | --- | --- |
| **goroutine 泄漏** | `runtime.NumGoroutine()` 线性增长,内存涨 | channel 永远不关 / select 没退出分支 / 无 ctx 控制 | 每个 G 都要有明确退出条件,加 ctx + done channel |
| **for 循环闭包变量复用**(Go 1.22 前) | N 个 G 全打印同一个值(最后一个) | for 的 `i` 是同一个变量,G 捕获的是引用 | `go func(i int){...}(i)` 或 `i := i`;Go 1.22+ 默认修复 |
| **panic 不能跨 G recover** | 子 G panic 直接整个进程 crash | 每个 G 有独立栈,recover 只对本 G 生效 | 每个 G 入口加 `defer recover()`,封装 GoSafe |
| **过度创建 G 没有 pool** | 调度延迟飙升 / OOM / GC 暴涨 | 无界 `for range tasks { go t() }`,百万 G 失控 | 用 worker pool / 信号量 / errgroup.SetLimit |
| **主 G 提前退出** | 子 G 还没跑完进程就退了 | main 函数 return 等于 `os.Exit(0)`,不等其他 G | `sync.WaitGroup` 或 `errgroup.Wait()` 阻塞主 G |
| **`wg.Add` 放在 `go` 之后** | `wg.Wait()` 偶发不等(竞态) | 主 G 已经 Wait,Add 还没执行,counter=0 立即放行 | **永远在 `go` 之前 Add** |
| **ctx 不传递** | 取消信号传不进 G,优雅停机失败 | 子 G 不知道父级要退出 | 函数签名第一个参数固定 `ctx context.Context`,select 监听 `<-ctx.Done()` |

---

## 四、面试常问(简答模板)

**Q1:goroutine 和线程的区别?**
goroutine 是 **runtime 调度的用户态协程**,栈 2KB 起动态伸缩,创建/切换都不进内核;OS 线程栈 1~8MB 固定,切换走内核态(~1μs)。一个进程开**百万 G** 没问题,开**几千线程**就吃不消。M:N 模型把 N 个 G 复用到 M 个 OS 线程上。

**Q2:G 的栈多大?怎么扩容?**
初始 **2KB**(早期 8KB),最大 **1GB**(amd64)。编译器在函数序言插入 `stackguard0` 检查,不够就调 `morestack`:**分配 2 倍新栈 + 复制旧栈内容 + 调整栈上指针**(precise stack copy)。GC 时使用率低于 1/4 还会**收缩**。

**Q3:GMP 调度模型一句话?**
**G 是 goroutine,M 是 OS 线程,P 是逻辑处理器(默认 = GOMAXPROCS = CPU 核数)**。P 持有本地 runq,M 必须绑定 P 才能跑 G;本地 runq 空了去全局 / 别的 P 偷(work-stealing)。

**Q4:百万 goroutine 真实开销?**
内存:100 万 × 2KB = **2GB 起步**(实际更多,栈会涨)。调度:G 多了调度延迟上升,GC 扫描栈也变慢。**生产上建议用 worker pool 把并发数压到 CPU 核数的几倍到几十倍**,不要无脑 `go`。

**Q5:goroutine 调度时机?抢占式还是协作式?**
**Go 1.14 起是基于信号的异步抢占**(之前是协作式,只在函数调用、chan 操作、syscall 等检查点切换)。现在 sysmon 监控到 G 跑超过 10ms 会发 SIGURG 强制抢占,避免死循环卡住调度。

**Q6:goroutine 泄漏怎么排查?**
1. `pprof.Lookup("goroutine").WriteTo(os.Stdout, 1)` 看 G 数量和堆栈
2. `go tool pprof http://localhost:6060/debug/pprof/goroutine` 看火焰图
3. 重点看 **waitReason**:`chan send`/`chan receive`/`select` 集中的就是泄漏点
4. 治理:每个 `go func()` 入口先答三问——**什么时候退出?怎么取消?panic 怎么办?**

---

## 五、深水区:原理与源码(被追问时看)

> 下面是 goroutine 的 g 结构、创建销毁、栈管理、状态机、八股速记、手写实现、踩坑实录等内容。**正常面试 Q1-Q6 够用**,只在被深追"g 结构里有啥 / morestack 怎么走 / GMP 偷 G 怎么偷"时才用到。

---

## 六、核心原理

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

## 七、八股速记

- M:N 调度，**N 个 goroutine 复用 M 个 OS thread**
- 栈 **2KB 起，动态伸缩**，最大 1GB
- 栈扩容用**复制法**，所以 g 上指针都要被 GC/runtime 准确识别
- 创建即 `go f()`，**没有 join，没有返回值**，要拿结果用 channel 或 errgroup
- g 没有 ID 方法（runtime.Goid 故意不暴露），是为了防止开发者依赖 g 局部状态做坑爹设计
- panic 不会跨 g 传播，子 g 的 panic 会让整个进程 crash（除非自己 recover）
- 主 g 退出整个程序退出，不等其他 g
- runtime.Gosched() 主动让出，runtime.Goexit() 立即终止当前 g

## 八、面试真题

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

## 九、手写实现

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

## 十、踩坑与最佳实践

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
