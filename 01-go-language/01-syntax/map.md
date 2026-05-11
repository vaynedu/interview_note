# map

> Go map：链式哈希表，bmap (bucket) + tophash 数组 + 溢出桶；非并发安全，迭代无序

## 〇、核心提炼（5 段式）

### 核心机制（4 条必背）

1. **hmap + bmap 两层结构** - hmap 是 map 的 header，包含若干 bmap（桶），每个 bmap 存 8 个 K/V
2. **tophash 加速查找** - 每个 bmap 头部存 8 个 tophash（哈希值高 8 位），先比 tophash 再比 key
3. **链式溢出桶** - 一个 bmap 满了 → 链一个 overflow bucket（避免立即扩容）
4. **渐进式扩容（增量迁移）** - 触发条件：装载因子 > 6.5 或溢出桶过多；扩容时**双倍 + 增量迁移**

### 核心本质（必懂）

> Go map 的本质是 **"链式哈希表 + 渐进式扩容"**：
>
> - **不是简单数组**：是 hmap + 多个 bmap（每个 bmap 存 8 个 K/V）+ overflow chain
> - **不是 java.HashMap**：Go map 用"开放定址 + 链式溢出"混合，不是纯链式
> - **不是线程安全**：并发读写会触发 fatal error（不是 panic，无法 recover）
>
> **关键事实**：
> - **迭代无序是故意的**：Go 在 1.0 就刻意打乱顺序，防止开发者依赖
> - **删除不释放 bucket**：DELETE 只标记 tophash = empty，bucket 不缩
> - **扩容是渐进的**：触发后不一次性迁移，每次 GET/SET 顺带迁移 1-2 个 bucket
> - **并发安全方案**：sync.Map（特定场景）/ map + Mutex（通用）

### 完整流程（面试必背）

```
hmap 结构（runtime/map.go）:
  type hmap struct {
      count     int            // map 大小
      B         uint8          // bucket 数量 = 2^B
      noverflow uint16         // 溢出桶数量
      hash0     uint32         // hash seed（防 hash 碰撞攻击）
      buckets   unsafe.Pointer // bucket 数组
      oldbuckets unsafe.Pointer // 扩容时的老 bucket
      nevacuate uintptr        // 已迁移的 bucket 数（渐进式扩容用）
  }

bmap 结构（单个 bucket）:
  type bmap struct {
      tophash [8]uint8  // 8 个 K/V 的哈希高 8 位
      // [8]K              // 8 个 key 紧凑排列
      // [8]V              // 8 个 value 紧凑排列
      // overflow *bmap    // 指向溢出桶
  }
  → 一个 bucket 容纳 8 个 K/V
  → K/V 分开存（不是 KVKV 交替）→ 内存对齐 + 缓存友好

写入流程 SET m[k] = v:
  1. hash = hash(k) ^ hmap.hash0  // 加 seed 防碰撞攻击
  2. bucketIdx = hash & (2^B - 1) // 低 B 位定位 bucket
  3. tophash = hash >> (64-8)      // 高 8 位作为 tophash
  4. 在 bucketIdx 对应 bmap 中遍历:
     - 找 tophash 匹配的位置（fast path）
     - tophash 匹配 → 比对完整 key
     - key 匹配 → 更新 value
  5. 没找到 + bmap 有空槽 → 写入空槽
  6. 没找到 + bmap 满了 → 检查 overflow bucket
  7. overflow 链上都满了 → 触发扩容 / 加新 overflow

查找流程 GET m[k]:
  1. 同上算 hash / bucketIdx / tophash
  2. 遍历 bucket + overflow chain
  3. tophash 匹配 → 比 key → 返回
  4. 全部不匹配 → 返回零值

扩容条件（满足任一）:
  - count / 2^B > 6.5（装载因子 > 6.5）
  - noverflow > 阈值（溢出桶太多）

扩容流程（渐进式）:
  1. 触发时不立即迁移
  2. oldbuckets = buckets, buckets = 分配新（2x 或同 size）
  3. 每次 GET/SET 操作顺带迁移 1-2 个 bucket（增量）
  4. nevacuate 记录已迁移数量
  5. 全部迁移完 → 释放 oldbuckets
```

