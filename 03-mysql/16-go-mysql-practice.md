# Go MySQL 工程实战

> 03-mysql 的 Go 落地篇：把原理沉到 **database/sql + GORM + sqlx + 连接池 + 超时 + 事务 + 埋点 + 重试 + 读写分离** 全套工程封装。
>
> 与 [04-redis/20-go-redis-best-practices.md](../04-redis/20-go-redis-best-practices.md) 对偶。
>
> 面试被问"你们 Go 里 MySQL 怎么用的"时，这篇能直接给答案。

---

## 一、为什么需要这一篇

```
典型面试追问链:
  "用什么 ORM？"
  → "为什么用 GORM/sqlx/原生？"
  → "连接池怎么调？"
  → "Context 超时能真的杀掉 SQL 吗？"
  → "事务里能调 RPC 吗？"
  → "读写分离怎么做？"
  → "批量插入怎么写？"
  → "DB 挂了你的服务能跑吗？"

只会 db.Query 的人答到第 3 问就卡住。
资深的答案应该是一套 SDK 级别的工程封装。
```

---

## 二、database/sql 连接池深度

### 2.1 核心概念

`database/sql` 不是连接，是**连接池管理器**。每个 `*sql.DB` 实例内部维护一池连接，并发安全。

```
全局单例:
  var db *sql.DB  // 全程序一个，不要每次新建

并发模型:
  N goroutine 共享一个 *sql.DB
  实际执行时 → 从池取连接 → 用完归还
```

### 2.2 四个关键参数

```go
db.SetMaxOpenConns(50)         // 最大连接数（含 InUse + Idle）
db.SetMaxIdleConns(10)         // 最大空闲连接（推荐 = MaxOpenConns 的 20-50%）
db.SetConnMaxLifetime(30 * time.Minute)  // 连接最大生命周期
db.SetConnMaxIdleTime(10 * time.Minute)  // 空闲多久后关闭
```

### 2.3 MaxOpenConns 怎么定

```
公式（Little's Law）:
  MaxOpenConns ≈ QPS × 平均 SQL RT(s) × 冗余系数

示例:
  单实例 QPS = 200
  平均 SQL RT = 5ms = 0.005s
  冗余 = 2x
  → MaxOpenConns = 200 × 0.005 × 2 = 2

实际更复杂（突发 + 慢查询波动）:
  - 小服务（QPS<100）:    MaxOpenConns = 20-50
  - 中等（QPS 100-1k）:   MaxOpenConns = 50-100
  - 大流量（QPS>1k）:     MaxOpenConns = 100-300

⚠️ 上限受 MySQL 限制:
  MySQL max_connections 默认 151
  N 个应用实例 × MaxOpenConns < MySQL max_connections × 0.8
```

### 2.4 MaxIdleConns vs MaxOpenConns 的坑

```
❌ MaxIdleConns 太小:
  每次都现建连接 → DialTimeout + TLS 握手 = 几十 ms
  RT 飙升

❌ MaxIdleConns = MaxOpenConns:
  空闲连接也占 MySQL 资源
  云上 MySQL 按连接收费

✓ 推荐:
  MaxIdleConns = MaxOpenConns × 0.2-0.5
```

### 2.5 ConnMaxLifetime 必设

```
为什么必须设？
  ① MySQL wait_timeout 默认 28800s（8h），会主动断
  ② 云上 LB / NAT 静默断连
  ③ MySQL 主从切换后老连接连的是旧主

不设 → 某天突然大面积 "broken pipe" / "invalid connection"

推荐:
  ConnMaxLifetime = 30min - 1h（< MySQL wait_timeout）
  ConnMaxIdleTime = 10min - 30min
```

### 2.6 监控连接池状态

```go
// 定时采集 db.Stats()
func exportDBStats(db *sql.DB) {
    s := db.Stats()
    metrics.Gauge("db.open").Set(float64(s.OpenConnections))
    metrics.Gauge("db.in_use").Set(float64(s.InUse))
    metrics.Gauge("db.idle").Set(float64(s.Idle))
    metrics.Counter("db.wait_count").Add(float64(s.WaitCount))
    metrics.HistogramObserve("db.wait_duration", s.WaitDuration.Seconds())
    metrics.Counter("db.max_idle_closed").Add(float64(s.MaxIdleClosed))
    metrics.Counter("db.max_lifetime_closed").Add(float64(s.MaxLifetimeClosed))
}
```

**关键指标**：
- `WaitCount` 持续增长 → 池不够，加 MaxOpenConns
- `WaitDuration` P99 > 100ms → 阻塞严重
- `MaxIdleClosed` 高 → IdleConns 太小，频繁建连
- `MaxLifetimeClosed` 高 → 正常（按预期老化）

---

## 三、超时分层设计

### 3.1 三层超时

