# channel

> CSP 通信原语：环形缓冲队列 + 两个等待队列 + 互斥锁；"不要通过共享内存来通信，要通过通信来共享内存"

## 一、核心原理

### 1.1 底层结构

```go
// runtime/chan.go
type hchan struct {
    qcount   uint           // 当前队列中元素数量
    dataqsiz uint           // 环形缓冲区大小(make 时指定的 cap)
    buf      unsafe.Pointer // 指向环形缓冲区
    elemsize uint16
    closed   uint32         // 是否已关闭
    elemtype *_type
    sendx    uint           // 下一个写入位置
    recvx    uint           // 下一个读取位置
    recvq    waitq          // 等待接收的 goroutine 队列(sudog 链表)
    sendq    waitq          // 等待发送的 goroutine 队列
    lock     mutex          // 保护以上所有字段
}
```

- **无缓冲 channel**：`dataqsiz=0`，发送必须等到有接收方
- **有缓冲 channel**：环形队列，`sendx` 和 `recvx` 模 `dataqsiz` 移动

### 1.2 发送流程 (`ch <- v`)

加锁后按以下顺序判断：

1. **chan 已关闭** → panic("send on closed channel")
2. **recvq 有等待者** → 直接把 v 拷给等待方的栈，唤醒它（绕过 buf，零拷贝优化）
3. **buf 未满** → 写入 buf[sendx]，sendx++，qcount++
4. **buf 满 / 无缓冲且无接收方** → 当前 g 包装成 sudog 入 sendq，gopark 挂起，让出 P

### 1.3 接收流程 (`v := <-ch`)

加锁后：

1. **chan 已关闭 且 buf 空** → 返回零值，ok=false
2. **sendq 有等待者**：
   - 无缓冲：直接从对方栈拷数据，唤醒发送方
   - 有缓冲：从 buf[recvx] 取数据，把发送方的数据写到 buf 末尾，唤醒发送方
3. **buf 非空** → 从 buf[recvx] 取，recvx++
4. **buf 空** → 当前 g 入 recvq，gopark 挂起

### 1.4 关闭 (`close(ch)`)

1. close 已关闭的 chan → panic
2. close nil chan → panic
3. 设置 `closed = 1`
4. 唤醒所有 recvq 中的 goroutine（它们收到零值，ok=false）
5. 唤醒所有 sendq 中的 goroutine（它们 panic("send on closed channel")）

### 1.5 select 与 channel

select 多路 case 时：
1. 随机化 case 顺序（防饥饿）
2. 加锁所有相关 chan，依次检查可立即执行的 case
3. 无可立即执行 → 把当前 g 挂到所有 chan 的等待队列，任一被唤醒后再清理其他队列

## 二、八股速记

- 底层 = **环形缓冲 + sendq + recvq + lock**
- 收发优先**直接 g 到 g 拷贝**，绕过 buf
- 关闭 chan：**唤醒所有等待方**，发送方 panic、接收方拿零值
- 三类 panic：
  - 关闭 nil chan
  - 关闭已关闭 chan
  - 向已关闭 chan 发送
- nil chan 收发**永久阻塞**（在 select 中可用作"屏蔽某个 case"）
- 接收**两返回值** `v, ok := <-ch`，ok=false 表示已关闭且 buf 空
- 无缓冲是同步的（rendezvous），有缓冲是异步队列
- 容量满时发送阻塞，容量空时接收阻塞
- range chan 直到 chan 关闭才退出

## 三、面试真题

**Q1：channel 发送到关闭 chan 会发生什么？**
panic("send on closed channel")。**实践原则：让发送方关闭，不要让接收方关闭**；多发送方场景需要额外协调（如 done chan + sync.Once）。

**Q2：从关闭的 channel 读取会怎样？**
1. buf 非空：依然能读出剩余数据，ok=true
2. buf 空：立即返回零值，ok=false（不阻塞）
所以 `for v := range ch` 在 close 后会读完剩余数据再退出。

**Q3：无缓冲 channel 的本质？**
**同步点**（rendezvous）。发送和接收必须同时就位，否则一方阻塞等另一方。常用于：(1) goroutine 间精确同步；(2) 无锁的所有权移交（数据所有权随 channel 传递）。

**Q4：channel 适合什么场景，不适合什么？**
适合：goroutine 间数据流、生产者消费者、扇出扇入、信号通知（done chan）、限流（带缓冲的 chan 当令牌桶）。
不适合：高频读写共享状态（用 mutex 更快）、需要查询当前状态（chan 没有 peek）。

