# LRU 缓存(Least Recently Used)

> **题目**:实现一个 LRU 缓存,支持 O(1) 的 `Get(key)` 和 `Put(key, val)`,容量满时淘汰**最久未使用**的。
>
> **LeetCode 146**——所有大厂手写题里出场频率第一名。
>
> 考查:**双向链表 + map 配合 + 哨兵节点 + 并发安全 + LRU vs LFU 取舍**。

---

## 一、考点拆解

| 考点 | 不及格 | 合格 | 优秀 |
| --- | --- | --- | --- |
| 数据结构 | 数组 / 单链表(O(N))| **双向链表 + map** | 解释为什么不能少任一个 |
| 复杂度 | Get/Put O(N) | **均 O(1)** | 能逐步推导 |
| 哨兵节点 | 没用,边界 if 满天飞 | **head/tail 哨兵** | 解释好处 |
| 并发安全 | 没考虑 | 整体 Mutex | 分段锁 / RWMutex 取舍 |
| 进阶 | - | TTL / 泛型 | LFU / W-TinyLFU / 分片 |

---

## 二、思路推导

### 2.1 为什么是"双向链表 + map"组合

| 单用 | 问题 |
| --- | --- |
| **只用 map** | 找 key 是 O(1),但**找不到"最老的 key"** → 淘汰要 O(N) 遍历 |
| **只用链表** | 找最老的 O(1)(尾部),但**找指定 key** 要 O(N) |
| **双向链表 + map** | map 定位 key → 拿到链表节点 → O(1) 移到头部 / 删尾部 ✅ |

> 单链表为啥不行?**删除一个节点需要知道前驱**——单链表得 O(N) 找前驱,双向链表 O(1)。

### 2.2 LRU 的两个操作如何 O(1)

```text
Get(k):
  1. map 查 k → node                    O(1)
  2. node 从链表当前位置删除            O(1) ← 双向链表才行
  3. node 移到链表头(最近使用)          O(1)

Put(k, v):
  if 存在: 更新 val + 移到头
  if 不存在:
    if 满: 删尾节点(map 也删)            O(1)
    新节点插入头 + map 加 k→node          O(1)
```

### 2.3 哨兵节点(关键技巧)

```text
不用哨兵:
  插入第一个节点要判 head==nil
  删尾节点要判 size==1
  → 一堆边界 if

用 head/tail 双哨兵(空节点):
  head ⇄ [真实节点们] ⇄ tail
  插入头:固定 head.next 操作
  删除尾:固定 tail.prev 操作
  → 永不为空,零边界判断
```

资深答题必用哨兵,**省一半代码,逻辑还清晰**。

---

## 三、完整实现(单线程,LeetCode 风格)

```go
package lru

type node struct {
    key, val   int
    prev, next *node
}

type LRUCache struct {
    cap   int
    m     map[int]*node
    head  *node // 哨兵:head.next 是最新
    tail  *node // 哨兵:tail.prev 是最老
}

func New(capacity int) *LRUCache {
    head := &node{}
    tail := &node{}
    head.next = tail
    tail.prev = head
    return &LRUCache{
        cap:  capacity,
        m:    make(map[int]*node, capacity),
        head: head,
        tail: tail,
    }
}

// Get 命中则移到头部,返回 -1 表示未命中
func (c *LRUCache) Get(key int) int {
    n, ok := c.m[key]
    if !ok {
        return -1
    }
    c.moveToFront(n)
    return n.val
}

func (c *LRUCache) Put(key, val int) {
    if n, ok := c.m[key]; ok {
        n.val = val
        c.moveToFront(n)
        return
    }
    // 容量满 → 删最老的
    if len(c.m) == c.cap {
        old := c.tail.prev
        c.remove(old)
        delete(c.m, old.key)
    }
    n := &node{key: key, val: val}
    c.addToFront(n)
    c.m[key] = n
}

// addToFront 插入到 head 之后
func (c *LRUCache) addToFront(n *node) {
    n.prev = c.head
    n.next = c.head.next
    c.head.next.prev = n
    c.head.next = n
}

// remove 摘除一个节点(双向链表 O(1))
func (c *LRUCache) remove(n *node) {
    n.prev.next = n.next
    n.next.prev = n.prev
    n.prev, n.next = nil, nil // 帮 GC
}

func (c *LRUCache) moveToFront(n *node) {
    c.remove(n)
    c.addToFront(n)
}
```

