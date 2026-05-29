# 跳表(Skip List)

> **题目**:实现一个跳表,支持 `Insert(score, key) / Delete(key) / Search(score) / RangeByScore(min, max)`,期望 O(log N)。
>
> 考查:**多级链表 + 随机高度**、**为什么 Redis ZSet 用跳表而不是红黑树**、可选无锁版(atomic CAS forward 指针)。
>
> LeetCode 1206。

跳表是 Redis ZSet 的底层结构(配合 dict),也是 LevelDB MemTable 的索引。**比红黑树简单 50% 代码量,性能相当,且天然支持范围查询**——这是它在工业界胜出的关键。

---

## 一、考点拆解

| 考点 | 不及格 | 合格 | 优秀 |
| --- | --- | --- | --- |
| 数据结构 | 不知道 | 多级链表 + 随机高度 | **每层都是有序链表,高层稀疏低层全量** |
| 高度生成 | 固定 | 随机抛硬币(p=0.5)| **几何分布,期望 log N** |
| 时间复杂度 | 不清楚 | 期望 O(log N) | **最坏 O(N) 但概率极低**(不像 RBT 严格保证) |
| vs 红黑树 | 不清楚 | 实现简单 | **范围查询天然 O(K) + 并发友好**(无锁可行) |
| Redis 应用 | 不知道 | ZSet 用跳表 | **跳表 + dict 双索引**,score 查 / member 查都 O(log N) / O(1) |
| 无锁版 | 没听过 | 知道存在 | **atomic CAS forward 指针**,Java ConcurrentSkipListMap |

---

## 二、为什么 Redis ZSet 用跳表

