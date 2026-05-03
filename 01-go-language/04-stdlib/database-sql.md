# database/sql

> Go 数据库抽象层：driver 接口 + 连接池 + 预处理语句 + 事务；本身不带 SQL，靠 driver（mysql/pgx/sqlite）实现

## 一、核心原理

### 1.1 整体架构

```mermaid
flowchart LR
    App[业务代码] --> DB[sql.DB]
    DB --> Pool[(连接池)]
    Pool --> Drv[Driver接口]
    Drv --> Real[mysql/pgx/sqlite]
    Real --> Net[(数据库)]

    style DB fill:#9f9
    style Pool fill:#ff9
```

`sql.DB` 不是单连接，是**连接池**句柄：
- 全局**一个就够**，不要每次请求 new
- 内部用 `sync.Mutex` + `chan` 管理空闲连接
- 并发安全，可被多 goroutine 共享

### 1.2 driver 接口

```go
import (
    "database/sql"
    _ "github.com/go-sql-driver/mysql"  // 注册 driver
)

db, err := sql.Open("mysql", dsn)
```

`_ "..."` 触发 driver 包的 `init()` 调用 `sql.Register("mysql", &MySQLDriver{})`。

`sql.Open` 不会真连数据库，只校验 DSN。第一次 `Query` / `Ping` 才真连。

### 1.3 连接池配置（必调）

```go
db.SetMaxOpenConns(100)              // 最大打开连接(含使用中)
db.SetMaxIdleConns(10)               // 最大空闲连接
db.SetConnMaxLifetime(time.Hour)     // 连接最长存活时间
db.SetConnMaxIdleTime(10 * time.Minute) // 空闲连接最长存活
```

| 参数 | 默认 | 推荐 |
| --- | --- | --- |
| MaxOpenConns | 0（无限） | 50~200（按 DB 承载力） |
| MaxIdleConns | 2 | 与 MaxOpenConns 同序 |
| ConnMaxLifetime | 0（永久） | 30min~1h（避开 DB/中间件断连） |
| ConnMaxIdleTime | 0 | 几分钟 |

> **MaxIdleConns 默认 2 是最大坑**，高并发下连接频繁开关。

### 1.4 三类调用

```go
// 1. 不返回结果集 (INSERT/UPDATE/DELETE/DDL)
res, err := db.ExecContext(ctx, "INSERT ... VALUES (?)", v)
n, _ := res.RowsAffected()

// 2. 返回多行
rows, err := db.QueryContext(ctx, "SELECT id, name FROM users WHERE age > ?", 18)
defer rows.Close()
for rows.Next() {
    var id int64; var name string
    rows.Scan(&id, &name)
}
if err := rows.Err(); err != nil { /* iteration 错误 */ }

// 3. 返回单行
err := db.QueryRowContext(ctx, "SELECT name FROM users WHERE id=?", 1).Scan(&name)
if errors.Is(err, sql.ErrNoRows) { /* 没记录 */ }
```

**重要**：`rows.Close()` 必须；`QueryRow` 返回 `*Row`（不是 `*Rows`），`Scan` 内部会自动释放连接。

### 1.5 预处理语句 Stmt

```go
stmt, err := db.PrepareContext(ctx, "SELECT name FROM users WHERE id=?")
defer stmt.Close()
for _, id := range ids {
    var n string
    stmt.QueryRowContext(ctx, id).Scan(&n)
}
```

好处：
- SQL 解析一次，多次执行
- 防 SQL 注入（参数化）

但 Go 的 Stmt 在连接池下有坑：每个连接需要单独 prepare。Driver（如 mysql）通常会自动管理。**业务代码倾向用普通 `Query`，driver 层会做 prepare**。

### 1.6 事务

```go
tx, err := db.BeginTx(ctx, &sql.TxOptions{
    Isolation: sql.LevelReadCommitted,
    ReadOnly:  false,
})
if err != nil { return err }
defer tx.Rollback()  // 兜底,Commit 后再 Rollback 是 no-op

if _, err := tx.ExecContext(ctx, "..."); err != nil { return err }
if _, err := tx.ExecContext(ctx, "..."); err != nil { return err }
return tx.Commit()
```

