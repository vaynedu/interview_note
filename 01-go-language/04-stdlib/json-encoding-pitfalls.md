# JSON 编解码线上坑

> `encoding/json` 简单好用，但线上常见坑很多：数字精度、空值语义、字段兼容、反射性能、大响应内存。

## 一、核心原则

```text
对外 API 用明确 struct。
金额和 ID 不要随便走 float64。
区分 nil、空数组和字段缺失。
大 JSON 做大小限制和流式处理。
热路径关注反射和分配。
```

## 二、数字精度坑

默认把未知 JSON 解到 `map[string]any` 时，数字会变成 `float64`。

```go
var m map[string]any
json.Unmarshal(data, &m)
fmt.Printf("%T\n", m["order_id"]) // float64
```

大整数可能丢精度，订单号、用户 ID、金额都很危险。

推荐：

```go
type Order struct {
    OrderID string `json:"order_id"`
    Amount  int64  `json:"amount"` // 单位：分
}
```

如果必须用动态结构：

```go
dec := json.NewDecoder(r)
dec.UseNumber()
```

再显式转换：

```go
var m map[string]any
if err := dec.Decode(&m); err != nil {
    return err
}
n, err := m["order_id"].(json.Number).Int64()
```

## 三、金额不要用 float

错误：

```go
type PayReq struct {
    Amount float64 `json:"amount"`
}
```

推荐：

```go
type PayReq struct {
    AmountCent int64 `json:"amount_cent"`
}
```

或者用 decimal 类型，但接口边界要统一。

## 四、omitempty 的语义坑

```go
type User struct {
    Age int `json:"age,omitempty"`
}
```

当 `Age=0` 时字段不会输出。但 0 可能是合法值，不等于“没有传”。

如果要区分缺失和零值：

```go
type UserPatch struct {
    Age *int `json:"age"`
}
```

语义：

```text
Age == nil      字段缺失
*Age == 0       明确传了 0
*Age == 18      明确传了 18
```

## 五、nil slice 和空数组

```go
type Resp struct {
    Items []int `json:"items"`
}

json.Marshal(Resp{}) // {"items":null}
```

如果前端或接口协议要求 `[]`：

```go
resp := Resp{Items: make([]int, 0)}
```

也可以统一在响应组装层把 nil slice 转为空 slice。

## 六、未知字段处理

默认 `json.Decoder` 会忽略未知字段。这对兼容性友好，但配置类或强校验接口可能不安全。

严格模式：

```go
dec := json.NewDecoder(r)
dec.DisallowUnknownFields()
```

适合：

- 管理后台配置。
- 内部强约束接口。
- 金融、支付、权限类请求。

不一定适合：

- 对外开放 API。
- 需要灰度兼容新老字段的接口。

## 七、大 JSON 内存问题

错误：

```go
body, _ := io.ReadAll(r.Body)
json.Unmarshal(body, &req)
```

问题：

- 未限制 body 大小。
- 大请求会一次性占用内存。
- 容易被异常请求打爆。

推荐：

```go
r.Body = http.MaxBytesReader(w, r.Body, 1<<20)
dec := json.NewDecoder(r.Body)
if err := dec.Decode(&req); err != nil {
    return err
}
```

大数组处理可以流式 decode：

```go
dec := json.NewDecoder(r)

tok, err := dec.Token()
if err != nil {
    return err
}
_ = tok // '['

for dec.More() {
    var item Item
    if err := dec.Decode(&item); err != nil {
        return err
    }
    handle(item)
}
```

## 八、时间字段

`time.Time` 默认使用 RFC3339 格式。

常见问题：

- 前端传时间戳，后端 struct 用 `time.Time` 直接解析失败。
- 时区没有统一。
- `omitempty` 对零值时间不一定符合预期。

建议：

- API 统一时间格式，优先 ISO8601/RFC3339 或毫秒时间戳。
- 服务内部统一 UTC 或明确时区。
- 数据库存储和展示转换分开。

## 九、性能坑

`encoding/json` 使用反射，普通业务足够，但热路径要关注：

- `map[string]any` 分配多。
- 反射成本高。
- `string` / `[]byte` 转换频繁。
- 大对象 marshal 造成内存峰值。

优化方向：

- 用明确 struct。
- 预分配 slice。
- 避免中间 `map[string]any`。
- 大响应用 `json.Encoder` 流式写。
- 极端热点再考虑高性能 JSON 库，但要评估兼容性。

## 十、线上案例

### 案例：订单号被解析成 float64 后精度丢失

背景：活动系统接收订单系统回调，为了快速兼容多种事件，先把 body 解成 `map[string]any`。

问题：

```go
orderID := fmt.Sprintf("%.0f", m["order_id"].(float64))
```

大订单号超过 JavaScript/float64 安全整数范围后，末尾几位变化，导致活动发奖查不到订单。

处理：

- 订单号统一改成 string。
- 动态解析使用 `Decoder.UseNumber()`。
- 回调结构改成明确 struct。
- 对关键 ID 增加格式校验和日志。

复盘：

```text
订单号、用户 ID、金额这类关键字段不要走 float64。
动态 JSON 只能作为兼容层，核心链路必须尽快收敛到明确 schema。
```

## 十一、面试真题

**Q1：`map[string]any` 解析 JSON 有什么坑？**

数字默认变成 float64，大整数可能丢精度；类型断言容易 panic；结构不明确，字段兼容和校验困难；分配和反射成本也更高。

**Q2：omitempty 有什么坑？**

它会省略零值，但零值不一定代表没传。比如年龄 0、库存 0、开关 false 都可能是有效业务值。需要区分缺失和零值时，用指针字段。

**Q3：大 JSON 请求怎么防护？**

用 `http.MaxBytesReader` 限制 body 大小，避免无脑 `io.ReadAll`；必要时用 `json.Decoder` 流式解析；对数组数量和字段大小做业务限制。

## 十二、面试表达

```text
我一般不会在核心链路长期使用 map[string]any 解析 JSON，尤其是订单号、金额、用户 ID 这类字段。
对外接口我会用明确 struct，金额用 int64 分，缺失和零值用指针区分，大请求用 MaxBytesReader 和 Decoder 控制内存。
如果 JSON 成为性能瓶颈，先减少中间对象和反射，再评估高性能库。
```
