# encoding

> Go 序列化包：json / xml / gob / base64 / hex；tag 控制字段映射，反射驱动，性能瓶颈常在此

## 一、核心原理

### 1.1 包覆盖

| 包 | 用途 |
| --- | --- |
| `encoding/json` | JSON 编解码（最常用） |
| `encoding/xml` | XML |
| `encoding/gob` | Go 专用二进制（仅 Go 之间通信用） |
| `encoding/base64` | Base64 编解码 |
| `encoding/hex` | 16 进制字符串 |
| `encoding/binary` | 二进制读写（uvarint、定长） |
| `encoding/csv` | CSV |

### 1.2 JSON 编解码

```go
type User struct {
    ID    int64  `json:"id"`
    Name  string `json:"name,omitempty"`     // 零值省略
    Email string `json:"-"`                   // 不序列化
    Date  string `json:"date,string"`         // 强制 string 化
}

// Marshal: struct → []byte
b, err := json.Marshal(u)

// Unmarshal: []byte → struct
err := json.Unmarshal(b, &u)

// 流式: io.Writer / Reader
json.NewEncoder(w).Encode(u)
json.NewDecoder(r).Decode(&u)
```

### 1.3 tag 选项

```go
`json:"name,omitempty,string"`
//      ^         ^         ^
//      字段名    零值省略   字段值再 string 化(数值用)
```

常用：
- `json:"id"` 重命名
- `json:"-"` 不序列化
- `json:",omitempty"` 零值不输出
- `json:",string"` 数值/bool 转 string（兼容 JS 大整数）

### 1.4 类型映射

| Go | JSON |
| --- | --- |
| bool | true/false |
| int/uint/float | number |
| string | string |
| nil | null |
| slice/array | array |
| map[string]T | object |
| struct | object |
| `interface{}` | 反射动态决定 |

`map` key 必须是 string 类型（或实现 `encoding.TextMarshaler`）。

### 1.5 自定义编解码

实现 `Marshaler` / `Unmarshaler` 接口：

```go
type Date time.Time

func (d Date) MarshalJSON() ([]byte, error) {
    return []byte(`"` + time.Time(d).Format("2006-01-02") + `"`), nil
}

func (d *Date) UnmarshalJSON(data []byte) error {
    s := strings.Trim(string(data), `"`)
    t, err := time.Parse("2006-01-02", s)
    if err != nil { return err }
    *d = Date(t)
    return nil
}
```

或更通用的 `TextMarshaler` / `TextUnmarshaler`（同时被 json/xml/yaml 等用）。

### 1.6 性能（标准库 vs 第三方）

| 库 | 速度 | 内存 | 备注 |
| --- | --- | --- | --- |
| `encoding/json` | 1x | 1x | 反射，标准 |
| `easyjson` | 5~10x | < | 代码生成 |
| `sonic` (字节跳动) | 5~10x | < | JIT，amd64 |
| `jsoniter` | 2~3x | < | 兼容 API |
| `goccy/go-json` | 3~5x | < | 反射优化 |

热路径序列化是常见瓶颈（接口聚合、日志结构化、消息队列编码）。

### 1.7 流式 vs 一次性

```go
// 一次性: 全部读到内存
json.Unmarshal(data, &v)

