# channel

> CSP 通信原语：环形缓冲队列 + 两个等待队列 + 互斥锁；"不要通过共享内存来通信，要通过通信来共享内存"

## 〇、核心提炼（5 段式）

### 核心机制（4 条必背）

1. **hchan 结构** - 环形缓冲数组（buf）+ 两个等待队列（sendq / recvq）+ 互斥锁（lock）
2. **三种 channel** - 无缓冲（同步交接 sender ↔ receiver）/ 有缓冲（buf 未满写入 / 未空读取）/ nil（永久阻塞）
3. **优雅关闭原则** - **只有 sender 能 close**，**多 sender 用 sync.Once 或独立 done channel**
4. **select 多路复用** - 随机选择就绪 case，default 非阻塞，结合 timer / context 做超时

### 核心本质（必懂）

> channel 的本质是 **"goroutine 间的同步队列 + 调度配合"**：
>
> - **不是普通队列**：是 goroutine 调度器深度集成的同步原语
> - **同步交接**：无缓冲 channel 是 sender goroutine 直接把数据**复制**到 receiver 的栈（不走 buf）
> - **阻塞用 G 挂起**：满 / 空时 sender / receiver 挂到 sendq / recvq，被对方唤醒
>
> **关键事实**：
> - **channel 是有锁的**（hchan.lock）：高频场景仍是性能瓶颈，原子操作（atomic）更快
> - **关闭已关闭的 channel 会 panic**：必须明确"由谁关闭"
> - **向已关闭的 channel 发送会 panic**：但接收仍可读到剩余数据 + 零值
> - **nil channel 永久阻塞**：用于动态关闭某个 select case

### 完整流程（面试必背）

```
hchan 结构（runtime/chan.go）:
  type hchan struct {
      qcount   uint           // buf 中元素数
      dataqsiz uint           // buf 容量（make 时定）
      buf      unsafe.Pointer // 环形缓冲数组指针
      elemsize uint16         // 元素大小
      closed   uint32         // 是否已关闭
      sendx    uint           // 发送索引（环形位置）
      recvx    uint           // 接收索引
      recvq    waitq          // 阻塞的 receiver 队列
      sendq    waitq          // 阻塞的 sender 队列
      lock     mutex
  }

发送 ch <- v 流程:
  1. lock(&ch.lock)
  2. 如果 ch.closed → panic
  3. 如果 recvq 非空（有 receiver 在等）:
     - 直接把 v 复制到 receiver 的栈
     - 唤醒该 receiver
     - unlock + 返回
  4. 如果 buf 未满（len < cap）:
     - 把 v 写入 buf[sendx]
     - sendx++, qcount++
     - unlock + 返回
  5. buf 满了（或无缓冲）:
     - 当前 G 加入 sendq
     - gopark 挂起 G（让出 M）
     - 被 receiver 唤醒后继续

接收 v := <-ch 流程（对称）:
  1. lock
  2. sendq 非空 + buf 满 → 取 buf[recvx]，从 sendq 复制数据进 buf，唤醒 sender
  3. buf 非空 → 取 buf[recvx]，recvx++, qcount--
  4. buf 空 + sendq 空 + 已关闭 → 返回零值 + ok=false
  5. buf 空 + sendq 空 + 未关闭 → 当前 G 加入 recvq → gopark

无缓冲 channel 的同步交接:
  sender 来时无 receiver → 挂起到 sendq
  receiver 来时找到 sendq 中的 sender:
    - 直接复制 sender 栈中的数据到 receiver 栈
    - 唤醒 sender
    - 没有经过 buf！

select 流程（多路复用）:
  1. 收集所有 case 的 channel
  2. 按地址排序加锁（防死锁）
  3. 检查每个 case 是否就绪
  4. 有就绪 → 随机选一个执行
  5. 都不就绪 + 有 default → 走 default
  6. 都不就绪 + 无 default:
     - G 加入所有 channel 的等待队列
     - gopark 挂起
     - 任一就绪 → 唤醒 G
```

### 4 条核心机制 - 逐点讲透

#### 1. hchan 结构

