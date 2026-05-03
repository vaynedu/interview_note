# 测试 (testing)

> Go 测试三件套：单元测试 + benchmark + example；table-driven 是 Go 标准范式；testify/mock/httptest/sqlmock 是常备工具

## 一、核心原理

### 1.1 文件命名与 testing 包

```go
// foo.go
package foo

func Add(a, b int) int { return a + b }

// foo_test.go
package foo  // 同包: 可测试未导出
// 或 package foo_test (黑盒测试: 只测公开 API)

import "testing"

func TestAdd(t *testing.T) {
    if got := Add(1, 2); got != 3 {
        t.Errorf("Add(1,2)=%d, want 3", got)
    }
}
```

**约定**：
- 测试文件 `*_test.go`，与被测文件同目录
- 测试函数 `Test<Name>(t *testing.T)`
- benchmark `Benchmark<Name>(b *testing.B)`
- example `Example<Name>()`，文档化用

### 1.2 table-driven（Go 标准范式）

```go
func TestAdd(t *testing.T) {
    tests := []struct{
        name    string
        a, b    int
        want    int
    }{
        {"basic", 1, 2, 3},
        {"zero", 0, 0, 0},
        {"negative", -1, 1, 0},
        {"overflow", math.MaxInt, 1, math.MinInt},
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            if got := Add(tc.a, tc.b); got != tc.want {
                t.Errorf("got %d, want %d", got, tc.want)
            }
        })
    }
}
```

**好处**：
- 加 case 只加一行
- `t.Run` 子测试可独立运行：`go test -run TestAdd/basic`
- 失败信息清晰

### 1.3 t 的核心方法

```go
t.Errorf("...")  // 报错继续
t.Fatalf("...")  // 报错并停止当前测试
t.Skip("...")    // 跳过测试 (如需要外部依赖)
t.Helper()       // 标记本函数为 helper, 报错行号指向调用方
t.Cleanup(fn)    // 注册清理 (推荐替代 defer)
t.Parallel()     // 并行运行
t.TempDir()      // 临时目录, 自动清理
t.Setenv(k, v)   // 测试期 env, 自动还原
t.Log(...)       // 调试输出 (-v 时才显示)
```

### 1.4 benchmark

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(1, 2)
    }
}

// 子 benchmark
func BenchmarkSort(b *testing.B) {
    sizes := []int{100, 1000, 10000}
    for _, n := range sizes {
        b.Run(fmt.Sprintf("n=%d", n), func(b *testing.B) {
            data := genData(n)
            b.ResetTimer()
            for i := 0; i < b.N; i++ {
                Sort(append([]int{}, data...))
            }
        })
    }
}
```

跑法：
```bash
go test -bench=. -benchmem -benchtime=3s
```

`-benchmem` 显示内存分配，`-benchtime=3s` 跑 3 秒（默认 1s）。

对比基准：`benchstat`：
```bash
go test -bench=. -count=10 > old.txt
# 改代码
go test -bench=. -count=10 > new.txt
benchstat old.txt new.txt
```

### 1.5 mock 策略

**接口 + mock**：

```go
type UserRepo interface {
    Get(ctx context.Context, id int64) (*User, error)
}

type Service struct{ repo UserRepo }

// 测试用 mock 实现
type mockRepo struct {
    users map[int64]*User
}
func (m *mockRepo) Get(_ context.Context, id int64) (*User, error) {
    if u, ok := m.users[id]; ok { return u, nil }
    return nil, ErrNotFound
}

func TestServiceLogin(t *testing.T) {
    svc := &Service{repo: &mockRepo{users: map[int64]*User{1: {Name: "x"}}}}
    err := svc.Login(context.Background(), 1)
    if err != nil { t.Fatal(err) }
}
```

或用工具自动生成：
- `mockery` / `gomock` 根据接口生成 mock 代码
- `testify/mock` 提供 mock helper

### 1.6 HTTP 测试 (httptest)

**测试 handler**：

```go
func TestHandler(t *testing.T) {
    req := httptest.NewRequest("GET", "/users/1", nil)
    rec := httptest.NewRecorder()

    handler.ServeHTTP(rec, req)

    if rec.Code != 200 { t.Fatalf("status=%d", rec.Code) }
    if !strings.Contains(rec.Body.String(), `"id":1`) { t.Fatal("bad body") }
}
```

**测试 client**：

```go
func TestClient(t *testing.T) {
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte(`{"name":"alice"}`))
    }))
    defer srv.Close()

    client := NewClient(srv.URL)
    u, err := client.GetUser(1)
    require.NoError(t, err)
    require.Equal(t, "alice", u.Name)
}
```

### 1.7 数据库测试

三种策略：

| 方式 | 优点 | 缺点 |
| --- | --- | --- |
| sqlmock | 快、不依赖 DB | 不能测真 SQL |
| docker + 真 DB | 真实 | 慢、CI 复杂 |
| testcontainers-go | 真实 + 自动管理 | 慢 |

```go
import "github.com/DATA-DOG/go-sqlmock"

