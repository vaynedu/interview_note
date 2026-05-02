# string

> Go 字符串：不可变字节序列（UTF-8 编码），底层 `{data, len}` 两字段；`[]byte` 转换可能涉及拷贝

## 一、核心原理

### 1.1 底层结构

```go
// runtime/string.go (reflect.StringHeader 等价)
type stringStruct struct {
    str unsafe.Pointer // 指向底层字节数组
    len int            // 字节长度(不是字符数)
}
```

- 只有**两个字**（指针 + 长度），比 slice 少一个 cap
- 字符串是**只读的**，底层字节数组不可修改
- 多个字符串可共享同一底层数据（编译期常量池、切片操作）

### 1.2 不可变性

字符串一旦创建，内容不可变：

```go
s := "hello"
// s[0] = 'H'  // 编译错误: cannot assign to s[0]
```

要修改只能通过 `[]byte` 转换：

```go
b := []byte(s)
b[0] = 'H'
s2 := string(b)  // "Hello"
```

### 1.3 与 []byte 的转换

两种转换都涉及**内存拷贝**（除非编译器优化掉）：

```go
s := string(b)  // []byte → string: 拷贝 b 的内容
b := []byte(s)  // string → []byte: 拷贝 s 的内容
```

为什么必须拷贝？因为：
- `[]byte` 是可变的，`string` 是不可变的
- 如果不拷贝共享底层数组，修改 []byte 会让 string 变，破坏不可变性

**编译器优化例外**（不拷贝）：
- `for i, c := range []byte(s)` — range 对 []byte(s) 的结果优化，不实际拷贝
- `m[string(b)]` — map 查找用 []byte 查 string key 不拷贝
- `string(b) == "literal"` — 比较时不拷贝

### 1.4 字符串与 rune

- 字符串是 UTF-8 编码的**字节序列**
- `len(s)` 返回**字节数**，不是字符数
- `s[i]` 返回第 i 个**字节**（byte = uint8）
- `range s` 按 **rune（码点）**迭代，自动解码 UTF-8
- `rune` 是 int32，表示一个 Unicode 码点

```go
s := "你好"
len(s)           // 6 (每个汉字 UTF-8 占 3 字节)
s[0]             // 228 (第一个字节, 不是 '你')

for i, r := range s {
    fmt.Printf("%d=%c ", i, r)  // 0=你 3=好
}

runes := []rune(s)  // [20320 22909]
len(runes)          // 2
```

### 1.5 拼接性能

```go
// 差: 每次 + 都创建新 string,O(n²)
s := ""
for _, p := range parts { s += p }

// 好: strings.Builder,内部 []byte,O(n)
var sb strings.Builder
for _, p := range parts { sb.WriteString(p) }
s := sb.String()

// 已知数量时 bytes.Buffer 也可以; strings.Join 最简洁
s := strings.Join(parts, "")
```

`strings.Builder` 的关键：`String()` 方法用 `unsafe.Pointer` 把内部 `[]byte` 转成 string 而不拷贝（因为 Builder 保证 []byte 不再被修改）。

## 二、八股速记

- 字符串 = `{ptr, len}` 两字段，**不可变**
- `len(s)` 是**字节数**，不是字符数
- UTF-8 编码，`range s` 按 rune 迭代自动解码
- `string ↔ []byte` 默认**拷贝**，编译器对少数场景有零拷贝优化
- 拼接用 **`strings.Builder`** 或 `strings.Join`，不要 `+=` 循环
- 字符串可以当 map key（因为不可变且可哈希）
- `s == t` 比较先比 len 再比内容，O(len)
- 字符串的切片 `s[i:j]` 共享底层数据（零拷贝），小心大字符串留住大 byte 数组

## 三、面试真题

**Q1：string 是值类型还是引用类型？**
官方说法是**值类型**（值传递，赋值复制的是 header）。但 header 里的指针指向只读的底层字节数组，多个 string 可共享。行为上既不是纯值类型（共享底层）也不是引用类型（不能修改）。

**Q2：`[]byte` 和 `string` 转换为什么要拷贝？**
因为 string 不可变，[]byte 可变。如果共享底层数组，修改 []byte 会破坏 string 的不可变语义（其他持有同一 string 的代码会突然"变脸"）。拷贝保证了语义安全。

**Q3：`len("你好")` 是多少？**
6。`len` 是字节数，`"你"` 和 `"好"` UTF-8 各占 3 字节。要算字符数用 `utf8.RuneCountInString(s)` 或 `len([]rune(s))`。

**Q4：`string(65)` 结果是什么？**
`"A"`。把 int 当 rune 转 string，等价于 `string(rune(65))`。Go 1.15+ `go vet` 会警告这种写法建议显式用 `strconv.Itoa(65)` 得 `"65"`。

**Q5：字符串拼接哪种最快？**

| 方法 | 复杂度 | 适用 |
| --- | --- | --- |
| `s += t` 循环 | O(n²) | 不推荐 |
| `strings.Builder` | O(n) | 不确定数量时 |
| `strings.Join` | O(n) | 已知切片 |
| `fmt.Sprintf` | 慢 | 格式化场景 |
| `bytes.Buffer` | O(n) | Builder 前的老写法 |
| `[]byte` 预分配 + `string(b)` | O(n) | 需要 byte 操作时 |

**Q6：字符串能不能直接修改某个字节？**
不能。要改必须转 `[]byte` 修改后再转回：

```go
b := []byte(s)
b[i] = 'X'
s = string(b)
```