### 4 条核心机制 - 逐点讲透

#### 1. hmap + bmap（为什么 8 个 K/V 一桶）

```
8 个 K/V 一桶的设计:
  - 减少桶数量（指针开销）
  - 利用 CPU 缓存行（典型 64 字节）
  - tophash 数组连续 → SIMD 友好

vs Java HashMap（一桶一节点）:
  Java: Entry[] + 链表 / 红黑树
  Go: bmap[] + overflow chain（一桶 8 元素）
  → Go 内存更紧凑 + 缓存友好

K/V 分开存（不是 KVKV）:
  type bmap {
      tophash [8]uint8
      keys [8]K
      values [8]V
      overflow *bmap
  }
  → 内存对齐（K 和 V 类型大小不同时避免 padding）
  → 缓存友好（连续访问 K 数组）
```

#### 2. tophash 加速

```
作用:
  避免每次都比完整 key（key 可能是 string，比对开销大）

流程:
  hash = hash(k)
  tophash = hash >> (64-8)  // 取高 8 位

  bucket 中：
  for i := 0; i < 8; i++ {
      if bucket.tophash[i] == tophash {
          // 才比完整 key
          if bucket.keys[i] == k {
              return bucket.values[i]
          }
      }
  }

特殊 tophash 值:
  emptyRest (0): 该槽位空 + 后续也空 → 提前结束
  emptyOne (1):  该槽位空（删除后状态）
  evacuatedX/Y (2/3): 扩容中迁移到新桶的标记

性能:
  tophash 比对是 1 字节比较（vs string 比对几十字节）
  → 快 10x+
```

#### 3. 溢出桶（避免立即扩容）

```
为什么有溢出桶:
  bucket 满了不一定要扩容（扩容贵）
  先加 overflow 缓冲一下

链式结构:
  bucket → overflow → overflow → ...

何时真正扩容:
  - 装载因子 > 6.5（平均每桶 6.5 个元素）
  - 或 noverflow > 阈值（溢出桶太多 = 退化为链表）

扩容触发后:
  - 装载因子高 → 双倍扩容（buckets 翻倍）
  - 溢出桶多但装载因子不高 → 等量扩容（重新分布 + 清理溢出桶）
```

#### 4. 渐进式扩容（避免卡顿）

```
传统 rehash 问题:
  一次性把所有元素重 hash 到新桶
  100w 元素 = 几百 ms 卡顿

Go 渐进式:
  1. 触发扩容: 分配新 buckets（2x），保留 oldbuckets
  2. 每次 GET/SET 顺带迁移 1-2 个 bucket
  3. 渐进完成

具体迁移:
  对 oldbucket[i] 中的每个元素:
  - hash 取低 (B+1) 位
  - 比原来多一位决定去 X 桶（新桶 i）还是 Y 桶（新桶 i + 2^B）
  - 标记 oldbucket tophash = evacuatedX/Y

期间访问:
  - GET: 先查新 bucket，没有则查 old（如果未迁移）
  - SET: 写新 bucket
  - 触发迁移 oldbucket[i] 和它的 overflow chain

代价:
  - 期间内存翻倍（new + old 并存）
  - 操作稍慢（要查两处）
  - 但避免单次大卡顿
```

### 一句话总结

> Go map 的核心是：**hmap + bmap 两层结构 + tophash 加速 + 链式溢出桶 + 渐进式扩容**，
> 本质是**链式哈希表 + 8 元素一桶 + 增量迁移**：内存紧凑、缓存友好、避免扩容卡顿。
> **三大坑**：非并发安全（并发读写 fatal）、迭代无序（故意打乱）、删除不缩内存。
> **并发方案**：读多写少 + 稳定 key 用 sync.Map；通用场景用 map + Mutex；超高频用 atomic.Pointer + 整体替换。

