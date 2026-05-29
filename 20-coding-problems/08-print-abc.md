# 3 个 goroutine 交替打印 ABC

> **题目**:开 3 个 goroutine,分别只打印 A、B、C,要求输出 **`ABCABCABC...`**,共打印 N 轮。
>
> 这是并发"热身题",看似简单,但**考点密度极高**:同步原语选择、信号传递方向、推广到 N 个的通用性、避免死锁、避免最后一轮卡死。

---

## 一、考点拆解

| 考点 | 不及格 | 合格 | 优秀 |
| --- | --- | --- | --- |
| 思路 | for 加 Sleep | 用同步原语 | **能讲 3 种以上**写法并比较 |
| 信号传递 | 全员抢一个 channel | **明确"传棒"方向** | 推广到 N 个 |
| 终止条件 | 多打了 / 死锁 | 最后一轮干净退出 | **避免漏 Done / 重复 close** |
| 同步原语 | 只会 channel | channel / Mutex+Cond / atomic | **三者性能与场景对比** |
| 推广 | 写死 3 个 | 改成 N 个 | 性能优化 / 公平性 |

> **关键认知**:"交替打印"的本质是**让 N 个 goroutine 按固定顺序轮流醒来**。**信号必须有方向**(A 醒后传给 B,B 传给 C,C 传回 A),不能让大家抢同一个信号。

---

## 二、思路推导

### 2.1 错误写法(典型陷阱)

```go
// ❌ 全员监听同一个 channel,抢到谁就谁打
sig := make(chan int)
go func() { for range sig { fmt.Print("A") } }()
go func() { for range sig { fmt.Print("B") } }()
go func() { for range sig { fmt.Print("C") } }()
for i := 0; i < 9; i++ { sig <- 1 }
```

输出可能是 `ACBABCBAC` —— **顺序不可控**,Go runtime 调度哪个 G 抢到都可能。

### 2.2 正确思路:三个有方向的信号

```text
A 拿到 a 信号 → 打 A → 给 B 发 b 信号
B 拿到 b 信号 → 打 B → 给 C 发 c 信号
C 拿到 c 信号 → 打 C → 给 A 发 a 信号
```

**形成环**:每个 goroutine 等自己的信号,打完后唤醒下家。

---

## 三、解法 1:三个 channel 串成环(推荐)

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    const rounds = 5
    a := make(chan struct{}, 1)
    b := make(chan struct{}, 1)
    c := make(chan struct{}, 1)
    var wg sync.WaitGroup
    wg.Add(3)

    go printer("A", a, b, &wg, rounds)
    go printer("B", b, c, &wg, rounds)
    go printer("C", c, a, &wg, rounds)

    a <- struct{}{} // 启动:先让 A 跑
    wg.Wait()
}

func printer(name string, in, out chan struct{}, wg *sync.WaitGroup, rounds int) {
    defer wg.Done()
    for i := 0; i < rounds; i++ {
        <-in
        fmt.Print(name)
        out <- struct{}{}
    }
}
```

**为什么 channel 容量是 1**:
- C 打完最后一轮要往 `a` 写 → 但 A 已经退出了 → **若 a 无缓冲会卡死**
- 容量 1 让最后一次 send **不阻塞**,gorotine 干净退出

**讲解时主动点出**:
- 三个 goroutine 用**同一份代码**,只是参数不同 → 推广到 N 个非常容易
- 启动用 `a <- struct{}{}` 投递第一个"令牌"
- WaitGroup 等三个都退出,主线程才能结束

---

## 四、解法 2:一个 channel + 轮次判断

```go
func main() {
    const rounds = 5
    ch := make(chan int, 1)
    var wg sync.WaitGroup
    wg.Add(3)

    print := func(name string, my, total int) {
        defer wg.Done()
        for {
            v := <-ch
            if v >= total*3 {
                ch <- v // 让别人也能退出
                return
            }
            if v%3 == my {
                fmt.Print(name)
                ch <- v + 1
            } else {
                ch <- v // 不是我的轮次,放回去
            }
        }
    }

    go print("A", 0, rounds)
    go print("B", 1, rounds)
    go print("C", 2, rounds)
    ch <- 0
    wg.Wait()
}
```

**评价**:
- ✅ 容易推广到 N(改 my, total)
- ❌ **CPU 浪费**:不是自己轮次时频繁"读了又写回",高频自旋
- ❌ 公平性差,运气好坏决定唤醒顺序

> 这种写法**不推荐**,但面试官有时会问"如果只能用一个 channel 怎么做",答这个并主动说**"性能不如多 channel 环"**。

---

## 五、解法 3:Mutex + Cond + 共享 turn 变量

```go
type Printer struct {
    mu    sync.Mutex
    cond  *sync.Cond
    turn  int // 0=A, 1=B, 2=C
    done  int // 已打印总数
    total int
}