Antirez 在 [Redis 早期讨论](http://web.archive.org/web/20141220033712/http://www.redislabs.com/blog/skip-list-vs-red-black-tree)里给了三个理由:

| 理由 | 跳表 | 红黑树 |
| --- | --- | --- |
| 范围查询 | **天然 O(log N + K)** | 需要中序遍历 |
| 实现复杂度 | **~200 行 C** | ~500 行 C(旋转 / 染色) |
| 内存可控 | **链表节点** | 节点更紧凑但旋转开销大 |
| 并发友好 | **无锁版成熟** | 旋转破坏多个节点,无锁难 |

**资深表达**:"Antirez 说'红黑树要写好太难,跳表用 200 行 C 搞定,性能差不多还支持范围查询'——Redis 选跳表是**工程取舍**,不是性能必然。"

---

## 三、思路:多级链表 + 随机高度

### 3.1 结构示意

```text
Level 3:  H ─────────────────────────────────► 7 ─────────────► nil
Level 2:  H ──────────► 3 ──────────────────► 7 ──────► 9 ────► nil
Level 1:  H ──────► 2 ► 3 ──────► 5 ────────► 7 ► 8 ── 9 ────► nil
Level 0:  H ► 1 ► 2 ► 3 ► 4 ► 5 ► 6 ► 7 ► 8 ► 9 ► 10 ► nil   (全量有序)

H = 头哨兵
```

- Level 0 是全量有序链表
- 每一层是上一层的"稀疏副本"(每个节点以概率 p=0.5 升级)
- 查找时**从最高层往下走**——高层跳得远,低层精细定位

### 3.2 查找 score=8

```text
L3: H → 7(< 8,继续)→ nil(降级到 L2)
L2: 从 7 → 9(> 8,降级到 L1)
L1: 从 7 → 8 ✓
```

每层期望走 1/p = 2 步,共 log₂N 层 → **期望 O(log N)**。

### 3.3 为什么"随机"高度也能 O(log N)

每个节点高度服从几何分布:
- P(高度 ≥ k) = p^(k-1)
- 期望高度 = 1/(1-p) = 2(取 p=0.5)
- N 个节点期望最大高度 ≈ log₁/ₚ N

**关键洞察**:跳表用**随机数代替平衡操作**——红黑树用复杂的旋转维持平衡,跳表用概率维持期望平衡。**代码简单很多**。

---

## 四、完整代码

```go
package skiplist

import (
    "math/rand"
)

const (
    maxLevel = 32       // 2^32 个节点够用
    pFactor  = 0.25     // 每层晋升概率(Redis 用 0.25)
)

type node struct {
    score   float64
    key     string
    forward []*node // 每层的下一个节点;len(forward) == 节点高度
}

type SkipList struct {
    head  *node
    level int // 当前最大已用层数(不是 maxLevel)
    size  int
}

func New() *SkipList {
    return &SkipList{
        head:  &node{forward: make([]*node, maxLevel)},
        level: 1,
    }
}

// randomLevel 几何分布:返回 1~maxLevel
func (s *SkipList) randomLevel() int {
    lvl := 1
    for lvl < maxLevel && rand.Float64() < pFactor {
        lvl++
    }
    return lvl
}

// Search 返回 score 对应的 key,没有返回 ""
func (s *SkipList) Search(score float64) (string, bool) {
    cur := s.head
    for i := s.level - 1; i >= 0; i-- {
        for cur.forward[i] != nil && cur.forward[i].score < score {
            cur = cur.forward[i]
        }
    }
    cur = cur.forward[0]
    if cur != nil && cur.score == score {
        return cur.key, true
    }
    return "", false
}

// Insert 插入一个 (score, key);允许 score 重复(Redis ZSet 用 score+member 双关键字,简化版只用 score)
func (s *SkipList) Insert(score float64, key string) {
    update := make([]*node, maxLevel) // 每层的"插入位置前驱"
    cur := s.head
    for i := s.level - 1; i >= 0; i-- {
        for cur.forward[i] != nil && cur.forward[i].score < score {
            cur = cur.forward[i]
        }
        update[i] = cur
    }

    lvl := s.randomLevel()
    if lvl > s.level {
        for i := s.level; i < lvl; i++ {
            update[i] = s.head // 新层从 head 起
        }
        s.level = lvl
    }

    newNode := &node{score: score, key: key, forward: make([]*node, lvl)}
    for i := 0; i < lvl; i++ {
        newNode.forward[i] = update[i].forward[i]
        update[i].forward[i] = newNode
    }
    s.size++
}

// Delete 删除第一个 score 匹配的节点
func (s *SkipList) Delete(score float64) bool {
    update := make([]*node, maxLevel)
    cur := s.head
    for i := s.level - 1; i >= 0; i-- {
        for cur.forward[i] != nil && cur.forward[i].score < score {
            cur = cur.forward[i]
        }
        update[i] = cur
    }
    target := cur.forward[0]
    if target == nil || target.score != score {
        return false
    }
    for i := 0; i < s.level; i++ {
        if update[i].forward[i] != target {
            break
        }
        update[i].forward[i] = target.forward[i]
    }
    // 顶层可能因为删空而下降
    for s.level > 1 && s.head.forward[s.level-1] == nil {
        s.level--
    }
    s.size--
    return true
}

// RangeByScore 返回 [min, max] 区间所有 key,按 score 升序
func (s *SkipList) RangeByScore(min, max float64) []string {
    var result []string
    cur := s.head
    // 在 L0 上从 min 起步:先用高层跳过 < min
    for i := s.level - 1; i >= 0; i-- {
        for cur.forward[i] != nil && cur.forward[i].score < min {
            cur = cur.forward[i]
        }
    }
    cur = cur.forward[0]
    for cur != nil && cur.score <= max {
        result = append(result, cur.key)
        cur = cur.forward[0]
    }
    return result
}
```

**讲解时主动点出的细节**:

| 设计 | 资深点 |
| --- | --- |
| `forward []*node` 每节点独立分配 | 节点高度 = `len(forward)`,**只占自己用的层数内存** |
| 头节点 forward 长度 = maxLevel | 头永远是最高的,简化插入逻辑 |
| `update[]` 数组 | **关键!**记录每层的"插入位置前驱",所有插入都基于它 |
| **p = 0.25**(Redis 选) | 经典是 0.5,Redis 改 0.25 让高层更稀疏,**总指针数更少**,缓存友好 |
| `s.level` 动态调整 | 删空后 level 下降,**避免无用层浪费查找时间** |
| 范围查询 O(log N + K) | 高层定位起点,L0 顺着 forward[0] 扫,**天然有序** |

---

## 五、并发版:两个层次

### 5.1 简单方案:全局 RWMutex

```go
type ConcurrentSkipList struct {
    mu sync.RWMutex
    sl *SkipList
}

func (c *ConcurrentSkipList) Search(s float64) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.sl.Search(s)
}

func (c *ConcurrentSkipList) Insert(s float64, k string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.sl.Insert(s, k)
}
```

读多写少场景**够用**,简单可靠。

### 5.2 无锁版(资深加分)

Java `ConcurrentSkipListMap` / Folly `ConcurrentSkipList` 是真无锁——**所有 forward 指针都是 atomic,用 CAS 修改**:

```go
type lockfreeNode struct {
    score   float64
    key     string
    forward []atomic.Pointer[lockfreeNode]
    marked  atomic.Bool // 标记已逻辑删除
}

// Insert 简化思路:
// 1. 先找好 update[] 和 successor[](每层后继)
// 2. 从底层往上,CAS pred.forward[i] 从 successor 改到 newNode
// 3. CAS 失败 → 别人改了,重新定位
//
// Delete 用"标记 + 后续清理":
// 1. 把节点 marked = true(原子)
// 2. CAS pred.forward 跳过该节点
// 3. 其他线程看到 marked=true 会主动帮忙清理
```

**关键挑战**:
- **逻辑删除 + 物理删除分离**:防止"读到一半节点被删导致 NPE"
- **多层 CAS 不能原子**:底层 CAS 成功后,中间被其他线程"看到了一半",但**跳表不要求层间一致**——查询时从高层降级,看不到也没关系,下次走 L0 还是能找到

工业级实现极其复杂(Folly 1500+ 行 C++),业务里**几乎不用自己写**,直接用 `sync.Map` 或 RBLock 包装。

---

## 六、和红黑树 / B+ 树对比

| 维度 | 跳表 | 红黑树 | B+ 树 |
| --- | --- | --- | --- |
| 查找 | 期望 O(log N) | 严格 O(log N) | O(log N) |
| 插入 | 期望 O(log N) | O(log N) | O(log N) |
| 范围查询 | **O(log N + K)** 天然 | 中序遍历 | **O(log N + K)** 天然 |
| 实现复杂度 | **低**(200 行) | 高(旋转 + 染色) | 高(分裂 / 合并) |
| 内存 | 较高(每节点多指针) | 紧凑 | 紧凑(节点装多 key) |
| 并发 | **无锁可行** | 旋转破坏多节点,难 | 难 |
| 磁盘友好 | ✗ | ✗ | **✓**(节点 = 页大小) |
| 典型用途 | Redis ZSet / LevelDB MemTable | C++ map / Linux 内核 | **MySQL InnoDB / 文件系统** |

> **跳表 vs B+ 树**:跳表纯内存场景胜出(简单 + 并发);**磁盘存储用 B+ 树**(节点 = 4KB 页,一次 IO 多 key,见 [03-mysql/22-innodb-internals.md](../03-mysql/22-innodb-internals.md))。

---

## 七、Redis ZSet 实现细节(资深加分)

Redis ZSet **不止跳表**,而是 **跳表 + dict 双索引**:

```text
ZSet
  ├─ zsl  *zskiplist      ← 跳表(按 score 排序,支持范围查询)
  └─ dict *dict           ← 字典(member → score),O(1) 找 score
```

为什么:
- `ZRANGEBYSCORE`(按 score 查)→ 跳表 O(log N + K)
- `ZSCORE member`(按 member 查)→ dict O(1)
- 没 dict,`ZSCORE` 要扫整个跳表 O(N)

**额外优化**:小 ZSet(元素 < 128 且每元素 < 64B)用 **listpack**(紧凑数组),省内存。元素多了再升级成 skiplist+dict——这是 Redis 的[编码转换](../04-redis/17-object-encoding-internals.md)。

**跳表节点改造**:
- 每个 forward 指针带一个 **span**(跳过多少节点)→ 支持 `ZRANK member`(O(log N) 算排名)
- 双向链表(forward + backward)→ 支持 `ZREVRANGEBYSCORE`

---

## 八、典型坑

| 坑 | 现象 | 修复 |
| --- | --- | --- |
| `forward` 数组用 maxLevel 大小 | 每节点都吃 maxLevel * 8 字节,内存爆 | **按实际高度分配**,len(forward) = 节点高度 |
| randomLevel 用 `rand.Intn(32)` | 各层等概率 → 不是几何分布 | **`for rand < p: lvl++`** |
| 顶层删空不下降 | 查询多绕几层 | 删除后 `for s.head.forward[level-1]==nil: level--` |
| 没 update[] 数组直接插 | 多层指针漏改 / 死循环 | **update[] 是模板写法** |
| score 相同时插入位置随意 | Redis 要求 score 相同时按 member 字典序 | 比较条件加 `|| (score == && member <)` |
| 范围查询从 head L0 扫起 | O(N),失去意义 | **先高层定位起点**,再 L0 扫 |
| 用 `sync.Mutex` 的 Search | 高并发读卡 | 用 RWMutex 或无锁版 |
| 直接持指针读 | 并发下 forward 改了,读到野指针 | atomic.Pointer 或锁 |

---

## 九、测试

```go
func main() {
    s := New()
    members := []struct {
        score float64
        key   string
    }{
        {1.5, "a"}, {3.0, "b"}, {2.0, "c"}, {5.5, "d"}, {4.0, "e"},
    }
    for _, m := range members {
        s.Insert(m.score, m.key)
    }

    fmt.Println(s.Search(3.0)) // → b, true
    fmt.Println(s.RangeByScore(2.0, 4.5)) // → [c b e]
    s.Delete(3.0)
    fmt.Println(s.Search(3.0)) // → "", false
}
```

---

## 十、现场表达模板

> "跳表是**多级有序链表 + 随机高度**,期望 O(log N),实现比红黑树简单太多——Redis ZSet、LevelDB MemTable、Java ConcurrentSkipListMap 都在用。
>
> 核心思想:**用随机数代替平衡操作**。每个节点抛硬币决定高度(几何分布),期望最大高度 ≈ log N。查找从最高层往下走,**高层跳得远,低层精细定位**。
>
> 实现关键:
> 1. **每节点 forward 数组按实际高度分配**,不是 maxLevel(否则内存爆)
> 2. **update[] 数组**:插入 / 删除时记录每层的前驱,所有指针修改都基于它
> 3. **p = 0.25**(Redis 选):比经典 0.5 让高层更稀疏,**总指针数少,缓存友好**
> 4. **顶层删空要下降 level**,避免无用层
>
> 为什么 Redis 选跳表不选红黑树:Antirez 原话——'**红黑树写好太难**,跳表 200 行 C 搞定,**性能差不多还天然支持范围查询**'。这是工程取舍。
>
> Redis ZSet 实际是**跳表 + dict 双索引**:跳表按 score 排序 / 范围查,dict 按 member O(1) 取 score。跳表节点还带 **span**(跳过多少元素)支持 ZRANK 排名查询。
>
> 并发:简单场景**RWMutex 包一层**就够;无锁版用 atomic.Pointer + CAS forward 指针 + 逻辑删除标记,实现极其复杂(Folly 1500+ 行),业务里几乎不自己写。
>
> vs B+ 树:**纯内存用跳表**(简单 + 并发友好),**磁盘用 B+ 树**(节点 = 页大小,一次 IO 多 key)。"

---

## 十一、一句话总结

> **跳表 = 多级有序链表 + 随机几何高度(p=0.25)**,期望 O(log N);
>
> - **核心创新**:用**随机数代替平衡操作**(红黑树用旋转,跳表用概率)
> - **范围查询天然 O(log N + K)**,这是跳表胜过红黑树的工程关键
> - **Redis ZSet = 跳表 + dict 双索引**;跳表节点带 span 支持 ZRANK,双向支持 REV 查询
> - **并发友好**:无锁版(atomic.Pointer + 逻辑删除)成熟,Java ConcurrentSkipListMap / Folly
> - **磁盘存储用 [B+ 树](../03-mysql/22-innodb-internals.md)**(页对齐),跳表只在内存场景胜出
> - LeetCode 1206 是简化版,工业级看 Redis `t_zset.c` / Folly `ConcurrentSkipList`
