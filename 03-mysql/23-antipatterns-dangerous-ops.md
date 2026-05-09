# MySQL 反模式与危险操作

> 把散落在各篇的"坑"集中起来，按危险等级排序。**生产事故的来源 90% 是这里面的**。
>
> 与 [04-redis/19-redis-design-tradeoffs.md 第六节](../04-redis/19-redis-design-tradeoffs.md) 对偶。

---

## 一、危险分级

```
P0 - 立刻拖垮线上 / 资损:
  大事务 / 长事务 / 误删数据 / DDL 锁雪崩 / FOR UPDATE 全表

P1 - 性能严重退化:
  隐式类型转换 / NULL 索引坑 / 函数索引失效 / 自增锁 / SELECT *

P2 - 设计债 / 长期问题:
  外键 / 超宽表 / VARCHAR 滥用 / 无 schema 演进
```

---

## 二、P0 危险操作

### 2.1 大事务

```sql
-- ❌ 危险：一个事务改 100w 行
BEGIN;
DELETE FROM orders WHERE created_at < '2024-01-01';  -- 100w 行
COMMIT;
```

**后果**：

```
1. undo log 暴涨 → ibdata 膨胀 → 磁盘满
2. 事务持锁久 → 阻塞其他业务
3. 主从延迟（binlog 大 + 从库重放慢）
4. 崩溃恢复时间长（要 rollback 100w 行）
5. purge 跟不上 → history list length 涨爆
```

**正确做法**：

```sql
-- ✓ 拆批，每批 1000，循环
DELETE FROM orders WHERE created_at < '2024-01-01' LIMIT 1000;
-- 应用层循环 + sleep 间隔，避免主从延迟
```

```go
for {
    res, err := db.ExecContext(ctx,
        "DELETE FROM orders WHERE created_at < ? LIMIT 1000",
        cutoff)
    if err != nil { return err }
    n, _ := res.RowsAffected()
    if n == 0 { break }
    time.Sleep(100 * time.Millisecond)  // 给从库喘息
}
```

### 2.2 长事务

```sql
-- ❌ 事务跑 5 分钟以上
BEGIN;
SELECT ...
-- 应用做大量计算 / 调外部
SELECT ...
COMMIT;
```

**后果**：

```
1. undo 链长，purge 不能回收
   → history list length 涨到几千万
2. MVCC 读穿透多层 undo，性能差
3. 主从延迟（持锁时间长）
4. ibdata 涨爆（无独立 undo 表空间时）
```

**排查长事务**：

```sql
-- 查执行 > 60s 的事务
SELECT
    trx_id,
    trx_started,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS duration_sec,
    trx_query,
    trx_rows_locked
FROM information_schema.INNODB_TRX
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 60
ORDER BY trx_started;

-- KILL 掉
KILL <thread_id>;
```

**预防**：

```
- 应用层事务超时（设 SET innodb_lock_wait_timeout = 5;）
- 监控 INNODB_TRX 表（> 1 分钟告警）
- 监控 history list length (> 100w 告警)
- 事务内禁止 RPC / Sleep / 外部调用
```

### 2.3 DDL 锁雪崩 (MDL Lock Storm)

```sql
-- ❌ 业务高峰期跑 ALTER
ALTER TABLE orders ADD COLUMN remark VARCHAR(255);
```

**后果链**：

```
T+0: ALTER 开始，先拿 MDL 写锁
T+0: 当前有长事务持 MDL 读锁 → ALTER 阻塞排队
T+1s: 新请求来 → 拿 MDL 读锁
       但写锁在排队，新读锁也排队
T+5s: 连接池被排队请求打满
T+10s: 服务大面积超时

→ 一个 ALTER 拖垮整个服务
```

**应急止血**：

```sql
-- 1. 找出阻塞 ALTER 的长事务
SELECT * FROM performance_schema.metadata_locks
WHERE OBJECT_TYPE = 'TABLE' AND OBJECT_NAME = 'orders';

-- 2. KILL 阻塞事务
KILL <blocking_thread_id>;

-- 3. 或 KILL ALTER
KILL <alter_thread_id>;
```

