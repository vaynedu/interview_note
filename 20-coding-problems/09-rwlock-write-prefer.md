# 读写锁(写优先 / 读优先 / 公平)

> **题目**:不用 `sync.RWMutex`,自己实现一个**写优先**的读写锁,支持 `RLock / RUnlock / Lock / Unlock`,保证写者不饿死。
>
> 考查:**Mutex + Cond + 计数器** 协作、**读者写者饥饿问题**、和 Go `sync.RWMutex` 对照。

读写锁是高读低写场景的经典优化(配置中心、路由表、缓存元数据)。默认实现往往**读者优先**,大流量读会**饿死写者**。资深面试常问"怎么改成写优先"。

---

## 一、考点拆解

| 考点 | 不及格 | 合格 | 优秀 |
| --- | --- | --- | --- |
| 概念 | 等于 Mutex | 多读单写 | 读读并发 / 读写互斥 / 写写互斥 |
| 饥饿 | 没想过 | 知道读者优先会饿死写者 | **三种策略**(读优先 / 写优先 / 公平 FIFO)对比 |
| 实现 | 只会用 sync.RWMutex | Mutex + Cond + 计数 | 区分 readerCount / writerWaiting / writerActive |
| sync.RWMutex 行为 | 不知道 | 说"读写互斥" | **Go 当前版本是写优先**(1.18+ semaphore + 待写者计数) |
| 升级 / 降级 | 不知道 | RUnlock 后再 Lock | **不能在持有 RLock 时直接 Lock**(死锁) |
| 用途 | 路由表 | 配置中心 / 缓存元数据 | **写极少读极多**才划算,否则用 Mutex 更快 |

---

## 二、读写锁三种调度策略

| 策略 | 行为 | 优点 | 缺点 |
| --- | --- | --- | --- |
| **读优先**(Reader-preferring) | 有读者就阻塞写者 | 读吞吐最大 | **写者饿死** |
| **写优先**(Writer-preferring) ⭐ | 有写者等待,新读者阻塞 | 写不饿死 | 读吞吐略降 |
| **公平**(FIFO / fair) | 严格按到达顺序 | 不饿死任何一边 | 实现复杂,吞吐最低 |

> **Go `sync.RWMutex` 在 1.18 之后是写优先**——内部用 `readerCount` 负值标记"有写者等待",新进读者看到负值会去 semaphore 排队。**所以直接用标准库就够,自己写是为了应付面试。**

---

## 三、思路推导

### 3.1 默认"读优先"的问题

```go
// ❌ 朴素实现:读优先
type RWMutex struct {
    mu sync.Mutex
    cond *sync.Cond
    readers int
    writer bool
}

func (l *RWMutex) RLock() {
    l.mu.Lock()
    for l.writer { l.cond.Wait() } // 只看写者
    l.readers++
    l.mu.Unlock()
}

func (l *RWMutex) Lock() {
    l.mu.Lock()
    for l.readers > 0 || l.writer { l.cond.Wait() } // 等所有读者走
    l.writer = true
    l.mu.Unlock()
}
```

问题:读流量持续涌入 → `readers` 永远 > 0 → 写者**永远 Wait**。

### 3.2 写优先的核心改造

只加**一个计数**:`writerWaiting`(等待中的写者数)。

```text
RLock 条件:  没有活动写者 且 没有等待中的写者
RUnlock:     readers-- → 如果 readers==0,唤醒等待的写者
Lock:        writerWaiting++,等到 readers==0 && writer==false 后开写
Unlock:      writer=false,Broadcast(让读者和后续写者重新竞争)
```

**不变式**:只要有写者在等,新读者**全部排队**——这就堵住了"读者源源不断"的饥饿路径。

---

## 四、完整代码(写优先)

