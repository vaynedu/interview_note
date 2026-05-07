# error

> Go 错误处理：error 接口 + 值对比，显式返回而非异常；Go 1.13+ 引入 wrap / Is / As 标准化链路

## 一、核心原理

### 1.1 error 接口

```go
type error interface {
    Error() string
}
```

**就一个方法**。任何实现了 `Error() string` 的类型都是 error。设计哲学：**错误是值**，不是控制流。

### 1.2 三种创建方式

```go
// 1. errors.New
err := errors.New("not found")

// 2. fmt.Errorf (可格式化, 可 wrap)
err := fmt.Errorf("query failed: %w", originalErr)

// 3. 自定义类型(可携带上下文)
type NotFoundError struct {
    Resource string
    ID       int64
}
func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s %d not found", e.Resource, e.ID)
}
```

### 1.3 wrap / unwrap（Go 1.13+）

`fmt.Errorf` 的 `%w` 动词可包装错误，形成链：

```go
err1 := errors.New("db closed")
err2 := fmt.Errorf("load user: %w", err1)
err3 := fmt.Errorf("handle request: %w", err2)
// err3 的链: err3 → err2 → err1
```

`errors.Unwrap(err)` 取出被包装的 err（没有则返回 nil）。

Go 1.20+ 支持**多错误包装** `fmt.Errorf("%w %w", e1, e2)` 以及 `errors.Join(e1, e2)`。

### 1.4 errors.Is / errors.As

```go
var ErrNotFound = errors.New("not found")

func get() error {
    return fmt.Errorf("load: %w", ErrNotFound)
}

err := get()
errors.Is(err, ErrNotFound)  // true, 沿 wrap 链递归比
```

- `errors.Is(err, target)`：**值相等**比较，递归 unwrap
- `errors.As(err, &target)`：**类型断言**，递归 unwrap，命中则赋值

```go
var nfe *NotFoundError
if errors.As(err, &nfe) {
    fmt.Println(nfe.ID)  // 拿到具体字段
}
```

**原则**：
- **sentinel error**（`ErrXxx` 变量）用 `Is`
- **自定义 struct error**（携带字段）用 `As`
- 不要直接 `err == ErrXxx`，被 wrap 过就失效

### 1.5 panic / recover

panic 是**严重错误**的表达，会 unwind 栈、执行沿途 defer，直到：
- 有 defer 里调用 `recover()` → 拦下，函数从该 defer 所在层返回
- 没 recover → 进程崩溃 + 打印堆栈

```go
func safe() (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)
        }
    }()
    dangerous()
    return
}
```

**recover 只在 defer 函数体内直接调用才有效**。

### 1.6 errors.Join（Go 1.20+）

聚合多个错误：

```go
errs := []error{
    validateName(n),
    validateEmail(e),
    validateAge(a),
}
return errors.Join(errs...)  // nil 会被忽略
```

`errors.Is/As` 对 joined error 依然有效。

## 二、八股速记

- error 是**接口**，只要实现 `Error() string` 即可
- **errors.New / fmt.Errorf / 自定义类型**三种创建方式
- **`%w` wrap** 形成链；`errors.Unwrap` 拆链
- `errors.Is` 比值，`errors.As` 比类型，都能穿透 wrap
- **sentinel error 用 Is，自定义 error 用 As**
- panic/recover **只用于严重错误**，不是业务错误控制流
- recover 必须在 **defer 函数体内直接调用**
- `err != nil` 检查是 Go 的主流程，别怕啰嗦

## 三、面试真题

**Q1：为什么 Go 不用异常而用 error 返回值？**
设计哲学差异：
- 异常容易隐藏控制流（一个函数可能抛任意异常）
- error 返回值让错误处理**显式**，代码可读性更强
- 编译器可强制处理（通过 `err != nil` 判断）
- 错误是**值**，可以传递、比较、包装、携带上下文

代价是代码啰嗦，但可预测性远高于异常。

