# context

> 跨 API 边界传递 deadline / cancel 信号 / 请求级值的标准方式；不可变树形结构，沿调用链向下传播

## 一、核心原理

### 1.1 接口定义

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

- **Done()**：返回一个 chan，**关闭时**表示该 ctx 被取消
- **Err()**：取消原因（`context.Canceled` / `context.DeadlineExceeded`）
- **Value()**：请求级 KV，沿父子链查找

### 1.2 四种基础 ctx

| 构造 | 用途 |
| --- | --- |
| `context.Background()` | 根 ctx，main、init、test 用 |
| `context.TODO()` | 不知道用啥时的占位（语义同 Background，但 lint 友好） |
| `WithCancel(parent)` | 派生可手动取消的 ctx |
| `WithDeadline/WithTimeout(parent)` | 派生有截止时间的 ctx |
| `WithValue(parent, k, v)` | 派生携带 KV 的 ctx |

### 1.3 树形传播

```
root (Background)
 ├─ A (WithCancel)
 │   ├─ A1 (WithTimeout 5s)
 │   └─ A2 (WithValue userID=1)
 └─ B (WithCancel)
     └─ B1 (WithCancel)
```

**取消传播规则**：
- 父被取消 → 所有子代立即被取消
- 子被取消 → **不影响父**
- 子的 deadline 不能晚于父（实际取较早的）

实现：每个 cancelCtx 维护 `children map[canceler]struct{}`，cancel 时遍历调用子代的 cancel。

### 1.4 取消的 chan 模型

```go
ctx, cancel := context.WithCancel(parent)
// 内部:
// cancelCtx.done = make(chan struct{})
// cancel() = close(cancelCtx.done)

go func() {
    select {
    case <-ctx.Done():  // 收到关闭信号
        return
    case <-time.After(time.Second):
        doWork()
    }
}()
```

`Done()` 返回的 chan 在 ctx 取消时被 close，所以所有监听者一起收到信号。

### 1.5 WithValue 的实现

链表查找：

```go
func (c *valueCtx) Value(key any) any {
    if c.key == key { return c.val }
    return c.Context.Value(key)  // 递归向上找
}
```

每次 Value 是 O(n) 链表遍历，但通常 n 很小（一两层）。**不要在 Value 里塞业务参数**。

## 二、八股速记

- **四方法接口**：Deadline / Done / Err / Value
- **不可变** + 派生：`With*` 返回新 ctx，不改父
- **取消是 chan close**，所有监听者同时唤醒
- **取消向下传播**，不向上
- WithCancel/Timeout/Deadline 必须 **defer cancel()**，否则 ctx 泄漏
- WithValue 用**自定义类型 key**（避免冲突），不要存大量数据
- **第一个参数**约定，不要塞 struct
- ctx 不要存到结构体字段，应作为函数参数显式传

## 三、面试真题

**Q1：context 设计的核心目的？**
解决两个问题：
1. **取消传播**：上游取消（用户断开连接、超时）能让所有下游 g 立即停下，避免做无用功 + 资源泄漏
2. **截止时间统一**：链路任意一环可设 deadline，下游各阶段自动尊重

附带：传请求级元数据（trace_id、auth）。

**Q2：为什么要 `defer cancel()`？**

```go
ctx, cancel := context.WithTimeout(parent, time.Second)
defer cancel()  // ← 必须
```

不调 cancel 也不致命（timer 到点会自己 close done chan），但：
- timer 资源在到期前一直占用
- 父 ctx 的 children map 里这个子 ctx 不会被移除 → 如果父长期存在，map 持续增长
- 静态分析工具会报警

`go vet` 直接检查未调 cancel。

**Q3：context.Value 应该存什么？**
**只存请求级元数据**：
- trace_id / span_id
- 认证用户信息
- request_id
- AB 实验分组

**不应存**：
- 业务参数（应作为函数参数）
- 大对象（每次查找遍历链表）
- 可变状态

key 必须是自定义类型避免冲突：

```go
type ctxKey struct{}
type userKey struct{}
ctx = context.WithValue(ctx, userKey{}, user)
```

**Q4：ctx 取消后还能继续用吗？**
不能。约定：ctx 一旦 Done()，所有依赖它的操作应**立即返回**，不再做业务。继续用属于 bug。

**Q5：WithTimeout 和 WithDeadline 区别？**
等价：`WithTimeout(p, d) ≡ WithDeadline(p, time.Now().Add(d))`。前者表示"再过 d 时间"，后者表示"到具体时刻"。重试场景常用 WithDeadline 保留剩余预算。

**Q6：父 ctx 取消，子 ctx 一定取消吗？反过来呢？**
- 父 → 子：是，立即传播
- 子 → 父：否
- 兄弟之间：不互相影响

**Q7：怎么在测试里 mock context？**
```go
ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
defer cancel()
err := myFunc(ctx)  // 验证超时行为
```
不需要 mock，标准库够用。

**Q8：ctx 应该在结构体字段里吗？**
**不应该**。Go 官方明确反对。原因：
- ctx 是请求级的，结构体生命周期通常更长
- 隐藏了取消行为，让人难以追踪
唯一例外：实现 `Server` 这类长期对象，可在内部用 cancelCtx 控制生命周期，但**不暴露**。

