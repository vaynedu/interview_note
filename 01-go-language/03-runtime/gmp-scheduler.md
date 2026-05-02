# GMP 调度器

> Go 调度的灵魂：G(协程) - M(内核线程) - P(逻辑处理器) 三角关系，work-stealing + 抢占式调度

## 一、核心原理

### 1.1 三个角色

| | 含义 | 数量 |
| --- | --- | --- |
| **G** (goroutine) | 用户协程，含 PC/SP/栈/状态 | 任意多 |
| **M** (machine) | OS 线程，真正执行体 | 默认上限 10000 |
| **P** (processor) | 逻辑处理器，持有可运行 G 的本地队列 | `GOMAXPROCS`（默认 = CPU 核数） |

**关系**：M 必须**绑定一个 P** 才能执行 G；P 是 M 与 G 之间的中介，承载本地 runq 和 mcache。

### 1.2 队列结构

```
全局: sched.runq (lock 保护, 全局可运行 G)
每个 P:
  - runnext: 1 个高优先级 G (最近 go 出来的, 局部性优化)
  - runq:    256 长度的环形 G 队列 (无锁)
每个 M: 当前正在执行的 g, mcache, 本地状态
```

### 1.3 调度循环 schedule()

M 进入 `schedule()` 函数找 G 执行，顺序：

1. **每 61 次调度**强制查一次全局 runq（防全局队列饥饿）
2. P 的 **runnext**
3. P 的 **runq**
4. **全局 runq**（拿一批到本地）
5. **netpoll**：检查就绪的网络 G
6. **work-stealing**：随机选另一个 P 偷一半 G 过来
7. 还没有 → M 进入休眠（park），P 移交或挂起

### 1.4 抢占机制演进

- **Go 1.13 及之前**：协作式抢占，函数序言里检查 `stackguard0`，纯计算无函数调用的 G 会一直占用 M（出名的 "for{}" 卡死）
- **Go 1.14+**：**异步抢占**，基于信号（Linux 上是 SIGURG）。M 收到信号后保存现场入队
- 抢占触发：G 占用 P 超过 **10ms**（sysmon 监控）

### 1.5 系统调用处理

G 进入系统调用时（如读文件、syscall）：
- **快速 syscall**（如 nanosleep、futex）：M 不放手 P，回来继续
- **慢 syscall**（阻塞型）：M 与 P 解绑，P 找其他 M（或新建 M）继续跑别的 G。M 系统调用回来后没 P 了，把 G 放回全局 runq，自己进入休眠

### 1.6 netpoll（网络轮询）

Go 把所有 fd 注册到 epoll/kqueue。G 读网络阻塞时不是真阻塞 M，而是：
1. G 状态 → `_Gwaiting`，离开 P
2. fd 注册到 netpoll
3. M 继续从 P 拿别的 G 跑
4. sysmon 或 schedule 周期性 `netpoll()` 拉就绪 fd
5. 把对应 G 标记 `_Grunnable`，放回 runq

这就是为什么 Go 可以"同步代码写异步性能"——网络阻塞不阻塞线程。

### 1.7 sysmon（系统监控线程）

独立的 M（不绑定 P），每 20μs ~ 10ms 巡检：
- 触发抢占（G 跑超过 10ms）
- 收回长时间 syscall 中 M 的 P
- 触发 netpoll
- 强制 GC（2 分钟没 GC 时）
- 归还闲置内存给 OS

## 二、八股速记

- **G-M-P** 三件套：G 是协程，M 是线程，P 是调度器局部队列+缓存
- P 数量 = GOMAXPROCS，决定**并行度**
- 调度循环优先级：runnext → 本地 runq → 全局 runq → netpoll → 偷
- **work-stealing**：本地空了去偷别人**一半**
- **异步抢占**（Go 1.14+）基于信号，10ms 时间片
- **netpoll** 用 epoll/kqueue 让网络 IO 不阻塞 M
- **sysmon** 后台 M 负责抢占、netpoll、GC 触发
- 慢 syscall 时 M-P 解绑，避免 P 闲置
- 快 syscall M 保留 P，回来继续

## 三、面试真题

**Q1：为什么需要 P 这个中间层？**
早期 Go（1.0）只有 G 和 M，M 共享一个全局 runq + 全局锁，伸缩性差。引入 P 后：
- 每个 P 有本地 runq（无锁访问），减少争用
- M-P 解耦，让 syscall 阻塞时 P 可以被其他 M 接管
- 限制并行度（GOMAXPROCS），避免线程泛滥
- mcache 挂在 P 上，内存分配也无锁