**预防**：

```sql
-- 1. ALTER 加超时
SET lock_wait_timeout = 5;
ALTER TABLE ... ;

-- 2. 业务低峰期
-- 3. gh-ost / pt-osc 大表必用
-- 4. 8.0 用 INSTANT 算法（仅适用部分操作）
ALTER TABLE orders ADD COLUMN x INT, ALGORITHM=INSTANT;
```

### 2.4 误删数据 / 误改数据

```sql
-- ❌ WHERE 条件错误 / 漏 WHERE
DELETE FROM orders;            -- 漏 WHERE 删全表
UPDATE users SET status = 0;   -- 漏 WHERE 改全表
DELETE FROM orders WHERE id = ANY(...);  -- 子查询出错变全删
```

**后果**：业务数据丢失，可能资损。

**预防**：

```
1. 启用安全模式（生产强烈推荐）:
   --safe-updates
   或 SET sql_safe_updates = 1;
   → 禁止无 WHERE / 无 LIMIT 的 UPDATE/DELETE

2. 客户端工具防护:
   MySQL Workbench: Edit → Preferences → SQL Editor → Safe Updates
   DBeaver: Settings → Database → SQL Editor → Confirm dangerous SQL

3. 流程:
   生产变更必须经过 SQL Review
   高危 SQL 提工单 + DBA 审核
   备份验证 + 灰度

4. 高危操作前先 SELECT 确认:
   SELECT * FROM orders WHERE status = 1 LIMIT 10;
   -- 看清楚再写 DELETE
```

**误删恢复**：

```bash
# binlog 闪回
mysqlbinlog --start-datetime='...' --stop-datetime='...' \
    --base64-output=DECODE-ROWS -v \
    /var/log/mysql/binlog.000001 \
    | python3 binlog_flashback.py > rollback.sql
mysql -u root -p < rollback.sql

# 工具:
binlog2sql (https://github.com/danfengcao/binlog2sql)
my2sql (Go 版)
```

### 2.5 FOR UPDATE 全表锁

```sql
-- ❌ 索引失效，FOR UPDATE 升级表锁
SELECT * FROM orders WHERE name = 'xxx' FOR UPDATE;  -- name 无索引
```

**后果**：

```
当 WHERE 字段无索引:
  InnoDB 退化为全表加 X 锁
  → 所有其他事务被阻塞
  → 表级死锁概率激增
  → 业务大面积阻塞
```

**正确做法**：

```sql
-- ✓ WHERE 必须命中索引
SELECT * FROM orders WHERE id = 1 FOR UPDATE;       -- 主键
SELECT * FROM orders WHERE status = 1 FOR UPDATE;   -- 索引列

-- 或换乐观锁
UPDATE orders SET status = 2, version = version + 1
WHERE id = ? AND status = 1 AND version = ?;
```

---

## 三、P1 性能反模式

### 3.1 隐式类型转换

```sql
-- ❌ phone 是 VARCHAR，传整数 → 全表
SELECT * FROM users WHERE phone = 13800000000;

-- 优化器看到类型不匹配:
--   每行 phone CAST 成数字比对
--   索引完全失效

-- ✓ 类型匹配
SELECT * FROM users WHERE phone = '13800000000';
```

**Go 中常见错误**：

```go
// ❌ 把 int 传给 varchar 字段
db.Query("SELECT * FROM users WHERE phone = ?", 13800000000)
// 实际上 driver 会拼成 "13800000000"，但应用代码可能写错

// ✓ 显式
db.Query("SELECT * FROM users WHERE phone = ?", "13800000000")
```

**字符集不匹配也会触发**：

```sql
-- 表 utf8，连接 utf8mb4 → JOIN 时转换索引失效
-- 解决：统一字符集 utf8mb4
```

### 3.2 函数 / 表达式索引失效

