# 泛型 (generics)

> Go 1.18+ 引入：类型参数 + 约束（constraint）+ type set；编译期单态化（部分 GC shape stenciling），不是 Java 擦除式

## 一、核心原理

### 1.1 基本语法

```go
// 类型参数列表 [T any]
func Map[T, U any](s []T, f func(T) U) []U {
    r := make([]U, len(s))
    for i, v := range s { r[i] = f(v) }
    return r
}

ints := []int{1, 2, 3}
doubled := Map(ints, func(x int) int { return x * 2 })  // 类型推断
```

### 1.2 约束（constraint）

约束是**接口**，用 type set 表达"哪些类型符合"：

```go
// 任意类型
type any = interface{}

// comparable: 可用 == 比较
type comparable interface{ ... }

// 自定义约束: 数值类型
type Number interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 |
    ~float32 | ~float64
}

func Sum[T Number](s []T) T {
    var sum T
    for _, v := range s { sum += v }
    return sum
}
```

**`~T`** 表示"底层类型是 T 的所有类型"，包括 `type MyInt int`。
不带 `~` 只匹配精确类型。

### 1.3 type set 语法

```go
type Stringer interface {
    String() string
}

type Numeric interface {
    int | float64 | string  // | 表示并集
}

type SignedNumeric interface {
    ~int | ~int64
    String() string  // 同时要求方法
}
```

约束 = **方法集合 ∩ 类型集合**。

### 1.4 标准库 constraints 包

```go
import "golang.org/x/exp/constraints"

constraints.Ordered  // 可比较大小: int/float/string
constraints.Integer  // 所有整型
constraints.Signed   // 有符号
constraints.Float    // 浮点
```

> 注意：`golang.org/x/exp/constraints` 长期是 experimental，主线没合入；常用约束自己定义就行。

### 1.5 类型推断

```go
Map(ints, func(x int) int { return x * 2 })  // 推断 T=int, U=int
Map[int, string](ints, strconv.Itoa)         // 显式指定
```

推断不出来时必须显式（如返回类型不在参数里）。

### 1.6 泛型类型

```go
type Stack[T any] struct {
    data []T
}

func (s *Stack[T]) Push(v T) { s.data = append(s.data, v) }
func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.data) == 0 { return zero, false }
    v := s.data[len(s.data)-1]
    s.data = s.data[:len(s.data)-1]
    return v, true
}

s := Stack[int]{}
s.Push(1)
```

### 1.7 实现机制（GC shape stenciling）

Go 不是 Java 擦除（Erasure），也不是 C++ 模板（每实例一份代码）。
Go 用 **GC shape stenciling**：
- 按"GC 形状"分组（同样大小 + 同样指针布局的类型共享一份代码）
- 比如 `Map[int]` 和 `Map[int64]` 共享代码，`Map[*T]` 和 `Map[*U]` 也共享

权衡：代码膨胀小，但每次调用要查 dictionary 找方法/类型信息，比直接调用慢一点。

### 1.8 限制

- **不支持泛型方法**（仅泛型函数 + 泛型类型上的方法）
- 方法不能有自己的类型参数：`func (s *Stack[T]) ToSlice[U any]() []U`  ❌
- 类型参数**不能用作值字面量**的类型（少数边界情况）
- 反射对泛型实例的支持有限

## 二、八股速记

- Go 1.18+ 引入，**类型参数 + 约束 + type set**
- 约束是接口，可包含方法集合 + 类型集合（`int | float64`）
- `~T` 匹配 underlying 是 T 的所有类型
- `comparable` 支持 ==，`any = interface{}`
- 类型推断常见，必要时显式 `[T]`
- 实现：**GC shape stenciling**（不是擦除也不是完全模板化）
- **泛型方法不支持**（仅泛型函数 + 泛型 receiver 上的普通方法）
- 性能比 interface 抽象快（避免装箱），但比直接代码略慢
- 常用场景：容器（Stack/Queue）、工具函数（Map/Filter/Reduce）、避免 interface{} 装箱

## 三、面试真题

**Q1：Go 为什么这么晚才引入泛型？**
设计权衡：
- Go 1.0 时代选择简洁性，避免 C++/Java 模板的复杂度
- 多年讨论后，1.18 选了 type parameter + constraint 方案，比 Java 擦除强（保留类型信息），比 C++ 模板简单（编译期单态化但用 GC shape 分组）

**Q2：类型参数和 interface 的区别？**

| | 泛型 | interface |
| --- | --- | --- |
| 类型检查 | 编译期 | 运行期 |
| 性能 | 接近直接代码 | 装箱 + 动态分派 |
| 代码 | 一份代码多类型 | 一份代码统一行为 |
| 灵活性 | 编译期已确定 | 运行期可动态 |