**讲解时主动点出的细节**:

| 设计 | 资深点 |
| --- | --- |
| `key, val` 都存在 node 上 | 淘汰时**必须知道 key 才能 `delete(m, key)`**,只存 val 不够 |
| 哨兵 `head`/`tail` | 不存数据,只为消除边界判断 |
| `remove` 里 `n.prev, n.next = nil, nil` | 不置 nil 也对,但**长链路下旧节点会被链表引用,GC 不掉** |
| `len(c.m) == c.cap` | 用 map 长度判容量,**别用单独维护的 size 字段**(容易和 map 不一致)|

---

## 四、并发安全版(资深加分)

### 4.1 最简单:整体 Mutex

```go
type SafeLRU struct {
    mu sync.Mutex
    c  *LRUCache
}

func (s *SafeLRU) Get(key int) (int, bool) {
    s.mu.Lock()
    defer s.mu.Unlock()
    v := s.c.Get(key)
    return v, v != -1
}

func (s *SafeLRU) Put(key, val int) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.c.Put(key, val)
}
```

> 注意:**不能用 RWMutex**!`Get` 看起来是读,实际上要**移动节点**(改链表结构),所以必须写锁。

### 4.2 分段锁(高并发优化)

整体一把锁在高 QPS 下成为瓶颈。分段:

```go
type ShardedLRU struct {
    shards [16]*SafeLRU
}

func (s *ShardedLRU) shard(key string) *SafeLRU {
    h := fnv.New32a()
    h.Write([]byte(key))
    return s.shards[h.Sum32()%16]
}

func (s *ShardedLRU) Get(key string) (interface{}, bool) {
    return s.shard(key).Get(key)
}
```

**代价**:
- 每个 shard 独立淘汰,**全局 LRU 序略有偏差**(可接受)
- 容量要除以 shard 数

工业级实现(`groupcache` / `bigcache` / `freecache`)都是分段的。

---

## 五、测试

```go
func main() {
    c := New(3)
    c.Put(1, 100)
    c.Put(2, 200)
    c.Put(3, 300)
    fmt.Println(c.Get(1))  // 100,1 升为最新
    c.Put(4, 400)          // 容量满,淘汰最老的 2
    fmt.Println(c.Get(2))  // -1
    fmt.Println(c.Get(3))  // 300
    fmt.Println(c.Get(4))  // 400
}
```

LeetCode 146 直接通过。

---

## 六、进阶变体

### 6.1 泛型版(Go 1.18+)

```go
type Cache[K comparable, V any] struct {
    cap  int
    m    map[K]*node[K, V]
    head, tail *node[K, V]
}

type node[K comparable, V any] struct {
    key  K
    val  V
    prev, next *node[K, V]
}
```

写法基本一样,把 `int` 换成 `K`/`V` 即可。资深表达必带泛型版本。

### 6.2 带 TTL 的 LRU(实战常见)

```go
type entry struct {
    key, val   int
    expireAt   int64 // 纳秒
    prev, next *entry
}

func (c *Cache) Get(key int) (int, bool) {
    n, ok := c.m[key]
    if !ok {
        return 0, false
    }
    if n.expireAt > 0 && time.Now().UnixNano() > n.expireAt {
        c.remove(n)
        delete(c.m, key)
        return 0, false
    }
    c.moveToFront(n)
    return n.val, true
}
```

**资深点**:TTL 过期是**惰性删除**(读时判断),不主动扫,**避免后台 goroutine**。如果要主动清理,加个定时器扫尾部即可。

### 6.3 LRU vs LFU vs W-TinyLFU(必懂对比)

| 算法 | 淘汰依据 | 优点 | 缺点 | 场景 |
| --- | --- | --- | --- | --- |
| **LRU** | 最久未访问 | 实现简单,O(1)| 偶发大查询**冲刷缓存** | 通用 |
| **LFU** | 访问频次最低 | 抗冲刷,热点不被挤掉 | 老热点"赖着不走"|"老用户"特征明显的场景 |
| **W-TinyLFU** | LFU + 时间衰减 + 准入过滤 | 兼具 LRU/LFU 优点 | 实现复杂(Caffeine / Ristretto) | **现代缓存首选** |