```sql
-- ❌ 索引失效
WHERE DATE(created_at) = '2026-05-09'
WHERE YEAR(created_at) = 2026
WHERE LOWER(name) = 'alice'
WHERE id + 1 = 100
WHERE SUBSTR(phone, 1, 3) = '138'

-- ✓ 改写
WHERE created_at >= '2026-05-09 00:00:00' AND created_at < '2026-05-10 00:00:00'
WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'
WHERE id = 99
WHERE phone LIKE '138%'

-- ✓ 或用 8.0 函数索引
ALTER TABLE users ADD INDEX idx_lower_name ((LOWER(name)));
WHERE LOWER(name) = 'alice'  -- 命中
```

### 3.3 NULL 索引坑

```sql
-- ❌ NULL 比较坑
WHERE col = NULL                -- 永远不等
WHERE col != NULL               -- 永远不等
WHERE col IN (1, 2, NULL)       -- NULL 不参与

-- ✓ 用 IS NULL / IS NOT NULL
WHERE col IS NULL

-- 索引可用（5.7+），但有性能影响:
-- COUNT(col) 跳过 NULL
-- ORDER BY 默认 NULL 在前
```

**字段尽量 NOT NULL**：

```sql
-- ❌
CREATE TABLE users (
    age INT,           -- 可 NULL
    name VARCHAR(50)   -- 可 NULL
);

-- ✓
CREATE TABLE users (
    age INT NOT NULL DEFAULT 0,
    name VARCHAR(50) NOT NULL DEFAULT ''
);
```

### 3.4 OR 条件索引失效

```sql
-- ❌ 一边没索引 → 全表
WHERE (a = 1 OR b = 2)  -- a 有索引 b 没

-- ✓ 都加索引 + UNION ALL
SELECT ... WHERE a = 1
UNION ALL
SELECT ... WHERE b = 2 AND a != 1  -- 去重
```

### 3.5 联合索引顺序错

```sql
-- 索引 (a, b, c)

WHERE a = 1                      -- ✓ 用 a
WHERE a = 1 AND b = 2            -- ✓ 用 ab
WHERE a = 1 AND b = 2 AND c = 3  -- ✓ 用 abc
WHERE b = 2                      -- ❌ 不用
WHERE b = 2 AND c = 3            -- ❌ 不用
WHERE a = 1 AND c = 3            -- ⚠️ 只用 a（c 用不上）
```

**设计原则**：

```
联合索引列顺序:
  1. 等值查询字段在前
  2. 范围查询字段在后
  3. 高选择性在前

错误:
  CREATE INDEX idx ON t(create_time, status, user_id);
  WHERE status = 1 AND user_id = 100 AND create_time > ...
  → 只用上 create_time，浪费

正确:
  CREATE INDEX idx ON t(status, user_id, create_time);
```

### 3.6 自增锁竞争

```sql
-- 高并发 INSERT
INSERT INTO orders (...) VALUES (...);
INSERT INTO orders (...) VALUES (...);
-- ...
```

**自增锁模式**：

```
innodb_autoinc_lock_mode:
  0 (TRADITIONAL):
    每次 INSERT 拿表级自增锁，等本语句完成
    → 串行，最差性能

  1 (CONSECUTIVE 默认):
    简单 INSERT 立刻分配 + 释放锁
    批量 INSERT 拿锁直到完成（保证连续）
    → 性能 OK，主从一致（STATEMENT 格式）

  2 (INTERLEAVED):
    所有 INSERT 立刻分配，ID 不连续
    → 最快，但 STATEMENT 复制可能不一致
    → 必须 ROW binlog
```

**建议**：

```
binlog = ROW: innodb_autoinc_lock_mode = 2 (推荐 8.0 默认)
binlog = STATEMENT: innodb_autoinc_lock_mode = 1
```

### 3.7 SELECT * 滥用