```
环形缓冲数组（buf）:
  有缓冲: 分配 cap × elemsize 字节
  无缓冲: buf = nil

  sendx / recvx 模 dataqsiz 形成环形:
    sendx 写入位置（生产者）
    recvx 读取位置（消费者）
    qcount 当前元素数

等待队列（waitq）:
  sendq: 阻塞的 sender G 链表
  recvq: 阻塞的 receiver G 链表

  每个 sudog 节点:
    - 阻塞的 G
    - 要发送/接收的数据指针

锁（mutex）:
  hchan.lock 保护所有字段
  → 每次操作必须加锁
  → 高并发是潜在瓶颈
```

#### 2. 三种 channel 行为

```
无缓冲 channel (make(chan T)):
  buf = nil, cap = 0
  sender 无 receiver → 立即阻塞
  receiver 无 sender → 立即阻塞
  → 同步交接，必须双方同时在场

有缓冲 channel (make(chan T, n)):
  buf 容量 n
  buf 未满 → 写入不阻塞
  buf 未空 → 读取不阻塞
  → 异步通信

nil channel:
  send / recv 永久阻塞
  close 会 panic

  用途:
  在 select 中动态"关闭"某个 case
  case <-doneCh: doneCh = nil  // 之后不再被选中
```

#### 3. 关闭原则（避免 panic）

```
原则 1: 只有 sender 能关闭
  receiver 不应该关闭（不知道 sender 是否还会写）
  关闭已关闭 → panic
  向已关闭写 → panic

原则 2: 多个 sender 时的关闭
  方案 A: 引入"总关闭信号" channel
    done := make(chan struct{})
    sender:
      select {
      case ch <- v:
      case <-done: return
      }
    // 关闭：close(done) → 所有 sender 退出
    // 不直接 close(ch)

  方案 B: sync.Once 包装
    var closeOnce sync.Once
    closeOnce.Do(func() { close(ch) })

原则 3: 检测已关闭
  v, ok := <-ch
  if !ok {
      // channel 已关闭且无数据
  }

  for v := range ch {
      // 自动结束 when ch 关闭且空
  }
```

#### 4. select 多路复用

```
特性:
  - 随机选择就绪 case（防饥饿）
  - default 实现非阻塞
  - 都阻塞时 G 挂起，任一就绪即唤醒

常见模式:

1. 超时控制:
   select {
   case v := <-ch:
   case <-time.After(1 * time.Second):
       return errors.New("timeout")
   }

2. context 取消:
   select {
   case <-ctx.Done(): return ctx.Err()
   case v := <-ch: process(v)
   }

3. 非阻塞发送:
   select {
   case ch <- v:  // 成功
   default:       // 没人接，丢弃
   }

4. 多源复用:
   for {
       select {
       case v := <-ch1: process1(v)
       case v := <-ch2: process2(v)
       case <-done: return
       }
   }

性能坑:
  time.After 每次都创建新 timer + 不释放（GC 前一直存在）
  长时间循环用 time.NewTimer + timer.Reset
```

### 一句话总结

> channel 的核心是：**hchan = 环形 buf + 两个等待队列 + 互斥锁**，
> 本质是 **goroutine 间同步队列 + 调度器深度集成**：无缓冲是同步交接（sender ↔ receiver 直接复制），有缓冲是异步通信。
> **三大坑**：关闭已关闭 channel panic / 向已关闭 channel 写 panic / nil channel 永久阻塞。
> **关闭原则**：只有 sender 能关闭、多 sender 用 done 信号 / sync.Once。
> **性能**：channel 有锁，高频小数据用 atomic 更快；channel 适合"任务流水线 + 信号通知 + 多路 select"场景。

---

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

## 六、源码深读（runtime/chan.go）

### 6.1 sudog：等待队列节点

```go
// runtime/runtime2.go
type sudog struct {
    g        *g              // 被挂起的 goroutine
    next     *sudog          // 链表指针
    prev     *sudog
    elem     unsafe.Pointer  // 指向待发送 / 接收的数据
    c        *hchan          // 关联的 chan
    isSelect bool            // 是否来自 select
    success  bool            // 被唤醒时是否成功接收（close 时为 false）
    waitlink *sudog          // 用于多 sudog 场景
    ...
}

type waitq struct {
    first *sudog
    last  *sudog
}
```

**关键**：每个阻塞在 channel 上的 goroutine 会被包装成 sudog，挂到 sendq 或 recvq。sudog 复用（sync.Pool）减少分配。

