# 逃逸分析

> 编译器决定变量在栈还是堆上的分析过程；栈分配免费且随函数退出自动释放，堆分配进入 GC

## 一、核心原理

### 1.1 什么是逃逸

> **"变量的生命周期超过当前函数 → 逃逸到堆"**

编译器在编译期静态分析，判定每个变量的生命周期：
- 仅在函数内用 → **栈分配**（函数返回即释放，零 GC 成本）
- 可能被外部引用 → **堆分配**（需 GC 管理）

### 1.2 核心判定依据

**指针不能指向即将销毁的内存。** 只要编译器无法静态证明变量不会被外部引用，就保守地放到堆上。

### 1.3 常见逃逸场景

#### (1) 返回局部变量指针

```go
func new() *Point {
    p := Point{X: 1}
    return &p  // p 必逃逸: 调用方会用它
}
```

#### (2) 赋值给 interface

```go
func log(v any) {}  // any = interface{}

log(42)  // 42 被装箱,可能逃逸
```

装入 interface 时若值大于一个指针（通常 8B），编译器需要把值复制到堆并存指针。

> 小优化：Go 1.9+ 对 `uint8`/`bool` 等小值做了 interned，避免每次逃逸。

#### (3) 闭包捕获变量

```go
func counter() func() int {
    n := 0
    return func() int { n++; return n }  // n 被闭包引用,逃逸
}
```

#### (4) 发送到 channel

```go
ch := make(chan *Point)
p := &Point{}  // p 逃逸: 下游 g 会用
ch <- p
```

#### (5) slice 扩容超限

```go
s := make([]int, 1<<20)  // 大 slice 可能直接堆分配(栈放不下)
```

编译器对大于某阈值（通常 64KB）或运行期大小不定的 slice/array 直接放堆。

#### (6) 动态大小分配

```go
n := someRuntimeValue()
s := make([]int, n)  // n 编译期不定 → 堆
```

#### (7) 方法的指针接收者 vs 值接收者

```go
func (p *P) Foo() {}  // 对 p 取地址时可能导致逃逸

var p P
p.Foo()  // 隐式 &p, 若 Foo 内逃逸了 p, 则 p 也逃逸
```

#### (8) 反射与 fmt

```go
fmt.Println(x)  // x 可能逃逸(fmt 用反射)
```

fmt / encoding/json 等都会让参数逃逸。热路径避免。

### 1.4 不会逃逸的场景

- 局部变量且只在函数内使用
- 小 struct 值传递
- slice `[]T` 作为参数且不被保存到外部
- map 查找结果（只读用时）
- 编译器能精确推断引用不会外泄

### 1.5 怎么看逃逸结果

```bash
go build -gcflags="-m" ./...
# 输出:
# ./main.go:10:6: moved to heap: p
# ./main.go:15:13: ... argument does not escape
# ./main.go:20:9: new(Point) escapes to heap

# 更详细
go build -gcflags="-m=2" ./...
```

关键输出：
- `moved to heap: xxx` — 明确逃逸
- `escapes to heap` — 表达式逃逸
- `does not escape` — 没逃逸
- `inlined` — 被内联

## 二、八股速记

- **生命周期超过函数 = 逃逸**，否则栈分配
- **栈分配 ≈ 免费**，堆分配要 GC 管
- 常见逃逸：返回指针、装入 interface、闭包捕获、送 channel、大对象、动态 size
- `go build -gcflags="-m"` 查看
- **逃逸不是 bug**，是编译器保守决策；关心的是**热点路径**逃逸成本
- sync.Pool、预分配、避免 interface 装箱是减少堆分配的三板斧
- 函数内联会影响逃逸判断（内联后上下文合并分析）

## 三、面试真题

**Q1：什么是逃逸分析？**
编译器静态分析每个变量的引用路径，判断它的生命周期是否会超出当前函数作用域。如果会，必须堆分配；否则可以栈分配。目标：尽量放栈上，减少 GC 压力。

**Q2：栈分配和堆分配的区别？**
| | 栈 | 堆 |
| --- | --- | --- |
| 分配成本 | 移动 SP，几乎免费 | 走分配器 + 可能触发 GC |
| 释放 | 函数返回自动 | 等 GC 标记清扫 |
| 空间局部性 | 好（连续） | 差（散落在 span） |
| 并发安全 | 每 g 独立栈 | 共享堆，受 GC 影响 |
| 容量 | 2KB 起动态伸缩，上限 1GB | 几乎无限 |

**Q3：返回局部变量地址会不会有问题？**

```go
func new() *Point {
    p := Point{}
    return &p
}
```

**不会**。Go 编译器通过逃逸分析把 `p` 放到堆上，生命周期由 GC 管理。C/C++ 这样写是悬空指针 bug，Go 不是。

**Q4：怎么判断是否逃逸？**

```bash
go build -gcflags="-m" main.go
```

输出里搜 `escapes to heap` 或 `moved to heap`。高级模式 `-m=2` 给理由。

**Q5：interface{} 为什么会导致逃逸？**
`interface{}` = `(type, data)`，`data` 是指针。把值装进 interface 时，如果值 > 一个指针的大小（通常 8B），就得把值复制到堆，让 `data` 指过去。

**小值优化**：`bool`、`int8`、某些常量会被 interned，不逃逸。