```
HTTP 请求超时   (3-10s)
    ↓
业务上下文超时  (1-3s)   ← context.WithTimeout
    ↓
单 SQL 超时    (200ms-1s) ← QueryContext / ExecContext
    ↓
连接获取超时   (受 PoolWait 影响)
    ↓
DialTimeout    (DSN: timeout=500ms)
```

### 3.2 DSN 配置完整版

```go
dsn := "user:pass@tcp(host:3306)/dbname?" +
    "charset=utf8mb4" +              // 字符集
    "&parseTime=true" +              // DATETIME → time.Time
    "&loc=Local" +                   // 时区
    "&timeout=500ms" +               // Dial 超时
    "&readTimeout=2s" +              // 读超时（保护 net.Conn 层）
    "&writeTimeout=2s" +             // 写超时
    "&interpolateParams=true" +      // 客户端拼参数（减少一次 RTT）
    "&maxAllowedPacket=16777216" +   // 16MB
    "&collation=utf8mb4_0900_ai_ci"
```

### 3.3 Context 超时的真相

```go
ctx, cancel := context.WithTimeout(parent, 200*time.Millisecond)
defer cancel()

row := db.QueryRowContext(ctx, "SELECT ... FROM big_table WHERE ...")
```

**重要事实**：

```
Context 超时触发时:
  1. database/sql 调用 driver.Conn.Cancel()
  2. go-sql-driver/mysql 在 *新连接* 上发 KILL QUERY <thread_id>
  3. 当前连接被标记 bad，归还池时丢弃

但 KILL 不立刻生效:
  - 正在 fsync / 网络 IO → 要等
  - 正在回滚 → 可能更久
  - 复杂查询拆分点之间 → 才会检测 KILL 信号

→ 业务感知超时，但 DB 端 SQL 仍在跑
→ 资源仍被占用
```

**应对策略**：

```
1. 应用层 Context 超时 << 预期 SQL RT × 2
2. 慢 SQL 优化（不能依赖 KILL 兜底）
3. 大查询用 LIMIT 拆批，每批快速完成
4. 监控 KILL 频率（高 = 业务设计有问题）
```

### 3.4 别用 db.Query / db.Exec 不带 Context

```go
// ❌ 永不超时，goroutine 泄漏风险
db.Query("SELECT ...")

// ❌ 用了 background ctx
db.QueryContext(context.Background(), "SELECT ...")

// ✅ 业务级超时
ctx, cancel := context.WithTimeout(reqCtx, 500*time.Millisecond)
defer cancel()
db.QueryContext(ctx, "SELECT ...")
```

---

## 四、ORM 选型：原生 / sqlx / GORM

### 4.1 三者对比

| 维度 | database/sql 原生 | sqlx | GORM |
| --- | --- | --- | --- |
| 性能 | 最快 | 接近原生 | 慢 20-50% |
| 易用性 | 啰嗦 | 简洁 | 最易 |
| SQL 控制 | 完全 | 完全 | 部分（生成 SQL）|
| 学习曲线 | 平 | 平 | 陡 |
| 调试 | 直观 | 直观 | 黑盒 |
| 反射 | 无 | 少量 | 大量 |
| 适合场景 | 极致性能 / 复杂 SQL | 多数业务 | 快速开发 / CRUD 多 |

### 4.2 sqlx 推荐用法

```go
import "github.com/jmoiron/sqlx"

type User struct {
    ID   int64  `db:"id"`
    Name string `db:"name"`
    Age  int    `db:"age"`
}

// 单行查询
var u User
err := db.GetContext(ctx, &u, "SELECT id, name, age FROM users WHERE id=?", 1)

// 多行查询
var us []User
err := db.SelectContext(ctx, &us, "SELECT id, name, age FROM users WHERE age > ?", 18)

// 命名参数
_, err := db.NamedExecContext(ctx,
    `INSERT INTO users (name, age) VALUES (:name, :age)`,
    map[string]interface{}{"name": "Alice", "age": 25})

// IN 查询
ids := []int64{1, 2, 3}
query, args, _ := sqlx.In("SELECT * FROM users WHERE id IN (?)", ids)
query = db.Rebind(query)  // ? → 对应驱动占位符
db.SelectContext(ctx, &us, query, args...)
```

### 4.3 GORM 深度使用要点

```go
import "gorm.io/gorm"

// ✅ Session 模式控制行为
db.WithContext(ctx).
    Session(&gorm.Session{
        PrepareStmt:            true,   // 启用 prepared cache
        QueryFields:            true,   // SELECT 显式列名
        AllowGlobalUpdate:      false,  // 禁止无 WHERE 全表更新
        SkipDefaultTransaction: true,   // 单条 query 不开事务（提速）
    }).
    First(&user, id)

// ✅ 显式选择字段（生产强烈推荐）
db.Select("id", "name", "age").Find(&users)

// ✅ 强制使用索引
db.Clauses(hints.UseIndex("idx_user_age")).Find(&users)

// ✅ 软删除关闭（默认 GORM 加 deleted_at IS NULL）
db.Unscoped().Find(&users)
```

