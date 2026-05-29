# WaitGroup(自己实现)

> **题目**:不用 `sync.WaitGroup`,自己实现 `Add(n) / Done() / Wait()`,支持**多 waiter 同时 Wait**、Add 负数 panic、计数归零唤醒所有 Waiter。
>
> 考查:**atomic + 信号量 / Mutex + Cond**、**counter 和 waiter 数为什么要原子合并**、Go 标准库源码细节。

WaitGroup 是 Go 并发基础组件,看似简单,实际**标准库实现里有 atomic uint64 高低 32 位合并的巧思**——资深面试常拿这个考"对源码的熟悉度"。

---

## 一、考点拆解

| 考点 | 不及格 | 合格 | 优秀 |
| --- | --- | --- | --- |
| 基础语义 | 不熟 | Add / Done / Wait | **Add 在 Go 之前** + Done 在 defer |
| 数据结构 | Mutex + count | Mutex + Cond + count | **atomic uint64(高 32 counter + 低 32 waiter)** |
| 多 waiter | 只支持一个 Wait | 用 Broadcast 唤醒所有 | 每个 waiter 拿自己的信号量 |
| 边界 | Add(-1) 没处理 | counter < 0 panic | **counter > 0 时不能 Wait → Wait → Add(+) 误用** |
| 复用 | 不能 | Wait 完后能重用 | **Wait 期间 Add 是 race**(标准库会 panic) |
| 性能 | Mutex 重 | atomic 比 Mutex 快 10× | runtime_Semacquire 比 channel 快 |

---

## 二、关键不变式

WaitGroup 内部不变式:

1. **counter ≥ 0** 任何时刻
2. **counter == 0 时,所有 Wait 必须立刻返回**
3. **counter > 0 时,Wait 必须阻塞**
4. **Wait 不能和 Add(+) 并发**(标准库会 panic,因为可能产生未定义行为)

---

## 三、实现一:Mutex + Cond(教学版)

最直观的写法,基本能用,但有几个**微妙坑**。

```go
package wg

import "sync"

type WaitGroup struct {
    mu      sync.Mutex
    cond    *sync.Cond
    counter int
}

func New() *WaitGroup {
    w := &WaitGroup{}
    w.cond = sync.NewCond(&w.mu)
    return w
}

func (w *WaitGroup) Add(delta int) {
    w.mu.Lock()
    defer w.mu.Unlock()
    w.counter += delta
    if w.counter < 0 {
        panic("WaitGroup: negative counter")
    }
    if w.counter == 0 {
        w.cond.Broadcast() // 唤醒所有 Wait
    }
}

func (w *WaitGroup) Done() {
    w.Add(-1)
}

func (w *WaitGroup) Wait() {
    w.mu.Lock()
    defer w.mu.Unlock()
    for w.counter > 0 {
        w.cond.Wait()
    }
}
```

**讲解时主动点出的细节**:

| 设计 | 资深点 |
| --- | --- |
| `for` 不是 `if` | **防虚假唤醒**(见 [01-blocking-queue.md](01-blocking-queue.md)) |
| **Broadcast** 不是 Signal | 可能有多个 waiter,**必须全部唤醒** |
| `counter < 0` 立刻 panic | **暴露调用方 bug**(多调了 Done) |
| Cond 用同一个 Mutex | Wait 内部会释放锁,Add 才能进来 |

**这版的问题**:每次 Add/Done 都拿 Mutex,**高频 Done 性能差**——百万级 goroutine 的服务,这里会成瓶颈。

---

## 四、实现二:atomic 高低位合并(标准库思路)

Go 标准库 `sync.WaitGroup` 的核心思路:**把 counter 和 waiter 数压到一个 uint64**,用 atomic 操作。

### 4.1 为什么要合并

朴素方案:`counter atomic.Int32 + waiterCount atomic.Int32`。
问题:Add 时要先 `counter.Add`,再判断**当前 waiter 数**——这两步**不是原子**的,有竞态:

```text
A: counter.Add(-1) → 0
B: 准备 Wait,刚读 counter == 0(还没注册 waiter)
A: 发现 counter == 0,但 waiter 数还是 0,不唤醒
B: waiterCount.Add(1),Wait
死锁:B 永远等不到唤醒
```

**修复**:把 counter 和 waiterCount 合并到一个 64 位,**单次 atomic 拿到一致快照**。

### 4.2 内存布局

```text
state uint64
├── 高 32 位: counter
└── 低 32 位: waiter count
```

Add(delta) → `atomic.AddUint64(&state, uint64(delta)<<32)`
- 高 32 位整体加 delta
- 解析:`counter := int32(state >> 32)`, `waiter := uint32(state)`

