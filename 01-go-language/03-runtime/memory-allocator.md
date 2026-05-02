# 内存分配器

> Go 内存分配：三级缓存（mcache/mcentral/mheap）+ size class，类 tcmalloc 设计，小对象无锁路径

## 一、核心原理

### 1.1 分层架构

```
对象分配
   ↓
┌──────────────────────────────────────────┐
│  mcache (per-P, 无锁)                     │  ← 首选
│  每个 P 持有,含 67 个 size class 的 mspan │
└──────────────┬───────────────────────────┘
               ↓ 本地空了
┌──────────────────────────────────────────┐
│  mcentral (全局, 按 size class 分组)      │
│  每个 class 一把锁, 含 full/partial 链表  │
└──────────────┬───────────────────────────┘
               ↓ 没有空闲 span
┌──────────────────────────────────────────┐
│  mheap (全局页堆)                         │
│  管理 8KB 页, 向 OS 申请                  │
└──────────────────────────────────────────┘
```

### 1.2 size class

对象按大小分 **67 个 size class**（8B, 16B, 32B, ..., 32KB），每个 class 对应一种 **mspan**（连续的页块）。

```
class 1:  8B   span = 8KB  = 1024 obj
class 2:  16B  span = 8KB  = 512 obj
...
class 67: 32KB span = 32KB = 1 obj
```

好处：
- 避免任意大小分配的碎片
- 同 class 对象在 span 内连续，缓存友好
- 分配 = 从 span 空闲链取一个 slot，O(1)

### 1.3 分配路径（小对象 < 32KB）

```go
p := &Point{X: 1}  // 分配 16B
```

1. 计算 size class（16B → class 2）
2. 从当前 P 的 mcache.alloc[2] 拿 span
3. 从 span 的空闲位图找一个 slot
4. 返回对象地址

**全无锁**（mcache 是 per-P 的）。本地 span 用完 → 去 mcentral 取。

### 1.4 mcentral

每个 size class 一个 mcentral，含两个链表：
- **partial**：有空闲 slot 的 span
- **full**：已分配满的 span

mcentral 有锁，但不同 class 之间无竞争。

### 1.5 mheap 与 page

mheap 管理所有 mspan，底层是 **8KB page** 粒度。向 OS 申请走 `mmap`（Linux），大块回收走 `madvise(MADV_FREE)`。

mheap 内部用**基数树 + pageAlloc**（Go 1.14+ 改进）管理空闲页。

### 1.6 大对象（> 32KB）

直接从 mheap 按页分配，**跳过 mcache 和 mcentral**。会有 mheap 锁竞争（频繁分配大对象时明显）。

### 1.7 tiny allocator（< 16B）

对 `*int`、`*bool`、短字符串等**非指针小对象**，mcache 有个 `tiny` 分配器：
- 一个 16B 的 "tiny block"
- 多个小对象打包到同一块
- 减少 size class 1-3 的分配次数和内存碎片

条件：对象大小 < 16B **且不含指针**（含指针要精确 GC 扫描，不能打包）。

### 1.8 与 GC 的配合

- span 在 class 内连续存储 → GC 扫描更高效
- `gcmarkBits` 每 bit 对应一个 slot，标记位图
- 清扫时批量回收未标记 slot 到 span 的 freelist

### 1.9 内存归还 OS

- `scavenger` 后台 goroutine 定期把长时间未用的 page 用 `madvise(MADV_FREE/DONTNEED)` 告诉 OS 可回收
- `GOGC` / `GOMEMLIMIT` 也影响归还积极性
- Go 1.16+ 默认 `MADV_DONTNEED`（立即返还 RSS），旧版 `MADV_FREE`（惰性归还，RSS 看着不降）

## 二、八股速记

- **三级缓存**：mcache (per-P, 无锁) → mcentral (全局分 class) → mheap (页堆)
- **67 个 size class**，小对象从对应 class 的 span 分配
- **小对象 < 32KB** 走三级；**大对象直接 mheap**
- **tiny allocator** 打包 < 16B 且无指针的对象
- 整体设计类 **tcmalloc**
- per-P mcache = 小对象分配**无锁**
- 底层 8KB page 粒度管理
- **scavenger** 后台归还内存给 OS

## 三、面试真题

**Q1：Go 的内存分配器为什么比 malloc 快？**
- **per-P mcache 无锁**：小对象分配命中 mcache 时无需任何锁
- **size class 预切好**：分配 = freelist 取一个 O(1)，不像 malloc 要动态管理任意大小
- **缓存友好**：同 class 对象在 span 内连续
- **与 GC 协作**：span 和标记位图配合，清扫批量

代价：内存利用率略低（size class 有 padding）。

**Q2：小对象和大对象分配路径区别？**
- **< 16B 无指针**：走 tiny allocator，打包到 16B block
- **16B ~ 32KB**：按 size class 从 mcache 分配（不命中走 mcentral → mheap）
- **> 32KB**：直接从 mheap 按页分配，mheap 锁竞争风险

**Q3：为什么有 mcentral 这一层？**
如果 mcache 本地空了直接去 mheap，mheap 会成为锁竞争瓶颈。mcentral 按 size class 分治，**不同 class 锁独立**，减少竞争。

**Q4：Go 有没有内存碎片问题？**
有两种碎片：
- **外部碎片**：span 之间的 page 碎片 → mheap 的 pageAlloc 管理（Go 1.14+ 改进了这块，减少碎片）
- **内部碎片**：size class 向上取整的 padding → 无法避免，设计权衡