```go
package rwlock

import "sync"

type WriterPreferRWMutex struct {
    mu             sync.Mutex
    readerCount    int  // 当前活动读者
    writerWaiting  int  // 等待中的写者
    writerActive   bool // 当前有写者持锁

    readerCond *sync.Cond // 读者等的条件
    writerCond *sync.Cond // 写者等的条件
}

func New() *WriterPreferRWMutex {
    l := &WriterPreferRWMutex{}
    l.readerCond = sync.NewCond(&l.mu)
    l.writerCond = sync.NewCond(&l.mu)
    return l
}

func (l *WriterPreferRWMutex) RLock() {
    l.mu.Lock()
    defer l.mu.Unlock()
    // 关键:有写者活动 或 有写者等待,都让读者排队
    for l.writerActive || l.writerWaiting > 0 {
        l.readerCond.Wait()
    }
    l.readerCount++
}

func (l *WriterPreferRWMutex) RUnlock() {
    l.mu.Lock()
    defer l.mu.Unlock()
    if l.readerCount == 0 {
        panic("RUnlock without RLock")
    }
    l.readerCount--
    if l.readerCount == 0 && l.writerWaiting > 0 {
        l.writerCond.Signal() // 最后一个读者走 → 唤醒一个写者
    }
}

func (l *WriterPreferRWMutex) Lock() {
    l.mu.Lock()
    defer l.mu.Unlock()
    l.writerWaiting++
    for l.readerCount > 0 || l.writerActive {
        l.writerCond.Wait()
    }
    l.writerWaiting--
    l.writerActive = true
}

func (l *WriterPreferRWMutex) Unlock() {
    l.mu.Lock()
    defer l.mu.Unlock()
    if !l.writerActive {
        panic("Unlock without Lock")
    }
    l.writerActive = false
    // 优先唤醒等待中的写者,没有再唤醒读者
    if l.writerWaiting > 0 {
        l.writerCond.Signal()
    } else {
        l.readerCond.Broadcast()
    }
}
```

**讲解时主动点出的细节**:

| 设计 | 资深点 |
| --- | --- |
| 两个 Cond 共用同一个 Mutex | 读者写者**等的条件不同**,分开 Cond 唤醒精准,避免惊群 |
| `for` 而不是 `if` | **防虚假唤醒**(见 [01-blocking-queue.md](01-blocking-queue.md)) |
| `writerWaiting++` 在 Wait 之前 | 让**新读者立刻看到**"有人在等写",马上排队 |
| Unlock 优先 Signal writer | **写优先核心策略**,有写就让写先来 |
| 没写者时 Broadcast 读者 | 一次唤醒所有等待读者,**读读并发** |
| readerCount == 0 才唤醒 writer | 最后一个读者负责"开门",其他 RUnlock 不用唤醒 |

---

## 五、测试:验证写不饿死

```go
func main() {
    rw := New()
    var counter atomic.Int64

    // 启动 10 个读者,持续 1s,每个读 100ms 一次
    for i := 0; i < 10; i++ {
        go func(id int) {
            for {
                rw.RLock()
                time.Sleep(10 * time.Millisecond)
                rw.RUnlock()
                time.Sleep(1 * time.Millisecond)
            }
        }(i)
    }

    // 等 50ms 后启动一个写者
    time.Sleep(50 * time.Millisecond)
    start := time.Now()
    rw.Lock()
    fmt.Printf("写者等待: %v\n", time.Since(start)) // 应远小于 1s
    counter.Add(1)
    rw.Unlock()
}
```

**读优先版本**:写者要等读流量彻底退去,可能 5s+ 甚至永远不来。
**写优先版本**:写者等不到 50ms 就拿到锁(等当前持锁的几个读者放手即可)。

---

## 六、进阶变体

### 6.1 读优先版(性能极致,但有饥饿风险)

只需改 `RLock`:**不看 writerWaiting**,只看 writerActive。

```go
func (l *ReaderPreferRWMutex) RLock() {
    l.mu.Lock()
    defer l.mu.Unlock()
    for l.writerActive { // 注意没有 writerWaiting
        l.readerCond.Wait()
    }
    l.readerCount++
}
```

适用:**读极多、写极少、可接受写者偶尔等久**(配置中心刷新场景)。

### 6.2 公平版(FIFO)

完全严格按到达顺序,需要**等待队列**(类似 [05-semaphore.md](05-semaphore.md) 的 Weighted 版):

```go
type waiter struct {
    isWriter bool
    ready    chan struct{}
}
```

`RLock` / `Lock` 都入队尾,`RUnlock`/`Unlock` 从队头唤醒。
**实现复杂、性能差,业务里几乎用不到**,Java `ReentrantReadWriteLock(true)` 就是这种。

### 6.3 可重入(reentrant)

记录每个 goroutine 持有的锁次数(用 `goroutine ID + map`),允许同 G 多次 RLock。
**反模式**:Go 官方明确不支持可重入,理由是**容易隐藏死锁 bug**,真要重入说明设计有问题。

### 6.4 RLock 升级 Lock

**直接升级会死锁**:

```go
rw.RLock()
defer rw.RUnlock()
// ... 决定要写
rw.Lock() // ❌ 死锁:自己等自己的 RUnlock
```

正确做法:**先放再抢**(注意中间状态会变化,要重新检查):

```go
rw.RLock()
v := check()
rw.RUnlock()

if needWrite {
    rw.Lock()
    defer rw.Unlock()
    // 必须重新检查!中间可能有人改过
    if check() {
        write()
    }
}
```

---

## 七、和 `sync.RWMutex` 的对比