```sql
-- ❌
SELECT * FROM users WHERE id = 1;
-- 后果:
--   1. 网络传输浪费（特别是有 BLOB / TEXT）
--   2. Buffer Pool 装载更多无用列
--   3. 覆盖索引失效（索引无法包含所有列）
--   4. 加字段后业务可能 panic（顺序变了 / Scan 错位）

-- ✓
SELECT id, name, age FROM users WHERE id = 1;
```

### 3.8 IN 列表过长

```sql
-- ❌ IN 几千个值
SELECT * FROM orders WHERE id IN (1, 2, 3, ..., 5000);

-- 后果:
--   优化器评估慢
--   SQL 长度大
--   网络传输慢
--   MAX_ALLOWED_PACKET 限制

-- ✓ 拆批 (每批 500-1000)
-- ✓ 临时表 + JOIN
CREATE TEMPORARY TABLE tmp_ids (id BIGINT PRIMARY KEY);
INSERT INTO tmp_ids VALUES (...);
SELECT * FROM orders o JOIN tmp_ids t ON o.id = t.id;
```

### 3.9 深分页 LIMIT N, M

```sql
-- ❌ LIMIT 1000000, 10
-- 实际扫 1000010 行，丢弃前 100w
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;

-- ✓ 游标分页
SELECT * FROM orders WHERE id > 1000000 ORDER BY id LIMIT 10;

-- ✓ 子查询定位
SELECT * FROM orders WHERE id IN (
    SELECT id FROM orders ORDER BY id LIMIT 1000000, 10
);
```

### 3.10 ORDER BY + LIMIT 没索引

```sql
-- ❌
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;
-- 如果 created_at 无索引 → filesort 全部数据

-- ✓
ALTER TABLE orders ADD INDEX idx_created (created_at);
-- 或覆盖索引
ALTER TABLE orders ADD INDEX idx_user_created_status (user_id, created_at, status);
```

---

## 四、P2 设计反模式

### 4.1 外键约束

```sql
-- ❌ 互联网业务不推荐
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

**为什么互联网普遍关闭**：

```
1. 写性能差（每次写要校验外键，加锁）
2. 分库分表后失效（跨库无法外键）
3. 数据迁移困难（要先迁主表）
4. 不能水平扩展
5. 删除主表行被外键阻塞

✓ 应用层维护关系
✓ 唯一索引 + 业务校验
```

### 4.2 超宽表

```sql
-- ❌ user 表 200+ 字段
CREATE TABLE users (
    id BIGINT,
    name VARCHAR(50),
    age INT,
    -- ... 200 个字段
    profile_json JSON,
    bio TEXT,
    avatar BLOB
);
```

**后果**：

```
1. 单行大 → 单 page 装不下几行 → IO 多
2. Buffer Pool 装载效率低
3. 全表扫描慢
4. 备份慢
5. 字段经常迁移 → DDL 频繁
```

**正确做法**：

```sql
-- ✓ 按冷热拆分
CREATE TABLE users (             -- 基础表（热）
    id BIGINT PRIMARY KEY,
    name VARCHAR(50),
    age INT,
    status TINYINT,
    created_at DATETIME
);

CREATE TABLE user_profiles (      -- 扩展表（冷）
    user_id BIGINT PRIMARY KEY,
    profile_json JSON,
    bio TEXT,
    avatar_url VARCHAR(255)       -- BLOB 走 OSS
);
```

### 4.3 VARCHAR 长度滥用

```sql
-- ❌ 所有字符串都用 VARCHAR(255)
name VARCHAR(255)
title VARCHAR(255)
phone VARCHAR(255)

-- 问题:
-- 1. utf8mb4 下 VARCHAR(255) 占 4*255+2 = 1022B 索引空间
-- 2. 索引前缀长度限制 (3072B InnoDB)
-- 3. 行存储预算紧
```

**正确**：

```sql
-- ✓ 按业务实际长度
name VARCHAR(50)
title VARCHAR(100)
phone CHAR(11)
```

### 4.4 时间字段类型坑

```sql
-- ❌ TIMESTAMP 时区坑
created_at TIMESTAMP

-- 问题:
--   主从时区不一致 → 数据不一致
--   Y2038 问题（2038-01-19 之后溢出）

