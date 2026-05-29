# 延迟队列(Delay Queue)

> **题目**:实现 `delayqueue.Offer(item, delay)` / `Take() (item, ok)`,Take 阻塞直到队首任务到期,可被 Close 唤醒。
>
> 考查:**小顶堆 + Mutex + Cond + 定时唤醒**、和**时间轮**的取舍、Java DelayQueue / Kafka 的对照。

延迟队列是订单超时关闭、消息延迟投递、Kafka purgatory、Java `DelayQueue` 的核心结构。资深面试常和[时间轮](06-timing-wheel.md)对比着问。

---

## 一、考点拆解

| 考点 | 不及格 | 合格 | 优秀 |
| --- | --- | --- | --- |
| 数据结构 | 链表 + 遍历 | **小顶堆** | 知道 `container/heap` 接口 |
| Take 阻塞 | for + Sleep 1s 自旋 | **Cond.WaitTimeout 等队首到期** | Go 没原生 WaitTimeout,用 channel 模拟 |
| 唤醒新插入 | 新 item 比队首早,旧 Wait 不知 | **Offer 时 Signal** | 区分"新 item 是新队首"才 Signal |
| 并发取 | 多个消费者抢同一 item | **Take 拿锁 + 重检** | 公平性(Cond 顺序) |
| 取消 | 没实现 | Close 让 Take 返回 | 已入队的 item 是否丢弃 |
| vs 时间轮 | 不知道 | 知道堆是 O(log N) | 海量任务用时间轮,百万级延迟选时间轮 |

---

## 二、思路推导

### 2.1 朴素错误写法

```go
// ❌ 自旋,CPU 100%
for {
    item := pq.Peek()
    if item != nil && time.Now().After(item.deadline) {
        return pq.Pop()
    }
    time.Sleep(10 * time.Millisecond)
}
```

问题:
- 自旋浪费 CPU
- 10ms 精度不准
- 新 item 比队首早,要等一轮 Sleep

### 2.2 正确思路:堆顶到期时间驱动等待

```text
Take():
  for {
    Lock
    if 堆为空: Cond.Wait()       // 永等,Offer 时唤醒
    else:
      top := 堆顶
      now := time.Now()
      if now >= top.deadline:    // 到期了
        Pop()
        Unlock
        return item
      else:                       // 等到期或被新 Offer 唤醒
        wait = top.deadline - now
        condWaitTimeout(wait)
    Unlock
  }

Offer(item, delay):
  Lock
  push 到堆
  if 新 item 成为新堆顶: Cond.Broadcast() // 让等的人重算 wait
  Unlock
```

**关键不变式**:
1. **堆顶变了就唤醒**——新 item 可能比当前堆顶更早到期
2. **Take 等待时不持锁**,otherwise Offer 进不来

---

## 三、Go 没有 `Cond.WaitTimeout` 怎么办

Go 的 `sync.Cond` **没有 WaitTimeout**——这是 Go 团队故意的(认为大多数场景应该用 channel)。

经典解法:**用 channel 包装 Cond**,或者**直接用 channel + timer**。

下面用第二种(更 Go-idiomatic)。

---

## 四、完整代码