> 资深表达:"业务里大数据扫描会**冲刷 LRU**——一次性扫了一堆冷数据把热数据顶出去。Redis 4.0+ 改用 **LFU** 解决这个问题。**Ristretto / Caffeine** 用的是 **W-TinyLFU**,综合最优。"

### 6.4 LFU 简易实现思路

```text
难点:LFU 要按"频次"排序,频次相同时按"时间"
双层结构:
  freqMap: freq → 双向链表(同频次按 LRU 排)
  keyMap:  key → node(包含 freq)
访问时:freq+1,从老链表移到新链表头
淘汰:从最小 freq 的链表尾删除
```

实现细节多,LeetCode 460 是经典题。

### 6.5 写多读少 vs 读多写少

| 模式 | 推荐 |
| --- | --- |
| 读多写少 | LRU 整体 Mutex 即可,Get 也是写操作所以不能 RWMutex |
| 写多读少 | 分段锁 / **无锁队列存"访问事件"+ 后台 goroutine 批量更新链表**(Caffeine 思路) |

---

## 七、典型坑

| 坑 | 现象 | 修复 |
| --- | --- | --- |
| 只存 val,不存 key 在 node 里 | 淘汰时不知道 map 删哪个 key | node 同时存 key |
| 不用哨兵 | 一堆 `if head == nil` 边界判断 | 加 head/tail 哨兵 |
| Get 用 RWMutex 加读锁 | 数据竞争(Get 要改链表)| 必须 Mutex |
| 容量用单独 size 字段 | 和 map 长度不一致 | 直接 `len(m)` |
| 单链表实现 | 删除节点要 O(N) 找前驱 | 必须双向链表 |
| remove 后不置 nil | 长跑下被链表残留引用,GC 不掉 | `n.prev = nil; n.next = nil` |
| 容量 0 / 负数 | panic | 构造时校验 |
| 并发场景共享底层 LRU | 数据错乱 | 整体 Mutex 或分段锁 |

---

## 八、为什么不用 `container/list`

标准库 `container/list` 是双向链表,功能够用,但:
- 节点类型是 `*list.Element`,值是 `interface{}` → **每次断言开销**
- 现代 Go 1.18+ 用泛型自己实现更优

LeetCode 提交里 **自己手写 + 哨兵 + 内嵌结构体** 是最快的写法。面试官也希望你**手写**而不是调标准库,**展示对数据结构的理解**。

---

## 九、现场表达模板

> "LRU 必须 **O(1)** Get 和 Put,所以选**双向链表 + map**:
>
> - **map** 提供 O(1) 查 key
> - **双向链表** 提供 O(1) 删除指定节点和淘汰尾部
> - 单用 map 找不到最老的,单用链表查 key 要 O(N),两者缺一不可
>
> 实现技巧是用 **head/tail 哨兵节点**,这样所有边界判断都不需要,代码量减半。
> 节点里要**同时存 key 和 val**,因为淘汰时要拿 key 回去删 map。
>
> 并发安全的话,**不能用 RWMutex**——Get 也要移动节点,本质是写操作。
> 高并发场景用**分段锁**,工业实现像 groupcache / bigcache 都是分段的。
>
> LRU 的痛点是**偶发大查询冲刷热点**,所以 Redis 4.0+ 用 **LFU**,
> Caffeine / Ristretto 用 **W-TinyLFU** 才是现代最优解。"

---

## 十、一句话总结

> **LRU = 双向链表 + map + 头尾哨兵**;
>
> - Get/Put **O(1)** 靠 map 定位 + 双向链表 O(1) 删除
> - node 同时存 **key 和 val**(淘汰时要删 map)
> - 并发版用 **Mutex(不是 RWMutex)** 或**分段锁**
> - LRU 抗不住扫描冲刷,真实工程用 **LFU / W-TinyLFU**(Caffeine / Ristretto / Redis)