**Q2：GOMAXPROCS 应该设多少？**
默认 = CPU 核数。一般场景**不要改**。容器化环境注意：
- Go 1.5~1.20 看不到 cgroup CPU limit，会拿到宿主机核数 → 用 [`uber-go/automaxprocs`](https://github.com/uber-go/automaxprocs) 自动按 limit 调整
- Go 1.25+ runtime 默认感知 cgroup（GOMAXPROCS=auto）
设小：单 G 任务跑慢；设大：调度抖动 + 锁争用变多

**Q3：work-stealing 怎么偷？**
M 当前 P 空了，随机选一个目标 P，**从目标 runq 尾部偷一半到自己 runq**。从尾偷是为了避免和目标 P 自己取（runq 头部）冲突。还顺带可能偷 timer。

**Q4：goroutine 阻塞时 M 一定会休眠吗？**
不一定。看阻塞原因：
- chan/锁/sleep 阻塞：G 进 `_Gwaiting`，M 继续跑别的 G（不休眠）
- 网络 IO：netpoll 接管，M 继续跑别的 G
- 慢 syscall：M 暂时放弃 P，可能休眠或抢别的活
- 没活可干：M park 休眠（pthread_cond_wait）

**Q5：runtime.Gosched() 做什么？**
当前 G 主动让出 P，进 P 的 runq 尾部，schedule 重新选 G。**不是 sleep**，立即可被再次调度。一般不需要主动调，runtime 自动抢占已经够用。

**Q6：异步抢占怎么实现？**
sysmon 发现某 G 占用 P 超过 10ms → 给绑定的 M 发信号（SIGURG on Linux）→ 信号 handler 修改 G 的 PC 跳到 `asyncPreempt` → asyncPreempt 保存所有寄存器（不只调用约定保存的）→ 进入 schedule。代价：保存全部寄存器比协作式重，但能抢占任何代码点。

**Q7：CGO 调用对调度有什么影响？**
CGO 调用相当于一次 syscall：
- M 与 P 解绑（C 代码可能阻塞任意时间）
- 其他 M 可以继续用这个 P
- C 代码运行在 M 上，**不能并行 GC**（GC 必须等 C 调用返回）
- 频繁 CGO 调用会创建大量 M，开销不小

**Q8：怎么观察调度行为？**
- `GODEBUG=schedtrace=1000` 每秒打印调度状态
- `GODEBUG=scheddetail=1` 打印每个 P/M/G 详情
- `runtime/trace` 包 + `go tool trace` 可视化

## 四、手写实现

调度器没法手写完整版，列几个**演示其核心思想**的小代码。

**1. 演示 GOMAXPROCS=1 时的协作式让出：**

```go
func main() {
    runtime.GOMAXPROCS(1)
    go func() {
        for i := 0; i < 5; i++ {
            fmt.Println("g1", i)
            runtime.Gosched()  // 让出
        }
    }()
    for i := 0; i < 5; i++ {
        fmt.Println("main", i)
        runtime.Gosched()
    }
    time.Sleep(100 * time.Millisecond)
}
// 不调 Gosched 时, GOMAXPROCS=1 会让 main 把全部 5 个跑完才轮到 g1
// (因为没有抢占点, 直到 main 退出循环)
```

**2. 简化的 work-stealing 队列（思想演示）：**

```go
type WorkQueue[T any] struct {
    mu    sync.Mutex
    items []T
}

func (q *WorkQueue[T]) PushHead(v T) {
    q.mu.Lock()
    q.items = append([]T{v}, q.items...)  // 真实 runq 是环形,这里简化
    q.mu.Unlock()
}

func (q *WorkQueue[T]) PopHead() (v T, ok bool) {
    q.mu.Lock()
    defer q.mu.Unlock()
    if len(q.items) == 0 { return }
    v, q.items = q.items[0], q.items[1:]
    return v, true
}

// 偷: 从尾部拿, 减少与本地 PopHead 的冲突
func (q *WorkQueue[T]) StealTail() (v T, ok bool) {
    q.mu.Lock()
    defer q.mu.Unlock()
    n := len(q.items)
    if n == 0 { return }
    v = q.items[n-1]
    q.items = q.items[:n-1]
    return v, true
}
```

## 五、踩坑与最佳实践

### 坑 1：容器里 GOMAXPROCS 错配

8 核物理机上的 1 核容器跑 Go 服务，runtime 默认 GOMAXPROCS=8 → 频繁切换 + GC 抖动 + 看似 CPU 没满负载但实际被 throttle。

**修复**：
- Go ≤1.24：`go.uber.org/automaxprocs` 在 `init()` 里读 cgroup
- Go ≥1.25：runtime 自带 cgroup 感知

### 坑 2：纯计算 G 卡死调度（老版本）

Go ≤1.13：

```go
go func() { for {} }()  // 老版本可能让其他 G 永久饿死
```

升级 Go 1.14+ 后基本不用担心，sysmon 能强抢。

### 坑 3：CGO 频繁调用导致 M 暴涨

每次 cgo 调用 M 与 P 解绑，并发场景下 M 数量飙升。监控 `runtime.NumThread()`，如果远大于 GOMAXPROCS 要警惕。

**优化**：
- 减少 cgo 调用频率（批量化）
- 设 `GOMAXPROCS` 与 cgo 并发数匹配
- 极端情况评估是否值得保留 CGO

### 坑 4：依赖 G 调度顺序

```go
go A()
go B()
// 假设 A 先跑 → 错!
```

调度顺序由 P 的 runq + steal + netpoll 决定，**没有任何顺序保证**。需要顺序就用 channel/锁。

### 坑 5：长时间 syscall 没准备好资源

阻塞 syscall 时 P 移交，但 M 不会自动销毁。如果你有上万个 g 同时跑阻塞 syscall（比如老式同步 DNS），会瞬间起几千个 M，超过 `debug.SetMaxThreads`（默认 10000）就 crash。

### 最佳实践

- 默认信任 runtime，不要乱调 GOMAXPROCS
- 容器场景必上 automaxprocs（或升级到 1.25+）
- 关注 `GODEBUG=schedtrace` 输出，观察 runqueue 长度和 idleprocs
- CPU 密集型用 `runtime.LockOSThread` + 调小 GOMAXPROCS 把热点固定到核
- 性能问题先看 trace（`go tool trace`），调度问题非常直观