```go
package delayqueue

import (
    "container/heap"
    "sync"
    "time"
)

// ---------- 堆元素 ----------
type item struct {
    value    interface{}
    deadline time.Time
    index    int
}

type pq []*item

func (p pq) Len() int            { return len(p) }
func (p pq) Less(i, j int) bool  { return p[i].deadline.Before(p[j].deadline) }
func (p pq) Swap(i, j int)       { p[i], p[j] = p[j], p[i]; p[i].index = i; p[j].index = j }
func (p *pq) Push(x interface{}) { *p = append(*p, x.(*item)); (*p)[len(*p)-1].index = len(*p) - 1 }
func (p *pq) Pop() interface{}   { old := *p; n := len(old); x := old[n-1]; *p = old[:n-1]; return x }

// ---------- 延迟队列 ----------
type DelayQueue struct {
    mu     sync.Mutex
    heap   pq
    wakeup chan struct{} // 新 item 成为队首 / Close 时被关闭信号
    closed bool
}

func New() *DelayQueue {
    return &DelayQueue{wakeup: make(chan struct{}, 1)}
}

// Offer 投递一个 delay 后到期的 item
func (q *DelayQueue) Offer(v interface{}, delay time.Duration) {
    q.mu.Lock()
    defer q.mu.Unlock()
    if q.closed {
        return
    }
    it := &item{value: v, deadline: time.Now().Add(delay)}
    heap.Push(&q.heap, it)
    // 如果新 item 是新的堆顶 → 唤醒等待者重算 wait
    if it.index == 0 {
        q.notify()
    }
}

// Take 阻塞获取已到期的 item;Close 后返回 ok=false
func (q *DelayQueue) Take() (interface{}, bool) {
    for {
        q.mu.Lock()
        if q.closed {
            q.mu.Unlock()
            return nil, false
        }
        if q.heap.Len() == 0 {
            q.mu.Unlock()
            // 队空 → 永等(直到 Offer / Close)
            <-q.wakeup
            continue
        }
        top := q.heap[0]
        wait := time.Until(top.deadline)
        if wait <= 0 {
            // 到期 → Pop 返回
            heap.Pop(&q.heap)
            q.mu.Unlock()
            return top.value, true
        }
        q.mu.Unlock()

        // 等到期 or 被新 Offer 唤醒 or Close
        timer := time.NewTimer(wait)
        select {
        case <-timer.C:
        case <-q.wakeup:
            timer.Stop()
        }
    }
}

// Close 关闭队列,所有 Take 立刻返回 ok=false
func (q *DelayQueue) Close() {
    q.mu.Lock()
    defer q.mu.Unlock()
    if q.closed {
        return
    }
    q.closed = true
    close(q.wakeup) // 关闭 channel → 所有等待者一起醒
}

// notify 非阻塞通知(防止 Offer 自己卡住)
func (q *DelayQueue) notify() {
    select {
    case q.wakeup <- struct{}{}:
    default:
    }
}
```

> ⚠️ `Close` 用 `close(channel)` 一次性唤醒所有等待者(它们 select 拿到零值会跳出)。**这之后不能再 `notify()`(向已关闭 channel 写会 panic)**——所以 Offer 里有 `q.closed` 检查。

**讲解时主动点出的细节**:

| 设计 | 资深点 |
| --- | --- |
| `container/heap` 接口 | Go 的"侵入式"堆接口,需要实现 `Push/Pop/Less` |
| `wakeup` 容量 1 + select default | **非阻塞通知**,多次 Offer 不重复堆积通知 |
| `it.index == 0` 才 notify | 不是新堆顶不用唤醒,**减少惊群** |
| Take 拿锁后**重检**堆顶 | 被唤醒后状态可能变了(被别人抢了) |
| `time.NewTimer + Stop` | 不用 `time.After`,**避免 goroutine 内存泄漏** |
| Close 用 `close(wakeup)` | 一次唤醒所有 Take,**channel 的关闭语义天然支持** |

---

## 五、测试

```go
func main() {
    q := New()
    start := time.Now()

    go func() {
        q.Offer("A", 300*time.Millisecond)
        q.Offer("B", 100*time.Millisecond) // 插队 → 应该先出
        q.Offer("C", 500*time.Millisecond)
    }()

    for i := 0; i < 3; i++ {
        v, _ := q.Take()
        fmt.Printf("%v @ %v\n", v, time.Since(start))
    }
}
// B @ ~100ms
// A @ ~300ms
// C @ ~500ms
```

---

## 六、进阶变体

### 6.1 支持取消单个 item

```go
type Handle struct {
    it *item
    q  *DelayQueue
}

func (q *DelayQueue) OfferWithHandle(v interface{}, delay time.Duration) *Handle {
    // ... 同 Offer
    return &Handle{it: it, q: q}
}

func (h *Handle) Cancel() bool {
    h.q.mu.Lock()
    defer h.q.mu.Unlock()
    if h.it.index < 0 { // 已被取走
        return false
    }
    heap.Remove(&h.q.heap, h.it.index)
    h.it.index = -1
    return true
}
```

注意 `container/heap.Remove` 是 O(log N),需要 item 知道自己的 index——这是为什么 item 结构里有 `index` 字段。

### 6.2 多消费者

上面的实现**已经支持多消费者**(都在 Take 里抢 Mutex)。
但有个公平性问题:多个 Take 等同一个 wakeup,channel 只 send 一个会被一个抢走——另一个继续等。
解决:**Close 时用 `close(channel)` 一次唤醒所有**(本实现已经这样做了)。

### 6.3 vs 时间轮