```go
import "sync"

var mu sync.RWMutex
mu.RLock();   defer mu.RUnlock()
mu.Lock();    defer mu.Unlock()

// RLocker() 返回一个 sync.Locker,把 RLock/RUnlock 适配成 Lock/Unlock
// 用于把 RWMutex 传给只接 Locker 接口的 API
locker := mu.RLocker()
cond := sync.NewCond(locker)
```

| 维度 | 标准库 `sync.RWMutex` | 自己写的 |
| --- | --- | --- |
| 调度策略 | **写优先**(Go 1.18+) | 可选 |
| 实现 | runtime semaphore + atomic | Mutex + Cond |
| 性能 | **快**(无锁路径多) | 慢(每次操作都 Mutex) |
| 写者饥饿 | 不会 | 取决于策略 |
| 可读性 | 黑盒 | 透明 |
| 适用 | **生产代码** | **学习 / 面试** |

**资深表达**:"业务代码直接用 `sync.RWMutex`——它在 Go 1.18 之后已经是写优先,通过 `readerCount` 负值标记 + semaphore 实现,性能比 Mutex+Cond 高一个数量级。自己写读写锁主要是面试展示对同步原语的理解。"

---

## 八、读写锁 vs 互斥锁:什么时候真划算

读写锁**不是免费**的——每次操作要维护 readerCount、走两次 atomic 至少。
经验阈值:

| 读写比 | 推荐 |
| --- | --- |
| 读 : 写 < 10 : 1 | **直接用 `Mutex`** |
| 读 : 写 > 100 : 1 且读耗时长 | `sync.RWMutex` |
| 写极频繁 | **Mutex**(RWMutex 反而更慢) |
| 写极少(< 1/min)| 考虑 `atomic.Value` 整体替换 |

**反模式**:别看到"读多写少"就上 RWMutex,**短临界区下 Mutex 几乎总是更快**。

---

## 九、典型坑

| 坑 | 现象 | 修复 |
| --- | --- | --- |
| 读优先实现 → 写饿死 | 持续读流量下写永远等不到 | 加 `writerWaiting` 计数,写优先 |
| `for` 写成 `if` | 虚假唤醒导致多写并发 | 必须 for 重检 |
| RUnlock 多于 RLock | counter 变负 / panic | 加校验 `if readerCount==0 panic` |
| RLock 中再 Lock | 死锁 | 先 RUnlock 再 Lock,重新检查 |
| Unlock 没分情况广播 | 写优先失效 / 读者卡死 | 有写者 Signal writer,否则 Broadcast reader |
| 高读写混合用 RWMutex | 比 Mutex 还慢 | 测了再决定,默认用 Mutex |
| 想搞可重入 | 隐藏死锁 | Go 不支持,设计上避免 |
| 持锁里干慢活(IO) | 卡死所有读者 | 临界区只放内存操作,IO 拿出去 |

---

## 十、现场表达模板

> "读写锁的核心是**读读并发、读写互斥、写写互斥**。默认实现往往**读优先**——只看活动写者,这样读流量大时**写者会饿死**。
>
> **写优先**的改法很简单:加一个 `writerWaiting` 计数器,新读者除了看活动写者,**还要看有没有写者在等**——只要有写者等,新读者就排队。这样写者最多等当前持锁的几个读者放手就能上。
>
> 实现上用 **Mutex + 两个 Cond**:读者等 readerCond,写者等 writerCond。Unlock 时**优先 Signal writer**——这是写优先的关键。读者 RUnlock 时只有 `readerCount==0` 才唤醒写者(最后一个读者负责开门)。
>
> 注意点:
> 1. 等待条件必须 **for 而不是 if**,防虚假唤醒
> 2. **不能在持有 RLock 时直接 Lock**——会死锁,必须先 RUnlock 再重新抢,中间状态要重新检查
> 3. **Go 1.18+ 的 sync.RWMutex 已经是写优先**,业务代码直接用就行,自己写是为了面试展示对 Mutex+Cond 的理解
> 4. **读写比不到 10:1 直接用 Mutex**,RWMutex 的维护开销吃掉了并发收益"

---

## 十一、一句话总结

> **写优先读写锁 = Mutex + 两个 Cond + readerCount + writerWaiting**;
>
> - **核心不变式**:有写者等待时,**新读者必须排队**(堵住饥饿路径)
> - **RUnlock 时只有最后一个读者负责唤醒写者**,Unlock 时优先 Signal writer
> - **Go 标准库 1.18+ 已是写优先**,业务直接用 `sync.RWMutex`
> - **读写比 < 10:1 时 Mutex 更快**,RWMutex 不是万能优化
> - **不支持可重入、不能持读升级写**,要重新抢并重检状态