---

## 一、核心原理

### 1.1 底层结构

```go
// runtime/map.go
type hmap struct {
    count     int            // 元素个数 (len(m) 直接读这里)
    flags     uint8          // 状态标志(写中/迁移中/迭代中)
    B         uint8          // bucket 数量 = 2^B
    noverflow uint16         // 溢出桶大致数量
    hash0     uint32         // 哈希种子(防 hash 攻击)
    buckets   unsafe.Pointer // 指向 2^B 个 bmap 的数组
    oldbuckets unsafe.Pointer // 扩容时的旧桶数组
    nevacuate uintptr        // 迁移进度
    extra     *mapextra      // 溢出桶链
}

type bmap struct {
    tophash [8]uint8  // 每个 key hash 的高 8 位
    // 后面紧接 8 个 key、8 个 value、1 个 overflow 指针(编译期生成)
}
```

- 一个 bucket 存 8 个 KV，超过用溢出桶链表延伸
- key/value 各自连续存储（不是 KV 交替），便于内存对齐
- `tophash` 用于快速比较，避免每次都比 key 全值

### 1.2 查找流程

1. 用 `hash0 + key` 计算哈希值 `h`
2. 取 `h` 低 B 位定位 bucket（`h & (2^B - 1)`）
3. 取 `h` 高 8 位作 tophash，依次比对 bucket 内 8 个槽
4. tophash 命中再比对 key 全值
5. 没命中走 overflow 链
6. 扩容中还要先去 oldbuckets 看一遍

### 1.3 扩容机制

触发条件：
- **装载因子超过 6.5**（count / 2^B > 6.5）→ 双倍扩容（B+1）
- **溢出桶过多**（noverflow ≥ 2^min(15,B)）→ 等量扩容（B 不变，重新整理）

策略是**渐进式迁移**：每次写操作搬 1~2 个 bucket，避免单次 STW。`oldbuckets` 在迁移完成后才置 nil。

### 1.4 删除

调用 `mapdelete`：找到对应槽，清空 key/value，把 tophash 置为 `emptyOne`。**不会立即释放 bucket**，count 减 1。

### 1.5 迭代无序

迭代起点：随机选一个 bucket、随机选一个槽起步。**Go 故意打乱顺序**，避免开发者依赖未定义行为。

## 二、八股速记

- 底层是 `hmap` + `bmap` 数组，每个 bucket 8 槽，溢出桶链式延伸
- `tophash` = 哈希高 8 位，用于快速预筛选
- 扩容：装载因子 > 6.5 双倍扩容，溢出桶过多等量扩容；**渐进式迁移**
- **非并发安全**：并发读写检测到会直接 `fatal error`（不是 panic，无法 recover）
- 迭代**故意无序**，每次起点随机
- key 必须是**可比较类型**：不能是 slice/map/func，可以是 array/struct（成员都可比较时）
- `len(m)` O(1)，直接读 hmap.count
- 删除 key 不会缩容，长期增删可能浪费内存

## 三、面试真题

**Q1：map 为什么是非并发安全的？**
读时如果碰到正在写（flags 含 hashWriting）会触发 `throw("concurrent map read and map write")`。这是 runtime fatal，不能 recover。设计上为了简化和性能，没用锁；并发场景请用 `sync.RWMutex` 或 `sync.Map`。

**Q2：map 的 key 可以是哪些类型？为什么 slice 不行？**
key 必须可比较（`==` 可用）。基本类型、指针、channel、interface（运行时检查具体类型可比较）、array、所有字段可比较的 struct 都可以。slice/map/function 不可比较，因为它们没有定义良好的相等语义。

