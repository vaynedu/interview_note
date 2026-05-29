# interface

> Go 接口：iface (有方法) / eface (空接口) 双指针结构；隐式实现，鸭子类型；运行时动态分派

## 一、一句话总结(背诵版)

> **interface 是 Go 的隐式多态机制**——鸭子类型,**只要方法集匹配就自动实现**;通过**小接口组合**(io.Reader/Writer)解耦上下游,是依赖反转、mock 测试、行为抽象的基石。

延伸(被追问时再展开):

> 它的特别之处是**不需要 implements 关键字**,接口定义在**使用方而不是实现方**(反 Java idiom)。底层是 **iface/eface 双指针结构**(类型 + 数据),itab 由 runtime 缓存,类型断言和 type switch 都是基于 itab 的指针比较。Go idiom:**accept interfaces, return structs**——参数收接口求解耦,返回值给结构体保留信息完整。

---

## 二、使用场景(场景化记忆)

| 场景 | 一句话 | 典型例子 |
| --- | --- | --- |
| **依赖注入 / 反转** | 上层定义接口,下层实现,运行期注入 | DDD 的 `UserRepo` 接口,生产用 MySQL 实现、测试用 mock 实现 |
| **mock 测试** | 把外部依赖抽成接口,测试时换实现 | `HTTPClient` / `Cache` / `DB` 都抽接口,测试不依赖真实服务 |
| **io 抽象(行为复用)** | 用统一接口描述"能读 / 能写"的行为 | `io.Reader` / `io.Writer`——文件、网络、bytes.Buffer 都能用 |
| **错误处理(error interface)** | `error` 是仅一个 `Error()` 方法的接口 | 自定义错误类型实现 `Error() string` 即可作为 error 返回 |
| **运行时类型分发(断言)** | 在通用容器里恢复具体类型做不同处理 | `any` + type switch 实现 JSON 解析、ORM 字段映射 |
| **按行为分组(sort.Interface)** | 不同类型只要实现 Len/Less/Swap 就能复用 sort | `sort.Sort(myColl)`——把"可排序"行为抽象成接口 |

**选接口的判断**:**有"换实现 / 多种实现 / 测试隔离"诉求才抽接口**;**单一实现不抽接口**(Go idiom 反对预先抽象)。

---

## 三、常见错误(高频踩坑)

| 错误 | 现象 | 根因 | 修复 |
| --- | --- | --- | --- |
| **nil interface 不等于 nil** | 返回 `(*MyErr)(nil)` 给 error,`err == nil` 是 false | iface = (tab, data),tab 非 nil 时整个接口就不等 nil | 显式 `return nil`,不要返回 typed nil |
| **类型断言失败 panic** | `v.(T)` 单返回值在类型不匹配时直接 crash | 单返回值断言失败语义就是 panic | 用双返回值 `v, ok := x.(T)`,或 type switch |
| **值 vs 指针方法集** | `var i I = s`(s 是值,方法在 `*S`)编译错 | T 的方法集只含值接收者方法,`*T` 才含全部 | 用 `&s` 赋值,或把方法定义为值接收者 |
| **空接口 any 滥用** | 所有参数都用 `any`,失去类型安全 + 装箱开销 | 用 any 等于放弃编译期检查,基本类型还触发堆分配 | 优先具体类型 / 泛型(Go 1.18+),只在真通用时用 any |
| **接口定义在实现方包** | 引入接口包导致下层依赖上层,循环依赖 | Go idiom 是**接口定义在使用方**,不是 Java 的"定义在实现包" | 接口跟着 consumer 走,implementer 包不导出接口 |
| **定义大接口违 ISP** | 一个接口 10+ 方法,mock 写起来痛苦 | 接口越大,实现成本越高,复用性越差 | 拆成小接口(io.Reader/Writer 各 1 个方法),用 embedding 组合 |
| **接口实现未编译期检查** | 改了接口签名,实现方漏改,运行时才发现 | Go 不强制声明"实现某接口",静默不实现 | 用 `var _ I = (*T)(nil)` 编译期断言 |

---

## 四、面试常问(简答模板)

**Q1:interface 底层结构?iface 和 eface 区别?**
**iface = (itab, data)** 带方法接口;**eface = (\_type, data)** 空接口。itab 含**接口类型 + 具体类型 + 方法表**,runtime 按 `(接口类型, 具体类型)` pair 缓存。eface 不需要方法表,省一次间接寻址。

**Q2:interface 是否实现是编译期还是运行期?**
**编译期**检查方法集匹配(赋值 / 传参那一刻)。**运行期**通过 itab.fun[i] 做动态分派(取函数指针 + 间接调用)。所以编译期就能发现"没实现接口",不会运行期才报错。

**Q3:类型断言的两种形式?**
**单返回值** `v.(T)` 失败直接 panic;**双返回值** `v, ok := v.(T)` 失败 `ok=false` 不 panic。生产代码用双返回值;明知不会失败时单返回值更简洁。

**Q4:空接口 any 和反射的关系?**
any 只是擦掉类型,保留 `(_type, data)`;**反射**通过 `reflect.TypeOf(v)` / `reflect.ValueOf(v)` 把这两个字段暴露给运行期操作。所以 any 是反射的入口,但 any 本身没反射开销,**真正慢的是 reflect 包的动态操作**(Type/Value 方法 + 字段查找)。

**Q5:接口值方法集 vs 指针方法集?**
`T` 的方法集 = **所有值接收者方法**;`*T` 的方法集 = **值接收者 + 指针接收者全部**。给接口赋值时,**值 T 不能调用指针方法**(值没有地址)。实践:**一般用 `*T` 实现接口**,除非 T 是值语义类型(time.Time / 小 struct)。

**Q6:接口零值为什么不等于 nil(陷阱)?**
接口值 = (类型指针, 数据指针)。**真 nil interface = (nil, nil)**;`return (*MyErr)(nil)` 装箱后 = (\*MyErr 类型, nil 数据)——**类型非 nil,所以 `== nil` 是 false**。这是 Go 最经典坑,error 处理常踩。修复:显式 `return nil`,不要返回 typed nil。

---

## 五、深水区:原理与源码(被追问时看)

> 下面是 interface 的底层数据结构、itab 缓存机制、动态分派开销、方法集规则、常见坑等源码级内容。**正常面试用不到**,只在被深追"iface 怎么设计的 / itab 怎么生成的 / 装箱开销多大"时才会用到。

## 六、核心原理

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

## 七、八股速记

- 两种结构：**eface (无方法) = type+data**，**iface (有方法) = itab+data**
- itab = 接口类型 + 具体类型 + 方法表，**runtime 缓存**
- **隐式实现**，鸭子类型
- 类型断言双返回值版本不会 panic
- 方法集规则：值接收者的方法 *T 也能调；指针接收者的方法 T 不能调
- **nil interface 不等于装着 nil 指针的 interface**（最经典坑）
- interface{} 装基本类型会装箱到堆 → 性能敏感场景慎用
- 接口设计 idiom：**接受接口，返回结构体**（accept interfaces, return structs）

## 八、面试真题

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

## 九、手写实现

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

## 十、踩坑与最佳实践

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