-- ✓ DATETIME（无时区转换）
created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
-- 应用层统一 UTC 或 Asia/Shanghai
```

### 4.5 浮点存金额

```sql
-- ❌ FLOAT/DOUBLE 不精确
amount FLOAT       -- 0.1 + 0.2 = 0.30000000000000004
balance DOUBLE     -- 累加误差

-- ✓ DECIMAL 或 BIGINT 存"分"
amount DECIMAL(20, 4)   -- 精确小数
balance_fen BIGINT      -- 存分（更快）
```

### 4.6 utf8 字符集

```sql
-- ❌ utf8 只支持 3 字节
CREATE TABLE messages (
    content VARCHAR(1000)
) CHARSET=utf8;

-- 存 emoji 报错: Incorrect string value
-- 截断 4 字节字符

-- ✓ utf8mb4
CREATE TABLE messages (
    content VARCHAR(1000)
) CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

### 4.7 索引过多

```sql
-- ❌ 每个查询加一个索引
CREATE INDEX idx_a ON t(a);
CREATE INDEX idx_b ON t(b);
CREATE INDEX idx_c ON t(c);
CREATE INDEX idx_a_b ON t(a, b);
CREATE INDEX idx_a_c ON t(a, c);
-- ... 10+ 索引

-- 后果:
-- 1. 写性能下降（每次 INSERT/UPDATE 要更新所有索引）
-- 2. 存储翻倍
-- 3. 优化器选错索引概率上升
-- 4. Buffer Pool 浪费

-- ✓ 联合索引 + 覆盖索引设计
-- 通常一个表 3-5 个索引足够
```

**索引精简原则**：

```
1. 联合索引覆盖多个查询场景
2. 主键 + 业务唯一键 + 2-3 个常用查询索引
3. 定期分析 sys.schema_unused_indexes 删未用索引
```

### 4.8 ENUM 滥用

```sql
-- ❌ ENUM 难维护
status ENUM('PENDING', 'PAID', 'SHIPPED', 'COMPLETED', 'CANCELLED')

-- 问题:
-- 1. 加状态要 ALTER TABLE
-- 2. 排序按定义顺序，不直观
-- 3. 跨语言不友好

-- ✓ TINYINT + 应用层映射
status TINYINT NOT NULL DEFAULT 0
-- Go 端定义常量映射
```

### 4.9 BLOB / TEXT 大字段

```sql
-- ❌
content TEXT  -- 几 MB 的文章内容

-- 后果:
-- 1. 行外存储 → IO 多
-- 2. 备份慢
-- 3. 主从复制慢
-- 4. SELECT * 时大量带宽

-- ✓ 选择
小 (<1KB):  VARCHAR
中 (<10KB): TEXT (但避免 SELECT *)
大 (>10KB): OSS 存内容，DB 存 URL
```

### 4.10 主键 UUID

```sql
-- ❌ 无序 UUID 主键
CREATE TABLE orders (
    id CHAR(36) PRIMARY KEY,  -- UUID
    ...
);

-- 问题:
-- 1. 36 字节大（vs BIGINT 8 字节）
-- 2. 二级索引 = key + 主键 → 二级索引也很大
-- 3. 随机插入 → 频繁页分裂
-- 4. Buffer Pool 命中率低

-- ✓ 自增 BIGINT 或 雪花 ID
id BIGINT PRIMARY KEY AUTO_INCREMENT
-- 或
id BIGINT PRIMARY KEY  -- 应用层雪花算法
```

---

## 五、SQL 编写反模式（速查）