**Q3：map 扩容为什么是渐进式？**
一次性搬 2^B 个 bucket 会导致单次写操作耗时不可控（百万级 key 时 STW 严重）。渐进式把搬迁成本均摊到后续每次写，最坏情况单次写多搬 1~2 个 bucket。

**Q4：map 取出的 value 能直接修改字段吗？**
```go
m := map[string]struct{ N int }{"a": {1}}
m["a"].N = 2 // 编译错误: cannot assign to struct field
```
因为 `m["a"]` 返回的是 value 的副本（且 map 内存可能在迁移）。改法：value 用指针 `map[string]*S`，或先取出改完再赋值回去。

**Q5：怎么实现一个有序遍历的 map？**
- 维护一个 `[]key` 切片记录插入顺序
- 或第三方库（如 `orderedmap`）
- Go 语言层面没有，迭代顺序故意随机化

**Q6：map 的删除会缩容吗？**
不会。`delete` 只清槽位和 tophash，bucket 数组不缩。如果一个 map 经历过大量增删导致内存占用大但 count 小，需要重建：`m2 := make(map[K]V, len(m)); for k,v := range m { m2[k]=v }`。

**Q7：sync.Map 适合什么场景？**
官方文档明确：**读多写少**或**多个 goroutine 操作不相交的 key 集合**。它内部用 read/dirty 双 map + atomic，写多场景反而比 `RWMutex+map` 慢。

## 四、手写实现

**并发安全 map（读多写少）：**

```go
type SafeMap[K comparable, V any] struct {
    mu sync.RWMutex
    m  map[K]V
}

func (s *SafeMap[K, V]) Get(k K) (V, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    v, ok := s.m[k]
    return v, ok
}

func (s *SafeMap[K, V]) Set(k K, v V) {
    s.mu.Lock()
    defer s.mu.Unlock()
    if s.m == nil {
        s.m = make(map[K]V)
    }
    s.m[k] = v
}
```

**有序 map（按插入顺序遍历）：**

```go
type OrderedMap[K comparable, V any] struct {
    keys []K
    m    map[K]V
}

func (o *OrderedMap[K, V]) Set(k K, v V) {
    if _, ok := o.m[k]; !ok {
        o.keys = append(o.keys, k)
    }
    o.m[k] = v
}

func (o *OrderedMap[K, V]) Range(f func(K, V) bool) {
    for _, k := range o.keys {
        if !f(k, o.m[k]) {
            return
        }
    }
}
```

## 五、踩坑与最佳实践

### 坑 1：并发读写直接 fatal

```go
m := map[int]int{}
go func() { for { m[1] = 1 } }()
go func() { for { _ = m[1] } }()
// fatal error: concurrent map read and map write
```
defer recover 救不回来，进程直接退出。

### 坑 2：value 是 struct 不能改字段

见 Q4。要么 `*Struct`，要么 `m[k] = newStruct`。

### 坑 3：`for k,v := range m` 修改 map

迭代过程中 `delete(m, k)` 是允许的，但**新增的 key 是否被遍历到未定义**。需要新增就先收集到 slice，循环外处理。

### 坑 4：把大量 key 删掉后 map 不缩容

监控服务持有一个长期 map，每分钟增删 1w，半年后内存占用 GB 级但 len() 只有几千。解决：定期重建。

### 坑 5：nil map 写入 panic

```go
var m map[string]int
m["a"] = 1 // panic: assignment to entry in nil map
```
读取 nil map 返回零值不 panic，但写入必须先 `make`。

### 最佳实践

- 已知容量 `make(map[K]V, n)` 预设 hint，避免多次扩容
- key 用 `string` 时短字符串性能好，长字符串考虑哈希后用 `[16]byte`
- 用 `map[K]struct{}` 当 set，比 `map[K]bool` 省内存
- 高并发读写：先评估 `sync.RWMutex+map`，确认是读多写少且 key 分布广再上 `sync.Map`
- 序列化用途用 `map[string]interface{}` 没问题，业务代码尽量用具体类型