```go
// 泛型: 编译期类型安全, 高性能
func Min[T constraints.Ordered](a, b T) T {
    if a < b { return a }
    return b
}

// interface: 灵活但慢
func MinAny(a, b interface{ Less(any) bool }) interface{} { ... }
```

**Q3：什么场景应该用泛型？**
1. **容器**：Stack/Queue/Set/LRU
2. **工具函数**：Map/Filter/Reduce/Contains
3. **算法**：排序、查找、数学
4. **避免 interface{} 装箱**：高频路径的小函数

**不适合**：
- 单一类型的简单函数（直接写）
- 需要运行时多态（用 interface）
- 反射操作

**Q4：`~int` 和 `int` 区别？**

```go
type MyInt int

type T1 interface { int }   // 只匹配 int 本身
type T2 interface { ~int }  // 匹配所有 underlying type 是 int 的, 包括 MyInt
```

`~` 表示"底层类型为"。绝大多数约束应该用 `~`，因为业务里常 `type Status int` 这样。

**Q5：comparable 是什么？**

```go
type comparable interface { ... }  // runtime 内置
```

满足 == 和 != 的类型：基本类型、可比较的 struct/array、指针、channel、interface、comparable 类型组合。

**不满足**：slice、map、function。

```go
func Set[T comparable]() map[T]struct{} {
    return make(map[T]struct{})
}
```

**Q6：Go 泛型怎么实现的？性能如何？**
**GC shape stenciling**（不是 Java 擦除，也不是 C++ 完全模板化）：
- 按"GC shape"分组：相同大小 + 相同指针布局的类型共享一份编译代码
- 通过 dictionary 在运行期查找类型相关信息（方法表、size 等）

性能：
- 比 interface 装箱快（无堆分配）
- 比直接代码慢一点（dictionary lookup）
- 通常足够好，热点路径可能要特化

**Q7：泛型方法为什么不支持？**

```go
type Stack[T any] struct{}
func (s *Stack[T]) Map[U any](f func(T) U) *Stack[U] {  // ❌ 编译错误
    ...
}
```

设计取舍：方法上加类型参数会让接口实现匹配极其复杂（要算所有类型实例化的方法集）。Go 团队故意限制。

**变通**：用顶层泛型函数

```go
func StackMap[T, U any](s *Stack[T], f func(T) U) *Stack[U] { ... }
```

**Q8：`any` 和 `interface{}` 区别？**
**完全等价**。Go 1.18+ `any` 是 `interface{}` 的别名，纯语法糖让代码更易读。

**Q9：泛型对反射的影响？**
反射看到的是**实例化后的具体类型**，不知道类型参数：

```go
func Print[T any](v T) {
    fmt.Println(reflect.TypeOf(v))  // 实际类型, 不是 "T"
}
Print[int](1)  // int
```

无法在运行时获取"原始的 T 是什么"。

**Q10：怎么写一个泛型 Map 函数？**

```go
func Map[T, U any](s []T, f func(T) U) []U {
    r := make([]U, len(s))
    for i, v := range s { r[i] = f(v) }
    return r
}
```

类型推断会自动从参数推出 T 和 U。

## 四、手写实现

**1. 泛型容器：**

```go
type Set[T comparable] struct {
    m map[T]struct{}
}

func NewSet[T comparable]() *Set[T] {
    return &Set[T]{m: make(map[T]struct{})}
}

func (s *Set[T]) Add(v T)       { s.m[v] = struct{}{} }
func (s *Set[T]) Has(v T) bool  { _, ok := s.m[v]; return ok }
func (s *Set[T]) Remove(v T)    { delete(s.m, v) }
func (s *Set[T]) Len() int      { return len(s.m) }
func (s *Set[T]) Slice() []T {
    r := make([]T, 0, len(s.m))
    for k := range s.m { r = append(r, k) }
    return r
}
```

**2. 高阶函数族：**

```go
func Map[T, U any](s []T, f func(T) U) []U {
    r := make([]U, len(s))
    for i, v := range s { r[i] = f(v) }
    return r
}

func Filter[T any](s []T, pred func(T) bool) []T {
    r := make([]T, 0, len(s))
    for _, v := range s { if pred(v) { r = append(r, v) } }
    return r
}

func Reduce[T, U any](s []T, init U, f func(U, T) U) U {
    acc := init
    for _, v := range s { acc = f(acc, v) }
    return acc
}

func Contains[T comparable](s []T, v T) bool {
    for _, e := range s { if e == v { return true } }
    return false
}
```

**3. Min/Max：**

