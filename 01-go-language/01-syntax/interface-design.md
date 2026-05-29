# Go 接口设计与工程实践

> `interface.md` 偏底层结构，这篇偏工程设计。面试里真正加分的是：知道接口怎么实现，也知道什么时候不该抽接口。

## 一、一句话总结(背诵版)

> **Go 接口设计四原则**——**小接口**(1-3 method)、**消费方定义**(由调用者声明所需行为)、**隐式实现**(无 implements)、**组合优于继承**(embedding 拼小接口),核心 idiom 是 **accept interfaces, return structs**。

延伸(被追问时再展开):

> Go 接口的哲学跟 Java 完全相反:Java 是"先设计一套类型体系,实现方 implements",Go 是"调用方按需声明能力,实现方自动满足"。这带来三个工程红利:**测试 mock 零成本**(每个 consumer 自己的小接口)、**依赖方向倒置**(business 不再依赖 infra 的大接口)、**避免预先抽象**(YAGNI——只在真正有多实现 / mock 需要时才抽接口)。落地标准就是 io.Reader/Writer——一个方法、组合无数次、所有 IO 类型都自动满足。

---

## 二、使用场景(场景化记忆)

| 场景 | 一句话 | 典型例子 |
| --- | --- | --- |
| **依赖反转(DIP)** | 高层定义接口,低层实现注入 | `UserService` 定义 `userGetter`,MySQL 实现注入 |
| **单元测试 mock** | 把外部依赖抽接口,测试换 mock 实现 | `HTTPClient` / `Cache` / `DB` 接口,测试用 fake |
| **插件化扩展** | 用接口定义扩展点,运行时注册不同实现 | `database/sql.Driver`、`encoding.Codec`、`hash.Hash` |
| **中间件链** | 同一接口串成处理链 | `http.Handler` 链、gRPC interceptor、Gin middleware |
| **Repository / Service 分层** | DDD 分层中用接口隔离领域和基础设施 | `UserRepo` 接口在 domain 层,实现在 infra 层 |
| **标准抽象(行为复用)** | 统一接口描述某类行为 | `io.Reader/Writer`、`sort.Interface`、`error` |

**判断要不要抽接口**:**有"换实现 / 测试 mock / 插件扩展"诉求才抽**;**单一实现别提前抽象**(YAGNI)。

---

## 三、常见错误(高频踩坑)

| 错误 | 现象 | 根因 | 修复 |
| --- | --- | --- | --- |
| **接口定义在实现方** | infra 包定义大接口,business 反向依赖 infra | 沿用 Java idiom"实现方声明 implements" | 接口跟 consumer 走,implementer 包不导出接口 |
| **定义过大接口违 ISP** | 一个 Repo 接口 10+ 方法,mock 写得痛苦 | 把"所有可能用到的方法"都堆进同一接口 | 按 consumer 拆小接口,embedding 组合 |
| **为了"扩展"提前抽象** | 单一实现也抽空接口,代码绕一圈才到实现 | 过度防御性设计,YAGNI 违反 | 单实现先用具体类型,真有多实现再抽 |
| **返回 interface 而非具体类型** | 调用方拿到接口后想用扩展方法,只能类型断言 | 违反 Go idiom "accept interfaces, return structs" | 构造函数返回 `*T` 具体类型,参数收接口 |
| **typed nil 装箱坑** | `var e *MyErr = nil; return e` 让 `err == nil` 为 false | iface = (type, data),type 非 nil 整个接口非 nil | 显式 `return nil`,不要返回 typed nil |
| **interface 嵌套过深** | 接口 A 嵌 B 嵌 C,改一个方法影响一片 | 滥用 embedding 当继承用 | 保持扁平,组合不超过 2 层 |
| **过度依赖运行时断言** | 接口里塞 `any`,调用方满地 `v.(T)` | 用接口反而退化成无类型 | 用泛型(Go 1.18+)或具体类型,断言只在边界用 |

---

## 四、面试常问(简答模板)

**Q1:为什么 Go 推崇小接口?**
**小接口实现成本低、组合能力强、mock 容易、依赖最小**。`io.Reader` 一个方法,所有"能读"的类型都自动满足——文件、网络、bytes.Buffer、gzip、加密流……拼成 `ReadWriter`、`ReadCloser` 也只是 embedding。大接口的代价是"调用方依赖一堆用不到的方法 + 实现方被迫提供全部 + mock 痛苦"。