### 4.4 GORM 的坑

```
1. 默认开事务（每个 SQL）
   → SkipDefaultTransaction: true

2. 全字段 UPDATE
   db.Save(&user) → UPDATE 所有列
   → 用 db.Updates(map{}) 只更指定列

3. AutoMigrate 生产慎用
   → 只在开发环境，生产用迁移工具（goose / flyway）

4. 默认 Logger 打全 SQL
   → 生产配 Silent / Warn

5. 反射性能开销
   → 高 QPS 路径用 sqlx / 原生

6. Scan 大量行 OOM
   → Cursor 模式 db.FindInBatches
```

### 4.5 实战选型建议

```
极致性能（QPS > 5k）:
  → sqlx 或原生
  → 关键路径手写 SQL

业务为主（中后台 / 低中并发）:
  → GORM 全用
  → 维护成本低

混合（推荐）:
  CRUD     → GORM
  报表 / 复杂 SQL → sqlx
  超热点    → 原生 prepared
```

---

## 五、Prepared Statement 与 SQL 注入

### 5.1 为什么用 Prepared Statement

```
普通拼接:
  fmt.Sprintf("SELECT * FROM users WHERE id = %d", id)
  ❌ SQL 注入
  ❌ 不能复用执行计划

Prepared Statement:
  db.Prepare("SELECT * FROM users WHERE id = ?")
  ✓ 参数化（防注入）
  ✓ 服务端缓存执行计划（重复执行省解析）
```

### 5.2 database/sql 的 Prepared 行为

```go
// 隐式 prepare（每次自动 prepare 一次）
db.QueryContext(ctx, "SELECT ... WHERE id = ?", 1)
// 实际过程：
//   1. PREPARE statement
//   2. EXECUTE with args
//   3. CLOSE statement
// → 3 次 RTT 浪费

// 显式 prepare（复用）
stmt, _ := db.PrepareContext(ctx, "SELECT ... WHERE id = ?")
defer stmt.Close()
for _, id := range ids {
    stmt.QueryContext(ctx, id)  // 只 1 次 RTT
}
```

### 5.3 客户端拼参数（interpolateParams）

```go
// DSN 加 ?interpolateParams=true
// → driver 在客户端把参数拼成 SQL
// → 一次 RTT 完成，无服务端 prepare 开销
// → 仍然防 SQL 注入（驱动转义）

✓ 短查询性能更好
✗ 大量重复结构 SQL 失去 prepare 缓存优势
```

### 5.4 PrepareStmt 缓存（GORM）

```go
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    PrepareStmt: true,  // 全局开启 prepared 缓存
})
// → 每个 SQL 模板只 prepare 一次，复用
// → 高 QPS 场景必开
```

---

## 六、事务封装

### 6.1 标准事务模板

```go
func WithTx(ctx context.Context, db *sql.DB, fn func(*sql.Tx) error) (err error) {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer func() {
        if p := recover(); p != nil {
            _ = tx.Rollback()
            panic(p)  // 保留 panic
        } else if err != nil {
            _ = tx.Rollback()
        } else {
            err = tx.Commit()
        }
    }()
    err = fn(tx)
    return
}

// 使用
err := WithTx(ctx, db, func(tx *sql.Tx) error {
    if _, err := tx.ExecContext(ctx, "UPDATE ..."); err != nil {
        return err
    }
    if _, err := tx.ExecContext(ctx, "INSERT ..."); err != nil {
        return err
    }
    return nil
})
```

### 6.2 GORM 事务

```go
db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&order).Error; err != nil {
        return err  // 自动 rollback
    }
    if err := tx.Model(&Stock{}).
        Where("id=?", itemID).
        UpdateColumn("count", gorm.Expr("count - ?", 1)).Error; err != nil {
        return err
    }
    return nil  // 自动 commit
})
```

### 6.3 事务隔离级别

```go
// 显式指定（不指定 = MySQL 默认 RR）
tx, _ := db.BeginTx(ctx, &sql.TxOptions{
    Isolation: sql.LevelReadCommitted,  // RC
    ReadOnly:  false,
})
```

### 6.4 Save Point（嵌套事务模拟）

```go
// database/sql 不直接支持嵌套，用 SAVEPOINT
tx, _ := db.BeginTx(ctx, nil)
tx.ExecContext(ctx, "SAVEPOINT sp1")
_, err := tx.ExecContext(ctx, "...")
if err != nil {
    tx.ExecContext(ctx, "ROLLBACK TO SAVEPOINT sp1")
    // 主事务继续
}

// GORM 原生支持
db.Transaction(func(tx *gorm.DB) error {
    tx.Transaction(func(tx2 *gorm.DB) error {
        // 内层失败只回滚到 savepoint
        return tx2.Create(&xxx).Error
    })
    return nil
})
```

