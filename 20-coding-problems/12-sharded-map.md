# 并发安全 map(分片锁)

> **题目**:实现一个并发安全的 map(`Set / Get / Delete / Range / Len`),要求**高并发下吞吐远超 `Mutex + map`**。
>
> 考查:**分片(sharding) + 每片 RWMutex + 哈希分桶**、和 `sync.Map` 的取舍、`Range` 一致性。

分片锁是高并发 K/V 容器的经典优化:Java `ConcurrentHashMap` 1.7、`go-cache`、`bigcache`、`concurrent-map`(orcaman)都是这套思路。

---

## 一、考点拆解

| 考点 | 不及格 | 合格 | 优秀 |
| --- | --- | --- | --- |
| 分片思路 | 不知道 | 多个 map 分散锁 | hash 决定 shard,**降低锁粒度** |
| 分片数 | 任意拍 | 16/32 | **2 的幂 + 位运算 (hash & (N-1))** 比取模快 |
| 哈希函数 | strings | **fnv32**(标准库) | 知道为什么不用 hash/maphash(用 G 内随机种子) |
| 每片锁选 | 都 Mutex | 读多用 RWMutex | 写多场景 RW 反而慢 |
| 和 sync.Map 对比 | 等同 | 知道 sync.Map 是**特殊优化场景** | **写多 / 频繁更新用分片;读多写少且 key 稳定用 sync.Map** |
| Range 一致性 | 想要全局快照 | **没办法**,只能"每片快照" | 接受"弱一致" |
| Len 实现 | 遍历所有片 | atomic 计数 / 每片 atomic | **强一致的 Len 几乎不可能,业务接受近似值** |

---

## 二、为什么不直接 `Mutex + map`

```go
// ❌ 全局一把锁:写入争抢严重
type SyncMap struct {
    mu sync.Mutex
    m  map[string]interface{}
}
```

压测数据(参考):
- 16 协程并发写,**Mutex + map**:200 万 op/s
- 16 协程并发写,**分 32 片**:**5000 万+ op/s**(粗略 20-30× 提升)

原因:**锁的临界区被 30+ 倍稀释**,16 协程往 32 片随机写,大概率不撞同一把锁。

---

## 三、思路推导

### 3.1 核心结构

```text
ShardedMap
  └─ shards [32]*shard      ← 固定数量(2 的幂)
        └─ shard
              ├─ mu RWMutex
              └─ m  map[string]V
```

### 3.2 路由公式

```go
shardIdx := fnv32(key) & (shardCount - 1)
```

- `& (N-1)` 比 `% N` **快几倍**,但要求 N 是 2 的幂
- `fnv32` 是 Go 标准库 `hash/fnv`,**确定性 + 速度快**

### 3.3 为什么分片数选 16/32

| 分片数 | 优点 | 缺点 |
| --- | --- | --- |
| 太少(4-8) | 内存小 | 锁竞争仍重 |
| **16-32** | **吞吐稳定** | 内存可接受 |
| 太多(256+) | 锁竞争极低 | 内存浪费、Range/Len 变慢 |

经验:**32 是甜点**,Java `ConcurrentHashMap` 1.7 默认也是 16 段。

---

## 四、完整代码

```go
package shardedmap

import (
    "hash/fnv"
    "sync"
)

const shardCount = 32 // 必须是 2 的幂

type shard struct {
    mu sync.RWMutex
    m  map[string]interface{}
}

type ShardedMap struct {
    shards [shardCount]*shard
}

func New() *ShardedMap {
    sm := &ShardedMap{}
    for i := 0; i < shardCount; i++ {
        sm.shards[i] = &shard{m: make(map[string]interface{})}
    }
    return sm
}

func (m *ShardedMap) getShard(key string) *shard {
    h := fnv.New32a()
    h.Write([]byte(key))
    return m.shards[h.Sum32()&(shardCount-1)]
}

func (m *ShardedMap) Set(key string, value interface{}) {
    s := m.getShard(key)
    s.mu.Lock()
    s.m[key] = value
    s.mu.Unlock()
}

func (m *ShardedMap) Get(key string) (interface{}, bool) {
    s := m.getShard(key)
    s.mu.RLock()
    v, ok := s.m[key]
    s.mu.RUnlock()
    return v, ok
}

func (m *ShardedMap) Delete(key string) {
    s := m.getShard(key)
    s.mu.Lock()
    delete(s.m, key)
    s.mu.Unlock()
}

// Range 遍历所有 key,弱一致:期间 Set/Delete 不阻塞,但**不保证**遍历到
// 返回 false 终止遍历
func (m *ShardedMap) Range(fn func(key string, value interface{}) bool) {
    for _, s := range m.shards {
        s.mu.RLock()
        for k, v := range s.m {
            if !fn(k, v) {
                s.mu.RUnlock()
                return
            }
        }
        s.mu.RUnlock()
    }
}

// Len 弱一致:遍历期间可能变化,结果是近似值
func (m *ShardedMap) Len() int {
    total := 0
    for _, s := range m.shards {
        s.mu.RLock()
        total += len(s.m)
        s.mu.RUnlock()
    }
    return total
}
```