**Q2:"accept interfaces, return structs" 什么意思?为什么?**
**参数收接口,返回值给结构体**。参数收接口是为了**解耦**——调用方传什么我都能用(file / net / mock);返回结构体是为了**保留信息**——调用方拿到具体类型可以调全部方法,也不会踩 typed nil 坑。反过来"接口参数"才是 idiomatic,"接口返回值"是反模式(除非 SDK 稳定边界)。

**Q3:接口应该由谁定义?消费方还是实现方?**
**消费方定义**(consumer-driven)。理由:**依赖方向**——business 不应依赖 infra,接口跟 consumer 走才能让 infra 反向依赖 business;**最小化**——每个 consumer 只声明自己需要的方法,而不是接受实现方"塞给我的大接口";**易测**——consumer 自己定义的小接口,mock 起来零成本。Java idiom 是实现方 implements 一个大接口,Go 反过来。

**Q4:Go 接口和 Java 接口的本质区别?**
**Java 是显式实现 + 名义类型**(implements 关键字 + 类型体系先行设计);**Go 是隐式实现 + 结构类型**(鸭子类型,方法集匹配就自动满足)。带来三个工程差异:Go 接口可以**事后抽取**(写完实现再定义接口照样工作);Go 接口由**消费方定义**(Java 通常实现方设计);Go 推崇**小接口组合**(Java 倾向大接口继承)。

**Q5:ISP(接口隔离原则)在 Go 怎么落地?**
**按 consumer 拆小接口 + embedding 组合**。具体落地:1)**每个 consumer 自己定义需要的接口**(不复用大接口);2)**单方法接口优先**(`io.Reader` 模式);3)**用 embedding 组合**而非定义大接口(`io.ReadWriter = Reader + Writer`);4)**编译期断言** `var _ Reader = (*MyType)(nil)` 防漏实现。反面:一个 `UserRepo` 接口 10 方法,所有用到 User 的服务都被迫依赖全部——这是把 ISP 当摆设。

**Q6:接口怎么驱动测试和设计?**
**先写 consumer 的接口和测试,再写实现**——这就是接口测试驱动。流程:1)在 service 里定义需要什么能力(`userGetter`);2)用 mock 写测试覆盖业务逻辑;3)最后在 infra 写 MySQL 实现,自动满足接口。好处:**业务逻辑测试不依赖真实 DB**(快 + 稳定);**接口边界由测试驱动**(只暴露 consumer 真正需要的方法);**实现可替换**(MySQL → Redis → mock 不动业务代码)。

---

## 五、深水区:原理与源码(被追问时看)

> 下面是接口设计的工程实战细节——为什么要这么做、踩过的坑、和 Java/Spring 的对比、SDK 边界设计等。**一般面试用不到**,只在被深追"你们项目接口怎么分层 / 大接口怎么重构 / 接口性能"时才会用到。

## 六、核心原则

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

## 七、为什么接口建议由使用方定义

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

## 八、接收接口，返回结构体

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

## 九、小接口为什么重要

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

## 十、接口与 nil 坑

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

## 十一、接口性能与取舍

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

## 十二、线上设计坑

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

## 十三、面试真题

**Q1：Go 接口和 Java 接口有什么区别？**

Java 接口需要显式 `implements`，通常作为类型体系的一部分先设计。Go 接口是隐式实现，只要方法集匹配就满足，更适合由使用方定义小接口。

**Q2：为什么说接口要小？**

接口越大，调用方依赖越重，实现和 mock 成本越高。Go 更推崇只描述当前调用方需要的最小能力，比如 `io.Reader` 只有一个方法。

**Q3：为什么“接收接口，返回结构体”？**

接收接口可以降低调用方和实现方耦合；返回结构体可以保留具体能力，避免过早隐藏实现，也减少 nil interface 坑。

**Q4：接口会不会影响性能？**

会有动态分派和装箱成本，可能阻碍内联。但绝大多数业务场景不是瓶颈。热路径需要 benchmark 和 pprof 判断，不要凭感觉优化。

## 十四、面试表达

```text
Go 的接口设计重点不是“抽象越多越好”，而是使用方定义最小能力。
我一般会让业务层定义小接口，基础设施层提供具体实现；函数参数接收接口，构造函数返回具体结构体。
这样既方便测试 mock，也能控制依赖方向。只有在 SDK、插件、框架扩展点这类稳定边界上，我才会主动对外暴露接口。
```