### 6.2 chansend 关键路径（有删减）

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, ...) bool {
    // 1. nil chan
    if c == nil {
        if !block { return false }
        gopark(...)  // 永久阻塞
    }

    // 2. 快速路径：非阻塞 + chan 未关闭 + 无法发送
    if !block && c.closed == 0 && full(c) {
        return false
    }

    lock(&c.lock)

    // 3. 已关闭 → panic
    if c.closed != 0 {
        unlock(&c.lock)
        panic("send on closed channel")
    }

    // 4. recvq 有等待者 → 直接拷贝到对方栈（绕过 buf，零拷贝）
    if sg := c.recvq.dequeue(); sg != nil {
        send(c, sg, ep, ...)  // 把 ep 数据拷到 sg.elem
        return true
    }

    // 5. buf 未满 → 写入 buf
    if c.qcount < c.dataqsiz {
        qp := chanbuf(c, c.sendx)
        typedmemmove(c.elemtype, qp, ep)
        c.sendx++
        if c.sendx == c.dataqsiz { c.sendx = 0 }
        c.qcount++
        unlock(&c.lock)
        return true
    }

    // 6. buf 满 / 无缓冲 + 无接收方 → 阻塞
    if !block {
        unlock(&c.lock)
        return false
    }

    // 当前 g 包装成 sudog 入 sendq
    gp := getg()
    mysg := acquireSudog()
    mysg.elem = ep
    mysg.g = gp
    c.sendq.enqueue(mysg)
    gopark(chanparkcommit, ...)  // 让出 P

    // 被唤醒后：如果是被 close 唤醒，panic
    if !mysg.success {
        panic("send on closed channel")
    }
    releaseSudog(mysg)
    return true
}
```

### 6.3 chanrecv 关键路径

对称：
- 已关闭且 buf 空 → 返回零值 + ok=false（不 panic）
- sendq 有等待者：
  - 有 buf：从 buf[recvx] 取，把 sudog 数据搬到 buf 末尾
  - 无 buf：直接从 sudog.elem 拷贝
- buf 非空 → 从 buf[recvx] 取
- buf 空 → 入 recvq 阻塞

### 6.4 closechan 关键路径

```go
func closechan(c *hchan) {
    if c == nil { panic("close of nil channel") }
    lock(&c.lock)
    if c.closed != 0 {
        unlock(&c.lock)
        panic("close of closed channel")
    }
    c.closed = 1

    // 1. 唤醒所有 recvq（它们拿零值，success=false，ok=false）
    var glist gList
    for {
        sg := c.recvq.dequeue()
        if sg == nil { break }
        if sg.elem != nil {
            typedmemclr(c.elemtype, sg.elem)  // 写零值
        }
        sg.success = false
        glist.push(sg.g)
    }

    // 2. 唤醒所有 sendq（它们 panic）
    for {
        sg := c.sendq.dequeue()
        if sg == nil { break }
        sg.success = false  // 被唤醒后会 panic
        glist.push(sg.g)
    }

    unlock(&c.lock)

    // 3. 批量唤醒 goroutine
    for !glist.empty() {
        gp := glist.pop()
        goready(gp, ...)
    }
}
```

**关键认知**：
- close 不释放 hchan 内存（由 GC）
- close 会唤醒所有等待者（批量 goready 减少唤醒开销）
- 唤醒后 recvq 拿零值，sendq 触发 panic

### 6.5 select 的 selectgo（简化版）

```go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
    // 1. 两套乱序：pollOrder（轮询顺序，fastrand 打乱）+ lockOrder（加锁顺序，地址排序防死锁）

    // 2. 按 lockOrder 加锁所有相关 chan

    // 3. 第一轮：按 pollOrder 扫描，找就绪的 case 立即执行
    //    找到 → 解锁 + 返回
    //    都没就绪 + 有 default → 走 default

    // 4. 把当前 g 包装成 sudog，挂到所有相关 chan 的 sendq/recvq
    //    gopark 挂起

    // 5. 被唤醒：某个 chan 有动静
    //    清理其他 chan 的 sudog
    //    执行对应 case
}
```

**性能要点**：
- 编译器对 1 case + default 优化为 `selectnbsend/selectnbrecv`（绕过 selectgo）
- N 个 case 时间复杂度 O(N)，大多场景够用
- **地址排序加锁** 避免不同 select 间的死锁（两个 select 用了同一批 chan）

### 6.6 直接 g-to-g 拷贝（零拷贝优化）

最能体现 Go 设计的地方：

```
场景：goroutine A 发送，goroutine B 正在 recvq 等待