**讲解时主动点出的细节**:

| 设计 | 资深点 |
| --- | --- |
| `shardCount = 32` 是 2 的幂 | **`& (N-1)` 替代 `% N`** 提速,且固定 const 编译器友好 |
| `fnv.New32a()` | 速度快、确定性、零依赖;**不用 `hash/maphash`** 因为它有 G 内随机种子,**不同 G 同 key 算出不同值** |
| 每片用 RWMutex | **读多写少场景**才有收益,写多场景换成 Mutex 更快 |
| Range/Len 是弱一致 | **强一致需要全局锁**,违背分片的目的 → 业务接受"近似值" |
| getShard 路径短 | 每次 Get/Set 多一次 fnv 调用,**确保 fnv 不成为热点**(实际几十 ns) |
| 没有泛型 | Go 1.18+ 可改 `ShardedMap[K comparable, V any]`,**面试时主动提**加分 |

---

## 五、泛型版(Go 1.18+)

```go
type ShardedMap[K comparable, V any] struct {
    shards [shardCount]*shard[K, V]
    hash   func(K) uint32
}

type shard[K comparable, V any] struct {
    mu sync.RWMutex
    m  map[K]V
}

func New[K comparable, V any](hash func(K) uint32) *ShardedMap[K, V] {
    sm := &ShardedMap[K, V]{hash: hash}
    for i := 0; i < shardCount; i++ {
        sm.shards[i] = &shard[K, V]{m: make(map[K]V)}
    }
    return sm
}

// 用法:
// hashStr := func(k string) uint32 { h := fnv.New32a(); h.Write([]byte(k)); return h.Sum32() }
// m := New[string, *User](hashStr)
```

**资深表达**:"非泛型版用 `interface{}` 牺牲类型安全;泛型版让调用方注入 hash 函数,**避免反射 + 类型断言**,性能更好。"

---

## 六、测试

```go
func main() {
    m := New()
    var wg sync.WaitGroup
    for i := 0; i < 32; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 10000; j++ {
                key := fmt.Sprintf("g%d-k%d", id, j)
                m.Set(key, j)
            }
        }(i)
    }
    wg.Wait()
    fmt.Println("len:", m.Len()) // → 320000
}
```

---

## 七、和 `sync.Map` 的对比(必懂)

`sync.Map` 是 Go 标准库为**特定场景**优化的并发 map,**不是通用方案**。

### 7.1 `sync.Map` 内部:double-map + read-only

```text
sync.Map
  ├─ read   atomic.Value(只读 map,无锁读)
  ├─ dirty  map[any]*entry(写时拷贝)
  └─ misses int(read miss 次数,触发 dirty 升级)
```

**读路径**:先 atomic 加载 read map,无锁拿到值。
**写路径**:写 dirty(要 Mutex),read 没命中时 misses++,**misses 超过阈值** → dirty 升级为新 read。

### 7.2 适用场景对比

| 场景 | `sync.Map` | 分片 map(本文) |
| --- | --- | --- |
| **写远多于读** | ❌ **慢**(每次写 dirty + 升级抖动) | ✅ |
| **读远多于写,key 集合稳定** | ✅ **极快**(无锁读) | 略慢(每次 RLock) |
| **写多读多平衡** | ❌ | ✅ |
| **key 频繁新增删除** | ❌ dirty 频繁升级 | ✅ |
| **类型固定** | interface{}(类型断言)| 泛型支持 |
| **典型用途** | **缓存元数据 / 单例注册表** | **通用 K/V 场景** |

**Go 官方注释明确**:
> The Map type is optimized for two common use cases:
> (1) when the entry for a given key is only ever written once but read many times, as in caches that only grow, or
> (2) when multiple goroutines read, write, and overwrite entries for disjoint sets of keys.

**资深表达**:"`sync.Map` 不是 'Mutex + map' 的更快版本——**它针对'读多写少且 key 稳定'优化**,通用并发场景下分片 map 更好。压测下一般业务用分片 map 比 sync.Map 快 30-50%。"

---

## 八、Range 的一致性问题

### 8.1 三种"一致性"语义

