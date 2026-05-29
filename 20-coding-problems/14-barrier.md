# 屏障(Barrier / CyclicBarrier)

> **题目**:实现一个屏障 `Await()`,N 个 goroutine 调用 Await 时都阻塞,**等齐 N 个**后**一起放行**;放行后**自动重置**,可以被下一轮 N 个 goroutine 再次使用(Cyclic)。
>
> 考查:**Mutex + Cond + generation 代数**、和 `WaitGroup` / `CountDownLatch` 的区别、为什么需要"代数"防误唤醒。

屏障是并行计算的经典原语:**分阶段并行**——所有 worker 完成本阶段才能进入下阶段。Java 有 `CyclicBarrier`,Go 标准库没有(社区有 `x/sync/...` 的相关探索)。

---

## 一、考点拆解

| 考点 | 不及格 | 合格 | 优秀 |
| --- | --- | --- | --- |
| 概念 | 类似 WaitGroup | **N 个互相等齐** | 区分 Latch(一次性)和 Barrier(可复用)|
| 数据结构 | Mutex + count | Mutex + Cond + count + capacity | **+ generation 代数** |
| 复用 | 不支持 | Reset 重置 | **自动 Reset**(Cyclic)|
| 代数防误唤醒 | 没想过 | 知道 spurious | **新一轮 Await 不能被旧 Broadcast 唤醒** |
| 提前退出 | 没考虑 | 一个 G 退出怎么办 | `Break` / context cancel,**全部唤醒报 BrokenBarrierError** |
| 用途 | 不知道 | 并行计算分阶段同步 | MapReduce / ML 训练 epoch 同步 |

---

## 二、屏障 vs WaitGroup vs CountDownLatch

| | `sync.WaitGroup` | `CountDownLatch`(Java)| `CyclicBarrier`(Java)|
| --- | --- | --- | --- |
| 等待方向 | **主等子**(单向) | 主等子 | **互相等**(双向) |
| 一次性 | 可复用但需谨慎 | **一次性** | **可循环** |
| 谁调 Done | **子 goroutine** | 子 | **每个 G 自己调 Await** |
| 提前结束 | ✗ | ✗ | **✓**(Break)|
| 典型用途 | 主线程等所有子完成 | 启动信号 / 并发测试 | **分阶段并行** |

**核心区别**:
- WaitGroup:**"我等你们都完成"**(主等 N 子)
- Barrier:**"我们一起到齐再一起走"**(N 个互等)

---

## 三、思路推导

### 3.1 朴素错误版

```go
// ❌ 没代数,会被旧 Broadcast 误唤醒
type Barrier struct {
    mu sync.Mutex
    cond *sync.Cond
    capacity int
    count int
}

func (b *Barrier) Await() {
    b.mu.Lock()
    defer b.mu.Unlock()
    b.count++
    if b.count == b.capacity {
        b.count = 0
        b.cond.Broadcast()
        return
    }
    b.cond.Wait()
}
```

**坑**:被 Broadcast 唤醒后,如果**新一轮**的 Await 刚好和"旧的最后一个 Wait 还没返回"穿插,旧的会被错误地认为是新一轮的成员。

虚假唤醒 + 复用场景下,**没有代数标记**根本判断不了"我等的是哪一轮"。

### 3.2 加上 generation

```text
每"凑齐一轮"算一个 generation。
Await 进来时记录自己的 generation g0,
被唤醒后判断 b.generation != g0 → 说明已经过了这一轮(新一轮)→ 安全返回
否则继续 Wait(虚假唤醒 / 没凑齐)
```

---

## 四、完整代码