func TestUserRepo(t *testing.T) {
    db, mock, _ := sqlmock.New()
    defer db.Close()

    rows := sqlmock.NewRows([]string{"id", "name"}).AddRow(1, "alice")
    mock.ExpectQuery("SELECT id, name FROM users WHERE id=").
        WithArgs(1).
        WillReturnRows(rows)

    repo := NewRepo(db)
    u, err := repo.Get(context.Background(), 1)
    require.NoError(t, err)
    require.Equal(t, "alice", u.Name)
    require.NoError(t, mock.ExpectationsWereMet())
}
```

### 1.8 Coverage

```bash
go test -cover ./...                    # 总览
go test -coverprofile=c.out ./...
go tool cover -html=c.out -o c.html     # 可视化
go tool cover -func=c.out               # 函数级覆盖率
```

业内常见目标：核心模块 > 80%，工具函数 > 90%。

### 1.9 testify

最常用的断言库：

```go
import "github.com/stretchr/testify/require"
import "github.com/stretchr/testify/assert"

require.NoError(t, err)              // 错则 t.Fatal
require.Equal(t, 1, x)               // 断言相等
assert.Contains(t, s, "hello")       // 断���不影响后续
assert.Len(t, slice, 3)
assert.True(t, b)
```

`require.*` 失败立即停止；`assert.*` 失败继续执行。

### 1.10 fuzz 测试 (Go 1.18+)

```go
func FuzzReverse(f *testing.F) {
    f.Add("hello")  // seed
    f.Fuzz(func(t *testing.T, s string) {
        r := Reverse(s)
        rr := Reverse(r)
        if s != rr { t.Errorf("mismatch") }
    })
}
```

跑：`go test -fuzz=FuzzReverse -fuzztime=30s`，自动生成各种输入测边界。

## 二、八股速记

- 测试文件 `*_test.go`，函数 `Test/Benchmark/Example<Name>`
- **table-driven** 是 Go 标准范式，配合 `t.Run` 子测试
- `t.Fatalf` 停止；`t.Errorf` 继续
- `t.Cleanup` 优于 defer，**自动逆序**
- benchmark 用 `b.N`，加 `-benchmem` 看分配
- mock 通过**接口注入**，工具：mockery / gomock
- HTTP 测试：`httptest.NewRequest` + `NewRecorder`（handler）/ `NewServer`（client）
- DB 测试：sqlmock 快、testcontainers 真
- coverage: `go test -cover` / `-coverprofile`
- testify `require`/`assert` 大幅简化断言
- **fuzz 测试** Go 1.18+ 自动生成边界输入

## 三、面试真题

**Q1：Go 测试为什么推荐 table-driven？**
- **加 case 成本极低**：一行结构体
- **子测试独立**：`go test -run TestX/case_name` 单独跑
- **失败信息清晰**：自动带 case name
- **覆盖边界**：列出所有边界一目了然

vs 多个独立 Test 函数：重复样板代码、case 散乱。

**Q2：t.Errorf vs t.Fatalf？**
- `Errorf`：报错但**继续执行后续代码**，适合多个独立断言
- `Fatalf`：报错并**停止当前测试**，适合关键前提（前置失败后续无意义）

```go
u, err := getUser()
if err != nil { t.Fatalf("getUser: %v", err) }  // u 没拿到, 不能继续
if u.Name != "alice" { t.Errorf("name=%s", u.Name) }
if u.Age != 18 { t.Errorf("age=%d", u.Age) }
```

**Q3：怎么测试一个调用 DB 的函数？**

3 种方式：

1. **sqlmock**：mock SQL 层，不真连 DB。优点快，缺点不能测真 SQL 行为
2. **testcontainers-go**：用 Docker 启真 DB，测试结束清理。最真实
3. **专用测试 DB**：CI 提供共享测试 DB（要清理数据 / 用事务回滚）

业务推荐 testcontainers，简单 case 用 sqlmock。

**Q4：怎么 mock 时间/随机数？**

```go
// 不可测
func IsExpired() bool { return time.Now().After(deadline) }