### 6.5 事务里的禁忌（生产事故来源）

```
❌ 调外部 RPC / HTTP
   tx.Begin()
   http.Post(...)        ← 几秒占行锁
   tx.Commit()
   → 死锁 + 连接池耗尽

❌ 调 Redis（可能慢）
   tx.Begin()
   redis.Get(...)        ← Redis 抖动 = DB 抖动
   tx.Commit()

❌ 长事务（> 5s）
   → undo 链长 + history list 涨 + 主从延迟

❌ 事务内 Sleep / 用户输入
   → 永远卡死

❌ 事务内 if/for 错误处理混乱
   → defer Rollback 兜底也救不了

✓ 黄金法则:
  事务 = 多个 DB 操作的原子组合
  快进快出，只做 DB
```

---

## 七、批量操作

### 7.1 批量 INSERT 三种方式

```go
// ❌ 慢：一行一次
for _, u := range users {
    db.Exec("INSERT INTO users (name) VALUES (?)", u.Name)
}
// 1000 行 = 1000 次 RTT

// ✅ 拼 VALUES（最快）
func bulkInsert(ctx context.Context, db *sql.DB, users []User) error {
    if len(users) == 0 {
        return nil
    }
    const batchSize = 500  // 每批 500 行
    for i := 0; i < len(users); i += batchSize {
        end := i + batchSize
        if end > len(users) {
            end = len(users)
        }
        batch := users[i:end]

        valueStrs := make([]string, 0, len(batch))
        valueArgs := make([]interface{}, 0, len(batch)*2)
        for _, u := range batch {
            valueStrs = append(valueStrs, "(?, ?)")
            valueArgs = append(valueArgs, u.Name, u.Age)
        }

        query := "INSERT INTO users (name, age) VALUES " + strings.Join(valueStrs, ",")
        if _, err := db.ExecContext(ctx, query, valueArgs...); err != nil {
            return err
        }
    }
    return nil
}

// ✅ GORM CreateInBatches
db.WithContext(ctx).CreateInBatches(users, 500)
```

### 7.2 批次大小怎么定

```
受 max_allowed_packet 限制:
  默认 4MB（5.7）/ 64MB（8.0）
  超过会报 "Got a packet bigger than 'max_allowed_packet'"

经验:
  小行（< 200B/行）→ 1000-5000 行/批
  中行（200B-1KB）→ 500-1000 行/批
  大行（> 1KB）→ 100-500 行/批

监控:
  - 单批 SQL 大小 < max_allowed_packet * 0.8
  - 单批耗时 < 200ms（避免长事务）
```

### 7.3 批量 UPDATE 技巧

```go
// ✅ CASE WHEN 一次更新多行不同值
query := `UPDATE users SET name = CASE id
    WHEN 1 THEN 'Alice'
    WHEN 2 THEN 'Bob'
    WHEN 3 THEN 'Charlie'
END WHERE id IN (1, 2, 3)`

// ✅ 临时表 JOIN
// 适合大批量（万行+）
```

### 7.4 ON DUPLICATE KEY UPDATE（幂等插入）

```go
// 存在则更新，不存在则插入
query := `INSERT INTO users (id, name, age) VALUES (?, ?, ?)
          ON DUPLICATE KEY UPDATE name = VALUES(name), age = VALUES(age)`
db.ExecContext(ctx, query, 1, "Alice", 25)

// 8.0+ 推荐写法（避免 deprecation）
query := `INSERT INTO users (id, name, age) VALUES (?, ?, ?) AS new
          ON DUPLICATE KEY UPDATE name = new.name, age = new.age`
```

---

## 八、读写分离封装

### 8.1 设计目标

```
- 写走主库
- 读默认走从库
- 写后立即读 → 强制主库
- 主从延迟过大 → 自动降级到主
- 业务无感知（注解 / Context 标记）
```

### 8.2 简单封装

```go
type DB struct {
    master *sqlx.DB
    slaves []*sqlx.DB
    rng    *rand.Rand
}

type ctxKey struct{}

// 业务标记强制读主
func WithForceMaster(ctx context.Context) context.Context {
    return context.WithValue(ctx, ctxKey{}, true)
}

func (d *DB) reader(ctx context.Context) *sqlx.DB {
    if v, _ := ctx.Value(ctxKey{}).(bool); v {
        return d.master
    }
    if len(d.slaves) == 0 {
        return d.master
    }
    return d.slaves[d.rng.Intn(len(d.slaves))]
}

func (d *DB) GetContext(ctx context.Context, dest interface{}, query string, args ...interface{}) error {
    return d.reader(ctx).GetContext(ctx, dest, query, args...)
}

func (d *DB) ExecContext(ctx context.Context, query string, args ...interface{}) (sql.Result, error) {
    return d.master.ExecContext(ctx, query, args...)
}
```