```go
package barrier

import (
    "errors"
    "sync"
)

var ErrBrokenBarrier = errors.New("barrier is broken")

type Barrier struct {
    mu        sync.Mutex
    cond      *sync.Cond
    capacity  int
    count     int  // 当前轮已到的人数
    generation int // 代数,每凑齐一轮 +1
    broken    bool // 被 Break 时所有等待者都失败退出
}

func New(n int) *Barrier {
    if n <= 0 {
        panic("barrier: capacity must > 0")
    }
    b := &Barrier{capacity: n}
    b.cond = sync.NewCond(&b.mu)
    return b
}

// Await 阻塞直到凑齐 N 个;Break 后返回 ErrBrokenBarrier
func (b *Barrier) Await() error {
    b.mu.Lock()
    defer b.mu.Unlock()

    if b.broken {
        return ErrBrokenBarrier
    }

    gen := b.generation
    b.count++

    if b.count == b.capacity {
        // 我是最后一个 → 凑齐了,放行所有人
        b.count = 0
        b.generation++
        b.cond.Broadcast()
        return nil
    }

    // 还没凑齐 → 等
    for b.generation == gen && !b.broken {
        b.cond.Wait()
    }
    if b.broken {
        return ErrBrokenBarrier
    }
    return nil
}

// Break 让所有等待者立刻返回 ErrBrokenBarrier;屏障"作废"
func (b *Barrier) Break() {
    b.mu.Lock()
    defer b.mu.Unlock()
    b.broken = true
    b.cond.Broadcast()
}

// Reset 恢复一个 Broken 的屏障;只能在没有 waiter 时调用
func (b *Barrier) Reset() {
    b.mu.Lock()
    defer b.mu.Unlock()
    b.broken = false
    b.count = 0
    b.generation++ // 让残留的 Wait(理论上不该有)也分得清
}
```

**讲解时主动点出的细节**:

| 设计 | 资深点 |
| --- | --- |
| `generation` 代数 | **关键!**用代数标记"我等的是哪一轮",新轮不会唤醒到旧轮 |
| **最后一个**触发 Broadcast | 不是"等到位的人"唤醒,是"补齐的人"统一唤醒 |
| 唤醒前 `count = 0, generation++` | 状态先重置,Broadcast 后醒来的人看到新一轮已开始 |
| `for b.generation == gen` | for 不是 if,**双重防护**:虚假唤醒 + 代数判断 |
| `broken` 标记 | 让所有等待者一起失败退出,**避免一个 G 提前死掉,其他永远凑不齐** |
| Reset 慎用 | 只能在确认没 waiter 时,否则可能有遗留逻辑错乱 |

---

## 五、带回调的屏障(Java CyclicBarrier 风格)

Java `CyclicBarrier` 支持构造时传一个**回调**,在"凑齐"瞬间(放行所有人之前)由**最后一个到达的线程**执行——常用于**轮次结算**:

```go
type BarrierWithAction struct {
    *Barrier
    action func() // 凑齐时的回调,由最后一个 Await 的 G 执行
}

func NewWithAction(n int, action func()) *BarrierWithAction {
    return &BarrierWithAction{Barrier: New(n), action: action}
}

func (b *BarrierWithAction) Await() error {
    b.mu.Lock()
    defer b.mu.Unlock()
    if b.broken { return ErrBrokenBarrier }

    gen := b.generation
    b.count++

    if b.count == b.capacity {
        // 凑齐 → 先跑 action,再放行
        if b.action != nil {
            // ⚠️ action 持锁内执行,要保证它快;
            // 实战可考虑提前释放锁,但要注意状态一致
            b.action()
        }
        b.count = 0
        b.generation++
        b.cond.Broadcast()
        return nil
    }
    for b.generation == gen && !b.broken {
        b.cond.Wait()
    }
    if b.broken { return ErrBrokenBarrier }
    return nil
}
```

**用途**:每轮 epoch 结束后由最后一个 worker 汇总指标 / 切换 batch / 持久化状态。

---

## 六、测试

```go
func main() {
    const N = 3
    const Rounds = 4
    b := New(N)
    var wg sync.WaitGroup

    for i := 0; i < N; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for r := 0; r < Rounds; r++ {
                time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
                fmt.Printf("[round %d] worker %d arrived\n", r, id)
                if err := b.Await(); err != nil {
                    fmt.Printf("worker %d broken at round %d\n", id, r)
                    return
                }
                fmt.Printf("[round %d] worker %d passed\n", r, id)
            }
        }(i)
    }
    wg.Wait()
}
```

预期:**每轮 3 个 worker arrive 完才会一起 passed**,然后进入下一轮。**4 轮重复使用同一个屏障**,这就是 Cyclic 的价值。

---

## 七、典型用途

### 7.1 MapReduce 分阶段同步

```text
N 个 worker 并行做 Map → Barrier → 并行做 Shuffle → Barrier → 并行做 Reduce
```

每个阶段所有 worker 必须**齐步走**,这是 barrier 的核心场景。

### 7.2 ML 训练 epoch 同步

数据并行训练:N 张卡各自跑一个 batch → Barrier 等齐 → All-Reduce 同步梯度 → 下一个 batch。