func (p *Printer) Print(name string, my int) {
    for {
        p.mu.Lock()
        for p.turn != my && p.done < p.total*3 {
            p.cond.Wait()
        }
        if p.done >= p.total*3 {
            p.mu.Unlock()
            p.cond.Broadcast() // 让其他 G 也能退出
            return
        }
        fmt.Print(name)
        p.done++
        p.turn = (p.turn + 1) % 3
        p.cond.Broadcast() // 唤醒下一个
        p.mu.Unlock()
    }
}

func main() {
    p := &Printer{total: 5}
    p.cond = sync.NewCond(&p.mu)
    var wg sync.WaitGroup
    wg.Add(3)
    go func() { defer wg.Done(); p.Print("A", 0) }()
    go func() { defer wg.Done(); p.Print("B", 1) }()
    go func() { defer wg.Done(); p.Print("C", 2) }()
    wg.Wait()
}
```

**资深点**:
- `for p.turn != my` 必须 for(防虚假唤醒,见 [01-blocking-queue.md](01-blocking-queue.md))
- 用 `Broadcast` 不用 `Signal`——Signal 可能唤醒到错误的 G(它会发现不是自己再睡回去)
- 退出时**再 Broadcast 一次**,防止其他 G 永久阻塞

**评价**:
- ✅ 不需要 channel,纯同步原语
- ❌ Broadcast 惊群,N 大时性能差
- ✅ 经典 Java/C++ 风格,展示对 Cond 的理解

---

## 六、解法 4:atomic + busy wait(性能极致)

```go
var turn atomic.Int32