### 8.3 GORM DBResolver

```go
import "gorm.io/plugin/dbresolver"

db.Use(dbresolver.Register(dbresolver.Config{
    Sources:  []gorm.Dialector{mysql.Open(masterDSN)},
    Replicas: []gorm.Dialector{mysql.Open(slave1DSN), mysql.Open(slave2DSN)},
    Policy:   dbresolver.RandomPolicy{},
}).SetMaxIdleConns(10).SetMaxOpenConns(50))

// 强制主
db.Clauses(dbresolver.Write).Find(&user, id)
```

### 8.4 主从延迟监控 + 降级

```go
// 定时探测从库延迟
func monitorSlaveLag(slaves []*sqlx.DB) {
    ticker := time.NewTicker(5 * time.Second)
    for range ticker.C {
        for _, s := range slaves {
            var status struct {
                SecondsBehindMaster sql.NullInt64 `db:"Seconds_Behind_Master"`
            }
            row := s.QueryRowx("SHOW SLAVE STATUS")
            // ...解析
            if status.SecondsBehindMaster.Int64 > 5 {
                // 标记从库不可用，路由到主
                markSlaveUnhealthy(s)
            }
        }
    }
}
```

### 8.5 写后读一致性

```
方案 1: 强制读主（最安全）
  ctx = WithForceMaster(ctx)
  读关键数据走主

方案 2: 缓存最近写入
  写后把数据写本地 cache 5s
  5s 内读取 cache + 主库

方案 3: GTID 等待（强一致）
  写后记 GTID
  读时等从库回放到 GTID
  → 复杂，少用
```

---

## 九、错误处理与重试

### 9.1 区分错误类型

```go
import "github.com/go-sql-driver/mysql"

func handleErr(err error) {
    if err == sql.ErrNoRows {
        // 业务空：不算错误
        return
    }

    var mysqlErr *mysql.MySQLError
    if errors.As(err, &mysqlErr) {
        switch mysqlErr.Number {
        case 1062:  // Duplicate entry
            // 唯一键冲突：业务幂等可吃掉
        case 1213:  // Deadlock found
            // 死锁：可重试
        case 1205:  // Lock wait timeout
            // 锁等待超时：慎重试
        case 2006, 2013:  // MySQL gone away / Lost connection
            // 连接断：重试 + 触发重连
        }
    }

    if errors.Is(err, context.DeadlineExceeded) {
        // 业务超时
    }
    if errors.Is(err, driver.ErrBadConn) {
        // 坏连接（database/sql 会自动剔除）
    }
}
```

### 9.2 重试策略

```go
import "github.com/cenkalti/backoff/v4"

func RetryableExec(ctx context.Context, db *sql.DB, query string, args ...interface{}) error {
    bo := backoff.WithContext(
        backoff.NewExponentialBackOff(),
        ctx,
    )
    bo.(*backoff.ExponentialBackOff).MaxElapsedTime = 3 * time.Second

    return backoff.Retry(func() error {
        _, err := db.ExecContext(ctx, query, args...)
        if err == nil {
            return nil
        }

        // 只重试瞬时错误
        if isRetryable(err) {
            return err  // 触发重试
        }
        return backoff.Permanent(err)  // 不重试
    }, bo)
}

func isRetryable(err error) bool {
    if errors.Is(err, driver.ErrBadConn) {
        return true
    }
    var me *mysql.MySQLError
    if errors.As(err, &me) {
        return me.Number == 1213 || me.Number == 2006 || me.Number == 2013
    }
    return false
}
```

### 9.3 写操作重试要幂等！

```
❌ 危险:
  INSERT 失败 → 重试 → 第一次实际成功了 → 重复插入

✓ 必须:
  唯一业务键 + DB 唯一索引
  幂等 token + 应用层去重
  状态机前置条件:
    UPDATE orders SET status=2
    WHERE order_no=? AND status=1
    → 影响 0 行 = 已处理过
```

---

## 十、可观测：埋点与追踪

### 10.1 埋点指标必须打

| 指标 | 类型 | 标签 | 说明 |
| --- | --- | --- | --- |
| `db.query.duration` | Histogram | sql_type, table | 各 SQL 耗时 |
| `db.query.errors` | Counter | sql_type, code | 错误计数 |
| `db.pool.open` | Gauge | - | 当前连接 |
| `db.pool.in_use` | Gauge | - | 使用中 |
| `db.pool.wait_count` | Counter | - | 等待次数 |
| `db.pool.wait_duration` | Histogram | - | 等待时长 |
| `db.tx.duration` | Histogram | - | 事务耗时 |
| `db.slow.count` | Counter | sql_pattern | 慢 SQL 计数 |

### 10.2 用 sqlhooks 包一层