传统做法:
  A → buf → B（两次拷贝）

Go 做法:
  A → B.sudog.elem（一次拷贝，绕过 buf）
  同时唤醒 B

优势:
  - 减少一次拷贝
  - 不占用 buf 空间
  - 少一次加锁
```

这就是为什么无缓冲 channel 在"有等待方"场景下比 buffered 还快。

### 6.7 hchan 内存布局

```
[hchan 头部（约 96 字节）] [buf 环形缓冲区 cap × elemsize]

整体在一次分配里（减少 GC 压力）
buf 在 hchan 之后连续排列
```

`make(chan int, 100)` 分配 ≈ 96 + 100 × 8 = 896 字节，一次分配。

## 七、性能与选型

### 7.1 channel vs mutex 性能对比

**微基准**（Intel i7，Go 1.22，GOMAXPROCS=8）：

| 操作 | 耗时 | 相对 |
| --- | --- | --- |
| 原子 atomic.AddInt64 | ~1 ns | 1x |
| Mutex Lock/Unlock（无竞争） | ~20 ns | 20x |
| Mutex Lock/Unlock（高竞争） | 100-1000 ns | 100-1000x |
| Buffered chan send+recv（无阻塞） | ~50 ns | 50x |
| Unbuffered chan（有等待方） | ~200 ns | 200x |
| Unbuffered chan（需挂起唤醒） | ~1 μs+ | 1000x+ |

**结论**：
- **频繁读写共享状态用 atomic / Mutex**，不要用 channel
- **跨 goroutine 数据流 / 控制流用 channel**

### 7.2 Buffered vs Unbuffered 选择

| 场景 | 推荐 |
| --- | --- |
| **严格同步**（完成交接后才继续） | unbuffered |
| **所有权移交**（值传完就安全） | unbuffered |
| **削峰填谷**（生产消费速度差） | buffered（容量 = 预期突发） |
| **信号通知**（done / shutdown） | `chan struct{}`（buf=0 或 1） |
| **固定并发度**（worker pool） | buffered（容量 = worker 数） |
| **避免生产者阻塞** | buffered（容量足够） |

**缓冲大小的坑**：
- 过大 → 掩盖同步问题 + 内存浪费 + 积压时延迟巨大
- 过小 → 生产者频繁阻塞
- **常见错误选 1**（看似 unbuffered 替代品，实际行为不同）

### 7.3 channel vs sync.Cond

```
sync.Cond: Wait / Signal / Broadcast
  优: 零内存开销，灵活
  缺: 容易写错（必须配合 Mutex + for 循环）

channel: close / recv 天然广播
  优: 代码清晰
  缺: 单次使用（关闭后不能重开）

实战:
  一次性广播（启动信号 / 关闭信号）→ channel + close
  反复通知（条件变量）→ sync.Cond
```

### 7.4 channel vs sync.WaitGroup

```
sync.WaitGroup: Add / Done / Wait
  优: 直观，性能好
  缺: 只能等所有 goroutine 完成

channel: 每完成发一个
  优: 可以逐个收集结果
  缺: 需要知道数量

实战:
  等一组 goroutine 完成 → WaitGroup
  收集结果 → channel + WaitGroup 组合
```

### 7.5 极致性能场景：避免 channel

**高性能队列**（每秒千万级）用 channel 会瓶颈在：
- hchan.lock 争抢
- sudog 分配 / 唤醒

**替代方案**：
- Disruptor 风格 ringbuffer（sync/atomic + 填充防 False Sharing）
- 无锁队列（CAS）
- sharded channel（N 个 chan 分散争抢）

Kitex、NATS 等高性能项目大量使用自研队列替代 channel。

### 7.6 什么时候不要用 channel

```
❌ 高频读写共享计数器 → atomic
❌ 保护结构体字段 → Mutex
❌ 缓存热点数据 → sync.Map / 分段 Map
❌ 单次信号量 → sync.Once
❌ 百万级并发消息 → 自研无锁队列
❌ 简单 fire-and-forget → 直接 go func()
```

## 八、关闭协议（Channel Closing Principle）

Go 没有语法保证关闭安全，但有公认的**关闭协议**（Dave Cheney 总结）。

### 8.1 核心原则

**"Don't close a channel from the receiver side and don't close a channel if the channel has multiple concurrent senders."**

- **永远由发送方关闭**
- **多发送方场景需要额外协调**（不能随便某个 sender 关）

### 8.2 场景 1：单发送方 + 单/多接收方（简单）

```go
// 发送方关闭即可
func producer(out chan<- int) {
    defer close(out)
    for i := 0; i < 10; i++ {
        out <- i
    }
}

