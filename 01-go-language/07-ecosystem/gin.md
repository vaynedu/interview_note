# Gin

> Go Web 框架：基于 httprouter 的 Radix 树路由，最快的主流 Go HTTP 框架之一；中间件链 + Context 复用 + 绑定校验

## 一、核心原理

### 1.1 整体架构

```mermaid
flowchart LR
    Req[HTTP Request] --> Engine[gin.Engine]
    Engine --> Tree[Radix 路由树]
    Tree --> MW[中间件链]
    MW --> H[Handler]
    H --> Resp[Response]

    style Tree fill:#9f9
    style MW fill:#ff9
```

`gin.Engine` 实现了 `http.Handler` 接口，可挂在 `http.Server` 上。

### 1.2 Radix Tree 路由

Gin 用 [httprouter](https://github.com/julienschmidt/httprouter) 思想，每个 HTTP 方法（GET/POST/...）一棵 radix tree：

```
GET 树:
    /
    ├── api/
    │   ├── users → handler1
    │   └── users/:id → handler2
    └── static/*filepath → handler3
```

特点：
- **O(log n)** 查找，不回溯
- 支持 `:param`（命名参数）和 `*catchAll`（通配）
- 不支持正则，不支持任意位置通配（业务路由够用）

### 1.3 Context

每个请求一个 `gin.Context`：

```go
type Context struct {
    Request *http.Request
    Writer  ResponseWriter
    Params  Params  // URL 参数 :id
    Keys    map[string]any  // 中间件传值
    Errors  errorMsgs
    // ...
}
```

**关键**：`Context` 通过 `sync.Pool` **复用**！每个请求 Get 一个，handler 完成后 Put 回去。

```go
// gin 内部
c := engine.pool.Get().(*Context)
c.reset()
engine.handleHTTPRequest(c)
engine.pool.Put(c)
```

**坑点**：handler 内启动 goroutine 时不能直接用 `c`，要 `c.Copy()`：

```go
go func() {
    log.Println(c.Request.URL.Path)  // ❌ c 可能已 reset
}()

go func() {
    cc := c.Copy()
    log.Println(cc.Request.URL.Path)  // ✓
}()
```

### 1.4 中间件

中间件是 `func(*Context)`，通�� `c.Next()` 显式控制顺序：

```go
func Logger() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()  // 调用后续中间件 + handler
        log.Printf("%s %s %v", c.Request.Method, c.Request.URL.Path, time.Since(start))
    }
}

r := gin.New()
r.Use(Logger(), Recovery())
```

`c.Abort()` 终止链：

```go
func Auth() gin.HandlerFunc {
    return func(c *gin.Context) {
        if !valid(c.GetHeader("Authorization")) {
            c.AbortWithStatusJSON(401, gin.H{"error": "unauthorized"})
            return
        }
        c.Next()
    }
}
```

### 1.5 路由分组

```go
r := gin.New()
api := r.Group("/api/v1", AuthMiddleware())
{
    api.GET("/users/:id", getUser)
    api.POST("/users", createUser)

    admin := api.Group("/admin", AdminCheck())
    admin.DELETE("/users/:id", deleteUser)
}
```

分组共享 prefix + 中间件，结构清晰。

### 1.6 参数绑定

```go
type CreateUserReq struct {
    Name  string `json:"name" binding:"required,min=2,max=50"`
    Email string `json:"email" binding:"required,email"`
    Age   int    `json:"age" binding:"gte=0,lte=200"`
}

func createUser(c *gin.Context) {
    var req CreateUserReq
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    // ...
}
```

绑定方法：
- `ShouldBindJSON` / `ShouldBindXML` / `ShouldBindYAML`
- `ShouldBindQuery`（query string）
- `ShouldBindUri`（URL 参数）
- `ShouldBind`（自动按 Content-Type）

`Bind*` 系列在错误时自动 abort + 400，不推荐（耦合错误响应格式）。**用 `ShouldBind*` 自己处理错误**。

### 1.7 性能特点

vs 其他框架（粗略基准）：
- gin / fiber / echo / chi：微秒级，差异小
- httprouter（gin 内核）原生：最快
- 标准库 ServeMux：约慢 30%

实际业务中**差异可忽略**：业务逻辑/DB/RPC 占主导。选 gin 不为性能，为生态和易用。

## 二、八股速记

- **基于 Radix tree** 路由（httprouter 内核）
- **Context 池化复用**，开 g 必 `c.Copy()`
- **中间件链**：`c.Next()` / `c.Abort()`
- **路由分组**：共享 prefix + 中间件
- **参数绑定**：用 `ShouldBindXxx` 自己处理错误
- **校验靠 validator** tag
- **gin.Engine** 实现 `http.Handler`，可包到 `http.Server` 用 Shutdown
- 性能微秒级，业务里和其他框架差异不显著

## 三、面试真题

**Q1：Gin 为什么快？**
1. **Radix tree 路由**：O(log n) 查找，不像正则那样回溯
2. **Context 池化**：减少 GC 压力
3. **零分配关键路径**：JSON 编解码、参数解析尽量复用 buffer
4. **基于 net/http**：享受标准库连接池和 netpoll

但实际业务差异不大，gin 的核心价值是**生态**（中间件、绑定、校验）。

**Q2：handler 里 `go func()` 用 `c` 为什么会出错？**
`*gin.Context` 来自 sync.Pool，handler 返回后被 reset 并 Put 回 pool，下个请求拿到时数据被覆盖。

**修复**：

```go
go func(c *gin.Context) {
    // ❌ 用了原始 c
}(c)

go func(c *gin.Context) {
    // ✓ 用 Copy
}(c.Copy())
```

`Copy()` 复制必要字段（Request、Keys 等），但**不能 Write Response**（因为响应已经返回）。

**Q3：中间件顺序怎么设计？**
典型链：

```go
r.Use(
    Recovery(),       // 最外层, 兜底 panic
    RequestID(),      // 注入 trace_id
    Logging(),        // 记录 request/response
    Metrics(),        // Prometheus
    RateLimit(),      // 限流
    Auth(),           // 认证
    BodySizeLimit(),  // 限制 body 大小
)
```

**原则**：
- 最外层 Recovery + Logging（必须看到所有请求）
- 认证/授权早做（拒了不浪费后续）
- 限流在 auth 前（防匿名打爆）

**Q4：怎么自定义错误响应？**

```go
type Resp struct {
    Code int    `json:"code"`
    Msg  string `json:"msg"`
    Data any    `json:"data,omitempty"`
}

func writeOK(c *gin.Context, data any) {
    c.JSON(200, Resp{Code: 0, Data: data})
}

func writeErr(c *gin.Context, code int, msg string) {
    c.JSON(httpStatus(code), Resp{Code: code, Msg: msg})
}
```

绑定校验失败统一处理：

```go
func bindOrAbort(c *gin.Context, v any) bool {
    if err := c.ShouldBindJSON(v); err != nil {
        writeErr(c, 40001, err.Error())
        c.Abort()
        return false
    }
    return true
}
```

**Q5：Gin 怎么做优雅停机？**

```go
srv := &http.Server{Addr: ":8080", Handler: r}
go srv.ListenAndServe()

ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
defer stop()
<-ctx.Done()

shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
srv.Shutdown(shutdownCtx)
```

`gin.Engine` 实现 `http.Handler`，所以用 `http.Server.Shutdown` 就行。

**Q6：怎么传 ctx 给业务层？**

```go
func handler(c *gin.Context) {
    // 用 c.Request.Context() 获取请求 ctx
    user, err := svc.Get(c.Request.Context(), id)
    // ...
}
```

或用 `c` 当 ctx（Gin 1.7+ Context 实现 context.Context 接口），但**不推荐**——业务层不应感知 web 框架。

**Q7：Gin 和 chi/echo 对比？**

| | Gin | Echo | chi |
| --- | --- | --- | --- |
| 路由 | radix tree | radix tree | radix tree |
| 中间件 | 自定义 | 自定义 | 兼容 net/http |
| 性能 | 高 | 高 | 中高 |
| 生态 | 最强 | 强 | 标准库系 |
| 易用 | 高 | 高 | 中 |

**业务首选 gin**：生态最好。
**追求标准库纯净**用 chi（中间件兼容标准库）。
echo 介于两者之间。

**Q8：怎么写单元测试？**

```go
func TestHandler(t *testing.T) {
    gin.SetMode(gin.TestMode)
    r := gin.New()
    r.GET("/users/:id", getUser)

    req := httptest.NewRequest("GET", "/users/1", nil)
    rec := httptest.NewRecorder()
    r.ServeHTTP(rec, req)

    require.Equal(t, 200, rec.Code)
    require.Contains(t, rec.Body.String(), `"id":1`)
}
```

**Q9：怎么处理大文件上传？**

```go
r.POST("/upload", func(c *gin.Context) {
    file, err := c.FormFile("file")
    if err != nil { c.JSON(400, err); return }
    if file.Size > 100<<20 { c.JSON(413, "too large"); return }

    dst := "/uploads/" + file.Filename
    if err := c.SaveUploadedFile(file, dst); err != nil {
        c.JSON(500, err); return
    }
    c.JSON(200, gin.H{"file": dst})
})
```

设 body 上限：`r.MaxMultipartMemory = 32 << 20  // 32MB`。或用 `http.MaxBytesReader`。

**Q10：Gin 用 panic recover 时丢请求信息怎么办？**

```go
func Recovery() gin.HandlerFunc {
    return func(c *gin.Context) {
        defer func() {
            if rec := recover(); rec != nil {
                log.Printf("panic %s %s [%s]: %v\n%s",
                    c.Request.Method, c.Request.URL.Path, c.GetString("trace_id"),
                    rec, debug.Stack())
                c.AbortWithStatusJSON(500, gin.H{"error": "internal error"})
            }
        }()
        c.Next()
    }
}
```

带请求方法、路径、trace_id，便于定位。

## 四、手写实现

**1. 完整 server 模板：**

```go
func main() {
    r := gin.New()
    r.Use(gin.Recovery(), Logging(), RequestID(), Metrics())

    api := r.Group("/api/v1")
    {
        api.GET("/health", health)
        api.POST("/users", AuthRequired(), createUser)
    }

    srv := &http.Server{
        Addr:              ":8080",
        Handler:           r,
        ReadHeaderTimeout: 5 * time.Second,
        WriteTimeout:      10 * time.Second,
    }

    go srv.ListenAndServe()

    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
    defer stop()
    <-ctx.Done()

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    srv.Shutdown(shutdownCtx)
}
```

**2. 中间件示例：**

```go
func RequestID() gin.HandlerFunc {
    return func(c *gin.Context) {
        tid := c.GetHeader("X-Trace-Id")
        if tid == "" { tid = uuid.NewString() }
        c.Set("trace_id", tid)
        c.Header("X-Trace-Id", tid)
        c.Next()
    }
}

func Logging() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()
        log.Printf("[%s] %s %s %d %v",
            c.GetString("trace_id"),
            c.Request.Method, c.Request.URL.Path,
            c.Writer.Status(), time.Since(start),
        )
    }
}

func RateLimit(rps int) gin.HandlerFunc {
    limiter := rate.NewLimiter(rate.Limit(rps), rps)
    return func(c *gin.Context) {
        if !limiter.Allow() {
            c.AbortWithStatusJSON(429, gin.H{"error": "too many requests"})
            return
        }
        c.Next()
    }
}
```

**3. 异步任务正确写法：**

```go
func handler(c *gin.Context) {
    cc := c.Copy()  // 必须

    go func() {
        time.Sleep(5 * time.Second)
        log.Printf("delayed: %s", cc.Request.URL.Path)
    }()

    c.JSON(200, "started")
}
```

## 五、踩坑与最佳实践

### 坑 1：goroutine 用原 c

```go
go func() { use(c) }()  // c 可能被复用
```

**修复**：`c.Copy()`。

### 坑 2：用 Bind 系列

```go
c.BindJSON(&req)  // 错误时自动 400 + 默认错误格式
```

**修复**：用 `ShouldBindJSON` 自己处理错误响应格式。

### 坑 3：handler 阻塞太久

```go
func handler(c *gin.Context) {
    time.Sleep(60 * time.Second)  // 占着 g + 连接
}
```

每请求一 g，长阻塞会让连接积压。**修复**：异步化（任务队列）+ 立即返回 202。

### 坑 4：日志没带 trace_id

发生错误后 log 找不到关联请求。**修复**：RequestID 中间件 + log 中间件统一带 trace_id。

### 坑 5：Recovery 不打堆栈

```go
gin.Recovery()  // 默认 recovery 不打 stack
```

自定义 recovery 加 `debug.Stack()`。

### 坑 6：路由冲突

```go
r.GET("/users/:id", ...)
r.GET("/users/list", ...)  // 冲突! :id 和 list 不能共存
```

Gin 的 radix tree 不允许同位置的具名参数和静态路径并存。
**修复**：改 path 设计，如 `/users/list` → `/users?action=list` 或换层级。

### 坑 7：body 大小没限

```go
c.ShouldBindJSON(&req)  // 如果 body 是 1GB 也吞下
```

**修复**：设 `r.MaxMultipartMemory` 或 `http.MaxBytesReader`。

### 坑 8：cookie / session 绑定 c.Set

c.Set 仅当前请求有效（Context 复用 + reset）。**不能用作长期存储**。session 用 cookie 或服务端存储。

### 最佳实践

- **中间件顺序**：Recovery → RequestID → Logging → Metrics → 业务
- **Context.Copy** 在 goroutine
- **ShouldBind** + 自定义错误响应
- **body / header / file 大小限制**
- **优雅停机**：`http.Server.Shutdown`
- **trace_id 贯穿**：log 全带
- **超时**：handler 拿 `c.Request.Context()` 加超时传业务层
- **不依赖框架特定类型**：业务函数参数用 `context.Context`，不用 `*gin.Context`
- **路由分组 + 版本化** `/api/v1/`