```go
import "github.com/qustavo/sqlhooks/v2"

type Hook struct{}

func (h *Hook) Before(ctx context.Context, query string, args ...interface{}) (context.Context, error) {
    return context.WithValue(ctx, "start", time.Now()), nil
}

func (h *Hook) After(ctx context.Context, query string, args ...interface{}) (context.Context, error) {
    start := ctx.Value("start").(time.Time)
    dur := time.Since(start)
    metrics.HistogramObserve("db.query.duration", dur.Seconds(),
        "sql_type", classifySQL(query))
    if dur > 100*time.Millisecond {
        log.Warn("slow sql", "dur", dur, "sql", query)
    }
    return ctx, nil
}

// 注册
sql.Register("mysqlWithHooks", sqlhooks.Wrap(&mysql.MySQLDriver{}, &Hook{}))
db, _ := sql.Open("mysqlWithHooks", dsn)
```

### 10.3 OpenTelemetry 集成

```go
import (
    "go.opentelemetry.io/contrib/instrumentation/database/sql/otelsql"
    semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
)

db, _ := otelsql.Open("mysql", dsn,
    otelsql.WithAttributes(semconv.DBSystemMySQL),
    otelsql.WithSpanOptions(otelsql.SpanOptions{
        Ping:                true,
        DisableErrSkip:      true,
        OmitConnectorConnect: true,
    }),
)
otelsql.RegisterDBStatsMetrics(db,
    otelsql.WithAttributes(semconv.DBSystemMySQL))
```

### 10.4 GORM Tracing

```go
import "gorm.io/plugin/opentelemetry/tracing"

db.Use(tracing.NewPlugin())  // 一行搞定 OTel
```

### 10.5 慢 SQL 应用侧拦截

```go
// GORM Logger
gormLogger := logger.New(
    log.New(os.Stdout, "\r\n", log.LstdFlags),
    logger.Config{
        SlowThreshold:             200 * time.Millisecond,
        LogLevel:                  logger.Warn,
        IgnoreRecordNotFoundError: true,
    },
)

db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    Logger: gormLogger,
})
```

---

## 十一、NULL 处理

### 11.1 三种方式

```go
// ① sql.Null* 类型
var name sql.NullString
row.Scan(&name)
if name.Valid {
    fmt.Println(name.String)
}

// ② 指针类型
var name *string
row.Scan(&name)
if name != nil {
    fmt.Println(*name)
}

// ③ COALESCE 兜底（推荐）
db.QueryRow("SELECT COALESCE(name, '') FROM users WHERE id = ?", 1).Scan(&name)
```

### 11.2 设计建议

```
✓ 字段尽量 NOT NULL DEFAULT ''/0
  - 索引可用
  - COUNT/SUM 不跳行
  - Go scan 不出错

✗ 真正语义需要 NULL（可选字段）才用
  - 用 sql.Null* 或指针
```

---

## 十二、Failover 与高可用

### 12.1 主从切换感知

```go
// MySQL Proxy (ProxySQL / MaxScale) 模式
// → 客户端连 Proxy，Proxy 处理 failover
// → 客户端无感知

// 客户端模式
// 用 go-sql-driver 的 multi-host DSN（不推荐，功能弱）
// 或用专业库 vitess / TiDB driver（自动 failover）
```

### 12.2 重试 + 重连

```go
// driver.ErrBadConn 时 database/sql 自动重新建连
// 但只对"无副作用"操作（Query / 第一次 Exec）

// 业务侧：写操作要在应用层判断 + 幂等重试
```

### 12.3 健康检查

```go
// 定时 Ping
go func() {
    ticker := time.NewTicker(30 * time.Second)
    for range ticker.C {
        ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
        if err := db.PingContext(ctx); err != nil {
            log.Error("db ping fail", "err", err)
            metrics.Counter("db.ping.fail").Inc()
        }
        cancel()
    }
}()
```

---

## 十三、典型线上事故

### 事故 1：连接池打满

```
现象:
  接口大面积超时
  db.Stats() WaitCount 暴涨
  应用日志 "context deadline exceeded"

排查:
  ① show processlist 看正在跑的 SQL
  ② 找到 Sleep 状态超长的连接 → 可能业务 rows 没 Close
  ③ 找到 Query 状态超长的 → 慢 SQL

根因:
  代码中漏了 defer rows.Close()
  rows 持有连接不归还

修复:
  - 紧急 KILL 超长连接
  - 加 defer rows.Close()
  - 加 lint 规则强制
  - 加 db.Stats 告警（WaitCount > 0）
```

### 事故 2：事务里调 Redis 导致死锁

```
现象:
  特定时段大量 UPDATE 死锁
  错误码 1213 飙升

排查:
  show engine innodb status\G
  → 死锁日志显示两个事务等同一行
  → 事务跑了 800ms（异常长）

根因:
  事务内调 Redis 拿配置
  Redis 偶发慢（200-500ms）
  → 事务持锁时间长 → 死锁概率激增

修复:
  - 把 Redis 调用移到事务外
  - 事务内只做 DB 操作
  - 设事务超时（500ms）
```