// 可测: 注入时间源
type Clock interface { Now() time.Time }
type realClock struct{}
func (realClock) Now() time.Time { return time.Now() }

type Service struct{ clock Clock }
func (s *Service) IsExpired() bool { return s.clock.Now().After(deadline) }

// 测试
type fakeClock struct{ t time.Time }
func (f fakeClock) Now() time.Time { return f.t }

svc := &Service{clock: fakeClock{t: ...}}
```

或用 `clockwork` 等库。

**Q5：t.Parallel 的注意事项？**

```go
func TestX(t *testing.T) {
    t.Parallel()  // 标记为并行
    // ...
}

func TestY(t *testing.T) {
    t.Parallel()
    // 与 TestX 并行运行
}
```

**注意**：
- 共享变量要并发安全
- `t.Setenv` 与 Parallel 不能共用
- 子测试也要各自 `t.Parallel()`
- 循环里启子测试要捕获循环变量（Go 1.22 前）

**Q6：怎么测有外部依赖的代码？**
**接口隔离 + 依赖注入**：

```go
// 业务代码: 依赖接口
type Service struct {
    cache  Cache
    db     DB
    api    ExternalAPI
}

// 测试: 传 mock 实现
svc := &Service{
    cache: &mockCache{...},
    db:    &mockDB{...},
    api:   &mockAPI{...},
}
```

**禁止**：
- 业务代码里 `time.Now()` 写死
- 直接 `http.Get` 不能 mock
- 全局变量难替换

**Q7：benchmark 怎么避免编译器优化？**

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(1, 2)  // 结果未用, 可能被优化掉
    }
}
```

修复：用结果

```go
var sink int

func BenchmarkAdd(b *testing.B) {
    var s int
    for i := 0; i < b.N; i++ {
        s = Add(1, 2)
    }
    sink = s  // 防止被优化
}
```

或用 `runtime.KeepAlive`。

**Q8：怎么测 panic？**

```go
func TestMustParse(t *testing.T) {
    defer func() {
        if r := recover(); r == nil {
            t.Fatal("expected panic")
        }
    }()
    MustParse("invalid")
}
```

或 `require.Panics(t, func() { MustParse("...") })`。

**Q9：测试覆盖率 100% 是好事吗？**
不一定。

- **核心业务**：争取高覆盖（80%+）
- **工具函数**：必须高覆盖（90%+）
- **handler 层**：覆盖主要路径即可（60~70%）
- **生成代码 / DTO**：可豁免

100% 覆盖率 ≠ 0 bug。**质量比覆盖率重要**：边界 case、异常路径、并发场景。

**Q10：怎么组织测试代码？**

```
internal/
├── biz/
│   ├── order.go
│   ├── order_test.go         # 单测
│   └── testdata/             # 测试数据 (Go 编译器自动忽略)
│       └── sample.json
└── ...
test/
├── e2e/                      # 端到端测试
└── integration/              # 集成测试
```

约定：
- 单测和源码同目录
- `testdata/` 是 Go 编译器约定的忽略目录，存测试数据文件
- e2e/集成测试单独目录，可单独跑（build tag）

## 四、手写实现

**1. 完整 table-driven 模板：**

```go
func TestParseAge(t *testing.T) {
    tests := []struct{
        name    string
        input   string
        want    int
        wantErr bool
    }{
        {"valid", "18", 18, false},
        {"zero", "0", 0, false},
        {"negative", "-1", 0, true},
        {"too large", "200", 0, true},
        {"non-number", "abc", 0, true},
        {"empty", "", 0, true},
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            got, err := ParseAge(tc.input)
            if (err != nil) != tc.wantErr {
                t.Fatalf("err=%v, wantErr=%v", err, tc.wantErr)
            }
            if got != tc.want {
                t.Errorf("got %d, want %d", got, tc.want)
            }
        })
    }
}
```

**2. mock 时间：**

```go
type clock interface { Now() time.Time }

type Service struct {
    clock clock
}

// 生产
type realClock struct{}
func (realClock) Now() time.Time { return time.Now() }

// 测试
type fakeClock struct{ t time.Time }
func (f *fakeClock) Now() time.Time { return f.t }
func (f *fakeClock) Advance(d time.Duration) { f.t = f.t.Add(d) }

func TestExpiry(t *testing.T) {
    fc := &fakeClock{t: time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC)}
    s := &Service{clock: fc}
    fc.Advance(time.Hour)
    require.True(t, s.IsExpired())
}
```

**3. HTTP handler 测试：**

