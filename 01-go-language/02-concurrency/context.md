# context

> 跨 API 边界传递 deadline / cancel 信号 / 请求级值的标准方式；不可变树形结构，沿调用链向下传播

## 一、一句话总结(背诵版)

> **context 用来跨 API 边界传递取消信号、超时和请求级元数据**;形成**父子树**,父取消则所有子立即取消,子取消不影响父;约定**第一个参数**,沿调用链显式传递。

延伸(被追问时再展开):

> 本质是 **"信号广播 + 截止时间统一"**——`Done()` 返回的 chan 在取消时 close,所有监听者一起收到通知;实现是 **不可变树形结构 + cancelCtx.children map**,父 cancel 时遍历调用子代 cancel,完成 O(子代数) 的级联传播。

---

## 二、使用场景(场景化记忆)

| 场景 | 一句话 | 最小代码 |
| --- | --- | --- |
| **HTTP 请求超时** | server handler 派生 timeout,超时统一中断 | `ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)` |
| **数据库调用 ctx 中断** | db driver 监听 Done,超时取消正在跑的 SQL | `db.QueryContext(ctx, sql, args...)` |
| **链路追踪 trace_id 传递** | WithValue 携带 trace_id / span_id 向下游透传 | `ctx = context.WithValue(ctx, traceKey{}, id)` |
| **取消批处理 / fan-out** | 任一子任务失败,cancel 通知兄弟立即退出 | `errgroup.WithContext(parent)` |
| **goroutine 生命周期绑定** | 后台 g select `<-ctx.Done()` 退出,避免泄漏 | `select { case <-ctx.Done(): return }` |
| **鉴权信息透传(慎用)** | 中间件解析 token 后塞入 ctx,handler 取出 | `ctx = WithUser(ctx, user)` — 仅元数据,不要塞业务参数 |

**判断要不要传 ctx**:**只要可能阻塞(I/O、锁、channel)或可能长跑就必须传 ctx**。

---

## 三、常见错误(高频踩坑)

| 错误 | 现象 | 根因 | 修复 |
| --- | --- | --- | --- |
| **WithCancel/Timeout 漏 defer cancel()** | timer 资源泄漏 + 父 children map 持续增长 | cancel 不仅停 timer,还要从父 map 解绑;`go vet` 报警 | 派生即 `defer cancel()`,无脑跟一行 |
| **ctx.Value 当容器塞业务参数** | 业务参数靠 key 字符串拿,O(n) 链表遍历 + 类型不安全 | WithValue 设计目标只为请求级元数据(trace/auth) | 业务参数走函数签名;Value key 用**自定义类型** + 配套 `WithFoo/FooFrom` |
| **ctx 不是函数第一参数 / 塞 struct 字段** | 调用方看不到 ctx 何时切换;隐藏取消语义 | Go 官方约定:ctx 永远第一参数,显式传递 | `func Foo(ctx context.Context, ...) error`;struct 内不存 ctx |
| **TODO/Background 混用没语义** | 代码里到处 `context.Background()` 起新根 → 与父链断裂 | Background = 真正的根(main/init/test);TODO = "暂不知道用啥"占位 | 后台 g 想继承元数据但去掉 cancel,用 `context.WithoutCancel(ctx)`(Go 1.21+) |
| **ctx 不传给下游(空 G 漏掉)** | `go func() { doWork(context.Background()) }()` 收不到父取消 | goroutine 默认不继承父 ctx,需显式传 | 闭包捕获父 ctx 或参数传入;errgroup 自动处理 |
| **超时设置过小导致级联失败** | 上游 100ms 超时,下游 5 次重试每次 500ms → 全部超时 | 没做超时预算下传(deadline budget) | 用 `WithDeadline` 而非 `WithTimeout`,保留剩余预算;重试前判 `ctx.Err()` |
| **ctx.Done() 触发后没 return** | `select { case <-ctx.Done(): log.Println("done") }` 死循环 | 只打日志没退出 case 后继续循环 | `case <-ctx.Done(): return ctx.Err()` |

---

## 四、面试常问(简答模板)

**Q1:为什么 ctx 必须是第一个参数?**
Go 官方约定 + 工具链强校验(`go vet`、linter)。**位置固定 → 一眼能看出函数是否可取消**;塞中间或塞 struct 会隐藏取消语义,让调用方不知道什么时候 ctx 会切换。所有标准库(`db.QueryContext`、`http.NewRequestWithContext`)都遵守这个约定。

**Q2:WithCancel / WithTimeout / WithDeadline 的区别?**
都派生可取消的子 ctx,区别在**触发条件**:
- `WithCancel`:**手动**调 cancel() 才取消
- `WithTimeout(p, d)`:**d 时长后**自动取消,等价于 `WithDeadline(p, time.Now().Add(d))`
- `WithDeadline(p, t)`:**到达时刻 t** 自动取消;**重试场景优先用 Deadline 保留剩余预算**(不会因每次重试重新计时)

三者都必须 `defer cancel()`。

**Q3:context.Value 该用吗?何时该用?**
**可以用,但只用于请求级元数据**:trace_id / span_id / 认证用户 / request_id / AB 实验分组。
**不要用**于业务参数(应作为函数参数)、大对象(链表 O(n) 遍历)、可变状态。
key 必须是**自定义类型**(`type ctxKey struct{}`)避免不同包冲突,并提供配套 `WithFoo / FooFrom` 类型安全封装。

**Q4:ctx 是值传递还是引用传递?**
**Go 只有值传递**。但 `context.Context` 是**接口**,接口内部含**类型指针 + 数据指针**(对于结构体指针实现是指针 + 指针),所以**传递接口值的开销很小**(2 个 word),且对底层 cancelCtx 的修改对所有持有者可见——**行为上接近"引用语义"**。

**Q5:取消信号怎么传播?**
**链式 done channel + children map**:
1. 每个 cancelCtx 内部 `done = make(chan struct{})` + `children map[canceler]struct{}`
2. 派生子 ctx 时,把子注册到父的 children
3. 调用 cancel() → `close(c.done)`(广播给所有 `<-ctx.Done()` 监听者)→ 遍历 children 递归 cancel
4. 子 ctx 也会监听父的 Done,父先取消时同步关闭自己

**Q6:Background 和 TODO 有啥区别?**
**实现完全一样**(都是 `emptyCtx`),只是**语义不同**:
- `Background()`:**根 ctx**——main、init、test、最上游入口处用
- `TODO()`:**占位**——"我知道这里需要 ctx,但暂时不知道用哪个父"

区别价值:**静态分析工具**可以识别 `TODO()` 标记未完成的 ctx 链,提醒补全。

---

## 五、深水区:原理与源码(被追问时看)

> 下面是 cancelCtx / timerCtx / valueCtx 的源码结构、children map 设计、Done chan 的 lazy 创建、`WithoutCancel` 等深度内容。**正常面试用不到**,只在被深追"context 怎么实现的 / 为什么这样设计"时才会用到。

---

## 六、核心原理

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

## 七、八股速记

- **四方法接口**：Deadline / Done / Err / Value
- **不可变** + 派生：`With*` 返回新 ctx，不改父
- **取消是 chan close**，所有监听者同时唤醒
- **取消向下传播**，不向上
- WithCancel/Timeout/Deadline 必须 **defer cancel()**，否则 ctx 泄漏
- WithValue 用**自定义类型 key**（避免冲突），不要存大量数据
- **第一个参数**约定，不要塞 struct
- ctx 不要存到结构体字段，应作为函数参数显式传

## 八、面试真题

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

## 九、手写实现

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

## 十、踩坑与最佳实践

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