| 维度 | 延迟队列(堆)| [时间轮](06-timing-wheel.md) |
| --- | --- | --- |
| 添加 | O(log N) | **O(1)** |
| 触发 | O(log N) | O(M),M 是 bucket 任务数 |
| 精度 | 纳秒(time.Timer) | **tick 级**(1ms-1s) |
| 海量任务 | 几十万 OK | **百万级仍 OK** |
| 取消任务 | O(log N) 找 index | **O(1)** list.Remove |
| 实现 | **几十行**(`container/heap`)| 几百行(rounds / 多级降级)|
| 典型用途 | 业务订单超时、Java DelayQueue | Kafka purgatory、Netty 心跳 |

**资深表达**:"百万级任务才上时间轮,几十万以内堆完全够用——而且**堆精度高**,业务订单超时一秒不差。"

### 6.4 Java DelayQueue 对照

```java
// Java 等价物
DelayQueue<Delayed> q = new DelayQueue<>();
q.offer(item);
Delayed taken = q.take(); // 阻塞直到队首到期
```

实现内部就是 `PriorityQueue` + `Condition.awaitNanos(delay)`。Java 有 `awaitNanos`,Go 没有,所以 Go 实现要 channel + timer 拼接。

---

## 七、Kafka purgatory(资深加分)

Kafka 的"purgatory"用于跟踪**延迟操作**(producer 等 ack、consumer 等 fetch),其实是个**延迟队列 + 时间轮的组合**:

```text
DelayedOperationPurgatory:
  - 时间轮负责"按时间触发"(O(1) 添加)
  - 但每个 DelayedOperation 还能被"事件驱动"提前完成(比如 ack 到了)
  - 所以同时维护 watchKey → DelayedOperation 映射,事件触发时主动 complete
```

→ **延迟 + 事件**双触发,这是 Kafka purgatory 的精髓,Go 业务里很少需要这么复杂。

---

## 八、典型坑

| 坑 | 现象 | 修复 |
| --- | --- | --- |
| 用 `time.After` | 长跑 goroutine 内存泄漏(timer 直到 deadline 才回收) | `NewTimer` + `Stop()` |
| Offer 完忘 notify | 新 item 是新队首,但 Take 还在等旧的 deadline | `index==0 时 notify` |
| Take 锁内等 | 死锁,Offer 进不来 | 解锁后再 select 等 |
| Close 后还 Offer | panic(向 closed channel 写)| Offer 里检查 `q.closed` |
| 用 `Cond.Signal` 替代 channel | Signal 没 timeout 能力,要拼 timer + Broadcast 很丑 | 直接 channel + timer |
| 多个消费者用 `wakeup <- struct{}{}` | 只唤醒一个,其他卡死 | Close 用 `close(channel)` 全部唤醒 |
| heap 没维护 index | Cancel 时无法 O(log N) 删 | `Swap/Push/Pop` 里都更新 index |
| 想用 RWMutex | 写多读少不如 Mutex | Take/Offer 都改状态,**用 Mutex** |

---

## 九、现场表达模板

> "延迟队列核心是**小顶堆按 deadline 排序 + Take 阻塞等队首到期**。
>
> 难点不是堆,是**唤醒**:
> 1. 队空时 Take 永等,Offer 进来要唤醒
> 2. 队非空时 Take 等 `deadline-now`,但**新 Offer 可能更早到期**——所以新 item 成为新堆顶时要唤醒已等的 Take 重算
> 3. Close 要让所有 Take 立刻返回
>
> Go 没有 `Cond.WaitTimeout`,所以**用 channel + time.NewTimer 拼接**:Take 解锁后 `select { timer.C / wakeup }`,Offer 时往 wakeup 写,Close 直接 `close(wakeup)` 一次唤醒所有。
>
> 和时间轮对比:**堆是 O(log N)、精度高、实现简单**,几十万任务以内首选;**时间轮 O(1)、精度低、实现复杂**,百万级才划算。
> Kafka purgatory 是两者结合——时间触发 + 事件驱动提前完成。
>
> 典型应用:订单 30 分钟超时关闭、消息延迟投递、缓存 TTL 队列。Java 直接用 `java.util.concurrent.DelayQueue`,Go 一般业务自己拼个简单版就够。"

---

## 十、一句话总结

> **延迟队列 = 小顶堆 + Mutex + channel + Timer**;
>
> - **唤醒条件**:新 item 成为新堆顶 / 队列从空到非空 / Close
> - Go 没 `Cond.WaitTimeout`,用 **channel + `time.NewTimer`** 拼出超时等待
> - Close 用 **`close(channel)`** 一次唤醒所有 Take,优雅退出
> - **几十万任务内堆够用**,百万级以上才考虑[时间轮](06-timing-wheel.md)
> - Kafka purgatory 是堆 + 时间轮 + 事件触发的混合,业务里一般不需要
