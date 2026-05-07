# context 超时、取消与链路控制

> `context` 不是万能参数袋。它的核心用途是取消、超时、截止时间和请求级值传递。线上很多泄漏和雪崩都和 context 使用不当有关。

## 一、context 解决什么问题

```text
一个请求进来后，可能会调用 DB、Redis、HTTP、MQ、goroutine。
当请求超时、客户端断开、服务关闭时，下游操作应该一起停止。
```

传播模型：

```text
入口请求 ctx
   |
   +--> DB 查询
   |
   +--> Redis 查询
   |
   +--> HTTP 下游调用
   |
   +--> 后台 goroutine
```

只要父 context 取消，子 context 都会收到取消信号。

## 二、基本用法

```go
func Handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    if err := svc.CreateOrder(ctx, req); err != nil {
        // ...
    }
}
```

超时：

```go
ctx, cancel := context.WithTimeout(parent, 2*time.Second)
defer cancel()
```

取消：

```go
ctx, cancel := context.WithCancel(parent)
defer cancel()
```

等待：

```go
select {
case <-done:
    return nil
case <-ctx.Done():
    return ctx.Err()
}
```

## 三、必须 defer cancel

错误：

```go
ctx, cancel := context.WithTimeout(parent, time.Second)
do(ctx)
```

正确：

```go
ctx, cancel := context.WithTimeout(parent, time.Second)
defer cancel()
do(ctx)
```

即使自然超时，也应该调用 `cancel`，它会释放定时器和父子 context 引用。

## 四、不要把 context 存到 struct

错误：

```go
type Service struct {
    ctx context.Context
}
```

推荐：

```go
type Service struct {
    repo Repo
}

func (s *Service) Create(ctx context.Context, req Request) error {
    return s.repo.Save(ctx, req)
}
```

原因：

- context 是请求级生命周期，不是对象生命周期。
- 存到 struct 容易复用过期 context。
- 取消边界不清晰。

## 五、context.Value 使用边界

适合放：

- trace_id。
- request_id。
- auth claims。
- 灰度标记。
- 租户 ID。

不适合放：

- DB 连接。
- logger 大对象。
- 业务可选参数。
- 配置对象。
- 函数依赖。

key 不要用普通 string，避免包间冲突：

```go
type traceIDKey struct{}

func WithTraceID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, traceIDKey{}, id)
}
```

## 六、超时设计不是越短越好

一个订单接口超时 1s，内部调用：

```text
订单校验 100ms
库存服务 300ms
优惠服务 300ms
支付预创建 400ms
DB 写入 100ms
```

不能每层都随便 `WithTimeout(1s)`，否则总耗时可能失控，也可能下游拿到的 deadline 已经没意义。

推荐：

```text
入口设置总 deadline。
下游基于剩余时间设置更小预算。
非核心依赖可以更短并允许降级。
```

## 七、线上案例

### 案例：请求超时后 DB 查询还在跑

背景：用户查询订单列表，接口超时 2s。

问题：

```go
rows, err := db.Query(sqlText)
```

入口请求已经超时返回，但 DB 查询没有绑定 context，MySQL 慢 SQL 仍然继续执行，占用连接池。高峰期连接池被慢 SQL 打满，正常请求也拿不到连接。

处理：

```go
rows, err := db.QueryContext(ctx, sqlText)
```

同时：

- SQL 优化和加索引。
- 设置 DB 连接池参数。
- 设置接口超时和 DB 查询超时。
- 对超时错误做可观测记录。

复盘：

```text
context 要从入口一路传到 DB、Redis、HTTP 和 goroutine。
只在 handler 里设置超时，但下游不用 ctx，等于没有真正取消。
```

## 八、常见坑

### 坑 1：用 Background 断开链路

```go
func (s *Service) Call(ctx context.Context) error {
    return s.client.Do(context.Background())
}
```

这样会丢失 trace、deadline 和取消信号。

### 坑 2：异步任务误用请求 context

```go
go sendEmail(r.Context(), userID)
```

请求结束后 context 会取消，邮件任务可能立刻失败。异步任务应该有自己的任务 context，并把必要字段显式传入。

### 坑 3：吞掉 ctx.Err

```go
case <-ctx.Done():
    return nil
```

应该返回 `ctx.Err()`，让上层知道是超时还是取消。

## 九、面试真题

**Q1：context 的作用是什么？**

主要是跨 API 边界传递取消信号、超时、deadline 和少量请求级元信息。它不是依赖注入容器，也不是业务参数 map。

**Q2：为什么 context 通常作为第一个参数？**

这是 Go 约定，说明该函数受请求生命周期控制，调用方可以一眼看到取消和超时会继续向下传递。

**Q3：context 超时后 goroutine 会自动杀掉吗？**

不会。context 只是关闭 `Done` channel，goroutine 必须主动监听 `ctx.Done()` 并返回。

**Q4：什么时候不能用请求 context？**

真正需要请求结束后继续执行的后台任务不能直接用请求 context。应该创建新的任务上下文，同时显式传递 trace_id、user_id 等必要字段。

## 十、面试表达

```text
我理解 context 是链路控制工具，不是参数袋。
在线上我会要求入口 ctx 一路传到 DB、Redis、HTTP 和 goroutine；所有可能阻塞的地方都要能响应 ctx.Done。
同时也要区分请求生命周期和后台任务生命周期，避免请求结束把异步任务一起取消。
```
