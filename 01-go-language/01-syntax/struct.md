# struct

> Go 结构体：值类型，内存连续，支持组合、tag、内存对齐；Go 没有继承，靠 embedding 实现复用

## 一、核心原理

### 1.1 值类型与内存布局

struct 是**值类型**：
- 赋值、函数传参都是**整体拷贝**
- 字段在内存中**连续排列**，顺序与声明一致
- 零值 = 所有字段都是对应零值

```go
type User struct {
    ID   int64   // 8B
    Name string  // 16B (ptr+len)
    Age  int32   // 4B
}
// 理论 28B,实际因对齐会是 32B
```

### 1.2 内存对齐

CPU 按字长对齐访问内存效率最高。编译器自动在字段间插入 **padding** 使每个字段的地址满足其对齐要求。

**对齐规则**（amd64）：
- `bool/int8/uint8`：1B 对齐
- `int16/uint16`：2B 对齐
- `int32/uint32/float32`：4B 对齐
- `int64/uint64/float64/指针/string/slice/map`：8B 对齐
- struct 的对齐 = 其最大字段的对齐

**字段顺序影响 size**：

```go
type Bad struct {
    A bool   // 1B + 7B padding
    B int64  // 8B
    C bool   // 1B + 7B padding
}
// sizeof: 24B

type Good struct {
    B int64  // 8B
    A bool   // 1B
    C bool   // 1B + 6B padding
}
// sizeof: 16B
```

**口诀**：大字段在前，小字段在后（或相同大小的挨在一起）。

### 1.3 struct tag

struct 字段后的反引号字符串，runtime 通过反射读取，用于序列化、校验、ORM 映射等。

```go
type User struct {
    ID    int64  `json:"id" db:"user_id"`
    Email string `json:"email" validate:"email"`
}
```

tag 格式：`key1:"value1" key2:"value2"`（空格分隔，key 和 value 间用 `:`，value 必须双引号）。

### 1.4 匿名字段（embedding）

Go 没有 extends，用嵌入结构体实现"继承"的效果：

```go
type Animal struct{ Name string }
func (a *Animal) Speak() string { return a.Name + " speaks" }

type Dog struct {
    Animal  // 匿名嵌入
    Breed string
}

d := Dog{Animal: Animal{Name: "Rex"}, Breed: "Labrador"}
d.Name       // "Rex" (提升字段)
d.Speak()    // "Rex speaks" (提升方法)
d.Animal.Name // 显式访问
```

**特点**：
- 字段和方法会"提升"到外层
- 同名冲突时外层优先，或必须显式指定
- 嵌入的类型名就是字段名（`d.Animal`）
- 嵌入 `*T` 或 `T` 都可以

### 1.5 方法集规则

| 接收者 | 值 T 能调用 | 指针 *T 能调用 |
| --- | --- | --- |
| `func (t T) Foo()` | ✓ | ✓ |
| `func (t *T) Bar()` | ✗（不能取地址时） | ✓ |

**实操规则**：
- 要修改字段用指针接收者
- 一个类型的方法**要么都值接收者要么都指针接收者**，不要混
- 大 struct 用指针避免拷贝

### 1.6 零值可用

好的 struct 设计应该**零值即可用**：

```go
var b bytes.Buffer     // 零值可用,直接 b.Write(...)
var mu sync.Mutex      // 零值可用,直接 mu.Lock()
var wg sync.WaitGroup  // 零值可用
```

这是 Go 的核心惯用法，避免到处写 `New()` 构造。

### 1.7 比较与哈希

- struct 所有字段都可比较时，struct 本身可比较（可作为 map key）
- 含 slice/map/func 字段的 struct 不可比较
- 空 struct `struct{}` size 为 0，常用作 set 或信号

```go
type Key struct { A, B int }
m := map[Key]string{}  // ✓

type Bad struct { S []int }
// m := map[Bad]string{}  // 编译错误
```

## 二、八股速记

- struct 是**值类型**，赋值传参都拷贝
- 内存**连续布局**，字段顺序影响 size（大字段在前节省 padding）
- tag 是反射元数据，json/db/validate 常用
- **embedding** 不是继承，字段/方法会提升到外层
- **零值可用**是 Go 惯用法
- 方法集：值接收者方法集小，指针接收者方法集大
- 空 struct 是 0 字节，可做 set 或信号
- 全可比较字段才可比较，才能当 map key