### 事故 3：Context 取消但 SQL 还在跑

```
现象:
  接口超时返回，但 DB CPU 持续 100%
  show processlist 看到大量 KILL 不掉的 query

根因:
  Context 触发 KILL QUERY
  但慢 SQL 在 fsync / 大量 IO 中
  KILL 信号要等检测点

修复:
  - 优化慢 SQL（加索引）
  - 应用 Context 超时 + 应用层硬切（不依赖 KILL）
  - 监控 KILL 失败率
```

### 事故 4：批量插入 packet too large

```
现象:
  大批量导入报 "Got a packet bigger than 'max_allowed_packet'"

根因:
  一次拼 5000 行 INSERT，SQL > 16MB

修复:
  - 拆批（每批 500-1000 行）
  - 调大 max_allowed_packet（治标）
  - 大数据用 LOAD DATA INFILE
```

### 事故 5：GORM AutoMigrate 把生产表改了

```
现象:
  生产环境上线后，发现某字段类型被改了
  数据出现异常

根因:
  开发环境的 db.AutoMigrate(&Model{}) 没去掉
  生产启动时执行

修复:
  - 立即从备份恢复
  - 删 AutoMigrate 代码
  - 用 goose / golang-migrate 做版本化迁移
  - 加 CI 检查（禁用 AutoMigrate）
```

---

## 十四、反模式清单

### 反模式 1：每请求新建 *sql.DB

```go
// ❌
func handler(w http.ResponseWriter, r *http.Request) {
    db, _ := sql.Open(...)  // 泄露连接池
    defer db.Close()
}

// ✓ 全局单例
var db *sql.DB
func init() { db, _ = sql.Open(...) }
```

### 反模式 2：拼接 SQL（注入）

```go
// ❌ SQL 注入
query := "SELECT * FROM users WHERE name = '" + name + "'"

// ✓ 参数化
db.Query("SELECT * FROM users WHERE name = ?", name)
```

### 反模式 3：事务里 RPC

```go
// ❌
tx.Begin()
http.Get(...)  // 几秒
tx.Commit()
```

### 反模式 4：忘记 rows.Close

```go
// ❌
rows, _ := db.Query(...)
for rows.Next() { ... }
// 忘 Close → 连接泄漏

// ✓
rows, _ := db.Query(...)
defer rows.Close()
```

### 反模式 5：GORM Save 全字段

```go
// ❌ 即使只想改一个字段，UPDATE 所有列
db.Save(&user)

// ✓
db.Model(&user).Update("name", "Alice")
db.Model(&user).Updates(map[string]interface{}{"name": "Alice", "age": 25})
```

### 反模式 6：不带 Context

```go
// ❌
db.Query("SELECT ...")

// ✓
db.QueryContext(ctx, "SELECT ...")
```

### 反模式 7：写操作无幂等就重试

```go
// ❌ 重试可能重复扣款
for i := 0; i < 3; i++ {
    _, err := db.Exec("UPDATE balance SET amount = amount - 100 WHERE user_id = ?", uid)
    if err == nil { break }
}

// ✓ CAS / 状态机 + 唯一业务 ID
```

### 反模式 8：长事务

```go
// ❌
tx.Begin()
for _, item := range items {  // 1000 行
    tx.Exec("INSERT ...")
}
time.Sleep(...)
tx.Commit()  // 持锁几分钟

// ✓ 分批短事务
```

### 反模式 9：SELECT *

```go
// ❌ 字段多 / 大 BLOB / 网络浪费 / 索引覆盖失效
SELECT * FROM users

// ✓ 显式列
SELECT id, name FROM users
```

### 反模式 10：FOR UPDATE 没索引

```go
// ❌ 升级表锁
SELECT * FROM users WHERE name = 'Alice' FOR UPDATE  // name 无索引

// ✓ WHERE 字段必须有索引
```

---

## 十五、推荐工程模板

### 15.1 项目结构

```
/internal
  /repository
    base.go          # 公共：连接池 / Tx 装饰器 / 埋点
    user_repo.go     # 用户表仓储
    order_repo.go    # 订单表仓储
  /db
    mysql.go         # 初始化 *sql.DB / *gorm.DB
    migrate.go       # 迁移脚本
```

### 15.2 仓储模板