func main() {
    const rounds = 5
    var wg sync.WaitGroup
    wg.Add(3)

    print := func(name string, my int32) {
        defer wg.Done()
        for i := 0; i < rounds; i++ {
            for turn.Load() != my {
                runtime.Gosched() // 让出 CPU,避免纯空转
            }
            fmt.Print(name)
            turn.Store((my + 1) % 3)
        }
    }

    go print("A", 0)
    go print("B", 1)
    go print("C", 2)
    wg.Wait()
}
```

**评价**:
- ✅ **最快**(无锁,只 CAS / Load)
- ❌ busy wait 浪费 CPU(用 `runtime.Gosched()` 缓解)
- ❌ 不适合大规模或长等待场景
- ✅ 面试加分项:**展示对 atomic 和调度器的理解**

> 实战不用这种,但**面试官如果问"无锁怎么写"**,直接拿这个答案。

---

## 七、推广到 N 个 goroutine

最自然的推广是**解法 1 的环**:

```go
func main() {
    const N = 5         // 5 个 goroutine
    const rounds = 3    // 每个打 3 轮
    chans := make([]chan struct{}, N)
    for i := range chans {
        chans[i] = make(chan struct{}, 1)
    }
    var wg sync.WaitGroup
    wg.Add(N)

    for i := 0; i < N; i++ {
        i := i
        go func() {
            defer wg.Done()
            name := fmt.Sprintf("G%d", i)
            in := chans[i]
            out := chans[(i+1)%N]
            for r := 0; r < rounds; r++ {
                <-in
                fmt.Print(name)
                out <- struct{}{}
            }
        }()
    }
    chans[0] <- struct{}{} // 启动
    wg.Wait()
}
```

环形结构 + 单令牌 = **可推广性最强**。资深面试主动给这个写法 +1。

---

## 八、四种解法对比

| 解法 | 实现复杂度 | 性能 | 可推广性 | 适用 |
| --- | --- | --- | --- | --- |
| **3 channel 环**(推荐) | 低 | 中(channel 调度) | 高 | **首选** |
| 1 channel + 轮次判断 | 低 | **差**(自旋读写) | 中 | 当被限制只能 1 channel |
| Mutex + Cond | 中 | 中(惊群)| 中 | 没 channel 限制时备用 |
| atomic + Gosched | 中 | **最快**(但烧 CPU)| 高 | 极短等待,展示无锁功底 |

---

## 九、典型坑

| 坑 | 现象 | 修复 |
| --- | --- | --- |
| 三个 channel 都无缓冲 | 最后一轮 C → a 写阻塞 → 死锁 | 容量给 1 |
| 全员监听同一 channel | 输出顺序乱 | 信号要有方向 |
| Mutex 版用 `if` 不用 `for` | 虚假唤醒打错顺序 | 必须 for |
| Mutex 版用 Signal 不 Broadcast | 唤醒到错误 G,陷入死等 | 用 Broadcast |
| WaitGroup 忘 Add / Done | 主线程提前退出 / 死锁 | Add 在 go 之前,Done 在 defer |
| 退出时没 Broadcast / 没让别人退 | 其他 G 永久阻塞 | 退出前广播一次 |
| atomic 版没 `Gosched` | CPU 100% 空转 | 加 `runtime.Gosched()` |
| 想用 `select` 做"多入" | 复杂化没必要 | 每 G 只听一个 in chan |

---

## 十、面试追问拓展

### 10.1 推广题:N 个 goroutine 交替打印 1~M

> "1 号打 1,2 号打 2,...,N 号打 N,然后 1 号打 N+1,...,直到 M"

直接用**第七节的环**,把 `name` 换成动态 counter:

```go
counter := atomic.Int64{} // 全局共享
print := func(in, out chan struct{}, max int64) {
    for {
        <-in
        v := counter.Add(1)
        if v > max {
            close(out) // 通知下家也退出
            return
        }
        fmt.Print(v, " ")
        out <- struct{}{}
    }
}
```

### 10.2 变体:奇数偶数交替打印

两个 goroutine,A 只打奇数,B 只打偶数,用 2 channel 环即可。

### 10.3 死锁分析

> "如果只用 2 个 channel(a, b),分别给 A→B、B→C 用,C 怎么通知 A?"

答:需要第 3 个 channel(c)闭环,**否则永远死锁**——这是"3 个互相依赖 → 必须有 3 条边"的有向图问题。

---

## 十一、现场表达模板

> "这道题的核心是**让 3 个 goroutine 按固定顺序醒来**。**新手陷阱**是让大家抢同一个 channel,这样调度顺序不可控。
>
> **正解**是**有方向的信号传递**——3 个 channel 串成环:A 拿到 a 信号 → 打 A → 给 B 发 b 信号 → B 打 → 给 C 发 c → C 打 → 回 A,循环。
>
> 关键细节:
> 1. **channel 容量必须 1**,否则最后一轮 C → A 写阻塞死锁
> 2. **WaitGroup** 让主线程等三个都退出
> 3. 启动时**手动投第一个令牌**到 a
>
> 推广到 N 个就是把 3 个 channel 换成 N 个 channel 的环。
>
> 如果限制只能用一个 channel,可以用'共享 turn 变量 + 不是自己就放回去',但这是自旋,性能差;
> 如果不能用 channel,可以 Mutex + Cond + turn 变量,但要用 Broadcast 防止虚假唤醒拉错 G;
> 想极致性能用 atomic + runtime.Gosched(),展示无锁功底但实战别用。"

---

## 十二、模块化封装版(工程化)

> 上面解法 1/3/4 是"白板能写出"的版本。**实战 / 二面追问**时,面试官会让你"把它抽成可复用的库"——这一节给三种方案的**结构体封装版**,每种都是一个独立的小型并发组件。

### 12.1 通用接口

```go
// Runner 抽象"N 个 worker 按固定顺序轮流执行 action"
type Runner interface {
    Run(ctx context.Context) error
}

// Config 三种方案通用的参数
type Config struct {
    N      int                    // worker 数量
    Rounds int                    // 每个 worker 跑多少轮
    Action func(id int, round int) // 轮到 id 时执行
}
```

### 12.2 方案 A:channel 环(ChannelRing)

```go
package alternate

import (
    "context"
    "fmt"
    "sync"
)

type ChannelRing struct {
    cfg  Config
    chs  []chan struct{}
}