// 流式: 边读边解析(适合大文件/网络流)
dec := json.NewDecoder(r)
for dec.More() {
    var item Item
    dec.Decode(&item)
}
```

大数据用流式避免内存峰值。

## 二、八股速记

- `json.Marshal` / `Unmarshal` 是入口；流式用 `Encoder` / `Decoder`
- struct tag 控制字段名、零值省略、字符串化
- 字段必须**导出**（首字母大写）才能编解码
- map key 默认必须 string
- 自定义类型实现 `MarshalJSON` / `UnmarshalJSON`
- 标准库 `encoding/json` 用反射，慢；热路径用 sonic / easyjson
- `json.Number` 防大整数精度丢失
- 解析未知结构用 `map[string]any` 或 `json.RawMessage` 延迟解析
- 大数据流式用 `json.Decoder`，避免一次性加载

## 三、面试真题

**Q1：未导出字段为什么序列化不出来？**
`encoding/json` 用反射读字段，反射不能访问未导出字段（小写开头）。设计上：未导出字段是包内细节，序列化通常给外部用，应该显式（导出）。

**修复**：字段大写 + 用 tag 改 JSON 名：

```go
type T struct {
    Name string `json:"name"`  // 大写,序列化为 "name"
}
```

**Q2：`omitempty` 怎么判断"空"？**
零值即省略：
- 数值 0
- 字符串 ""
- nil 指针/slice/map/interface
- 长度 0 的 slice/map（注意：**空但非 nil 的 slice 也算空**）
- bool false

**坑**：bool 字段不能用 omitempty，false 会被省略导致前端拿不到字段。

**Q3：解析未知 JSON 结构怎么办？**

```go
var raw map[string]any
json.Unmarshal(data, &raw)
// raw["key"] 是 any, 类型断言取值
```

或用 `json.RawMessage` 延迟解析子树：

```go
type Msg struct {
    Type string          `json:"type"`
    Data json.RawMessage `json:"data"`  // 留 []byte, 之后按 type 决定解析成什么
}
```

**Q4：JSON 数字默认解析成什么？**
默认 `float64`。问题：超过 `2^53` 的整数精度丢失。

**修复**：

```go
dec := json.NewDecoder(r)
dec.UseNumber()  // 数字解析成 json.Number(string), 不丢精度
```

或目标字段直接用 `int64`，json 库会按目标类型解析。

**Q5：自定义类型怎么序列化？**

```go
type Status int
const (
    StatusActive Status = iota
    StatusInactive
)

func (s Status) MarshalJSON() ([]byte, error) {
    switch s {
    case StatusActive: return []byte(`"active"`), nil
    case StatusInactive: return []byte(`"inactive"`), nil
    }
    return nil, fmt.Errorf("unknown status %d", s)
}
```

**Q6：`time.Time` 怎么自定义格式？**
`time.Time` 默认输出 RFC3339（`"2006-01-02T15:04:05Z07:00"`）。要自定义：

```go
type Date time.Time

func (d Date) MarshalJSON() ([]byte, error) {
    return []byte(`"` + time.Time(d).Format("2006-01-02") + `"`), nil
}
```

或改用 `string` 字段手动转。

**Q7：`json.Marshal` 和 `Encoder.Encode` 区别？**
- `Marshal` 返回 `[]byte`，整体在内存
- `Encoder.Encode(v)` 直接写到 `io.Writer`，加上换行符 `\n`

写 HTTP 响应：

```go
json.NewEncoder(w).Encode(v)  // 比 Marshal+Write 少一次内存拷贝
```

**Q8：序列化性能怎么优化？**
1. **换库**：sonic / easyjson（代码生成）/ goccy/go-json
2. **复用 Encoder**：`json.Encoder` 内部有 buffer
3. **流式**：大对象用 Encoder 直接写 Writer
4. **减少嵌套**：扁平化 struct
5. **避免 interface{}**：用具体类型
6. **用 RawMessage 延迟解析**：只解析需要的部分

热点接口的 JSON 编解码可能占 CPU 30%+，pprof 看 `reflect.Value.Interface` / `encoding/json.*` 是信号。

**Q9：`UnmarshalJSON` 的接收者是值还是指针？**
**必须指针**（要修改内部字段）：

```go
func (d *Date) UnmarshalJSON(data []byte) error { ... }
```

否则 Unmarshal 调不到（值方法不在 *T 的方法集需求里 → 不对，反着说：值方法在指针方法集里。但 Unmarshal 需要修改对象，必须指针接收者）。

**Q10：`gob` 和 `json` 的区别？**

| | gob | json |
| --- | --- | --- |
| 通用性 | 仅 Go | 跨语言 |
| 体积 | 小（二进制+元数据） | 大（文本） |
| 速度 | 快 | 慢 |
| 调试 | 难（二进制） | 易（人可读） |
| 类型安全 | 强（自带类型信息） | 弱（按字段名） |

gob 用于 Go 服务间通信（不如直接用 protobuf）。json 是 Web/RPC 通用格式。

## 四、手写实现

**1. 流式处理大 JSON 数组：**

```go
func processStream(r io.Reader) error {
    dec := json.NewDecoder(r)
    // 读 [
    if _, err := dec.Token(); err != nil { return err }

    for dec.More() {
        var item Item
        if err := dec.Decode(&item); err != nil { return err }
        process(item)
    }

    // 读 ]
    if _, err := dec.Token(); err != nil { return err }
    return nil
}
```

**2. 多类型消息分发（type field + RawMessage）：**

```go
type Envelope struct {
    Type string          `json:"type"`
    Data json.RawMessage `json:"data"`
}