**Q6：闭包什么时候让变量逃逸？**
当闭包的生命周期超过当前函数：返回闭包、把闭包存进变量/channel/map、goroutine 启动闭包。

```go
// 不逃逸(闭包立即执行)
func() { fmt.Println(x) }()

// 逃逸(闭包被返回)
return func() { fmt.Println(x) }

// 逃逸(闭包进 g)
go func() { fmt.Println(x) }()
```

**Q7：切片扩容会逃逸吗？**
视情况：
- `make([]int, 10)` 小且长度固定，通常栈分配
- `make([]int, n)` n 动态，逃逸
- `s = append(s, ...)` 扩容时新底层数组在堆

**Q8：怎么优化逃逸？**
1. 热点路径用具体类型而不是 interface
2. 预估 slice/map 大小，`make(T, 0, n)` 预分配
3. 传值 vs 传指针：小 struct 值传反而不逃逸
4. 字符串拼接用 `strings.Builder`
5. sync.Pool 复用（对象仍在堆，但减少分配次数）
6. 避免不必要的 defer 闭包捕获

**Q9：一个变量一定是栈或堆吗？**
是。编译期决定。一次编译后对同一个变量只有一种分配位置。但同一函数不同变量可以一部分栈一部分堆。

**Q10：函数内联和逃逸分析的关系？**
内联会把被调用函数的代码展开到调用方，逃逸分析在展开后的上下文中重新判断。所以某些情况下加 `//go:noinline` 禁止内联，反而让变量逃逸（用于测试分析）。

## 四、手写实现

**1. 对比：逃逸 vs 不逃逸**

```go
// 逃逸: 返回指针
func escape1() *int {
    x := 42
    return &x
}

// 不逃逸: 值返回
func noEscape1() int {
    x := 42
    return x
}

// 逃逸: 装入 interface
func escape2() {
    var x int = 42
    fmt.Println(x)  // x 逃逸(fmt 参数 any)
}

// 不逃逸: 格式化参数能被编译器证明不外泄
func noEscape2(x int) int {
    return x * 2
}

// 逃逸: slice 大小运行期决定
func escape3(n int) []int {
    return make([]int, n)
}

// 不逃逸: 小 slice 且大小固定
func noEscape3() {
    s := make([]int, 10)
    _ = s
}
```

验证：

```bash
go build -gcflags="-m" main.go 2>&1 | grep -E "escape|heap"
```

**2. benchmark 验证优化前后：**

```go
// 优化前
func BenchmarkWithEscape(b *testing.B) {
    for i := 0; i < b.N; i++ {
        p := new(Point)  // 堆分配
        _ = p
    }
}

// 优化后
func BenchmarkNoEscape(b *testing.B) {
    for i := 0; i < b.N; i++ {
        p := Point{}    // 栈分配
        _ = p
    }
}
```

跑 `go test -bench=. -benchmem`，对比 `allocs/op`。

**3. sync.Pool 复用减少逃逸 GC 压力：**

```go
var pool = sync.Pool{New: func() any { return &Buffer{} }}

// 虽然 Buffer 仍堆分配,但频繁复用减少 new 次数
func process() {
    buf := pool.Get().(*Buffer)
    defer pool.Put(buf.Reset())
    // ...
}
```

## 五、踩坑与最佳实践

### 坑 1：热路径用 interface{}

```go
// 高频调用, 每次 v 逃逸
func Add(v any) { sum += v.(int) }
```

**修复**：针对主要类型特化 `AddInt(v int)`。

### 坑 2：defer 闭包捕获大 struct

```go
func process(big BigStruct) {
    defer func() { log.Println(big) }()  // big 被闭包捕获,逃逸
    // ...
}
```

**修复**：只捕获需要的字段。

### 坑 3：小 struct 用指针反而更慢

```go
type Small struct { X, Y int }

func f(p *Small) {}  // 指针接收者, 可能导致调用方 Small 逃逸
func g(s Small)  {}  // 值传递, 栈上两个 int 复制
```

小 struct 值传递更快且不逃逸。大 struct 才必须指针。

### 坑 4：string + []byte 来回转

```go
s := string(b)      // 拷贝
b2 := []byte(s)     // 拷贝
```

每次都可能逃逸。设计 API 时确定主要使用哪种类型。

### 坑 5：误以为减少逃逸就一定更快

有时堆分配 + 好的缓存行为比栈分配 + 复杂栈帧更快。**必须 benchmark 验证**。别迷信 "everything on stack"。

### 坑 6：闭包隐式持有大对象

```go
func load() func() int {
    data := loadHugeData()  // 100MB
    return func() int { return data[0] }  // 闭包持有整个 100MB
}
```

返回闭包用到只是 1 个字节，但整个 data 逃逸到堆。**修复**：只捕获必要字段 `x := data[0]; return func() int { return x }`。

### 最佳实践

- **先 profile 再优化**：`pprof allocs top` 找分配热点，再针对性减少逃逸
- **`-gcflags="-m"`** 作为日常检查工具，代码 review 时顺手看
- **热路径避免 interface**，用具体类型或泛型（Go 1.18+）
- **预分配 slice/map**，知道大小就 `make(T, 0, n)`
- **sync.Pool** 复用临时对象（buffer、解析中间态）
- **小 struct 值传递**，大 struct 用指针
- **字符串拼接用 Builder**，循环 `+=` 禁用
- **不要过度优化**：多数业务代码逃逸无所谓，只关心 top 热点
