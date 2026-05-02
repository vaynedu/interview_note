# map

> Go map：链式哈希表，bmap (bucket) + tophash 数组 + 溢出桶；非并发安全，迭代无序

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