## 四、手写实现

**1. 实现一个简化的 cancelCtx（理解原理）：**

```go
type myCancelCtx struct {
    parent  context.Context
    done    chan struct{}
    err     error
    once    sync.Once
}

func WithCancel(parent context.Context) (context.Context, func()) {
    c := &myCancelCtx{parent: parent, done: make(chan struct{})}
    // 监听父取消
    go func() {
        select {
        case <-parent.Done():
            c.cancel(parent.Err())
        case <-c.done:
        }
    }()
    return c, func() { c.cancel(context.Canceled) }
}

func (c *myCancelCtx) cancel(err error) {
    c.once.Do(func() {
        c.err = err
        close(c.done)
    })
}

func (c *myCancelCtx) Done() <-chan struct{} { return c.done }
func (c *myCancelCtx) Err() error { return c.err }
func (c *myCancelCtx) Deadline() (time.Time, bool) { return c.parent.Deadline() }
func (c *myCancelCtx) Value(k any) any { return c.parent.Value(k) }
```

> 真实实现用 children map 替代 g 监听父，避免每个 ctx 起一个 g。

**2. 业务里的标准用法：**

```go
func handle(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
    defer cancel()

    user, err := userSvc.Get(ctx, userID)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    // ...
}

func (s *UserSvc) Get(ctx context.Context, id int64) (*User, error) {
    return s.repo.QueryRow(ctx, id)  // ctx 沿调用链传下去
}
```

**3. fan-out 后任意失败即取消：**

```go
func parallel(parent context.Context, urls []string) error {
    ctx, cancel := context.WithCancel(parent)
    defer cancel()

    errCh := make(chan error, len(urls))
    for _, u := range urls {
        go func(u string) {
            errCh <- fetch(ctx, u)  // 任一失败 cancel,其他立刻退出
        }(u)
    }

    for range urls {
        if err := <-errCh; err != nil {
            cancel()
            return err
        }
    }
    return nil
}
```

更好的写法：`golang.org/x/sync/errgroup`。

**4. 类型安全的 Value 包装：**

```go
type ctxKey int
const userIDKey ctxKey = 1

func WithUserID(ctx context.Context, id int64) context.Context {
    return context.WithValue(ctx, userIDKey, id)
}

func UserIDFrom(ctx context.Context) (int64, bool) {
    id, ok := ctx.Value(userIDKey).(int64)
    return id, ok
}
```

## 五、踩坑与最佳实践

### 坑 1：忘记 defer cancel

```go
func bad() {
    ctx, _ := context.WithTimeout(context.Background(), time.Second)  // _ 丢弃 cancel
    doWork(ctx)
}
```

`go vet` 报警。timer 资源泄漏（虽然到期会清，但严重场景下父 ctx 的 children map 一直增长）。

### 坑 2：把 ctx 存到 struct

```go
type Service struct {
    ctx context.Context  // 反模式
}
```

让调用方搞不清 ctx 何时变化。每次方法调用应**显式传入**。

### 坑 3：用 ctx.Value 传业务参数

```go
ctx = context.WithValue(ctx, "userID", 123)  // string 作 key 也是反模式
ctx = context.WithValue(ctx, "amount", 100)
ctx = context.WithValue(ctx, "currency", "USD")
```

参数应作为函数参数。Value 仅传请求级元数据。

### 坑 4：ctx 链断裂

```go
func handle(parentCtx context.Context) {
    go func() {
        ctx := context.Background()  // 错: 创建新根, 与 parentCtx 失联
        doWork(ctx)
    }()
}
```

后台 g 收不到父取消信号 → 业务超时但 g 还在跑。除非业务逻辑确实独立于请求生命周期，否则应继承父 ctx。

### 坑 5：`select { case <-ctx.Done(): }` 之后没立即 return

```go
for {
    select {
    case <-ctx.Done():
        log.Println("done")  // 只 log 不 return → 死循环
    case msg := <-ch:
        handle(msg)
    }
}
```

ctx.Done() 后必须 break/return 退出循环。

### 坑 6：在 ctx.Done() 触发后还做长操作

```go
case <-ctx.Done():
    cleanupTakingMinutes()  // ctx 已经超时了, 还做这么久?
    return
```

清理也要受控（用 `context.WithTimeout(context.Background(), cleanupTime)` 派生新 ctx）。

### 最佳实践

- **第一个参数总是 ctx**：`func Foo(ctx context.Context, ...) error`
- **每层自己 defer cancel**：派生即 defer
- **不存到 struct**，作为参数显式传递
- **Value key 用自定义类型**，提供 `WithFoo` / `FooFrom` 配套函数
- HTTP server 用 `r.Context()`，gRPC 用 `stream.Context()`，作为最上游
- 数据库/HTTP client 调用必传 ctx：`db.QueryContext`, `http.NewRequestWithContext`
- 后台 g 想超过请求生命周期：用 `context.WithoutCancel(ctx)`（Go 1.21+）保留 Value 但去掉 cancel