### 4.3 简化实现

```go
package wg

import (
    "runtime"
    "sync"
    "sync/atomic"
)

type AtomicWaitGroup struct {
    state atomic.Uint64 // hi32 = counter, lo32 = waiter

    mu sync.Mutex
    ch chan struct{} // 所有 waiter 等同一个 channel(Close 唤醒所有)
}

func NewAtomic() *AtomicWaitGroup {
    return &AtomicWaitGroup{ch: make(chan struct{})}
}

func (w *AtomicWaitGroup) Add(delta int) {
    state := w.state.Add(uint64(delta) << 32)
    counter := int32(state >> 32)
    waiter := uint32(state)

    if counter < 0 {
        panic("WaitGroup: negative counter")
    }
    if delta > 0 && counter == int32(delta) && waiter != 0 {
        // 上来就 Add 时 waiter 不应该 > 0;否则是误用(Wait 后又 Add)
        panic("WaitGroup misuse: Add called concurrently with Wait")
    }
    if counter > 0 || waiter == 0 {
        return
    }
    // counter == 0 且有 waiter → 唤醒所有
    w.mu.Lock()
    // 重置 state 和 channel,允许复用
    w.state.Store(0)
    close(w.ch)
    w.ch = make(chan struct{})
    w.mu.Unlock()
}

func (w *AtomicWaitGroup) Done() {
    w.Add(-1)
}

func (w *AtomicWaitGroup) Wait() {
    for {
        state := w.state.Load()
        counter := int32(state >> 32)
        if counter == 0 {
            return // 没活儿,直接走
        }
        // 注册一个 waiter:CAS 把低 32 位 +1
        if w.state.CompareAndSwap(state, state+1) {
            w.mu.Lock()
            ch := w.ch
            w.mu.Unlock()
            <-ch // 等 Add(counter=0) 时 close
            return
        }
        // CAS 失败 → 有人改了 state,重读
    }
}
```

**讲解时主动点出的细节**:

| 设计 | 资深点 |
| --- | --- |
| `state atomic.Uint64` 高低位 | counter 和 waiter 一次原子读 / 写,**消除竞态** |
| Wait 用 CAS 注册 | **完全无锁的 Wait 注册** |
| close(channel) 唤醒所有 | **天然 broadcast**(close 后所有 `<-ch` 立刻返回零值) |
| 唤醒后**重建 channel** | 支持复用 |
| **misuse panic** | 标准库会检测"Wait 期间 Add",直接 panic |
| `w.mu` 只保护 channel 替换 | 不在 Add/Wait 主路径上 |

> ⚠️ 这是**教学版**——标准库实现里 `sync.WaitGroup` 用 `runtime_Semacquire`(直接调度 G,比 channel 快),还有 race detector 钩子。生产代码直接用 `sync.WaitGroup` 即可。

---

## 五、标准库 `sync.WaitGroup` 源码要点

Go 1.20+ `sync/waitgroup.go` 关键代码片段(简化):

```go
type WaitGroup struct {
    noCopy noCopy           // 防拷贝(go vet 会报错)
    state  atomic.Uint64    // 高 32 counter + 低 32 waiter
    sema   uint32           // runtime_Semacquire 用
}

func (wg *WaitGroup) Add(delta int) {
    state := wg.state.Add(uint64(delta) << 32)
    v := int32(state >> 32)
    w := uint32(state)
    if v < 0 {
        panic("sync: negative WaitGroup counter")
    }
    if w != 0 && delta > 0 && v == int32(delta) {
        panic("sync: WaitGroup misuse: Add called concurrently with Wait")
    }
    if v > 0 || w == 0 {
        return
    }
    // counter==0 && waiter>0 → 唤醒所有 waiter
    wg.state.Store(0)
    for ; w != 0; w-- {
        runtime_Semrelease(&wg.sema, false, 0)
    }
}

func (wg *WaitGroup) Wait() {
    for {
        state := wg.state.Load()
        v := int32(state >> 32)
        if v == 0 { return }
        if wg.state.CompareAndSwap(state, state+1) {
            runtime_Semacquire(&wg.sema)
            if wg.state.Load() != 0 {
                panic("sync: WaitGroup is reused before previous Wait has returned")
            }
            return
        }
    }
}
```

**资深要点**:
1. `noCopy` 嵌入字段 + `go vet` 检测 → **WaitGroup 不能值拷贝**
2. `runtime_Semrelease` 唤醒**单个** G,需要循环唤醒 w 次(不像 channel close 一次广播)
3. **misuse 检测**:Wait 后 Add 直接 panic(不要"复用"理解错了)

### 5.1 `noCopy` 是什么

```go
type noCopy struct{}
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
```