**Q2：`errors.New` vs `fmt.Errorf` vs 自定义 error？**
- **errors.New**：固定字符串，最简单，通常作 sentinel（`var ErrXxx = errors.New(...)`）
- **fmt.Errorf**：需要格式化参数，或要 wrap 上层错误
- **自定义类型**：需要携带字段（ID、状态码、堆栈）让调用方 `errors.As` 取出

**Q3：典型的 nil error 坑**

```go
type MyErr struct{}
func (*MyErr) Error() string { return "x" }

func do() error {
    var e *MyErr  // nil
    return e       // 被装进 error 接口, 接口非 nil
}

err := do()
fmt.Println(err == nil)  // false !
```

原因：接口 `= (type, data)`，type 非 nil 就不等于 nil（即使 data nil）。
**修复**：显式 `return nil`，或判断后返回：

```go
if e != nil { return e }
return nil
```

**Q4：错误怎么 wrap 和判断？**

```go
var ErrNotFound = errors.New("not found")

func load() error {
    return fmt.Errorf("load user 42: %w", ErrNotFound)
}

err := load()
errors.Is(err, ErrNotFound)  // true
err == ErrNotFound            // false (被 wrap)
```

**原则**：调用链路每一层加自己的上下文（`fmt.Errorf("query user: %w", err)`），最底层定义 sentinel。

**Q5：errors.Is 和 errors.As 什么时候用？**

```go
// sentinel: 用 Is
if errors.Is(err, sql.ErrNoRows) { ... }

// 自定义类型: 用 As 取出字段
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    log.Println(pathErr.Path)
}
```

`As` 的第二个参数必须是 `*Target` 指针。

**Q6：panic 能代替 error 吗？**
**不能**。panic 用于：
- 不可恢复错误（配置缺失、初始化失败）
- 程序员错误（index out of range、nil pointer）
- 库内部约定（如 JSON 编码遇到循环引用可能 panic）

业务逻辑错误一律用 error。把 panic 当异常用会让代码难以维护。

**Q7：什么场景应该 panic？**
- **无法继续运行**：启动时读不到必要配置
- **调用方错误**：给 API 传了非法参数且没有合理默认值（可以 return error 更好）
- **内部不变量被破坏**：断言式 panic，说明代码 bug

原则：**库代码倾向返回 error，应用代码可以 panic 并在顶层统一 recover**。

**Q8：中间件里怎么 recover？**

```go
func Recovery(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("panic: %v\n%s", r, debug.Stack())
                http.Error(w, "internal error", 500)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

**每个 goroutine 都需要自己的 recover**（panic 不跨 g 传播）。

**Q9：errors.Join 什么时候用？**
并行收集多个独立校验的错误、批量操作的错误聚合：

```go
func validate(u User) error {
    return errors.Join(
        validateName(u.Name),
        validateEmail(u.Email),
        validateAge(u.Age),
    )
}
// Error() 输出每行一条
// errors.Is(err, ErrBadEmail) 仍然工作
```

**Q10：stack trace 怎么拿？**
标准库不带。常用方案：
- `github.com/pkg/errors`（已较老，但 `Wrap` 自动采集栈）
- `github.com/cockroachdb/errors`
- Go 1.13+ 自己在 Error 返回时用 `runtime.Callers` 采集
- `debug.Stack()` 打当前完整堆栈（排障用）

## 四、手写实现

**1. 自定义 error 类型 + errors.As：**

```go
type NotFoundError struct {
    Resource string
    ID       int64
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s %d not found", e.Resource, e.ID)
}

func getUser(id int64) (*User, error) {
    u, ok := db[id]
    if !ok {
        return nil, &NotFoundError{Resource: "user", ID: id}
    }
    return u, nil
}

// 调用方
_, err := getUser(42)
var nfe *NotFoundError
if errors.As(err, &nfe) {
    http.Error(w, nfe.Error(), 404)
}
```

**2. 带 code 的 error 用于跨层传递：**

```go
type CodedError struct {
    Code    int
    Message string
    Cause   error
}