所以 `unsafe.Sizeof(x)` 和实际占用可能不同（实际是 class 大小）。

**Q5：GOMEMLIMIT 对分配器有什么影响？**
Go 1.19+ 的软内存上限：
- 接近 limit 时强制 GC（不等 GOGC 触发）
- 减少 scavenger 的归还延迟（更积极还给 OS）
- 容器化部署强烈推荐设为容器限制的 80~90%

**Q6：为什么容器里 `runtime.MemStats` 看到堆不多但 RSS 很高？**
几种可能：
- Go 向 OS 申请后**没及时归还**（scavenger 惰性）
- 老版本（<1.16）用 `MADV_FREE`，RSS 延迟下降
- **栈内存**和 **runtime 元数据**也算 RSS
- goroutine 栈用完缩到最小值但 M 的线程栈还在

**修复**：Go 1.16+ 默认 `MADV_DONTNEED`；设 `GOMEMLIMIT` 让 GC 更积极。

**Q7：sync.Pool 和分配器是什么关系？**
Pool 是**用户层的对象复用**，避开分配器走自己的池：
- `Get` 先看本地 P 的 pool
- Put 归还本地
- GC 时清空（所以不是持久缓存）

适合**临时对象**（buffer、解析中间态），减少对分配器和 GC 的压力。

**Q8：怎么观察分配行为？**
- `runtime.ReadMemStats(&m)` 看 `HeapAlloc`、`Mallocs`、`HeapObjects`
- `pprof heap`：`go tool pprof http://localhost:6060/debug/pprof/heap`
- `pprof allocs`：查所有分配热点（含已释放的）
- `go build -gcflags="-m"` 看逃逸分析
- `GODEBUG=gctrace=1` 每次 GC 打印内存信息

## 四、手写实现

无需手写分配器，演示几个**减少分配**的技巧：

**1. sync.Pool 复用：**

```go
var bufPool = sync.Pool{
    New: func() any { return make([]byte, 0, 4096) },
}

func handle() {
    buf := bufPool.Get().([]byte)
    defer func() {
        bufPool.Put(buf[:0])
    }()
    // 使用 buf
}
```

**2. 预分配 slice/map：**

```go
// 差: 多次扩容,触发多次分配
result := []int{}
for _, v := range src { result = append(result, v*2) }

// 好: 一次分配到位
result := make([]int, 0, len(src))
for _, v := range src { result = append(result, v*2) }
```

**3. 避免 interface 装箱（小对象尤其）：**

```go
// 差: int 逃逸到堆(装入 interface{})
log("count:", n)  // 若 log 签名 func(args ...any)

// 好: 热路径用具体类型
logInt("count", n)
```

**4. 字符串拼接避免临时对象：**

```go
// 差: 每次 + 都新分配 string
s := ""
for _, p := range parts { s += p }

// 好: Builder 内部扩容, 最终一次 string()
var sb strings.Builder
sb.Grow(expectedLen)
for _, p := range parts { sb.WriteString(p) }
s := sb.String()
```

**5. 避免返回局部变量指针（能栈分配就不上堆）：**

```go
// 差: p 必逃逸到堆
func new() *Point {
    p := Point{X: 1}
    return &p
}

// 好: 返回值类型,调用方决定
func new() Point { return Point{X: 1} }
```

## 五、踩坑与最佳实践

### 坑 1：`map` 装大量元素后 delete 不缩容

```go
m := map[int]*Big{}
for i := 0; i < 1e6; i++ { m[i] = big() }
for k := range m { delete(m, k) }
// len(m) = 0, 但 bucket 数组 + 溢出桶还占 GB 级内存
```

**修复**：重建 `m = make(map[int]*Big)`。

### 坑 2：切片截取阻止底层 GC

```go
big := readFile()   // 100MB []byte
small := big[:10]   // 共享底层
```

`big` 置 nil 也没用，small 持有引用。**修复**：`small := append([]byte{}, big[:10]...)`。

### 坑 3：sync.Pool 放大对象

Pool 每轮 GC 挪到 victim，再一轮 GC 才清。大对象在 Pool 里会**延长常驻内存**。建议 ≤ 几十 KB 才用 Pool。

### 坑 4：interface 包装让对象逃逸

```go
func log(v any) {}  // 参数是 any, 调用时 v 装箱
log(42)             // 42 被拷贝到堆
```

pprof allocs 能看到大量 `int` 分配。热路径用具体类型或 fmt 的 interned 整数优化（< 255 的小 int 不逃逸）。

### 坑 5：大量 goroutine 的栈也吃内存

每个 g 初始栈 2KB，高水位可能到几 MB。10 万 g × 8KB 平均 = 800MB。goroutine 数量要控。

### 坑 6：容器 RSS 下不去

Go ≤ 1.15 默认 `MADV_FREE`，RSS 看起来不降但实际 OS 可回收；Go ≥ 1.16 默认 `MADV_DONTNEED`。升级版本；或设 `GODEBUG=madvdontneed=1`（老版本）。

### 最佳实践

- **减少分配比优化分配器更重要**：预分配、Pool 复用、避免 interface 装箱
- **容器化**：Go 1.19+ 必设 `GOMEMLIMIT`，Go 1.16+ 确认 MADV_DONTNEED
- **定位分配热点**：`pprof allocs` 看 `top`，关注 `cum`（累计）前几名
- **逃逸优化**：`go build -gcflags="-m=2"` 看编译器判断
- **字段对齐**：密集数据结构按字段大小排序减少 padding
- **大对象池化慎用**：Pool 适合中小对象高频复用