它什么也不做,只是让 `go vet` **静态检查**到"这个对象被值拷贝了"时报警告。**所有标准库同步原语(Mutex / WaitGroup / Cond / Pool)都有 noCopy**。

---

## 六、典型坑(高频考点)

| 坑 | 现象 | 修复 |
| --- | --- | --- |
| **`go` 内部 Add** | 主线程已 Wait,Add 慢了一步 → counter 还是 0 → Wait 直接返回 → 漏等 | **`wg.Add(1)` 必须在 `go func()` 之前** |
| Done 多调 | counter 变负 → panic | 严格配对,defer Done |
| 复制 WaitGroup | counter / sema 复制后两边状态错乱 | `go vet` 报警告,**永远传指针** |
| Wait 期间 Add | 标准库 panic | 全部 Add 必须发生在 Wait 之前 |
| **复用**(Wait 后继续 Add)| 标准库允许,但要等 Wait 完全返回 | Wait 返回后再 Add 才安全 |
| 忘 defer Done | panic 后 Done 没执行 → Wait 永远卡 | **必须 `defer wg.Done()`** |
| 用 Cond 实现没 Broadcast | 只唤醒一个 waiter | Cond 版用 Broadcast |
| 0 个任务也 Wait | 立即返回(counter==0) | 没问题,这是设计 |

### 反模式示例

```go
// ❌ Add 在 go 内部
go func() {
    wg.Add(1) // 太晚了!主线程可能已经 wg.Wait() 通过
    defer wg.Done()
    work()
}()
wg.Wait()

// ✅ Add 在 go 之前
wg.Add(1)
go func() {
    defer wg.Done()
    work()
}()
wg.Wait()
```

---

## 七、和其他同步原语对比

| | `WaitGroup` | `errgroup` | `channel close` |
| --- | --- | --- | --- |
| 等待 N 个完成 | ✓ | ✓ | ✓ |
| 错误传播 | ✗ | **✓** | ✗ |
| 取消 | ✗ | **✓** | ✗ |
| 限并发 | ✗ | **✓**(SetLimit) | ✗ |
| 实现难度 | 简单 | 中 | 简单 |
| 适用 | **纯等待** | 并行任务 + 错误 | 简单信号 |

> [11-errgroup.md](11-errgroup.md) 内部就是 `sync.WaitGroup + sync.Once + CancelFunc`,**只在纯等待场景用 WG,需要错误传播就用 errgroup**。

---

## 八、测试

```go
func main() {
    wg := NewAtomic()
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
        }(i)
    }
    wg.Wait()
    fmt.Println("all done")
}
```

---

## 九、现场表达模板

> "`sync.WaitGroup` 看似简单,**内部用 atomic uint64 把 counter 和 waiter 数压到一个 64 位**——为什么要合并?
>
> 朴素方案分两个 atomic 字段,Add 时**判断'有没有 waiter'**和**改 counter** 这两步不是原子的,会出现:
> 1. A 做最后一次 Done,counter 归 0
> 2. B 准备 Wait,读到 counter == 0(还没注册 waiter)
> 3. A 看 waiter == 0,不唤醒
> 4. B 注册 waiter,等永远
>
> 合并到 uint64 后,**一次 atomic 拿到 counter 和 waiter 的一致快照**,消除竞态。
>
> 唤醒用 `runtime_Semrelease` 循环唤醒 w 次——比 channel close 慢一点(channel 是一次广播),但**和 runtime 调度器深度集成**,延迟更稳。
>
> 关键的坑:
> 1. **Add 必须在 go 之前**,不能在 goroutine 内部 Add——会和外面的 Wait 形成竞态
> 2. **Done 必须 defer**,panic 也要保证 Done 被调
> 3. **WaitGroup 不能值拷贝**,内置 noCopy 字段,go vet 会检测
> 4. **Wait 期间 Add 是 misuse**,标准库直接 panic
>
> 业务直接用 `sync.WaitGroup`,自己写是为了展示对'高低位合并 + 信号量 + race 检测'的理解。"

---

## 十、一句话总结

> **`sync.WaitGroup` = atomic uint64(高 32 counter + 低 32 waiter)+ runtime_Semacquire 信号量**;
>
> - **counter 和 waiter 数必须合并**到一个 atomic 字段,否则 Add/Wait 之间有不可消除的竞态
> - **唤醒用循环 Semrelease**(不像 channel close 一次广播),和调度器深度集成
> - 三大铁律:**Add 在 go 之前 / Done 必 defer / 不能值拷贝**(noCopy)
> - **Wait 期间 Add 是 misuse**,标准库直接 panic
> - **纯等待用 WG,要错误传播用** [errgroup](11-errgroup.md)
