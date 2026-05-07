# Go 接口设计与工程实践

> `interface.md` 偏底层结构，这篇偏工程设计。面试里真正加分的是：知道接口怎么实现，也知道什么时候不该抽接口。

## 一、核心原则

```text
小接口。
使用方定义接口。
接收接口，返回结构体。
不要为了“以后可能扩展”提前抽象。
```

Go 的接口不是 Java 那种“先设计一套顶层类型体系”，更像是调用方对依赖能力的最小声明。

```go
type UserStore interface {
    GetByID(ctx context.Context, id int64) (*User, error)
}

type UserService struct {
    store UserStore
}
```

`UserService` 不关心底层是 MySQL、Redis、HTTP RPC 还是 mock，只关心“能按 ID 查用户”。

## 二、为什么接口建议由使用方定义

错误做法：

```go
// infra/mysql/user_repo.go
type UserRepository interface {
    GetByID(ctx context.Context, id int64) (*User, error)
    Create(ctx context.Context, u *User) error
    Update(ctx context.Context, u *User) error
    Delete(ctx context.Context, id int64) error
    List(ctx context.Context, offset, limit int) ([]*User, error)
}
```

这个接口由实现方定义，容易变成“大而全契约”。上层只用 `GetByID`，却被迫依赖全部方法。

推荐：

```go
// app/order/service.go
type userGetter interface {
    GetByID(ctx context.Context, id int64) (*User, error)
}

type OrderService struct {
    users userGetter
}
```

依赖方向：

```text
业务服务 ---> 小接口 <--- MySQL 实现
              ^
              |
            Mock 实现
```

好处：

- 测试更容易 mock。
- 依赖更小，改动影响面更小。
- 不会把 infra 层的大接口泄漏到业务层。
- 更符合依赖倒置。

## 三、接收接口，返回结构体

### 1. 接收接口

函数参数可以接收接口，表达“我只需要你具备这个能力”。

```go
func Decode(r io.Reader) (*Config, error) {
    b, err := io.ReadAll(r)
    if err != nil {
        return nil, err
    }
    // ...
    return cfg, nil
}
```

调用方可以传文件、网络连接、字符串 reader。

### 2. 返回结构体

构造函数通常返回具体类型。

```go
func NewUserService(repo userGetter) *UserService {
    return &UserService{users: repo}
}
```

不要轻易这样写：

```go
func NewUserService(...) UserServiceInterface
```

原因：

- 返回接口会隐藏具体能力，不方便调用扩展方法。
- 可能制造 nil interface 坑。
- 实现方不应该替调用方决定抽象边界。

例外：框架插件、明确需要隐藏实现细节的 SDK、跨包稳定 ABI 风格接口。

## 四、小接口为什么重要

Go 标准库最经典的小接口：

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

小接口的价值：

- 实现成本低。
- 组合能力强。
- 测试替身简单。
- 调用方依赖最小能力。

组合接口：

```go
type ReadWriter interface {
    Reader
    Writer
}
```

## 五、接口与 nil 坑

最经典：

```go
type MyError struct{}

func (*MyError) Error() string { return "x" }

func do() error {
    var e *MyError = nil
    return e
}

func main() {
    err := do()
    fmt.Println(err == nil) // false
}
```

接口值由两部分组成：

```text
(动态类型, 动态值)
```

只有两个都为 nil，接口才等于 nil。

```text
nil interface:
type = nil
data = nil

装了 nil 指针的 interface:
type = *MyError
data = nil
```

正确写法：

```go
func do() error {
    var e *MyError = nil
    if e != nil {
        return e
    }
    return nil
}
```

## 六、接口性能与取舍

接口调用成本主要来自：

- 动态分派，可能无法内联。
- 值装入 interface 可能发生装箱。
- 逃逸到堆会增加 GC 压力。

但普通业务代码不要过早优化。更合理的判断：

| 场景 | 建议 |
| --- | --- |
| 业务服务依赖外部组件 | 用小接口 |
| 测试需要 mock | 用接口 |
| 热路径循环内每次调用 | 优先具体类型，必要时 benchmark |
| 数据结构内部泛型算法 | 优先泛型或具体类型 |
| SDK 对外暴露稳定能力 | 可以暴露接口，但保持小 |

## 七、线上设计坑

### 坑 1：大接口导致 mock 很痛苦

```go
type Repo interface {
    Get(...)
    Save(...)
    Delete(...)
    Count(...)
    List(...)
    Tx(...)
}
```

测试一个 `Get`，却要实现全部方法。解决：按使用方拆小接口。

### 坑 2：接口定义在实现层

业务层导入 infra 包里的接口，结果 infra 包一改接口，业务层全跟着动。解决：业务层定义自己需要的最小接口，infra 层只提供具体实现。

### 坑 3：返回接口隐藏错误能力

```go
func NewClient() Client {
    return &clientImpl{}
}
```

后续想暴露 `Close()`、`Stats()`，接口不包含就没法用。除非这是对外稳定 SDK，否则优先返回 `*Client` 具体类型。

### 坑 4：nil interface 判断错误

常发生在 `error`、`io.Reader`、自定义接口返回值里。要么返回真正的 `nil`，要么避免返回接口类型。

## 八、面试真题

**Q1：Go 接口和 Java 接口有什么区别？**

Java 接口需要显式 `implements`，通常作为类型体系的一部分先设计。Go 接口是隐式实现，只要方法集匹配就满足，更适合由使用方定义小接口。

**Q2：为什么说接口要小？**

接口越大，调用方依赖越重，实现和 mock 成本越高。Go 更推崇只描述当前调用方需要的最小能力，比如 `io.Reader` 只有一个方法。

**Q3：为什么“接收接口，返回结构体”？**

接收接口可以降低调用方和实现方耦合；返回结构体可以保留具体能力，避免过早隐藏实现，也减少 nil interface 坑。

**Q4：接口会不会影响性能？**

会有动态分派和装箱成本，可能阻碍内联。但绝大多数业务场景不是瓶颈。热路径需要 benchmark 和 pprof 判断，不要凭感觉优化。

## 九、面试表达

```text
Go 的接口设计重点不是“抽象越多越好”，而是使用方定义最小能力。
我一般会让业务层定义小接口，基础设施层提供具体实现；函数参数接收接口，构造函数返回具体结构体。
这样既方便测试 mock，也能控制依赖方向。只有在 SDK、插件、框架扩展点这类稳定边界上，我才会主动对外暴露接口。
```