// 接收方
for v := range out { ... }
```

### 8.3 场景 2：多发送方 + 单接收方

**错误做法**（某 sender 关 → 其他 sender panic）：
```go
// ❌
for i := 0; i < N; i++ {
    go func() {
        ch <- work()
        close(ch)  // N 个 goroutine 抢着 close
    }()
}
```

**正确做法**：接收方用 `done chan` 通知所有发送方退出，再让**接收方关闭** done（不关 ch）：

```go
func coordinator() {
    ch := make(chan int)
    done := make(chan struct{})
    var wg sync.WaitGroup

    // N 个发送方
    for i := 0; i < N; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case <-done:
                    return
                case ch <- work():
                }
            }
        }()
    }

    // 接收方
    for v := range ch {
        if shouldStop(v) {
            close(done)  // 接收方关 done，不关 ch
            break
        }
    }
    wg.Wait()
    // 现在所有 sender 退出，可以安全 close ch（可选，不关也行）
    close(ch)
}
```

**关键**：`ch` 永远不被多方 close（要么不关，要么 wg.Wait 后单点关）。

### 8.4 场景 3：多发送方 + 多接收方

**最复杂**。经典方案：额外加一个 signal chan，用"**moderator 模式**"。

```go
type Result struct{ Val int }

func runAll(N, M int) {
    ch := make(chan Result)
    stopCh := make(chan struct{})  // 关闭信号
    var stopOnce sync.Once
    stop := func() { stopOnce.Do(func() { close(stopCh) }) }

    var wg sync.WaitGroup

    // N 个发送方
    for i := 0; i < N; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                r := Result{Val: rand.Int()}
                select {
                case <-stopCh:
                    return
                case ch <- r:
                }
            }
        }()
    }

    // M 个接收方
    var rwg sync.WaitGroup
    for i := 0; i < M; i++ {
        rwg.Add(1)
        go func() {
            defer rwg.Done()
            for {
                select {
                case <-stopCh:
                    return
                case r := <-ch:
                    if r.Val == 42 {
                        stop()  // 任何接收方都可以触发停止
                        return
                    }
                }
            }
        }()
    }

    rwg.Wait()
    stop()
    wg.Wait()
}
```

**核心**：ch 永远不 close，通过 stopCh + sync.Once 保证关闭信号只触发一次，所有方看到 stopCh 关闭即退出。

### 8.5 关闭协议总结表

| 发送方数 | 接收方数 | 谁关闭 | 特殊处理 |
| --- | --- | --- | --- |
| 1 | 1 | 发送方 | 无 |
| 1 | N | 发送方 | 无 |
| N | 1 | **不关 ch**，接收方关 done | 发送方 select done |
| N | M | **不关 ch**，用 stopCh + sync.Once | moderator 模式 |

### 8.6 检测 chan 是否关闭（不优雅）

```go
// ❌ 没有"安全"的方式检测 chan 是否关闭（除非你能读它）
// v, ok := <-ch 是唯一检测方式，但会消耗数据

// ❌ 这种 trick 是反模式：
func isClosed(ch <-chan int) bool {
    select {
    case _, ok := <-ch:
        return !ok
    default:
        return false  // 有数据或无数据都返回 false
    }
}

// ✅ 正确思路：用单独的 done chan 表达"关闭意图"
type Closable struct {
    done chan struct{}
    once sync.Once
}
func (c *Closable) Close() { c.once.Do(func() { close(c.done) }) }
func (c *Closable) Done() <-chan struct{} { return c.done }
```

这也是 `context.Context` 的设计思路。

## 九、常见陷阱深挖

### 陷阱 1：nil channel 永久阻塞

```go
var ch chan int  // nil
ch <- 1          // 永久阻塞
<-ch             // 永久阻塞
```

**妙用**：select 中动态屏蔽某路：
```go
var in, out chan int
if condition {
    in = actualCh  // 启用
} else {
    in = nil       // 屏蔽
}
select {
case v := <-in:    // in=nil 时此 case 永不被选中
    ...
case <-out:
    ...
}
```

### 陷阱 2：range 不会退出（未 close）

```go
ch := make(chan int)
go func() {
    for i := 0; i < 3; i++ { ch <- i }
    // 忘了 close(ch)
}()