**Tx 持有一个连接直到 Commit/Rollback**。所有操作必须用 `tx.XxxContext`，不要混用 `db.XxxContext`（那会从池里另拿连接，不在事务里）。

### 1.7 context 集成

```go
ctx, cancel := context.WithTimeout(reqCtx, 200*time.Millisecond)
defer cancel()
db.QueryContext(ctx, "...")
```

ctx 取消时：
- driver 调用底层 cancel（mysql 用 KILL QUERY）
- 返回 `context.Canceled` / `context.DeadlineExceeded`

> **业务代码必须用 Context 版本** (`QueryContext` / `ExecContext`)，不要用 `Query` / `Exec`。

## 二、八股速记

- `sql.DB` 是**连接池**，全局共享
- `sql.Open` 不真连，第一次用时才连
- **MaxIdleConns 默认 2 必须调大**
- `MaxOpenConns` 防打爆 DB
- `ConnMaxLifetime` 配合中间件/DB 断连策略
- `rows.Close()` 必须，否则连接泄漏
- 单行 `QueryRow.Scan`，没记录返回 `sql.ErrNoRows`
- `Tx` 占用一个连接直到结束，操作必须走 `tx.Xxx`
- 用 `Context` 版本，永远传 ctx 控制超时
- driver 通过 `_ "package"` 注册

## 三、面试真题

**Q1：sql.DB 是单连接吗？为什么全局只要一个？**
不是单连接，是**连接池**。`sql.DB` 内部维护 idle conn 链表，调用方来要连接时从池取，用完归还。设计上**线程安全**，多 goroutine 共享一个 `*sql.DB`。

每次请求 new 一个 `sql.DB` 是反模式：池被打散，每次都重新连。

**Q2：连接池满了会怎样？**
当前 g 阻塞在 `getConn` 直到：
- 有连接归还
- ctx 超时（返回 ctx.Err）
- 没设 ctx 超时 → 永久阻塞，业务夯住

**生产强制**：必传 ctx + 必设超时。

**Q3：`MaxIdleConns` 和 `MaxOpenConns` 区别？**
- MaxOpen：总数上限（含正在使用的 + 空闲的）
- MaxIdle：空闲池上限（不含使用中）

约束：MaxIdle ≤ MaxOpen。MaxIdle 太小 → 连接频繁开关；太大 → 长期占用 DB 连接配额。

**Q4：什么是 `ConnMaxLifetime`，为什么要设？**
连接打开后最长存活时间。到期归还时直接销毁，下次创建新连接。

**为什么必设**：
- DB（如 MySQL）默认 `wait_timeout=28800s`（8h），到期断连。Go 池里的旧连接下次用直接报错
- 中间件（ProxySQL、HAProxy）可能更早断
- 主从切换时，老连接还连着旧 master

**推荐**：30min ~ 1h，比 DB/中间件超时小。

**Q5：QueryRow 返回 `*Row`，没找到怎么判断？**

```go
var name string
err := db.QueryRowContext(ctx, sql, id).Scan(&name)
switch {
case errors.Is(err, sql.ErrNoRows):
    // 没记录
case err != nil:
    // 其他错
default:
    // 拿到了
}
```

`Scan` 在没行时返回 `sql.ErrNoRows`，要用 `errors.Is`（可能被 wrap）。

**Q6：rows.Close() 不调用会怎样？**
- 连接没归还到池 → 池逐渐枯竭
- 高并发下表现：QPS 突然暴跌、`getConn` 超时
- pprof 看 goroutine 卡在 `database/sql.(*Rows).Next`

**养成习惯**：`defer rows.Close()` 紧跟 `Query`，即使后面 `for rows.Next()` 自然遍历完也写。

**Q7：事务怎么写正确？**

```go
func transfer(ctx context.Context, db *sql.DB, from, to int64, amt int) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil { return err }
    defer tx.Rollback()  // 关键: Commit 之后 Rollback 是 no-op

    if _, err := tx.ExecContext(ctx, "UPDATE accounts SET balance=balance-? WHERE id=?", amt, from); err != nil {
        return err  // defer Rollback 收尾
    }
    if _, err := tx.ExecContext(ctx, "UPDATE accounts SET balance=balance+? WHERE id=?", amt, to); err != nil {
        return err
    }
    return tx.Commit()
}
```

