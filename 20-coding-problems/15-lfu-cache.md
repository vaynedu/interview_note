# LFU 缓存(Least Frequently Used)

> **题目**:实现 `LFUCache(capacity)`,支持 `Get(key) / Put(key, value)`,**O(1)** 时间复杂度,满时**淘汰访问频次最低**的;频次相同则**淘汰最久未用**。
>
> LeetCode 460,**Hard**。考查:**频次桶(双向链表 of 双向链表)+ map + minFreq**、和 LRU 的本质区别、W-TinyLFU(Caffeine / Ristretto)。

LFU 是缓存淘汰算法里最考"数据结构功底"的一道——比 [LRU](04-lru-cache.md) 难一档,关键是怎么把"找最小频次 + LRU 内部排序"都做到 **O(1)**。

---

## 一、考点拆解

| 考点 | 不及格 | 合格 | 优秀 |
| --- | --- | --- | --- |
| 朴素方案 | 用堆,O(log N) | 知道朴素堆不达标 | **频次桶 + map**,严格 O(1) |
| 数据结构 | 一个 map | map + 频次 map | **三件套**:kvMap + freqMap + minFreq |
| 同频次排序 | 没考虑 | **频次内部 LRU** | 每个频次桶都是双向链表(最新插在头) |
| minFreq 维护 | 每次扫一遍 | Put/Get 时维护 | 删空时**不一定改 minFreq**(Put 新值会重置)|
| vs LRU | 不清楚 | LFU 看次数,LRU 看时间 | **频次不会衰减**问题 + W-TinyLFU 解决 |
| 并发 | 没想 | 大锁 | 分片 / Read-Through Cache 模式 |

---

## 二、为什么不能用堆

朴素方案:**小顶堆**按频次排序。
- Get:更新频次 → 调整堆位置 → **O(log N)**
- 淘汰:取堆顶 → **O(log N)**

题目要求 **O(1)**,堆**直接 fail**。必须用频次桶。

---

## 三、思路推导:频次桶

### 3.1 核心数据结构

```text
LFUCache
  ├─ kvMap     map[Key]*node       ← key → 节点
  ├─ freqMap   map[int]*DLinkedList ← 频次 → 该频次下的所有节点(LRU 顺序)
  ├─ minFreq   int                  ← 当前最小频次(用于 O(1) 找淘汰目标)
  └─ capacity  int

每个 node:
  - key, value
  - freq      ← 当前访问频次
  - prev/next ← 在所在频次桶的链表里的位置
```

### 3.2 Get(key) 流程

```text
1. kvMap[key] 没有 → 返回 -1
2. 有 → 拿到 node
   a. 从 freqMap[node.freq] 链表删掉 node
   b. node.freq++
   c. 加到 freqMap[node.freq] 链表头
   d. 如果原 freqMap[oldFreq] 空了 且 oldFreq == minFreq → minFreq++
   e. 返回 node.value
```

### 3.3 Put(key, value) 流程

```text
case A: key 已存在 → 更新 value + Get 一遍升频
case B: key 不存在
   B1: 容量未满 → 直接加到 freqMap[1] 头,minFreq = 1
   B2: 满 → 淘汰 freqMap[minFreq] 链表尾(最久未用),再加新节点
       新节点 freq = 1,加到 freqMap[1] 头,minFreq = 1
```

**关键不变式**:
- 每次插入新 key,**minFreq 必须重置为 1**(新 key 频次是 1)
- minFreq 只在"原 minFreq 的桶空了"时**才可能 ++**(Get 升频导致)
- 不需要扫描找 minFreq,**全程维护**

---

## 四、完整代码(LeetCode 460 标准解法)

```go
package lfu

type node struct {
    key, value, freq int
    prev, next       *node
}

type list struct {
    head, tail *node // 哨兵
    size       int
}

func newList() *list {
    h, t := &node{}, &node{}
    h.next, t.prev = t, h
    return &list{head: h, tail: t}
}

func (l *list) pushFront(n *node) {
    n.prev = l.head
    n.next = l.head.next
    l.head.next.prev = n
    l.head.next = n
    l.size++
}

func (l *list) remove(n *node) {
    n.prev.next = n.next
    n.next.prev = n.prev
    l.size--
}

func (l *list) popTail() *node {
    if l.size == 0 {
        return nil
    }
    n := l.tail.prev
    l.remove(n)
    return n
}

// ---------- LFUCache ----------

type LFUCache struct {
    capacity int
    minFreq  int
    kv       map[int]*node
    freq     map[int]*list
}

func New(capacity int) *LFUCache {
    return &LFUCache{
        capacity: capacity,
        kv:       make(map[int]*node),
        freq:     make(map[int]*list),
    }
}

func (c *LFUCache) Get(key int) int {
    n, ok := c.kv[key]
    if !ok {
        return -1
    }
    c.touch(n)
    return n.value
}

func (c *LFUCache) Put(key, value int) {
    if c.capacity <= 0 {
        return
    }
    if n, ok := c.kv[key]; ok {
        n.value = value
        c.touch(n)
        return
    }
    // 新 key
    if len(c.kv) >= c.capacity {
        // 淘汰 minFreq 链表的尾(最久未用)
        lru := c.freq[c.minFreq]
        evict := lru.popTail()
        delete(c.kv, evict.key)
    }
    n := &node{key: key, value: value, freq: 1}
    c.kv[key] = n
    if c.freq[1] == nil {
        c.freq[1] = newList()
    }
    c.freq[1].pushFront(n)
    c.minFreq = 1 // 新 key 必然让 minFreq 回到 1
}

// touch:把 node 从老频次桶搬到新频次桶,并维护 minFreq
func (c *LFUCache) touch(n *node) {
    oldList := c.freq[n.freq]
    oldList.remove(n)
    if oldList.size == 0 {
        delete(c.freq, n.freq)
        if c.minFreq == n.freq {
            c.minFreq++ // 原最小频次的桶空了 → 顺势 ++
        }
    }
    n.freq++
    if c.freq[n.freq] == nil {
        c.freq[n.freq] = newList()
    }
    c.freq[n.freq].pushFront(n)
}
```

