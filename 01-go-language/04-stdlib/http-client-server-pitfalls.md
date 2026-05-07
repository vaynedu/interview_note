# net/http 线上使用坑

> Go 的 `net/http` 很强，但默认值偏通用。线上服务最常见的问题不是不会发请求，而是超时、连接池、Body 关闭和服务端保护没做好。

## 一、HTTP Client 核心原则

```text
复用 http.Client。
必须设置超时。
必须关闭 resp.Body。
必须限制连接池。
请求必须带 context。
```

不要每次请求都创建 client：

```go
func bad(url string) error {
    client := &http.Client{}
    _, err := client.Get(url)
    return err
}
```

推荐统一封装：

```go
var client = &http.Client{
    Timeout: 3 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 20,
        IdleConnTimeout:     90 * time.Second,
        TLSHandshakeTimeout: 3 * time.Second,
    },
}
```

## 二、超时要分层理解

| 超时 | 含义 |
| --- | --- |
| `http.Client.Timeout` | 整个请求生命周期超时 |
| `DialContext.Timeout` | 建立 TCP 连接超时 |
| `TLSHandshakeTimeout` | TLS 握手超时 |
| `ResponseHeaderTimeout` | 等响应头超时 |
| `ExpectContinueTimeout` | 100-continue 等待超时 |
| `context timeout` | 业务链路取消和截止时间 |

推荐：

```go
ctx, cancel := context.WithTimeout(parent, 2*time.Second)
defer cancel()

req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
if err != nil {
    return err
}

resp, err := client.Do(req)
if err != nil {
    return err
}
defer resp.Body.Close()
```

## 三、Body 不关闭的坑

错误写法：

```go
resp, err := client.Get(url)
if err != nil {
    return err
}
body, err := io.ReadAll(resp.Body)
```

忘记 `Close` 会导致连接无法复用，严重时 fd 泄漏、连接池打满。

正确：

```go
resp, err := client.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close()

if resp.StatusCode >= 500 {
    return fmt.Errorf("remote 5xx: %d", resp.StatusCode)
}
```

如果要复用连接，需要读完 body 或丢弃剩余内容：

```go
defer func() {
    io.Copy(io.Discard, resp.Body)
    resp.Body.Close()
}()
```

大 body 不要无脑 `io.ReadAll`，要限制大小：

```go
body, err := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
```

## 四、连接池配置坑

默认 `MaxIdleConnsPerHost` 很小，高并发请求单一域名时容易频繁建连。

关键参数：

| 参数 | 作用 |
| --- | --- |
| `MaxIdleConns` | 全局最大空闲连接 |
| `MaxIdleConnsPerHost` | 每个 host 最大空闲连接 |
| `MaxConnsPerHost` | 每个 host 最大总连接 |
| `IdleConnTimeout` | 空闲连接保留时间 |

如果下游扛不住，可以用 `MaxConnsPerHost` 做客户端侧限流，避免把下游打死。

## 五、HTTP Server 保护

默认 server：

```go
http.ListenAndServe(":8080", mux)
```

线上不推荐裸用，因为缺少超时保护。

推荐：

```go
srv := &http.Server{
    Addr:              ":8080",
    Handler:           mux,
    ReadHeaderTimeout: 2 * time.Second,
    ReadTimeout:       5 * time.Second,
    WriteTimeout:      10 * time.Second,
    IdleTimeout:       60 * time.Second,
}

if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
    return err
}
```

慢客户端攻击、网络异常、请求体过大都可能拖住 goroutine 和连接。

## 六、优雅关闭

```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

if err := srv.Shutdown(ctx); err != nil {
    return err
}
```

`Shutdown` 会停止接收新连接，并等待已有请求处理完成。注意：

- 长请求必须响应 context。
- 后台 worker 也要有独立退出机制。
- Kubernetes 里要配合 `preStop` 和 `terminationGracePeriodSeconds`。

## 七、线上案例

### 案例：下游接口变慢导致本服务 goroutine 堆积

背景：订单服务调用营销服务查询优惠信息。

问题代码：

```go
resp, err := http.Get(url)
```

营销服务偶发慢查询，HTTP 请求没有超时，订单服务 goroutine 卡在 IO wait。高峰期连接池、goroutine、内存一起上涨，最终订单接口 P99 抖动。

处理：

- 统一替换为带超时的 client。
- 请求 context 继承入口请求 deadline。
- 对营销服务加熔断和降级。
- 通过 `MaxConnsPerHost` 限制对单一下游并发。
- 监控下游状态码、耗时、超时数、连接池等待。

## 八、面试真题

**Q1：Go HTTP client 为什么要复用？**

`http.Client` 内部通过 Transport 管理连接池。频繁创建 client/transport 会导致连接无法复用，增加 TCP/TLS 建连成本，甚至造成 fd 和端口资源压力。

**Q2：线上 HTTP 调用卡住怎么排查？**

看 goroutine pprof 是否大量停在 `net/http`、`IO wait`；看下游耗时、超时、连接池指标；检查 client 是否设置 Timeout、Transport 参数、context deadline，以及 Body 是否关闭。

**Q3：服务端为什么要设置 ReadHeaderTimeout？**

防止客户端慢慢发送请求头占住连接和 goroutine，降低慢请求攻击或异常网络对服务的影响。

## 九、面试表达

```text
我在线上会统一封装 HTTP client，复用 Transport，设置整体超时和连接池参数；每个请求都带 context，并确保 resp.Body 关闭。
服务端不会直接裸用 ListenAndServe，而是配置 ReadHeaderTimeout、ReadTimeout、WriteTimeout 和优雅关闭。
HTTP 问题排查时，我会同时看 goroutine pprof、下游耗时、状态码、超时和连接池指标。
```