**关键**：
- `defer tx.Rollback()` 兜底（Commit 后无副作用）
- 所有操作走 `tx.Xxx`
- 错误立即 return，不要继续

**Q8：什么是 SQL 注入？怎么防？**
拼接用户输入到 SQL：

```go
// 错: SQL 注入
db.Query("SELECT * FROM users WHERE name='" + name + "'")
// name = "' OR 1=1 --" 直接拖库

// 对: 参数化
db.Query("SELECT * FROM users WHERE name=?", name)
```

参数化让 driver 把值当数据传，不当 SQL 解析。

**Q9：怎么实现读写分离？**
方法 1：**两个 DB 实例**

```go
type DB struct {
    Master *sql.DB
    Slave  *sql.DB
}
func (d *DB) Read(ctx context.Context, ...) { d.Slave.QueryContext(...) }
func (d *DB) Write(ctx context.Context, ...) { d.Master.ExecContext(...) }
```

方法 2：**driver 自带支持**（如 `mysql-driver` 的多 host）。

**注意**：写后立刻读可能读不到（主从延迟），关键路径仍走主。

**Q10：怎么排查"DB 连接打满"？**

```go
stats := db.Stats()
fmt.Printf("Open=%d InUse=%d Idle=%d WaitCount=%d WaitDuration=%v\n",
    stats.OpenConnections, stats.InUse, stats.Idle,
    stats.WaitCount, stats.WaitDuration)
```

监控指标：
- `WaitCount` / `WaitDuration` 持续上升 → 池子小或 SQL 慢
- `OpenConnections` 接近 `MaxOpenConns` → 池满

排查：
1. 慢 SQL（DB 侧 `SHOW PROCESSLIST`）
2. 没释放（`rows.Close()` / `tx.Rollback`）
3. 池配置太小
4. 长事务

## 四、手写实现

**1. 标准连接池配置：**

```go
func NewDB(dsn string) (*sql.DB, error) {
    db, err := sql.Open("mysql", dsn)
    if err != nil { return nil, err }

    db.SetMaxOpenConns(100)
    db.SetMaxIdleConns(20)
    db.SetConnMaxLifetime(30 * time.Minute)
    db.SetConnMaxIdleTime(5 * time.Minute)

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    if err := db.PingContext(ctx); err != nil {
        db.Close()
        return nil, err
    }
    return db, nil
}
```

**2. 通用查询封装：**

```go
type User struct {
    ID   int64
    Name string
    Age  int
}

func GetUser(ctx context.Context, db *sql.DB, id int64) (*User, error) {
    var u User
    err := db.QueryRowContext(ctx,
        "SELECT id, name, age FROM users WHERE id=?", id,
    ).Scan(&u.ID, &u.Name, &u.Age)
    if errors.Is(err, sql.ErrNoRows) {
        return nil, ErrNotFound
    }
    if err != nil { return nil, err }
    return &u, nil
}

func ListUsers(ctx context.Context, db *sql.DB, minAge int) ([]User, error) {
    rows, err := db.QueryContext(ctx,
        "SELECT id, name, age FROM users WHERE age>=?", minAge,
    )
    if err != nil { return nil, err }
    defer rows.Close()

    var users []User
    for rows.Next() {
        var u User
        if err := rows.Scan(&u.ID, &u.Name, &u.Age); err != nil {
            return nil, err
        }
        users = append(users, u)
    }
    return users, rows.Err()
}
```

**3. 事务辅助函数：**

```go
func WithTx(ctx context.Context, db *sql.DB, fn func(*sql.Tx) error) (err error) {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil { return err }
    defer func() {
        if p := recover(); p != nil {
            tx.Rollback()
            panic(p)
        } else if err != nil {
            tx.Rollback()
        } else {
            err = tx.Commit()
        }
    }()
    err = fn(tx)
    return
}

// 使用
err := WithTx(ctx, db, func(tx *sql.Tx) error {
    if _, err := tx.ExecContext(ctx, "..."); err != nil { return err }
    if _, err := tx.ExecContext(ctx, "..."); err != nil { return err }
    return nil
})
```