func NewChannelRing(cfg Config) (*ChannelRing, error) {
    if cfg.N <= 0 || cfg.Rounds <= 0 || cfg.Action == nil {
        return nil, fmt.Errorf("invalid config")
    }
    chs := make([]chan struct{}, cfg.N)
    for i := range chs {
        chs[i] = make(chan struct{}, 1) // 容量 1 防最后一轮死锁
    }
    return &ChannelRing{cfg: cfg, chs: chs}, nil
}

func (r *ChannelRing) Run(ctx context.Context) error {
    var wg sync.WaitGroup
    wg.Add(r.cfg.N)
    errCh := make(chan error, 1)

    for i := 0; i < r.cfg.N; i++ {
        go r.worker(ctx, i, &wg, errCh)
    }

    r.chs[0] <- struct{}{} // 启动令牌

    done := make(chan struct{})
    go func() { wg.Wait(); close(done) }()

    select {
    case <-done:
        return nil
    case err := <-errCh:
        return err
    case <-ctx.Done():
        return ctx.Err()
    }
}

func (r *ChannelRing) worker(ctx context.Context, id int, wg *sync.WaitGroup, errCh chan<- error) {
    defer wg.Done()
    in := r.chs[id]
    out := r.chs[(id+1)%r.cfg.N]
    for round := 0; round < r.cfg.Rounds; round++ {
        select {
        case <-in:
        case <-ctx.Done():
            return
        }
        r.cfg.Action(id, round)
        select {
        case out <- struct{}{}:
        case <-ctx.Done():
            return
        }
    }
}
```

**模块化亮点**:
- `Config` 把可变参数封起来,**Action 由调用方注入**(打印 / 写日志 / 发请求都行)
- **支持 context 取消**,worker select 同时监听 in 和 ctx.Done
- 错误通过 errCh 单向回传,**不污染 worker 签名**
- channel 容量 1 不变,保证收尾不死锁

### 12.3 方案 B:Mutex + Cond(CondPrinter)

```go
type CondPrinter struct {
    cfg  Config
    mu   sync.Mutex
    cond *sync.Cond
    turn int
}

func NewCondPrinter(cfg Config) (*CondPrinter, error) {
    if cfg.N <= 0 || cfg.Rounds <= 0 || cfg.Action == nil {
        return nil, fmt.Errorf("invalid config")
    }
    p := &CondPrinter{cfg: cfg}
    p.cond = sync.NewCond(&p.mu)
    return p, nil
}

func (p *CondPrinter) Run(ctx context.Context) error {
    var wg sync.WaitGroup
    wg.Add(p.cfg.N)

    // 监听 ctx 取消时唤醒所有 waiter
    stop := make(chan struct{})
    go func() {
        select {
        case <-ctx.Done():
            p.mu.Lock()
            p.cond.Broadcast()
            p.mu.Unlock()
        case <-stop:
        }
    }()

    for i := 0; i < p.cfg.N; i++ {
        go p.worker(ctx, i, &wg)
    }

    wg.Wait()
    close(stop)
    return ctx.Err()
}

func (p *CondPrinter) worker(ctx context.Context, id int, wg *sync.WaitGroup) {
    defer wg.Done()
    for round := 0; round < p.cfg.Rounds; round++ {
        p.mu.Lock()
        for p.turn != id && ctx.Err() == nil {
            p.cond.Wait()
        }
        if ctx.Err() != nil {
            p.mu.Unlock()
            return
        }
        p.cfg.Action(id, round)
        p.turn = (p.turn + 1) % p.cfg.N
        p.cond.Broadcast()
        p.mu.Unlock()
    }
}
```

**模块化亮点**:
- **ctx 取消时主动 Broadcast**——Cond 本身不支持超时,需要外部触发唤醒(这是 Cond + ctx 的标准套路)
- `for p.turn != id && ctx.Err() == nil` 双重退出条件
- Broadcast 而非 Signal:Signal 可能唤错 G 导致死等

### 12.4 方案 C:atomic + Gosched(AtomicPrinter)

```go
import (
    "context"
    "runtime"
    "sync"
    "sync/atomic"
)

type AtomicPrinter struct {
    cfg  Config
    turn atomic.Int32
}

func NewAtomicPrinter(cfg Config) (*AtomicPrinter, error) {
    if cfg.N <= 0 || cfg.Rounds <= 0 || cfg.Action == nil {
        return nil, fmt.Errorf("invalid config")
    }
    return &AtomicPrinter{cfg: cfg}, nil
}