| 语义 | 实现 | 代价 |
| --- | --- | --- |
| **强一致快照** | Range 时全局加锁 | **吞吐降到 Mutex+map 水平**——违背分片目的 |
| **片内一致** | Range 一片时持该片 RLock | **片间不一致**(本实现) |
| **快照拷贝** | Range 开始时拷贝所有 key | **内存 ×2 + Range 期间还能变化** |

### 8.2 业务怎么处理

```go
// 方案 1:接受弱一致(99% 场景够用)
m.Range(func(k string, v interface{}) bool {
    process(k, v)
    return true
})

// 方案 2:版本号 + 重试
for {
    ver1 := m.Version()
    snap := collectAll(m)
    ver2 := m.Version()
    if ver1 == ver2 { break } // 期间没人改 → 快照有效
}
```

**结论**:业务里要全局快照很少,99% 场景接受片内一致就够。

---

## 九、典型坑

| 坑 | 现象 | 修复 |
| --- | --- | --- |
| 分片数不是 2 的幂 | 用了 `%` 慢 / 用 `&` 算错位 | 强制 2 的幂,如 32 |
| 用 `hash/maphash` | 不同 G 同 key 算到不同片 | **必须用 `hash/fnv`** 或自实现 |
| Range 内调 Set/Delete | **死锁**(自己持 RLock 又抢 Lock) | Range 内不要写;真要改先收集再统一处理 |
| 每次 getShard 创建 fnv | 内存分配开销 | 可改用 inline 的 fnv1a 计算 |
| 用 RWMutex 写多 | 比 Mutex 慢 | 写多直接 Mutex |
| 加 atomic Count 字段 | 跨 shard 累加器变热点 | 每片自己计数,Len 时累加 |
| 拿到 value 后改它 | 改的是 map 里的指针,**数据竞争** | 拷贝出来再改 / 用不可变值 |
| 直接拿 `sync.Map` 替代 | 写多场景反而慢 | 看场景选 |

---

## 十、和 Java ConcurrentHashMap 对照

| 阶段 | Java CHM | Go 类比 |
| --- | --- | --- |
| **JDK 1.7** | Segment 数组(默认 16)+ ReentrantLock | **本文的分片 map** |
| **JDK 1.8** | 数组 + CAS + synchronized(锁链表头节点) | sync.Map(双 map)/ 没有完全对应 |

Go 没有官方 ConcurrentHashMap,生产用 `sync.Map` 或开源 `orcaman/concurrent-map`(后者就是分片实现)。

---

## 十一、现场表达模板

> "并发安全 map 的核心是**降低锁粒度**——全局一把 Mutex 在高并发下会被打爆。**分片(sharding)**:把数据 hash 到 32 个独立的 map,每片自己的 RWMutex。
>
> 关键细节:
> 1. **分片数选 2 的幂**(32),用 `hash & (N-1)` 替代 `% N`,几倍加速
> 2. **hash 函数用 `hash/fnv`**,不用 `hash/maphash`——后者有 G 内随机种子,不同 G 算出不同值
> 3. **每片选 RWMutex 还是 Mutex** 取决于读写比,**读多写少用 RW**,写多 Mutex 更快
> 4. **Range 和 Len 是弱一致**,强一致快照需要全局锁,违背分片初衷
>
> 和 `sync.Map` 对比:`sync.Map` **不是通用方案**,它针对'**读多写少 + key 集合稳定**'优化(双 map + atomic 无锁读)。
> - **写多读多平衡** / key 频繁变 → **分片 map**
> - **读极多写极少 + key 稳定**(缓存元数据)→ **sync.Map**
> - **写远多于读** → **不要用 sync.Map**(dirty 升级抖动)
>
> 业务里 99% 用分片 map,sync.Map 只在符合官方文档那两种场景时用。Java `ConcurrentHashMap` 1.7 就是这套分片思路,1.8 改为 CAS + synchronized 锁桶头节点。"

---

## 十二、一句话总结

> **分片 map = N 个独立小 map + 每片独立 RWMutex + hash 路由**;
>
> - **分片数选 2 的幂(32)**,用 `& (N-1)` 替代 `% N`,几倍加速
> - **hash 必须用 `fnv` 不用 `maphash`**(后者跨 G 不一致)
> - **Range/Len 弱一致**,强一致快照违背分片目的
> - `sync.Map` 仅适合**读多 + key 稳定**(缓存元数据 / 注册表),通用场景仍是分片
> - 写多场景**每片用 Mutex 比 RWMutex 快**
> - Java `ConcurrentHashMap` 1.7 是同思路,1.8 改 CAS + 锁桶头