## 三、面试真题

**Q1：struct 是值类型，那传个大 struct 给函数不就拷贝一份？**
是的，所以大 struct（几十字段或含大数组）通常**传指针**：
- 避免拷贝开销
- 允许函数修改
- 避免方法集问题

小 struct（几个字段）值传递反而快，因为可以存寄存器、避免间接寻址和 GC 扫描指针。

**Q2：以下两个 struct 内存占用多少？**

```go
type A struct {
    a bool
    b int32
    c bool
}
type B struct {
    b int32
    a bool
    c bool
}
```

- A：1 + 3(pad) + 4 + 1 + 3(pad) = **12 字节**
- B：4 + 1 + 1 + 2(pad) = **8 字节**

按字段大小从大到小排可减少 padding。可用 `unsafe.Sizeof(v)` 验证。

**Q3：匿名字段会导致方法冲突吗？**

```go
type A struct{}
func (A) Foo() {}

type B struct{}
func (B) Foo() {}

type C struct{ A; B }
```

- `c.Foo()` 编译错误：ambiguous selector
- `c.A.Foo()` / `c.B.Foo()` 显式调用 OK
- 外层定义 `func (C) Foo()` 会覆盖两个匿名字段的 Foo

**Q4：struct tag 怎么读？**

```go
type User struct {
    Name string `json:"name" db:"user_name"`
}

t := reflect.TypeOf(User{})
f, _ := t.FieldByName("Name")
fmt.Println(f.Tag.Get("json"))  // "name"
fmt.Println(f.Tag.Get("db"))    // "user_name"
```

tag 格式错了 runtime 读不到但编译不报错（Go 1.13+ vet 会检查 json/xml tag 格式）。

**Q5：结构体能比较吗？能当 map key 吗？**

- 所有字段都可比较 → struct 可比较，可作 key
- 含 slice/map/func → 不可比较
- 指针字段可以（比较指针值）
- array 可以（元素可比较时）

```go
type P struct { X, Y int }
p1 == p2  // 按字段依次比

type Q struct { S []int }
// q1 == q2  // 编译错误
```

**Q6：空 struct 有什么用？**

```go
set := map[string]struct{}{}       // 比 map[string]bool 省内存
done := make(chan struct{})         // 信号,不携带数据
var dummy struct{}                   // 占位,0 字节
```

**Q7：embedding 和组合的区别？**
- **组合**：显式命名字段
- **embedding**：匿名字段，方法/字段会提升

```go
// 组合
type S1 struct { a A }
s1.a.Foo()

// embedding
type S2 struct { A }
s2.Foo()  // 直接调
```

embedding 用于"is-a" 式复用（Dog is an Animal），组合用于 "has-a"（Order has a Customer）。

**Q8：为什么推荐零值可用？**
省去 `New()` 构造器的心智负担，也符合 Go 简洁哲学。例：

```go
// 零值不可用
cfg := &Config{}  // 还得 cfg.Init() 或 NewConfig()
// 零值可用
var sb strings.Builder; sb.WriteString("x")  // 直接用
```

**Q9：方法接收者用值还是指针？**
原则：
- 需要**修改字段** → 必须指针
- **大 struct**（几十字段或含数组）→ 指针，避免拷贝
- 含 `sync.Mutex` 等**禁止拷贝**的字段 → 指针
- 其他场景：**一致性优先**，一个类型要么全值要么全指针

小 struct 没有 mutex 时值接收者更简单，性能甚至更好（避免指针逃逸）。

## 四、手写实现

**1. 按字段大小排序优化内存：**

```go
// 优化前: sizeof = 32
type Order1 struct {
    Paid   bool     // 1 + 7 pad
    Amount float64  // 8
    Done   bool     // 1 + 7 pad
    User   *User    // 8
}

// 优化后: sizeof = 24
type Order2 struct {
    Amount float64  // 8
    User   *User    // 8
    Paid   bool     // 1
    Done   bool     // 1 + 6 pad
}
```

**2. 零值可用的配置对象：**

