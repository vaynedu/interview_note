# 手写代码题(Coding Problems)

> 资深 Go 面试**必考**:现场写代码,不让用 channel / 不让用标准库 / 不让用第三方,考你对**同步原语 + 数据结构 + 并发模式**的真功底。
>
> 本目录按**类型**收录高频题,每篇按"题目 → 考点 → 思路 → 关键决策 → 完整代码 → 进阶变体"组织,保证现场能复述、能写。

---

## 一、为什么单独开这个专题

| 普通"知识点"文件 | 本目录(手写题)|
| --- | --- |
| 讲原理 / 设计 / 取舍 | 讲**如何在 30 分钟内写出可运行代码** |
| 答题模板偏概念表达 | 答题模板偏**白板节奏 + 边写边讲** |
| 量化数据 + 业务场景 | **边界条件 + 死锁 + 虚假唤醒** |

资深面试在算法题之外,几乎都会带 1 道这种"小型系统实现题"。考察点:
- **同步原语熟练度**(Mutex / RWMutex / Cond / WaitGroup / atomic)
- **不依赖 channel 的并发设计**(很多面试官故意禁用)
- **边界处理**(空 / 满 / 关闭 / 超时 / panic 安全)
- **现场调试能力**(死锁 / 数据竞争如何规避)

---

## 二、题目分类

### 2.1 同步原语类

实现各种 sync 包的轮子,考你对锁 / 条件变量 / atomic 的理解。

| 题 | 涉及 | 文件 |
| --- | --- | --- |
| **阻塞队列**(无 channel)| Mutex + Cond × 2 | [01-blocking-queue.md](01-blocking-queue.md) |
| **协程池**(worker pool) | buffered channel + WaitGroup + recover | [02-worker-pool.md](02-worker-pool.md) |
| **读写锁(写优先)** | Mutex + 两个 Cond + writerWaiting | [09-rwlock-write-prefer.md](09-rwlock-write-prefer.md) |
| **WaitGroup** | atomic uint64(高 32 counter + 低 32 waiter) | [13-waitgroup.md](13-waitgroup.md) |
| **errgroup**(无第三方) | WaitGroup + sync.Once + CancelFunc + 信号量 | [11-errgroup.md](11-errgroup.md) |
| **信号量**(Semaphore) | channel / Cond + FIFO 队列 | [05-semaphore.md](05-semaphore.md) |
| **屏障(Barrier / CyclicBarrier)** | Mutex + Cond + generation 代数 | [14-barrier.md](14-barrier.md) |

### 2.2 数据结构类(并发安全)

| 题 | 涉及 | 文件 |
| --- | --- | --- |
| 并发安全 **LRU** | 双向链表 + map + Mutex | [04-lru-cache.md](04-lru-cache.md) |
| **并发安全 LFU** | 频次桶(双向链表 of 双向链表)+ map + minFreq | [15-lfu-cache.md](15-lfu-cache.md) |
| **并发安全 map(分片)** | 32 片 + RWMutex + fnv hash | [12-sharded-map.md](12-sharded-map.md) |
| **跳表(线程安全)** | 多级链表 + 随机高度 + atomic CAS | [16-skiplist.md](16-skiplist.md) |
| 环形缓冲(SPSC / MPMC) | atomic 头尾指针 | 待补 |

### 2.3 并发模式类

| 题 | 涉及 | 文件 |
| --- | --- | --- |
| 协程池(worker pool) | task queue + worker × N | [02-worker-pool.md](02-worker-pool.md) |
| pipeline(N 阶段流水线) | 多 channel / 多队列串联 | 待补 |
| fan-out / fan-in | 任务分发 + 结果合并 | 待补 |
| **单飞**(singleflight) | map + WaitGroup + Mutex | [07-singleflight.md](07-singleflight.md) |

### 2.4 限流 / 调度类

| 题 | 涉及 | 文件 |
| --- | --- | --- |
| **令牌桶 / 漏桶 / 滑动窗口** | atomic 时间戳 + 计数 / 环形数组 + 时间戳 | [03-rate-limiter.md](03-rate-limiter.md) |
| **时间轮**(timing wheel) | 多级数组 + tick / rounds | [06-timing-wheel.md](06-timing-wheel.md) |
| **延迟队列** | 小顶堆 + Mutex + channel + Timer | [10-delay-queue.md](10-delay-queue.md) |

### 2.5 序号 / 协作类

> 高频小题,经常作为热身。

| 题 | 涉及 | 文件 |
| --- | --- | --- |
| **N 个 goroutine 交替打印 ABC** | channel 环 / Cond / atomic 三解 | [08-print-abc.md](08-print-abc.md) |
| 父协程等子协程全部退出 | WaitGroup / channel close | 待补 |
| 多个生产者一个消费者 | channel close 信号 | 待补 |

---

## 三、统一答题节奏(白板版)

每道题面试都按这 5 步讲,不要直接埋头写:

```text
1. 复述题目 + 反问(30s)
   "我理解的是 X,边界是 Y,容量上限是 Z,对吗?"
   反问点:容量是否有界 / 是否需要超时 / 是否需要 Close / 是否多生产多消费

2. 给出核心数据结构 + 同步原语(1min)
   "我会用 Mutex 保护数据,2 个 Cond 分别处理满和空"
   先告诉面试官设计思路,避免边写边改

3. 写主结构 + 构造函数(1min)
   把 struct 和 New 函数先写出来,让面试官看到骨架

4. 写核心方法(8~10min)
   边写边讲不变式:"这里 for 不能换 if,因为可能虚假唤醒"
   "Signal 必须在锁内,Wait 会自动释放锁"

5. 自测 + 边界(2min)
   主动说:"我再过一遍——空 / 满 / 并发抢 / Close 行为"
   最后写一段 main 跑一下
```

**反模式**(资深扣分点):
- ❌ 直接开始写 `func Push(...)` — 先讲设计
- ❌ `if size == cap { wait }` — 必须 `for` 防虚假唤醒
- ❌ `cond.Signal()` 放锁外 — 行为未定义
- ❌ 不讲 Close / panic 路径 — 暴露并发设计不完整
- ❌ 通篇 `interface{}` 不提泛型 — Go 1.18+ 该用泛型

---

## 四、复习建议

1. 第一遍:按分类**手写一遍**,卡在哪里就看答案——重点是肌肉记忆
2. 第二遍:**变体改造**(加超时 / 加 Close / 改泛型 / 改无锁)——考的就是这些
3. 第三遍:**讲给别人听**——能讲清楚"为什么 for 不是 if"才算过关

---

## 五、和其他目录的关系

| 目录 | 关注点 |
| --- | --- |
| `01-go-language/` | 语言**原理**(GMP / GC / channel 源码) |
| **`20-coding-problems/`**(本目录)| **现场手写实现** |
| `10-system-design/` | 系统**架构设计**(画图为主) |
| `99-meta/` | 速记题集 + 综合追问 |

> 手写题 ≠ 系统设计 ≠ 语言原理。三者结合才是资深的护城河。