```go
func TestGetUser(t *testing.T) {
    h := NewHandler(&mockRepo{users: map[int64]*User{1: {Name: "alice"}}})

    tests := []struct {
        name   string
        url    string
        status int
        body   string
    }{
        {"ok", "/users/1", 200, `"alice"`},
        {"not found", "/users/2", 404, ""},
        {"bad id", "/users/abc", 400, ""},
    }
    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            req := httptest.NewRequest("GET", tc.url, nil)
            rec := httptest.NewRecorder()
            h.ServeHTTP(rec, req)
            require.Equal(t, tc.status, rec.Code)
            if tc.body != "" {
                require.Contains(t, rec.Body.String(), tc.body)
            }
        })
    }
}
```

**4. cleanup 优于 defer：**

```go
func TestWithDB(t *testing.T) {
    db := setupDB(t)
    t.Cleanup(func() {
        db.Close()
    })

    t.Cleanup(func() {
        os.Remove("/tmp/test.db")
    })

    // Cleanup 自动 LIFO 执行, 不需要 defer
    // 子测试也能继承父 cleanup
}
```

**5. benchmark 对比：**

```go
func BenchmarkConcat(b *testing.B) {
    parts := []string{"a", "b", "c", "d", "e"}

    b.Run("plus", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            s := ""
            for _, p := range parts { s += p }
            sink = s
        }
    })

    b.Run("builder", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            var sb strings.Builder
            for _, p := range parts { sb.WriteString(p) }
            sink = sb.String()
        }
    })

    b.Run("join", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            sink = strings.Join(parts, "")
        }
    })
}
var sink string
```

## 五、踩坑与最佳实践

### 坑 1：t.Errorf 后继续访问 nil

```go
u, err := getUser()
if err != nil { t.Errorf("err: %v", err) }
fmt.Println(u.Name)  // u 可能 nil, panic
```

**修复**：用 `t.Fatalf`，或先判 `nil`。

### 坑 2：循环变量捕获（Go 1.22 前）

```go
for _, tc := range tests {
    t.Run(tc.name, func(t *testing.T) {
        t.Parallel()
        check(tc.input)  // 老版本 tc 可能是最后一个
    })
}
```

**修复**：`tc := tc`。Go 1.22+ 已修复。

### 坑 3：t.Setenv 在 Parallel 测试

```go
func TestX(t *testing.T) {
    t.Parallel()
    t.Setenv("X", "1")  // panic: t.Setenv called after t.Parallel
}
```

Setenv 修改进程级 env，与并行冲突。

### 坑 4：测试有顺序依赖

```go
func TestA(t *testing.T) { create(...) }
func TestB(t *testing.T) { read(...) }  // 依赖 A 先跑
```

`go test` 不保证顺序。**修复**：每个测试独立，setup/teardown 在自己里。

### 坑 5：测试用真实外部依赖

```go
func TestPaymentGateway(t *testing.T) {
    pay(realCreditCard)  // 真扣钱?!
}
```

绝对不要。用 mock / sandbox。

### 坑 6：sleep 等异步

```go
go doWork()
time.Sleep(100 * time.Millisecond)  // 脆弱, 慢机器超时
assert.True(t, done)
```

**修复**：用 channel 或 `require.Eventually`：

```go
require.Eventually(t, func() bool { return done }, 5*time.Second, 10*time.Millisecond)
```

### 坑 7：测试不清理状态

```go
func TestWriteFile(t *testing.T) {
    os.WriteFile("/tmp/x", data, 0644)
    // 没清理, 下次跑残留
}
```

**修复**：`t.TempDir()` 或 `t.Cleanup`。

### 坑 8：覆盖率追求 100% 写无意义测试

```go
func TestGetterSetter(t *testing.T) {
    o := &Order{}
    o.SetID(1)
    require.Equal(t, int64(1), o.GetID())  // 测了个寂寞
}
```

聚焦业务逻辑、边界、异常路径。

### 最佳实践

- **table-driven** + `t.Run` 是标准范式
- **接口隔离** + 依赖注入让代码可测
- 优先 `t.Cleanup`，胜于 defer
- 优先 `t.TempDir()`，胜于硬编码 /tmp
- testify `require/assert` 简化断言
- benchmark 加 `-benchmem`，对比用 `benchstat`
- HTTP 用 `httptest`，DB 用 sqlmock 或 testcontainers
- Cleanup / TempDir 让测试自洁
- `t.Parallel` 加速，注意状态隔离
- coverage 追求**有意义**的覆盖，不是数字
- **fuzz 测试**（Go 1.18+）找边界 bug
- 集成 / e2e 测试单独 build tag，CI 分阶段跑