**4. 健康检查 + 重连：**

```go
func StartHealthCheck(db *sql.DB) {
    ticker := time.NewTicker(30 * time.Second)
    go func() {
        for range ticker.C {
            ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
            if err := db.PingContext(ctx); err != nil {
                log.Printf("db ping failed: %v", err)
                // 此处可上报 metrics, ConnMaxLifetime 会自然让旧连接被替换
            }
            cancel()
        }
    }()
}
```

## 五、踩坑与最佳实践

### 坑 1：MaxIdleConns 默认 2

```go
db, _ := sql.Open(...)
// 默认 MaxIdleConns=2, 高并发会持续创建/销毁连接
```

QPS 高时频繁 TCP 三次握手 + 鉴权握手，CPU 和 RT 都飙。**修复**：`SetMaxIdleConns(20+)`。

### 坑 2：rows 没 Close 导致泄漏

```go
rows, _ := db.Query("...")
for rows.Next() {
    if shouldStop { return }  // 没 Close, 连接没归还
    rows.Scan(...)
}
```

**修复**：`defer rows.Close()` 在 Query 之后立即写。

### 坑 3：sql.ErrNoRows 当成普通错误

```go
err := db.QueryRow(sql).Scan(&v)
if err != nil {
    log.Errorf("db error: %v", err)  // ErrNoRows 也被打成 error
    return err
}
```

**修复**：先判断 `errors.Is(err, sql.ErrNoRows)`，业务上区分对待。

### 坑 4：tx 里混用 db 操作

```go
tx, _ := db.BeginTx(ctx, nil)
db.QueryContext(ctx, "SELECT ...")  // 错: 这是另一个连接,不在事务里
tx.ExecContext(ctx, "INSERT ...")
tx.Commit()
```

**修复**：tx 范围内**只用 tx.Xxx**。

### 坑 5：长事务

```go
tx, _ := db.BeginTx(ctx, nil)
data := callRemoteAPI()  // 几秒 RPC
tx.ExecContext(ctx, "INSERT ...", data)
tx.Commit()
```

事务期间持有 DB 连接 + 行锁，几秒不释放，导致：
- 连接池枯竭
- 死锁概率↑
- 主从延迟↑

**修复**：先做远程 IO，最后只在事务里做必需的 SQL。

### 坑 6：忘记 defer tx.Rollback()

```go
tx, _ := db.BeginTx(ctx, nil)
if _, err := tx.Exec("..."); err != nil {
    return err  // tx 没 rollback, 持有连接直到 GC
}
tx.Commit()
```

**修复**：紧跟 BeginTx 的下一行 `defer tx.Rollback()`。Commit 后再 Rollback 是 no-op，安全。

### 坑 7：Prepare 在循环外

```go
stmt, _ := db.Prepare("...")
for _, id := range ids {
    rows, _ := stmt.Query(id)  // OK
    rows.Close()
}
stmt.Close()
```

业务里**通常直接用 `db.Query` 即可**，driver 内部会管理 prepare。手动 Prepare 主要在确实热点 + 大量循环时考虑。

### 坑 8：ConnMaxLifetime=0（永久）

DB / LB 8 小时断连，Go 池里的"死连接"下次取出报 `bad connection`。Go 1.17+ 的 retryable 会自动重试一次，但极端场景仍可能报错。**修复**：设 30min~1h。

### 最佳实践

- 全局一个 `*sql.DB`，依赖注入到各层
- **必调连接池四件套**：MaxOpen/MaxIdle/ConnMaxLifetime/ConnMaxIdleTime
- **必用 Context 版本**，每次操作都传 ctx 带超时
- `rows.Close()` 紧跟 Query；`tx.Rollback()` 紧跟 BeginTx（defer）
- 错误必判 `sql.ErrNoRows`
- 监控 `db.Stats()`，关注 `WaitCount` / `WaitDuration`
- ORM（GORM/ent/sqlc）在简单 CRUD 加速，但复杂查询直接写 SQL 更可控
- 准备语句的 `?` / `$N` 一律参数化，防注入
- 健康检查（每 30s Ping）+ Prometheus 指标（连接数/慢查询）