**Q7：`strings.Builder` 为什么比 `+=` 快？**
1. 内部维护 `[]byte`，append 到末尾（类似 slice 扩容），而不是每次分配新 string
2. `String()` 方法用 `unsafe` 把 []byte 零拷贝转 string
3. 禁止值拷贝（go vet 会报），保证底层 []byte 不被外部篡改

**Q8：`strings.Builder` 和 `bytes.Buffer` 区别？**
Builder 专为构造 string 设计：
- 禁止拷贝（值拷贝会 panic）
- `String()` 零拷贝（buffer 的 `String()` 要拷贝）
- 不支持读取（buffer 是双向的）

构造字符串用 Builder，字节流用 Buffer。

**Q9：字符串作为 map key 有什么要注意的？**
性能上没问题（hash O(len)）。但要注意：
- **长 key 哈希慢**：超长 string 作 key 时考虑哈希后用 `[32]byte`
- **key 生命周期**：如果 string 来自大 byte 数组切片，会阻止原数组 GC

**Q10：字符串的切片 `s[i:j]` 会拷贝吗？**
不会。共享底层数据，新 string 只是新 header（指向 `s` 数据的 i 偏移）。所以大字符串截取小段可能留住大内存：

```go
huge := readBigFile()       // 1GB string
small := huge[0:10]         // 共享底层 1GB 数据
// huge = "" 后, small 依然持有 1GB 的引用
```

**修复**：`small := string([]byte(huge[0:10]))` 强制拷贝。

## 四、手写实现

**1. 反转字符串（处理 UTF-8）：**

```go
// 按字节反转 - 对 ASCII 正确, 对中文会乱
func reverseBytes(s string) string {
    b := []byte(s)
    for i, j := 0, len(b)-1; i < j; i, j = i+1, j-1 {
        b[i], b[j] = b[j], b[i]
    }
    return string(b)
}

// 按 rune 反转 - 正确处理 Unicode
func reverseRunes(s string) string {
    rs := []rune(s)
    for i, j := 0, len(rs)-1; i < j; i, j = i+1, j-1 {
        rs[i], rs[j] = rs[j], rs[i]
    }
    return string(rs)
}
```

**2. 判断字符串是否为 UTF-8：**

```go
import "unicode/utf8"

func isValidUTF8(s string) bool {
    return utf8.ValidString(s)
}
```

**3. 高性能拼接：**

```go
func concat(parts []string) string {
    n := 0
    for _, p := range parts { n += len(p) }

    var sb strings.Builder
    sb.Grow(n)  // 预分配,避免扩容
    for _, p := range parts { sb.WriteString(p) }
    return sb.String()
}
```

**4. 不拷贝的 string ↔ []byte（unsafe，仅在确认只读时用）：**

```go
// string → []byte 零拷贝(但返回的 []byte 不能写, 否则 UB)
func stringToBytes(s string) []byte {
    return unsafe.Slice(unsafe.StringData(s), len(s))
}

// []byte → string 零拷贝(但 []byte 之后不能再改)
func bytesToString(b []byte) string {
    return unsafe.String(unsafe.SliceData(b), len(b))
}
```

> ⚠️ 使用 `unsafe` 转换必须保证生命周期和不变性，否则程序行为未定义。只在热点路径用，普通代码不要这么搞。

## 五、踩坑与最佳实践

### 坑 1：以为 len 是字符数

```go
s := "hello世界"
len(s)  // 11 (5 + 3 + 3), 不是 7
```

面向用户展示的字符数用 `utf8.RuneCountInString(s)`。

### 坑 2：`s[i]` 不是字符

```go
s := "你好"
s[0]  // 228 (0xE4, '你' 的第一个 UTF-8 字节)
```

要取字符用 `[]rune(s)[i]`。

### 坑 3：`+=` 循环拼接性能炸裂

```go
// 拼 1 万个 10 字节的 string
s := ""
for _, p := range parts { s += p }  // O(n²),可能几十 ms
```

必须用 `strings.Builder`。

### 坑 4：`strings.Builder` 被值拷贝

```go
func build() strings.Builder {  // 返回值拷贝 Builder
    var sb strings.Builder
    sb.WriteString("x")
    return sb  // 返回后继续用会 panic: illegal use of non-zero Builder
}
```

Builder 内部有 "no copy" 检查。返回 string，或传 `*Builder`。

### 坑 5：截取大字符串导致内存泄漏

```go
logs := readLogs()              // 100MB
errMsg := logs[offset:offset+100]
// logs 可能被 GC? 不, errMsg 引用着底层数组
```

**修复**：`errMsg := string([]byte(logs[offset:offset+100]))` 强拷贝。

### 坑 6：`string(int)` 当 `strconv.Itoa`

```go
n := 65
s := string(n)  // "A", 不是 "65"
```

Go 1.15+ vet 会提示。整数转字符串用 `strconv.Itoa` / `strconv.FormatInt`。

### 坑 7：字符串不能比较 nil

```go
var s string
if s == nil { }  // 编译错误: cannot convert nil to string
if s == "" { }   // 正确
```

string 零值是 `""`，不是 nil。

### 最佳实践

- 循环拼接一律 `strings.Builder`（或 `strings.Join` 已知切片时）
- 判空用 `s == ""` 或 `len(s) == 0`
- 处理中文/emoji 用 `[]rune` 而不是 `[]byte`
- 热路径避免 `string ↔ []byte` 来回转，预先确定哪边最终用
- map key 用 string 时，长度过大考虑哈希
- 大字符串切片要警惕内存驻留，必要时强制拷贝
- `unsafe` 零拷贝转换只在压测验证瓶颈后再用，且充分测试生命周期