```sql
-- ❌ 这些都要避免

-- 1. SELECT *
SELECT * FROM users;

-- 2. 无 WHERE / LIMIT 的 UPDATE/DELETE
DELETE FROM orders;

-- 3. 隐式类型转换
WHERE phone = 138...

-- 4. 函数包索引列
WHERE DATE(created_at) = ...

-- 5. 前缀模糊
WHERE name LIKE '%xxx'

-- 6. NOT IN / != （索引失效概率大）
WHERE status NOT IN (1, 2)

-- 7. OR 条件，一边没索引
WHERE a = 1 OR b = 2

-- 8. 子查询代替 JOIN（5.7+ 优化器改进，但仍要警惕）
WHERE id IN (SELECT id FROM ...)

-- 9. 多表 JOIN 无索引
SELECT * FROM a JOIN b ON a.x = b.x  -- a.x b.x 无索引

-- 10. 深分页
LIMIT 100000, 10

-- 11. ORDER BY RAND()
SELECT * FROM products ORDER BY RAND() LIMIT 10  -- 全表排序

-- 12. COUNT(列) vs COUNT(*)
SELECT COUNT(name) FROM users  -- 跳 NULL，不一定快
SELECT COUNT(*) FROM users     -- MySQL 优化过

-- 13. GROUP BY 无索引
GROUP BY 多列 + 没覆盖索引

-- 14. UNION 而非 UNION ALL
UNION 自动去重，要排序，慢
明确不重复用 UNION ALL
```

---

## 六、运维操作反模式

### 6.1 业务高峰期 DDL

```
❌ 白天加索引 / 改字段
✓ 凌晨低峰 + gh-ost
```

### 6.2 批量导入未关索引 / 双写

```sql
-- ❌ 直接导入 1000w 行
LOAD DATA INFILE ...

-- ✓ 先关二级索引 + binlog
SET sql_log_bin = 0;
ALTER TABLE t DROP INDEX ...;
LOAD DATA INFILE ...;
ALTER TABLE t ADD INDEX ...;
SET sql_log_bin = 1;
```

### 6.3 主库直接跑慢 SQL

```
❌ 主库跑分析 / 报表
✓ 从库专门跑 / 读写分离
✓ binlog → 数仓
```

### 6.4 不备份就 DROP / TRUNCATE

```
❌ DROP TABLE / TRUNCATE 不可恢复
✓ 先 RENAME 到 _bak 表 + 观察 7 天
✓ DROP 前确认有备份 + 测试恢复
```

### 6.5 修改 my.cnf 不重启就改 Global

```
❌ 改 my.cnf 文件不重启 → 重启后丢
✓ SET GLOBAL ... + 同步改 my.cnf
```

### 6.6 不限制结果集

```
❌ 直接 SELECT 几百万行到内存
✓ Cursor / LIMIT 分批
```

---

## 七、Go 代码反模式

详见 [16-go-mysql-practice.md 第十四节](16-go-mysql-practice.md)。

```
❌ 每请求 sql.Open
❌ 拼接 SQL 不参数化
❌ 事务里调 RPC / Redis
❌ 不带 Context
❌ 写操作无幂等就重试
❌ rows 不 Close
❌ NULL 字段直接 Scan
❌ GORM Save 全字段
❌ AutoMigrate 上生产
❌ 大批量不拆批
```

---

## 八、防御措施

### 8.1 强制规范

```
代码层:
  □ Lint: golangci-lint + 自定义规则
  □ Code Review 必检查 SQL
  □ 禁止 SELECT * / 无 WHERE / 不带 Context

DB 层:
  □ 开启 sql_safe_updates
  □ 设置 max_execution_time（5.7+ 防慢查询雪崩）
  □ 设置 lock_wait_timeout = 5（DDL 防 MDL 雪崩）

监控:
  □ 长事务告警（> 60s）
  □ MDL 等待告警
  □ history list length 告警
  □ 慢 SQL 告警（> 1s）
```

### 8.2 SQL Review 流程

```
高危 SQL 必须走流程:
  □ DDL（任何 ALTER）
  □ DML 修改 > 1000 行
  □ 新建索引 / 删索引
  □ 改字段类型 / 长度
  □ 数据迁移

流程:
  开发提工单 → DBA 评估 → 走灰度 → 生产执行
  → 备份 + 回滚预案
```

### 8.3 工具