**Q5：select 的 case 选择顺序是？**
随机化。多个 case 同时可执行时，runtime 用伪随机选一个，避免某个 case 始终被优先。

**Q6：`for-select` 退出方式？**
```go
for {
    select {
    case <-ctx.Done():
        return
    case msg := <-ch:
        handle(msg)
    }
}
```
统一用 ctx 或 done chan 通知退出，不要依赖 close 数据 chan。

**Q7：怎么避免 goroutine 因 channel 阻塞泄漏？**
- 发送方用 `select + default`（non-blocking send）
- 加 `ctx` 控制超时
- 确保接收方在 sender 退出前不退出（或用 buffered chan 避免最后一个发送方阻塞）

## 四、手写实现

**1. 用 channel 实现信号量（限制并发）：**

```go
type Semaphore chan struct{}

func NewSemaphore(n int) Semaphore {
    return make(chan struct{}, n)
}

func (s Semaphore) Acquire() { s <- struct{}{} }
func (s Semaphore) Release() { <-s }

// 使用
sem := NewSemaphore(10)
for _, task := range tasks {
    sem.Acquire()
    go func(t Task) {
        defer sem.Release()
        t.Run()
    }(task)
}
```

**2. 生产者消费者（带优雅关闭）：**

```go
func pipeline(ctx context.Context) {
    out := make(chan int, 16)
    var wg sync.WaitGroup

    // 生产者
    wg.Add(1)
    go func() {
        defer wg.Done()
        defer close(out)
        for i := 0; ; i++ {
            select {
            case <-ctx.Done():
                return
            case out <- i:
            }
        }
    }()

    // 消费者(可多个)
    for v := range out {
        fmt.Println(v)
    }
    wg.Wait()
}
```

**3. 扇入（多路合并）：**

```go
func merge[T any](chans ...<-chan T) <-chan T {
    out := make(chan T)
    var wg sync.WaitGroup
    wg.Add(len(chans))
    for _, c := range chans {
        go func(c <-chan T) {
            defer wg.Done()
            for v := range c {
                out <- v
            }
        }(c)
    }
    go func() { wg.Wait(); close(out) }()
    return out
}
```

**4. 超时控制：**

```go
select {
case res := <-doSomething():
    return res
case <-time.After(2 * time.Second):
    return ErrTimeout
}
```

> ⚠️ `time.After` 在 select 重复调用时会泄漏 timer，长循环里要用 `time.NewTimer` + 显式 Stop。

## 五、踩坑与最佳实践

### 坑 1：多发送方关闭 chan 导致 panic

```go
// 错误: 任何一个发送方退出时 close 都会让其他发送方 panic
go func() { ch <- 1; close(ch) }()
go func() { ch <- 2; close(ch) }()  // 可能 panic
```

**修复**：用 `sync.Once` 包装 close，或用 done chan 通知所有发送方退出，由协调者关闭。

### 坑 2：`for-select` 中 `time.After` 内存泄漏

```go
for {
    select {
    case <-ch:
    case <-time.After(time.Second): // 每轮新建 timer,旧 timer 直到触发才回收
    }
}
```

**修复**：

```go
t := time.NewTimer(time.Second)
defer t.Stop()
for {
    select {
    case <-ch:
        if !t.Stop() { <-t.C }
        t.Reset(time.Second)
    case <-t.C:
        t.Reset(time.Second)
    }
}
```

### 坑 3：无缓冲 chan 发送方先退出导致死锁

```go
ch := make(chan int)
go func() { ch <- 1 }()
// 主 g 还没开始读,但子 g 已退出?
// 不会泄漏(子 g 会阻塞在 send 直到主 g 接收)
// 但如果主 g 提前 return,子 g 永久阻塞 → 泄漏
```

### 坑 4：用 channel 取代 mutex 保护状态

简单计数器用 atomic 比 channel 快几十倍。channel 不是万能锁，是通信原语。

### 坑 5：close 接收方维护的 chan

接收方关闭 chan 后，发送方写入会 panic。原则：**永远由发送方关闭**。

### 最佳实践

- **谁创建谁关闭**，且通常是发送方关闭
- 信号通知用 `chan struct{}`（零内存）
- 没有缓冲需求别加缓冲，避免掩盖同步 bug
- chan + ctx 是 goroutine 控制的标准组合
- 用 `select { case x: ... default: ... }` 实现 non-blocking 操作
- nil chan 屏蔽 select case 是个 trick（动态启用/禁用某路）