```go
type Config struct {
    mu       sync.Mutex
    settings map[string]string
}

func (c *Config) Set(k, v string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.settings == nil {
        c.settings = make(map[string]string)  // 延迟初始化
    }
    c.settings[k] = v
}

// 使用: var cfg Config; cfg.Set("k", "v") 直接可用
```

**3. 嵌入实现"继承"：**

```go
type Logger struct{ prefix string }
func (l *Logger) Log(msg string) { fmt.Println(l.prefix+":", msg) }

type Service struct {
    *Logger        // 嵌入指针类型
    name   string
}

s := &Service{Logger: &Logger{prefix: "svc"}, name: "orders"}
s.Log("started")  // 直接调用,输出 "svc: started"
```

**4. 用 tag 做 JSON 序列化：**

```go
type User struct {
    ID       int64  `json:"id"`
    Name     string `json:"name,omitempty"`
    Password string `json:"-"`  // 不序列化
}

b, _ := json.Marshal(User{ID: 1, Name: "Alice"})
// {"id":1,"name":"Alice"}
```

## 五、踩坑与最佳实践

### 坑 1：值接收者方法修改不生效

```go
type C struct { count int }
func (c C) Inc() { c.count++ }  // 值接收者,改的是副本

var c C
c.Inc()
fmt.Println(c.count)  // 0
```

**修复**：改指针接收者 `func (c *C) Inc()`。

### 坑 2：把含 mutex 的 struct 值拷贝

```go
type Counter struct {
    mu sync.Mutex
    n  int
}

c1 := Counter{}
c2 := c1  // 错! mutex 被复制,保护的是不同的 n
```

`go vet` 会报 "assignment copies lock value"。解决：用指针。

### 坑 3：tag 语法错误运行时才暴露

```go
type T struct {
    A string `json:"a"; db:"a"`  // 错: 多个 tag 用空格分隔, 不是分号
}
```

Go 1.13+ vet 会检查常见 tag（json、xml）。上线前跑 `go vet`。

### 坑 4：embedding 字段同名冲突

```go
type A struct{ Name string }
type B struct{ Name string }
type C struct{ A; B }

var c C
c.Name  // ambiguous selector c.Name
```

外层定义同名字段覆盖，或显式 `c.A.Name`。

### 坑 5：指针接收者不能从值调用未寻址的方法

```go
type M map[int]int
func (m M) put(k, v int) { m[k] = v }
func (m *M) init() { *m = make(M) }

func f() {
    // M(nil).init()  // 编译错误: cannot call pointer method on non-addressable
    m := M(nil)
    m.init()  // OK, m 可寻址
}
```

map 字面量不可寻址，返回值/常量也不可寻址。

### 坑 6：struct 含 slice/map 时，浅拷贝共享底层

```go
type S struct{ Items []int }

s1 := S{Items: []int{1, 2, 3}}
s2 := s1                 // 浅拷贝, Items 共享底层数组
s2.Items[0] = 99
s1.Items[0]              // 99 ← 被改了
```

深拷贝要自己写 `copy(dst.Items, src.Items)`。

### 坑 7：字段顺序影响 size

结构体数组时影响显著：

```go
type Bad struct { A bool; B int64; C bool }   // 24B
type Good struct { B int64; A bool; C bool }  // 16B

var bads [1e6]Bad   // 24MB
var goods [1e6]Good // 16MB
```

百万级数据量的场景优化字段顺序收益明显。工具：`fieldalignment` (golang.org/x/tools/go/analysis/passes/fieldalignment)。

### 最佳实践

- **大字段在前**减少 padding；密集数据场景跑 fieldalignment
- **方法接收者全一致**：一个类型要么全值接收者，要么全指针
- **禁拷贝标记**：含 `sync.Mutex`/`sync.WaitGroup` 的 struct 一律用指针
- **零值可用**：设计 struct 时考虑零值语义，延迟初始化内部 map/slice
- **tag 统一风格**：`json:"name" db:"col"` 字段名用 snake_case
- **不要暴露字段**：除 DTO 外，业务 struct 字段小写 + 提供方法
- **组合优于继承**：embedding 节制使用，避免方法/字段冲突难排查