type LoginEvent struct{ User string `json:"user"` }
type LogoutEvent struct{ User string `json:"user"` }

func handle(b []byte) error {
    var env Envelope
    if err := json.Unmarshal(b, &env); err != nil { return err }
    switch env.Type {
    case "login":
        var e LoginEvent
        json.Unmarshal(env.Data, &e)
        return onLogin(e)
    case "logout":
        var e LogoutEvent
        json.Unmarshal(env.Data, &e)
        return onLogout(e)
    }
    return fmt.Errorf("unknown type %s", env.Type)
}
```

**3. 大数字保留精度：**

```go
type Order struct {
    ID    json.Number `json:"id"`     // 解析成 string-form,不丢精度
    Price string      `json:"price"`  // 钱用 string 是惯例(decimal)
}
```

**4. 通用 ToJSON 辅助：**

```go
func MustJSON(v any) string {
    b, err := json.Marshal(v)
    if err != nil { panic(err) }
    return string(b)
}

// 调试日志
log.Printf("user: %s", MustJSON(u))
```

## 五、踩坑与最佳实践

### 坑 1：字段没大写

```go
type T struct { name string }  // 小写
b, _ := json.Marshal(T{name: "x"})
// b = "{}" 字段被忽略
```

### 坑 2：bool + omitempty

```go
type Config struct {
    Enabled bool `json:"enabled,omitempty"`
}
json.Marshal(Config{Enabled: false})  // {} ← false 被省略, 前端拿不到字段
```

**修复**：去掉 omitempty，或用 `*bool`（nil 表示未设置）。

### 坑 3：大整数精度丢失

```go
b := []byte(`{"id": 9007199254740993}`)  // 大于 2^53
var m map[string]any
json.Unmarshal(b, &m)
fmt.Println(m["id"])  // 9.007199254740992e+15 ← 精度丢失
```

**修复**：目标 struct 用 `int64`，或解析时 `dec.UseNumber()`。

### 坑 4：未指定 `Decoder.DisallowUnknownFields`

```go
type T struct { ID int }
data := []byte(`{"id": 1, "extra": "x"}`)
json.Unmarshal(data, &t)  // 默默忽略 "extra"
```

如果想严格匹配：

```go
dec := json.NewDecoder(bytes.NewReader(data))
dec.DisallowUnknownFields()
dec.Decode(&t)  // 报错: json: unknown field "extra"
```

API 校验场景必开。

### 坑 5：嵌套 struct 用 omitempty 失效

```go
type Inner struct { X int }
type Outer struct {
    I Inner `json:"i,omitempty"`  // omitempty 对 struct 无效!
}
json.Marshal(Outer{})  // {"i":{"x":0}} ← I 不会被省略
```

`omitempty` 对 struct 字段无效。**修复**：用 `*Inner`，nil 时省略。

### 坑 6：tag 写错

```go
`json:"name "`     // 错: 多空格
`json"name"`        // 错: 缺冒号
```

`go vet` 能查出 json/xml tag 格式错。

### 坑 7：循环引用 panic

```go
type Node struct { Self *Node }
n := &Node{}
n.Self = n
json.Marshal(n)  // panic: runtime error: stack overflow
```

JSON 不支持循环引用。涉及循环结构（如 graph）需要自定义 MarshalJSON 或用 ID 引用替代。

### 坑 8：默认时间格式不符

后端默认 RFC3339 (`2006-01-02T15:04:05Z`)，前端可能要 `2006-01-02 15:04:05`。

**修复**：自定义类型 + MarshalJSON，或 string 字段。

### 最佳实践

- **DTO 字段都大写 + tag** 显式指定
- **bool 不加 omitempty**，需要省略用 `*bool`
- **大数字用 int64 或 string**
- **API 入参严格匹配**：`DisallowUnknownFields()`
- **写响应用 Encoder**：少一次内存拷贝
- **大文件用流式**：`json.NewDecoder(r)`
- **热点路径换 sonic/easyjson**：标准库 5~10x 性能差距
- **time/decimal/UUID 等用自定义类型** + MarshalJSON 统一格式
- 消息分发用 `RawMessage` 延迟解析
- **DTO 和领域模型分离**：DTO 带 json tag，领域对象不带