func (e *CodedError) Error() string {
    if e.Cause != nil {
        return fmt.Sprintf("[%d] %s: %v", e.Code, e.Message, e.Cause)
    }
    return fmt.Sprintf("[%d] %s", e.Code, e.Message)
}

func (e *CodedError) Unwrap() error { return e.Cause }

// 构造
func WrapCode(code int, msg string, cause error) *CodedError {
    return &CodedError{Code: code, Message: msg, Cause: cause}
}
```

**3. goroutine 安全包装（GoSafe）：**

```go
func GoSafe(f func()) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("panic recovered: %v\n%s", r, debug.Stack())
            }
        }()
        f()
    }()
}
```

**4. sentinel error + wrap 链：**

```go
// 领域包
var (
    ErrUserNotFound = errors.New("user not found")
    ErrUserBanned   = errors.New("user banned")
)

// Repo 层
func (r *Repo) Get(id int64) (*User, error) {
    if /* db 查不到 */ {
        return nil, ErrUserNotFound
    }
    return u, nil
}

// Service 层
func (s *Service) Login(id int64) error {
    u, err := s.repo.Get(id)
    if err != nil {
        return fmt.Errorf("login: %w", err)
    }
    if u.Banned {
        return ErrUserBanned
    }
    return nil
}

// Handler 层
err := svc.Login(id)
switch {
case errors.Is(err, ErrUserNotFound):
    return http.StatusNotFound, "user not found"
case errors.Is(err, ErrUserBanned):
    return http.StatusForbidden, "banned"
case err != nil:
    return http.StatusInternalServerError, "internal error"
}
```

## 五、踩坑与最佳实践

### 坑 1：nil 指针被装成非 nil 接口

见 Q3。返回 error 时显式 `return nil`。

### 坑 2：吞错误

```go
data, _ := os.ReadFile(path)  // 忽略错误
```

除非确定不可能错（或故意丢弃），否则必须处理。`errcheck` 工具可以扫出。

### 坑 3：日志 + return err 导致重复记录

```go
if err != nil {
    log.Printf("error: %v", err)  // 每层都打
    return err
}
```

上层也打一次 → 日志翻倍。**原则**：要么一层 wrap + 顶层 log，要么底层 log 然后返回。不要两头都打。

### 坑 4：panic 跨 goroutine 不传播

```go
go func() {
    panic("oops")  // 整个进程 crash, 主 g 的 recover 救不到
}()
```

每个 g 都要自己 recover。统一用 `GoSafe` 包装。

### 坑 5：err.Error() 拿来对比

```go
if err.Error() == "file not found" { ... }  // 脆弱,错误文案变了就失效
```

用 `errors.Is(err, ErrNotFound)`。

### 坑 6：把 error 当 int / bool 用

```go
if err != nil { return -1 }  // 错误信息全丢
```

老老实实把 err 返回上去，上层可能需要判断类型。

### 坑 7：defer 里吞 panic

```go
defer func() {
    recover()  // 空白 recover, panic 直接没了,上层无感
}()
```

除非你确信要丢弃（比如 worker 容错），否则 recover 后要**至少打日志**或转 error。

### 最佳实践

- **从头定义 sentinel**：领域错误在包内 `var ErrXxx = errors.New(...)`，外层用 `errors.Is` 判断
- **需要携带字段用自定义类型 + errors.As**
- **每层加上下文**：`fmt.Errorf("layer action: %w", err)`，形成清晰链路
- **panic 只在无法继续时用**，中间件统一 recover
- **goroutine 入口必加 recover + stack trace**
- **error 不要打印两次**，上层决定 log
- **不要靠 err.Error() 文案匹配**，用 Is/As
- 严肃服务用 `cockroachdb/errors` 或类似库，自动带 stack trace 和结构化信息