**讲解时主动点出的细节**:

| 设计 | 资深点 |
| --- | --- |
| `freq[f]` 是双向链表 | **同频次按 LRU**——头是最新,尾是最久未用 |
| 哨兵节点 head/tail | **避免空链表特判**,删尾不用判空 |
| `popTail` 淘汰 | **同频次 LRU**,O(1) |
| `minFreq` 三处维护 | (1) Put 新 key → minFreq=1 (2) touch 时原桶空 → minFreq++ (3) 其他不动 |
| Put 新 key 必 reset minFreq | **新 key 频次 1,minFreq 不可能比 1 还小** |
| `delete(c.freq, n.freq)` 空桶清理 | 防止 freqMap 无限膨胀 |

---

## 五、minFreq 的微妙之处

**为什么 minFreq 不需要"全局扫描重算"**:

1. 只有两个事件会改变 minFreq:
   - **Put 新 key** → `minFreq = 1`(强制重置)
   - **touch 让原 minFreq 桶变空** → `minFreq++`

2. 其他情况下 minFreq 必然正确:
   - touch 一个 freq > minFreq 的节点 → minFreq 不变
   - 淘汰 minFreq 桶尾 → 即使桶空,**立刻会插入新 key 把 minFreq 设回 1**,所以中间状态不重要

**踩过的坑**:有人在淘汰时也想 `minFreq++`——错!淘汰后**马上会插新 key**,minFreq 会变 1。

---

## 六、LFU vs LRU 对比

| 维度 | [LRU](04-lru-cache.md) | LFU |
| --- | --- | --- |
| 淘汰依据 | **最久未用** | **最少使用** |
| 数据结构 | map + 双向链表 | map + 频次 map + minFreq |
| 实现难度 | 中(LeetCode 146 Medium)| 难(LeetCode 460 Hard)|
| 时间复杂度 | O(1) | O(1) |
| 抗"扫描攻击" | **差**(一次大扫描污染整个缓存)| **好**(扫描的 key 频次 1,优先淘汰)|
| 适应性 | **强**(热点变化快) | **差**(老热点频次高,新热点上不来)|
| 内存开销 | 一个链表 | **多个链表**(频次桶) |

### 6.1 LRU 的痛点:扫描污染

```text
缓存里有 100 个热 key,被高频访问。
来一个 1000 万 key 的全表扫描 → LRU 把热 key 全挤掉
扫描结束后缓存命中率暴跌
```

LFU 应对:扫描 key 频次都是 1,**热 key(频次几千)不会被淘汰**。

### 6.2 LFU 的痛点:频次不衰减

```text
老 key 频次累积到 100 万,但已经 1 小时没访问了
新热点 key 频次 1000,**永远干不掉老 key**
```

→ 老热点"霸座",新热点上不来。

### 6.3 解决:W-TinyLFU(Caffeine / Ristretto)

W-TinyLFU 是现代缓存的事实标准(Caffeine 默认、Ristretto 默认):
- **window LRU**(承接新访问)+ **main LFU**(承接稳定热点)
- LFU 用 **Count-Min Sketch** 估算频次(不存精确值,内存极小)
- **频次定期衰减**(每 N 次访问全员频次 / 2)
- 解决了 LFU 的"老 key 霸座"+ LRU 的"扫描污染"

