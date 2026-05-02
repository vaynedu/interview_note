# interface

> Go 接口：iface (有方法) / eface (空接口) 双指针结构；隐式实现，鸭子类型；运行时动态分派

## 一、核心原理

### 1.1 两种底层结构

```go
// runtime/runtime2.go
// 空接口 (interface{} / any)
type eface struct {
    _type *_type         // 类型信息
    data  unsafe.Pointer // 指向实际数据
}

// 带方法的接口
type iface struct {
    tab  *itab           // 方法表 + 类型信息
    data unsafe.Pointer  // 指向实际数据
}

type itab struct {
    inter *interfacetype // 接口类型描述
    _type *_type         // 实际类型
    hash  uint32         // _type.hash 副本(用于类型断言快速比较)
    _     [4]byte
    fun   [1]uintptr     // 方法表(变长,实际有 N 个)
}
```

**两个指针**：一个指向类型元信息（含方法表），一个指向数据。

### 1.2 隐式实现（鸭子类型）

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type File struct{ ... }
func (f *File) Read(p []byte) (int, error) { ... }

var r Reader = &File{}  // 编译期检查 *File 是否实现 Reader 的所有方法
```

不需要 `implements` 关键字，**只要方法集满足就自动是**。

### 1.3 itab 缓存

`itab` 由 runtime 在首次类型转换时生成并缓存（按 `(接口类型, 具体类型)` pair）。后续同样的转换直接查表，不用再计算。

### 1.4 类型断言

```go
// 单返回值: 失败 panic
file := r.(*File)

// 双返回值: 失败 ok=false, 不 panic
file, ok := r.(*File)
```

实现：比较 `iface.tab._type` 是不是目标类型。

### 1.5 type switch

```go
switch v := r.(type) {
case *File:
    // v is *File
case io.Reader:
    // v is io.Reader
default:
    // v is interface{} (原 r)
}
```

编译为多次 itab 比较 + 跳转表。

### 1.6 方法集规则

| 接收者 | 值 T 能调用的方法 | 指针 *T 能调用的方法 |
| --- | --- | --- |
| `func (t T)` | ✓ | ✓ |
| `func (t *T)` | ✗（值没法改自己） | ✓ |

赋值给接口时同理：

```go
type S struct{}
func (s *S) Foo() {}

var s S
var i interface{ Foo() } = s   // ❌ S 的方法集不含 Foo
var i interface{ Foo() } = &s  // ✓
```

### 1.7 动态分派开销

- itab 已缓存：一次间接寻址（fun[i]）+ 一次间接调用，约几个 ns
- inline 失效：编译器无法内联接口方法，热路径用具体类型更快
- 装箱：基本类型存入 interface{} 时若大于一个指针会逃逸到堆

## 二、八股速记

- 两种结构：**eface (无方法) = type+data**，**iface (有方法) = itab+data**
- itab = 接口类型 + 具体类型 + 方法表，**runtime 缓存**
- **隐式实现**，鸭子类型
- 类型断言双返回值版本不会 panic
- 方法集规则：值接收者的方法 *T 也能调；指针接收者的方法 T 不能调
- **nil interface 不等于装着 nil 指针的 interface**（最经典坑）
- interface{} 装基本类型会装箱到堆 → 性能敏感场景慎用
- 接口设计 idiom：**接受接口，返回结构体**（accept interfaces, return structs）

## 三、面试真题

**Q1：iface 和 eface 区别？**
- `eface`：空接口 `interface{}`，只存 (type, data)，主要用于通用容器
- `iface`：带方法的接口，存 (itab, data)，itab 含方法表用于动态分派
设计差异：eface 不需要方法表，省一次间接寻址。

**Q2：以下输出？**

```go
type MyError struct{}
func (e *MyError) Error() string { return "x" }

func mayFail() error {
    var p *MyError = nil
    return p  // 注意!
}