```bash
# SQL 静态检查
soar     # 小米开源
sqlcheck # SQL 反模式检测

# Schema 演进
goose / golang-migrate / atlas

# 大表 DDL
gh-ost / pt-online-schema-change

# binlog 闪回
binlog2sql / my2sql

# 死锁分析
pt-deadlock-logger

# 慢日志
pt-query-digest
```

---

## 九、面试常被问到的"踩坑案例"

### 案例 1：误删全表

```
背景: 凌晨手动清理测试数据
SQL: DELETE FROM users WHERE test = 1;
事故: 漏了 LIMIT，删了 50w 真实用户

恢复:
  ① 立即停服防写入
  ② binlog 闪回（binlog2sql）
  ③ 30 分钟内恢复

复盘:
  - 生产 SQL Review 强制
  - sql_safe_updates 开启
  - 高危 SQL 工单 + DBA 审核
```

### 案例 2：MDL 雪崩

```
背景: 业务高峰期加字段
SQL: ALTER TABLE orders ADD COLUMN ...;
事故:
  T+0: ALTER 开始
  T+0: 长事务阻塞 ALTER
  T+5s: 后续读写全部排队
  T+30s: 服务大面积超时
  T+1min: KILL ALTER 后恢复

复盘:
  - 大表 DDL 必用 gh-ost
  - 设 lock_wait_timeout = 5
  - 业务低峰期变更
```

### 案例 3：自增 ID 跳号 / 重复

```
背景: 主从切换 + 自增模式 = 2
事故: 切换后部分订单 ID 重复

复盘:
  - innodb_autoinc_lock_mode 主从必须一致
  - 业务订单号用雪花算法（不依赖自增）
  - 或开启 GTID
```

### 案例 4：金额计算错乱

```
背景: 用 FLOAT 存余额
SQL: UPDATE users SET balance = balance + 0.1;
事故: 累加 100 次 ≠ 10.0（浮点误差）

修复:
  - 改 DECIMAL(20, 4)
  - 历史数据校正
```

### 案例 5：长事务把 ibdata 撑爆

```
背景: 开发误操作开了事务没提交
事故:
  - 跑了 6h
  - undo 暴涨 200GB
  - ibdata 满 → 业务报错

恢复:
  - KILL 长事务
  - 等 purge 慢慢回收（不会缩 ibdata 文件）
  - 重做实例迁移到 innodb_undo_tablespaces 模式

复盘:
  - INNODB_TRX 监控告警
  - 应用层事务超时
  - 独立 undo 表空间（5.6+）
```

---

## 十、面试加分点

- **大事务 / 长事务 / DDL 锁** 三大 P0
- **能讲 MDL 雪崩** 完整链路
- **sql_safe_updates / lock_wait_timeout** 防御参数
- **gh-ost 必用** 大表 DDL
- **binlog 闪回工具**（binlog2sql / my2sql）
- **隐式类型转换** 索引失效
- **联合索引顺序** 等值在前 / 范围在后
- **NOT NULL DEFAULT** 设计原则
- **utf8mb4 必选**
- **金额必 DECIMAL / 时间必 DATETIME**
- **互联网外键关闭**
- **超宽表拆分** 冷热分离
- **索引 3-5 个够用** 不滥加
- **真实事故 STAR + 5 Whys**

---

## 十一、关联阅读

```
本目录:
- 02-index.md                索引原理
- 03-transaction-lock-mvcc.md 事务并发
- 06-slow-sql-optimization.md 慢 SQL
- 10-production-cases.md     生产案例
- 14-explain-optimizer.md    EXPLAIN
- 15-online-ddl-mdl.md       Online DDL / MDL
- 16-go-mysql-practice.md    Go 实战
- 18-field-types-charset-time.md 字段类型
- 20-mysql-senior-answers.md 答题模板
- 21-mysql-design-tradeoffs.md 设计边界
- 22-innodb-internals.md     源码深水

跨模块:
- 04-redis/19-redis-design-tradeoffs.md  Redis 反模式（对偶）
```