[W-TinyLFU 论文](https://arxiv.org/abs/1512.00727) — 资深主动提一句:"业务里我用 [Ristretto](https://github.com/dgraph-io/ristretto)"。

---

## 七、并发安全 LFU

朴素方案:**全局 Mutex 包一层**,所有 Get/Put 串行。

```go
type ConcurrentLFU struct {
    mu  sync.Mutex
    lfu *LFUCache
}

func (c *ConcurrentLFU) Get(k int) int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.lfu.Get(k)
}
```

> **注意:不能用 RWMutex**——因为 Get 也会**改频次(touch)**,本质是写操作。和 LRU 完全一样,见 [04-lru-cache.md](04-lru-cache.md)。

**进阶优化**:
- **分片**([12-sharded-map.md](12-sharded-map.md) 思路):按 key hash 分 N 个 LFU,每片独立 Mutex
- **批量更新**:Get 时不立刻 touch,**用环形 buffer 收集**,定期批量更新频次(Caffeine 的做法)

---

## 八、测试(LeetCode 460 样例)

```go
func main() {
    c := New(2)
    c.Put(1, 1)
    c.Put(2, 2)
    fmt.Println(c.Get(1))    // 1, freq(1)=2
    c.Put(3, 3)              // 容量满,淘汰 freq 最小的 → key=2(freq=1)
    fmt.Println(c.Get(2))    // -1
    fmt.Println(c.Get(3))    // 3, freq(3)=2
    c.Put(4, 4)              // 满,freq=2 有 {1,3},淘汰最久未用 → key=1
    fmt.Println(c.Get(1))    // -1
    fmt.Println(c.Get(3))    // 3
    fmt.Println(c.Get(4))    // 4
}
```

---

## 九、典型坑

| 坑 | 现象 | 修复 |
| --- | --- | --- |
| Get 当只读 → 用 RWMutex | 数据竞争(touch 改频次) | 用 Mutex,Get 也是写 |
| 淘汰后没把 minFreq=1 | 下次 Put 找不到正确淘汰目标 | Put 新 key 强制 `minFreq = 1` |
| 没维护 minFreq | 每次淘汰要 O(N) 找最小 | 增量维护(三处更新) |
| 空桶不删 | freq map 无限膨胀 | `if list.size==0 → delete` |
| 同频次没 LRU | 同频次任选一个淘汰,不符合题目 | 每频次桶是双向链表 |
| Put 0 容量 | panic / 死循环 | 校验 `capacity > 0` |
| 频次不衰减 | 老热点霸座 | 用 W-TinyLFU(Ristretto) |
| 没哨兵 | popTail 要判空 | head/tail 哨兵节点 |

---

## 十、和工业级缓存的差距

| 维度 | 本文实现 | Caffeine / Ristretto |
| --- | --- | --- |
| 算法 | 经典 LFU | **W-TinyLFU** |
| 频次存储 | int 精确 | Count-Min Sketch(4-bit × N)|
| 频次衰减 | 没有 | **定期 / 2** |
| 并发模型 | 全局 Mutex | **批量更新 + 异步驱逐 + 分片** |
| 命中率 | 一般 | **接近最优替换** |
| 内存 | 高(精确频次) | 低 |

资深表达:"面试写经典 LFU 主要展示数据结构功底。**生产用 Ristretto / Caffeine**——它们用 W-TinyLFU + Count-Min Sketch + 异步驱逐,**命中率接近 Belady 最优**,内存还小。"

---

## 十一、现场表达模板

> "LFU 比 LRU 难一档,核心是**怎么把'找最小频次'+'同频次 LRU'都做到 O(1)**。
>
> 数据结构三件套:
> 1. `kv map[Key]*node`:key → 节点
> 2. `freq map[int]*DList`:频次 → 该频次下所有节点(**双向链表,头是最新,尾是最久未用**)
> 3. `minFreq int`:当前最小频次,**全程维护**,O(1) 找淘汰目标
>
> Put / Get 都通过 `touch(n)`:从 freq[n.freq] 删 → freq++ → 加到 freq[n.freq] 头。
> minFreq 三处维护:
> - **Put 新 key → minFreq = 1**(强制)
> - touch 让原 minFreq 桶变空 → minFreq++
> - 淘汰不动 minFreq(立刻插新 key 会重置)
>
> 同频次桶用**双向链表 + 哨兵节点**,popTail 淘汰最久未用,O(1)。
>
> 关键的对比:
> - **LRU 防不了扫描攻击**(一次大扫描污染缓存),**LFU 防得住**(扫描 key 频次 1,优先淘汰)
> - **LFU 防不了'老 key 霸座'**(累计频次太高,新热点上不来)
> - 现代缓存用 **W-TinyLFU**(Caffeine / Ristretto)双重解决:window LRU + main LFU + Count-Min Sketch + 定期衰减
>
> 并发:不能用 RWMutex(Get 也改频次),要么全局 Mutex,要么分片 + 异步批量更新。"

---

## 十二、一句话总结

> **LFU = kv map + freq map(每频次一个双向链表)+ minFreq**,严格 O(1);
>
> - **同频次桶按 LRU**——头是最新,尾是最久未用,O(1) popTail 淘汰
> - **minFreq 全程增量维护**:Put 新 key 必 reset 为 1;touch 让原桶空时 ++;其他不动
> - **不能用 RWMutex**(Get 也写)
> - **LFU 防扫描攻击**,但**老 key 容易霸座** → 工业用 [W-TinyLFU](https://arxiv.org/abs/1512.00727)(Caffeine / Ristretto)
> - 比 [LRU](04-lru-cache.md) 难一档,LeetCode 460 Hard;面试写经典 LFU,生产用 Ristretto