for v := range ch {  // 收完 3 个后永久阻塞
    fmt.Println(v)
}
```

**修复**：range 的生产方必须 close。

### 陷阱 3：循环变量捕获（Go 1.22 前）

```go
// Go 1.22 之前
for _, ch := range chans {
    go func() {
        v := <-ch  // ch 可能是最后一个循环变量
    }()
}

// 修复（Go 1.22 之前）
for _, ch := range chans {
    ch := ch  // 遮蔽
    go func() { v := <-ch }()
}
```

Go 1.22+ 自动每轮独立变量，此坑消失。

### 陷阱 4：unbuffered chan 和 goroutine 启动时序

```go
ch := make(chan int)
go func() {
    <-ch
    fmt.Println("received")
}()
ch <- 1  // OK，发送方等接收方
// 但换个顺序:
ch := make(chan int)
ch <- 1  // ❌ 死锁（无人接收）
go func() { <-ch }()  // 永远到不了
```

**教训**：unbuffered chan 的发送必须**确保接收方已启动**。

### 陷阱 5：select 中的 case 表达式副作用

```go
select {
case ch <- compute():  // compute() 在 select 启动时就求值（即使 case 没选中）
case <-done:
}
```

**教训**：select 启动时**所有 case 的表达式都求值**一次，耗时操作要预先算好。

### 陷阱 6：goroutine 通过 chan 传递错误的指针

```go
// ❌
for i := range items {
    go func() {
        result <- &items[i]  // 可能返回同一个地址？实际不会，但容易误解
    }()
}

// ❌ 真正错的场景：传循环变量地址
for i := 0; i < N; i++ {
    x := i
    go func() {
        ch <- &x  // 每次新 x，OK
    }()
}
// 但如果 x 在外面定义：
var x int
for i := 0; i < N; i++ {
    x = i
    go func() { ch <- &x }()  // 都指向同一个 x
}
```

### 陷阱 7：关闭信号 chan 的两次 close

```go
// ❌
close(done)
close(done)  // panic

// ✅ sync.Once
var once sync.Once
once.Do(func() { close(done) })
```

### 陷阱 8：buffered chan 不等于"异步"

```go
ch := make(chan int, 1)
ch <- 1
ch <- 2  // 阻塞！buf 满了

// 误区：以为 buffer=1 可以无限发
// 正确：buffer 只是削峰，满了依然阻塞
```

### 陷阱 9：chan 作为函数返回值被 GC

```go
func worker() <-chan int {
    ch := make(chan int)
    go func() {
        defer close(ch)
        for i := 0; i < 3; i++ {
            ch <- i
        }
    }()
    return ch  // 只要外部还持有 ch，goroutine 能正常退出
}

// 陷阱: 如果调用方没读完就丢弃返回值
ch := worker()
v := <-ch  // 只读了 1 个
// 后续没人读 → 内部 goroutine 阻塞在 ch <- 1
// 直到 ch 被 GC 回收（因为外部没引用）? 实际上不会被 GC，因为 goroutine 内部还持有 ch
// → goroutine 泄漏
```

**修复**：用 ctx 通知结束。

## 十、面试加分点（进阶）

- 知道 **sudog** 是什么，channel 如何用它组织等待队列
- 理解**直接 g-to-g 拷贝**优化（绕过 buf）
- 能说清 **close 的批量唤醒**（glist + goready）
- 知道 **select 的两套 order**（pollOrder 随机 + lockOrder 地址排序防死锁）
- 知道 **关闭协议**（1:1 / 1:N / N:1 / N:M 各自的正确关闭方式）
- 知道 **channel vs mutex 性能差 10-1000x**，不要滥用 channel
- 能说出 **buffered 不等于异步**，满了照样阻塞
- 知道 **nil chan 妙用**（select 动态屏蔽 case）
- 知道**高性能场景下 channel 瓶颈**（hchan.lock + sudog 分配），替代方案（Disruptor / 自研）
- 知道 **chan 关闭不是"检测"工具**，用 context 表达意图