### 7.3 并发测试(让多个 G 同时起跑)

```go
b := New(workerCount + 1)
for i := 0; i < workerCount; i++ {
    go func() {
        b.Await() // 等所有 G 准备好
        hammer() // 一起冲
    }()
}
b.Await() // 主线程触发起跑
```

比 `time.Sleep + 起跑` 更精准的"同时刻"。

---

## 八、典型坑

| 坑 | 现象 | 修复 |
| --- | --- | --- |
| **没有代数**(`if b.count==N broadcast`)| 新轮被旧 Broadcast 串扰 | 必须有 generation |
| 等待用 `if` 不是 `for` | 虚假唤醒打乱节奏 | for + 代数双判断 |
| Reset 时有 waiter | 残留状态不一致 | 只在无 waiter 时 Reset(或用 generation++) |
| 一个 worker panic 后没 Break | 其他 worker 永远凑不齐 | **`defer recover() → b.Break()`** |
| capacity != N(实际 G 数)| 永远凑不齐 | 严格匹配 |
| action 持锁太久 | 阻塞所有 G | action 只放轻量逻辑 |
| 当 WaitGroup 用 | 语义不对(主等子)| 主等子用 WaitGroup |
| 上下文取消没集成 | ctx 取消时屏障还在等 | 集成 `AwaitCtx`(下面) |

---

## 九、带 context 的扩展

```go
func (b *Barrier) AwaitCtx(ctx context.Context) error {
    // 用一个 done channel 模拟 Cond.WaitTimeout(Go 没有,见 10-delay-queue.md)
    done := make(chan struct{})
    var err error
    go func() {
        err = b.Await()
        close(done)
    }()
    select {
    case <-done:
        return err
    case <-ctx.Done():
        b.Break() // 取消整个屏障
        <-done
        return ctx.Err()
    }
}
```

**注意**:这种"包一层 goroutine"的写法**会有一次额外的 G 创建**——业务里如果对 ctx 敏感,需要更深度的集成(改 Cond 为 channel 风格)。

---

## 十、和 `sync.Once` 的对比

| | `sync.Once` | Barrier |
| --- | --- | --- |
| 触发 | 任意 G 执行,只跑一次 | N 个 G 齐步 |
| 复用 | **不能**(一次性) | **可循环** |
| 用途 | 单例初始化 / 懒加载 | 分阶段并行 |

虽然都涉及"协调",但**完全不是一个语义**。

---

## 十一、现场表达模板

> "屏障和 WaitGroup 最大区别:**WaitGroup 是主等子(单向),Barrier 是 N 个互相等(双向)**。所有 G 凑齐才一起放行,常用于 MapReduce、ML 训练 epoch 同步。
>
> 实现核心:`Mutex + Cond + count + capacity + generation`。
>
> 关键的'**代数(generation)**'是面试加分点——**没有它,新一轮 Await 会被旧 Broadcast 串扰**。每凑齐一轮就 `generation++`,Await 进来记下自己的 gen,被唤醒后判断 `b.generation != gen` 才算"这轮过了"。
>
> 别的细节:
> 1. **最后一个**到达的 G 负责 Broadcast(不是"等齐的人"),先重置 count 和 generation 再唤醒
> 2. **`Break`** 让所有等待者一起失败,避免一个 G 提前挂掉、其他永远凑不齐——`defer recover` → `b.Break()` 是惯用法
> 3. Java `CyclicBarrier` 有**轮次回调**,在凑齐瞬间由最后一个 G 执行,适合 epoch 收尾
> 4. Go 标准库没有屏障,**业务里一般用 `errgroup.Wait()` 分阶段调用** 替代(简单但每阶段都要重新建 group)"

---

## 十二、一句话总结

> **屏障 = Mutex + Cond + count + capacity + generation**;
>
> - **核心创新是 generation**——防止新一轮 Await 被旧 Broadcast 错误唤醒
> - **最后一个到达**的 G 重置状态 + Broadcast,所有人在新 generation 下醒来
> - **Break + defer recover** 是必备:避免一个 G 挂掉拖死整个屏障
> - 和 [WaitGroup](13-waitgroup.md) 的区别:**互等(双向)+ 可循环**;WG 是主等子
> - 典型场景:MapReduce 分阶段、ML epoch 同步、并发测试同时起跑