```go
type UserRepo struct {
    db *sqlx.DB
}

func (r *UserRepo) GetByID(ctx context.Context, id int64) (*User, error) {
    var u User
    err := r.db.GetContext(ctx, &u, "SELECT id, name, age FROM users WHERE id = ?", id)
    if errors.Is(err, sql.ErrNoRows) {
        return nil, ErrUserNotFound
    }
    if err != nil {
        return nil, fmt.Errorf("get user %d: %w", id, err)
    }
    return &u, nil
}

func (r *UserRepo) Create(ctx context.Context, u *User) error {
    res, err := r.db.NamedExecContext(ctx,
        `INSERT INTO users (name, age) VALUES (:name, :age)`,
        u,
    )
    if err != nil {
        return err
    }
    u.ID, _ = res.LastInsertId()
    return nil
}
```

### 15.3 配置模板

```go
type Config struct {
    DSN              string        `yaml:"dsn"`
    MaxOpenConns     int           `yaml:"max_open_conns" default:"50"`
    MaxIdleConns     int           `yaml:"max_idle_conns" default:"10"`
    ConnMaxLifetime  time.Duration `yaml:"conn_max_lifetime" default:"30m"`
    ConnMaxIdleTime  time.Duration `yaml:"conn_max_idle_time" default:"10m"`
    SlowThreshold    time.Duration `yaml:"slow_threshold" default:"200ms"`
}

func New(cfg Config) (*sql.DB, error) {
    db, err := sql.Open("mysql", cfg.DSN)
    if err != nil {
        return nil, err
    }
    db.SetMaxOpenConns(cfg.MaxOpenConns)
    db.SetMaxIdleConns(cfg.MaxIdleConns)
    db.SetConnMaxLifetime(cfg.ConnMaxLifetime)
    db.SetConnMaxIdleTime(cfg.ConnMaxIdleTime)

    if err := db.Ping(); err != nil {
        return nil, err
    }
    go monitorPoolStats(db)
    return db, nil
}
```

---

## 十六、面试表达模板

### 16.1 "你们 MySQL 怎么用的"

```
我们在 SDK 层做了这几件事：

1. 连接池: MaxOpenConns 按 QPS × RT 估算，监控 db.Stats() 调整
2. 超时: DSN 配 dial/read/write，业务用 Context 三层
3. ORM: 按场景选——CRUD 用 GORM，热路径 sqlx
4. 事务: 装饰器封装 Begin/Commit/Rollback，
        recover panic + 禁止事务内 RPC
5. 批量: VALUES 拼 + 分批，单批 < max_allowed_packet
6. 读写分离: dbresolver 路由，写后强制读主，监控延迟
7. 埋点: sqlhooks + OpenTelemetry，所有 SQL P99 / 错误 / 慢日志
8. 重试: 区分错误类型，写操作必须幂等才重试
9. Failover: 主从延迟监控，超阈值降级
10. 反模式: Lint + Code Review 拦截（事务内 RPC / SELECT * / 无 Context）
```

### 16.2 "DB 挂了你们的服务能跑吗"

```
分情况:
- 主库挂: 切流到备份（DBA 切换 30s）
  期间写请求失败，业务限流 + 排队
  读请求降级到从库 + 缓存

- 从库挂: 自动剔除该从库
  读请求走其他从 / 主库
  无业务感知

- 网络抖动: 重试 + 熔断 + 降级
  关键查询有缓存兜底
  非关键直接返回默认值

不能完全无感知，但能做到分钟级恢复 + 业务核心可用
```

---

## 十七、面试加分点

- **MaxOpenConns 按 QPS × RT 公式算** 不是拍脑袋
- **db.Stats() WaitCount / WaitDuration** 监控指标
- **ConnMaxLifetime < MySQL wait_timeout** 防被动断
- **Context 超时 ≠ KILL QUERY 立刻生效**
- **事务里禁止 RPC / Redis / Sleep**
- **GORM SkipDefaultTransaction / PrepareStmt** 性能优化
- **批量 INSERT 拼 VALUES + 分批**
- **dbresolver 读写分离 + 强制主**
- **sqlhooks / otelsql 埋点**
- **错误分类重试**（1213 死锁 / 1062 重复 / driver.ErrBadConn）
- **写操作重试必须幂等**（DB 唯一索引 + 状态机）
- **NULL 字段统一 NOT NULL DEFAULT**
- **interpolateParams=true** 减少 RTT
- **AutoMigrate 禁生产** 用 goose 做迁移
- **典型事故案例**（连接池打满 / 事务死锁 / KILL 失效）

---

## 十八、关联阅读

```
本目录:
- 03-transaction-lock-mvcc.md  事务并发原理
- 06-slow-sql-optimization.md  慢 SQL
- 09-distributed-transaction.md 分布式事务
- 10-production-cases.md       8 个真实案例
- 14-explain-optimizer.md      EXPLAIN
- 17-consistency-reconciliation.md 对账
- 20-mysql-senior-answers.md   答题模板

跨模块:
- 04-redis/20-go-redis-best-practices.md  Redis Go 实战（对偶）
- 06-distributed/06-rate-limit-circuit.md 限流熔断
- 13-engineering/04-observability-integration.md 可观测
```