```go
type Ordered interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 |
    ~float32 | ~float64 | ~string
}

func Min[T Ordered](a, b T) T {
    if a < b { return a }
    return b
}

func Max[T Ordered](a, b T) T {
    if a > b { return a }
    return b
}
```

> Go 1.21+ 标准库 `cmp.Ordered`、`min`/`max` 内置函数已可用。

**4. 类型安全的 channel 工具：**

```go
func Merge[T any](chans ...<-chan T) <-chan T {
    out := make(chan T)
    var wg sync.WaitGroup
    wg.Add(len(chans))
    for _, c := range chans {
        go func(c <-chan T) {
            defer wg.Done()
            for v := range c { out <- v }
        }(c)
    }
    go func() { wg.Wait(); close(out) }()
    return out
}
```

**5. 通用 LRU：**

```go
type LRU[K comparable, V any] struct {
    cap  int
    m    map[K]*list.Element
    list *list.List
}

type entry[K comparable, V any] struct {
    K K
    V V
}

func NewLRU[K comparable, V any](cap int) *LRU[K, V] {
    return &LRU[K, V]{cap: cap, m: make(map[K]*list.Element), list: list.New()}
}

func (c *LRU[K, V]) Get(k K) (V, bool) {
    var zero V
    if e, ok := c.m[k]; ok {
        c.list.MoveToFront(e)
        return e.Value.(*entry[K, V]).V, true
    }
    return zero, false
}

func (c *LRU[K, V]) Put(k K, v V) {
    if e, ok := c.m[k]; ok {
        c.list.MoveToFront(e)
        e.Value.(*entry[K, V]).V = v
        return
    }
    if c.list.Len() >= c.cap {
        oldest := c.list.Back()
        delete(c.m, oldest.Value.(*entry[K, V]).K)
        c.list.Remove(oldest)
    }
    c.m[k] = c.list.PushFront(&entry[K, V]{K: k, V: v})
}
```

## 五、踩坑与最佳实践

### 坑 1：方法上加类型参数

```go
type Container[T any] struct{}
func (c *Container[T]) Convert[U any]() *Container[U] { ... }  // ❌ 编译错误
```

**修复**：用顶层泛型函数

```go
func Convert[T, U any](c *Container[T]) *Container[U] { ... }
```

### 坑 2：约束忘记 `~`

```go
type Number interface { int | float64 }  // 只匹配纯 int / float64

type MyInt int
Sum[MyInt](nil)  // 编译错误: MyInt 不是 int
```

**修复**：

```go
type Number interface { ~int | ~float64 }
```

### 坑 3：`any` slice 不是 `[]any`

```go
ints := []int{1, 2, 3}
var anys []any = ints  // ❌ 编译错误
```

切片不能因元素类型转换而互转。要泛型函数处理：

```go
func ToAny[T any](s []T) []any {
    r := make([]any, len(s))
    for i, v := range s { r[i] = v }
    return r
}
```

### 坑 4：泛型代码可读性下降

过度泛型化让 API 难懂：

```go
func DoStuff[T any, U comparable, V Ordered](...) ... { ... }
```

**原则**：泛型解决具体问题，不是显示技术力。简单场景用具体类型。

### 坑 5：编译时间增长

泛型实例化会让编译变慢，大项目大量使用泛型可能感受明显。如果编译速度敏感，少用泛型。

### 坑 6：泛型不能完全替代 interface

```go
// 泛型: 编译期确定类型
func Print[T fmt.Stringer](v T) { fmt.Println(v.String()) }

// interface: 运行期可动态
func PrintI(v fmt.Stringer) { fmt.Println(v.String()) }
```

需要存到一个容器里多种类型时，仍要用 interface。

### 坑 7：`comparable` 不接受 interface 字段的 struct

```go
type S struct { X any }
type T interface { comparable }
// S 不满足 comparable, 因为 any 字段可能含不可比较值, 运行时 panic
```

Go 1.20+ 放宽了部分限制（接口实现 comparable 时允许）。

### 最佳实践

- **从需求出发**：先有 2~3 个相似类型的代码，再考虑抽象成泛型，不要预先设计
- **优先标准库**：`slices` / `maps` 包（Go 1.21+）已经有 `Sort` / `Index` / `Contains` 等
- **`any` 替代 `interface{}`**：可读性更好
- **约束用 `~`**：兼容自定义底层类型
- **泛型 vs interface**：编译期确定类型用泛型，运行期多态用 interface
- **不要泛型化所有东西**：单一类型的简单函数直接写
- **dictionary 开销**：性能敏感热点 benchmark 验证，必要时手动特化
- 容器、工具函数、算法是泛型的甜区