func (p *AtomicPrinter) Run(ctx context.Context) error {
    var wg sync.WaitGroup
    wg.Add(p.cfg.N)
    for i := 0; i < p.cfg.N; i++ {
        go p.worker(ctx, i, &wg)
    }
    wg.Wait()
    return ctx.Err()
}

func (p *AtomicPrinter) worker(ctx context.Context, id int, wg *sync.WaitGroup) {
    defer wg.Done()
    my := int32(id)
    next := int32((id + 1) % p.cfg.N)
    for round := 0; round < p.cfg.Rounds; round++ {
        for p.turn.Load() != my {
            if ctx.Err() != nil {
                return
            }
            runtime.Gosched() // 让出 P,避免空转烧 CPU
        }
        p.cfg.Action(id, round)
        p.turn.Store(next)
    }
}
```

**模块化亮点**:
- 全程**无锁**,只 atomic Load/Store
- **busy-wait 内嵌 ctx 检查**——比 channel/Cond 的取消更及时(每次 spin 都查)
- `Gosched()` 是关键,否则 P 被占满其他 G 饿死

### 12.5 统一调用方式

```go
func main() {
    cfg := Config{
        N:      3,
        Rounds: 5,
        Action: func(id, round int) {
            fmt.Printf("%c", 'A'+rune(id))
        },
    }

    runners := []struct {
        name string
        new  func(Config) (Runner, error)
    }{
        {"ChannelRing", func(c Config) (Runner, error) { return NewChannelRing(c) }},
        {"CondPrinter", func(c Config) (Runner, error) { return NewCondPrinter(c) }},
        {"AtomicPrinter", func(c Config) (Runner, error) { return NewAtomicPrinter(c) }},
    }

    for _, r := range runners {
        runner, err := r.new(cfg)
        if err != nil {
            log.Fatal(err)
        }
        ctx, cancel := context.WithTimeout(context.Background(), time.Second)
        fmt.Printf("\n=== %s ===\n", r.name)
        if err := runner.Run(ctx); err != nil {
            fmt.Printf("err: %v\n", err)
        }
        cancel()
    }
}
```

**调用方完全不关心底层是 channel / Cond / atomic**——这就是**依赖倒置**的价值:测试时可以 mock Runner,业务可以按场景换实现。

### 12.6 三种封装对比

| 维度 | ChannelRing | CondPrinter | AtomicPrinter |
| --- | --- | --- | --- |
| 同步原语 | channel 环 | Mutex + Cond | atomic.Int32 |
| 内存占用 | N 个 channel | 1 个 Cond | 1 个 int32 |
| ctx 取消响应 | **快**(select 直接收) | **慢**(需外部 Broadcast 唤醒) | **快**(每 spin 查) |
| CPU 占用 | 低(阻塞挂起) | 低(阻塞挂起) | **高**(busy-wait,Gosched 缓解) |
| 可推广 N | ✅ 自然推广 | ✅ 自然推广 | ✅ 自然推广 |
| 适用 | **首选**,工程级默认 | 无 channel 限制时 | 极短切换 / 性能敏感 |

### 12.7 模块化收益

| 收益 | 体现 |
| --- | --- |
| **接口统一** | 三种实现都实现 `Runner`,调用方零成本切换 |
| **可测试** | Action 注入,**可断言"id=0 时调用一次,id=1 时调用一次..."** |
| **可观测** | 在 Runner 内加 metrics(每 worker 等待时长 / 切换次数) |
| **可扩展** | 加超时 / 加优先级 / 加暂停-恢复都在结构体内部完成,不破坏接口 |
| **隔离 bug** | Cond 版的虚假唤醒、atomic 版的烧 CPU,都被封在各自实现里,调用方看不到 |

> 资深表达:"白板上写裸 channel 演示思路;**进入生产代码,一定抽成 Runner 接口 + Config 注入 Action**——三种底层实现可以按场景换,业务层完全无感。"

---

## 十三、一句话总结

> **N 个 goroutine 交替执行 = N 个 channel 串成环 + 单个令牌循环传递**;
>
> - **信号必须有方向**(每 G 监听自己的 in,写到下家的 out),不能让大家抢同一个
> - **channel 容量 = 1**,防最后一轮死锁
> - **WaitGroup** 兜底主线程等所有 G 退出
> - 可推广到 N、可改 Mutex+Cond 版、可改 atomic 版,**多种写法都要会**