func main() {
    err := mayFail()
    fmt.Println(err == nil)  // ?
}
```

输出 **false**。
原因：`return p` 把 `(*MyError)(nil)` 装入 error 接口 → `iface{tab: itabFor(error,*MyError), data: nil}`。`tab` 不为 nil，`err == nil` 比较的是整个接口值，所以 false。

**修复**：

```go
func mayFail() error {
    var p *MyError = nil
    if p != nil { return p }
    return nil  // 显式返回 nil interface
}
```

**Q3：interface 调用方法的开销？**
- 编译期：接口类型擦除，无法内联具体实现
- 运行期：itab.fun[i] 取方法地址 + 间接调用，比直接调用慢约 2~5x（绝对值仍是 ns 级）
- 装箱：值类型放入 interface 可能堆分配
热路径（百万级 QPS 内层循环）用具体类型；普通业务别在意。

**Q4：什么时候用接口？**
- **依赖反转**：上层定义接口，下层实现（DDD repo 模式）
- **测试 mock**：把外部依赖（DB、HTTP client）抽成接口
- **行为多态**：不同类型的统一处理（io.Reader / Writer）
不要为了"将来可能要换"凭空抽接口，**接口跟着具体类型出现，自然提取出来**（Go idiom: small interfaces, accepted by consumers）。

**Q5：方法集 vs 实现接口？**
判断 T 是否实现接口 I：
- I 的所有方法都在 T 的方法集里 → 实现
- T 的方法集 = 所有值接收者方法
- *T 的方法集 = 值接收者方法 + 指针接收者方法

实践：通常用 `*T` 实现接口（除非 T 是值语义类型如 time.Time）。

**Q6：interface 怎么实现多态？**
- 编译期：变量类型是接口，编译器只检查方法签名
- 运行期：iface.tab.fun[i] 指向具体方法的代码
- 调用：取出函数指针，传入 data 作为 receiver

**Q7：type assertion 和 type switch 哪个快？**
单一类型断言略快（一次比较）。type switch 多 case 时编译器可能用 hash 表优化。差异通常 ns 级，不必在意。

**Q8：怎么判断一个类型是否实现接口（编译期检查）？**

```go
var _ io.Reader = (*MyType)(nil)  // 编译期断言
```

如果 `*MyType` 没实现 `io.Reader` 编译报错。常见 idiom 用于库设计。

## 四、手写实现

**1. 简化的策略模式（多态）：**

```go
type PaymentMethod interface {
    Charge(amount int) error
}

type Alipay struct{}
func (a *Alipay) Charge(amount int) error { /* ... */; return nil }

type WeChat struct{}
func (w *WeChat) Charge(amount int) error { /* ... */; return nil }

func Pay(m PaymentMethod, amount int) error {
    return m.Charge(amount)
}
```

**2. 测试 mock：**

```go
type UserRepo interface {
    GetByID(ctx context.Context, id int64) (*User, error)
}

// 生产
type mysqlRepo struct{ db *sql.DB }
func (r *mysqlRepo) GetByID(...) { ... }

// 测试
type mockRepo struct{ users map[int64]*User }
func (r *mockRepo) GetByID(_ context.Context, id int64) (*User, error) {
    if u, ok := r.users[id]; ok { return u, nil }
    return nil, ErrNotFound
}
```

**3. 安全的接口转换（option 模式）：**

```go
type Closer interface{ Close() error }

func tryClose(v any) {
    if c, ok := v.(Closer); ok {
        _ = c.Close()
    }
}
```

**4. 编译期实现检查：**

```go
type RedisClient struct{ ... }

var _ Cacher = (*RedisClient)(nil)  // 编译期保证 RedisClient 实现 Cacher
```

## 五、踩坑与最佳实践

### 坑 1：返回 nil 指针被装成非 nil interface（典型）

见 Q2。处理 error 时常发：

```go
func do() error {
    var err *MyErr  // nil
    // ... 没赋值
    return err  // 装箱后 != nil
}
```

**修复**：返回 `error` 时显式 `return nil`，或类型断言后判 `data == nil`。

### 坑 2：循环里把 interface 比较当类型判断

```go
if v == nil { ... }  // 接口的 nil: type 和 data 都 nil
```

只有显式 `var v MyInterface` 或 `return nil` 才得到真 nil interface。

### 坑 3：把 []T 强转成 []Interface

```go
ints := []int{1,2,3}
var anys []any = ints  // 编译错误
```

Go 不支持，必须显式转换：

```go
anys := make([]any, len(ints))
for i, v := range ints { anys[i] = v }
```

### 坑 4：interface{} 装小对象的装箱开销

```go
m := map[string]any{}
for i := 0; i < 1e6; i++ {
    m[strconv.Itoa(i)] = i  // 每个 int 装箱到堆
}
```

百万级数据这里会有几百 MB 堆分配。改用 `map[string]int`。

### 坑 5：embedding 接口导致方法冲突

```go
type A interface{ Foo() }
type B interface{ Foo() int }  // 签名不同
type C interface{ A; B }  // 编译错误
```

Go 1.14+ 允许同名同签名 embedding，签名不同仍报错。

### 最佳实践

- **小接口**：`io.Reader`/`io.Writer` 一两个方法，组合成大接口
- **接受接口，返回结构体**：函数参数用接口（解耦），返回值用具体类型（信息完整）
- **接口定义在使用方**，不是在实现方（Go 反 Java 习惯）
- 编译期 `var _ I = (*T)(nil)` 检查实现关系
- 性能热路径避免接口，泛型（Go 1.18+）是更好的零开销抽象
- error wrap 用 `errors.Is/As` 判类型，不要类型断言（兼容嵌套 wrap）
