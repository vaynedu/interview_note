# MySQL 资深面试题（20 题）

> 架构 / 索引 / 事务 / 锁 / MVCC / 日志 / 复制 / 慢 SQL / 分库分表
>
> 格式：题目 / 标准答案 / 易错点 / 追问点 / 背诵版

## 目录

1. [MySQL 整体架构？](#q1)
2. [InnoDB vs MyISAM？](#q2)
3. [B+ 树索引为什么用 B+ 不用 B / 红黑树 / Hash？](#q3)
4. [聚簇索引 vs 二级索引？回表是什么？](#q4)
5. [覆盖索引 / 最左前缀 / 索引下推？](#q5)
6. [事务隔离级别？幻读怎么解？](#q6)
7. [MVCC 原理？快照读 vs 当前读？](#q7)
8. [行锁 / 间隙锁 / Next-Key Lock？](#q8)
9. [死锁怎么排查？](#q9)
10. [redo log / undo log / binlog 区别？](#q10)
11. [两阶段提交？为什么需要？](#q11)
12. [主从复制原理？延迟怎么处理？](#q12)
13. [半同步复制 vs 异步复制 vs 全同步？](#q13)
14. [慢 SQL 怎么排查和优化？](#q14)
15. [explain 关键字段？type / key / Extra？](#q15)
16. [InnoDB Buffer Pool 是什么？](#q16)
17. [Online DDL / MDL 锁？大表加字段怎么做？](#q17)
18. [分库分表怎么做？跨库 join 怎么解？](#q18)
19. [分布式 ID 方案对比？](#q19)
20. [线上数据一致性怎么对账？](#q20)

---

## <a id="q1"></a>1. MySQL 整体架构？

### 标准答案

```
连接层（连接管理 + 鉴权 + 安全）
    ↓
SQL 层（解析器 + 优化器 + 执行器）
    ↓
存储引擎层（InnoDB / MyISAM / Memory）
    ↓
文件系统
```

**关键组件**：
- **解析器**：SQL → AST
- **优化器**：选执行计划（cost-based）
- **执行器**：调存储引擎接口
- **InnoDB**：默认引擎，事务 + 行锁 + MVCC
- **Buffer Pool**：内存缓存数据页

**redo log / undo log / binlog**：跨层协作（详见后题）。

### 易错点
- 误以为 SQL 优化器一定选最优（基于统计信息估算，可能错）
- 不知道 binlog 在 server 层，redo 在 InnoDB 层

### 追问点
- 优化器看什么？→ 统计信息（行数 / 索引 cardinality）
- 怎么强制走某索引？→ FORCE INDEX / IGNORE INDEX

### 背诵版
**连接层 → SQL 层（解析/优化/执行）→ 存储引擎（InnoDB/MyISAM）**。Buffer Pool 缓存数据页。**优化器基于统计信息**。

---

## <a id="q2"></a>2. InnoDB vs MyISAM？

### 标准答案

| | InnoDB | MyISAM |
| --- | --- | --- |
| 事务 | ✅ ACID | ❌ |
| 锁粒度 | 行锁 | 表锁 |
| 外键 | ✅ | ❌ |
| 崩溃恢复 | ✅（redo log） | ❌ |
| MVCC | ✅ | ❌ |
| 索引结构 | 聚簇索引（数据 = 索引） | 非聚簇（独立数据文件） |
| 全文索引 | ✅（5.6+） | ✅ |
| 适合 | 事务 + 高并发写 | 读多写少 + 简单查询 |

**MySQL 5.5+ 默认 InnoDB**。MyISAM 已基本被淘汰。

### 易错点
- 还在用 MyISAM（除老业务）
- 误以为 MyISAM 查询快（其实 InnoDB 也快，且支持事务）

### 追问点
- InnoDB 主键约束？→ 必有（无显式主键自动生成 ROWID）
- 表锁有什么用？→ 全表 DDL / 老 MyISAM 简单查询

### 背诵版
**InnoDB 默认**：事务 / 行锁 / MVCC / 崩溃恢复 / 聚簇索引。MyISAM 已淘汰，仅老业务。

---

## <a id="q3"></a>3. B+ 树索引为什么用 B+ 不用 B / 红黑树 / Hash？

### 标准答案

| | B+ 树 | B 树 | 红黑树 | Hash |
| --- | --- | --- | --- | --- |
| 树高 | 低（3-4 层撑亿级） | 较低 | 高（log n） | - |
| 范围查询 | **高效（叶子链表）** | 较弱 | 较弱 | ❌ |
| 等值查询 | O(log n) 磁盘 IO 少 | O(log n) | O(log n) | O(1) |
| 范围更新 | 顺序写盘 | 随机 | 随机 | - |

**B+ 树优势**：
- **磁盘 IO 友好**：每节点一个页（16KB），3-4 层撑亿级
- **范围查询高效**：叶子节点串链表（`WHERE id BETWEEN 1 AND 100`）
- **顺序写盘**：插入删除局部调整

**为什么不用 Hash**：不支持范围查询，仅适合等值（Memory 引擎用）。

**为什么不用红黑树**：树高太高（百万数据 20 层），磁盘 IO 爆炸。

### 易错点
- 混淆 B 树和 B+ 树（B+ 数据只在叶子，B 树非叶也有）
- 误以为 Hash 一定快（范围查询不支持）

### 追问点
- B+ 树多少层？→ 单页 16KB，主键 4-8 字节 + 指针 6 字节，每页 ~1000 个；3 层 = 1000³ = 10 亿
- 为什么不用 LSM-Tree？→ MySQL 是 OLTP 优先读，LSM 适合写多（如 RocksDB）

### 背诵版
**B+ 树 = 磁盘友好 + 范围查询 + 树矮**。Hash 不支持范围，红黑树太高，B 树叶子无链表。**3-4 层撑亿级**。

---

## <a id="q4"></a>4. 聚簇索引 vs 二级索引？回表是什么？

### 标准答案

**聚簇索引（Clustered Index）**：
- 数据按主键 B+ 树组织（**叶子节点存完整数据行**）
- InnoDB 表必有（无主键自动生成 ROWID）
- 一表一个

**二级索引（Secondary Index）**：
- 叶子节点存**主键值**（不是数据）
- 多个

**回表（Bookmark Lookup）**：
```sql
SELECT * FROM users WHERE name = 'Alice';
-- 1. 走 name 二级索引找到主键
-- 2. 用主键在聚簇索引找完整行 ← 回表
```

**避免回表**：
- 覆盖索引（select 字段全在二级索引中）
- 主键查询直接走聚簇

### 易错点
- 误以为二级索引存数据（其实存主键）
- `SELECT *` 必回表（即使只用了二级索引字段）

### 追问点
- 主键为什么用 BIGINT 自增？→ 顺序插入，B+ 树不分裂；UUID 随机插入分裂多
- 主键过长有什么影响？→ 二级索引存主键，主键大 → 二级索引也大

### 背诵版
**聚簇索引叶子存数据，二级索引叶子存主键，访问完整行需回表**。**覆盖索引避免回表**。主键用 BIGINT 自增。

---

## <a id="q5"></a>5. 覆盖索引 / 最左前缀 / 索引下推？

### 标准答案

**覆盖索引**：select 字段全在索引中，**不回表**。
```sql
-- 索引: (name, age)
SELECT name, age FROM users WHERE name = 'A';  -- 覆盖，不回表
SELECT * FROM users WHERE name = 'A';          -- 不覆盖，回表
```

**最左前缀**：联合索引按从左到右匹配。
```sql
-- 索引: (a, b, c)
WHERE a = 1                    -- ✅ 用索引
WHERE a = 1 AND b = 2          -- ✅ 用索引
WHERE a = 1 AND c = 3          -- 部分用（只用 a）
WHERE b = 2 AND c = 3          -- ❌ 不用索引（缺 a）
```

**索引下推（ICP, MySQL 5.6+）**：
- 老版本：先按最左前缀走索引，回表后过滤其他条件
- 新版本：把后续条件下推到存储引擎，**过滤后再回表**

```sql
-- 索引: (name, age)
WHERE name LIKE 'A%' AND age = 30
-- 老: 找所有 A* → 回表 → 过滤 age
-- 新（ICP）: 找 A* + age=30 → 回表（少很多）
```

### 易错点
- 联合索引顺序乱设（不按选择性 / 等值优先）
- `SELECT *` 失去覆盖索引优势
- ICP 关闭时机（`optimizer_switch`）

### 追问点
- 联合索引顺序怎么定？→ 等值条件在前，范围在后；选择性高的在前
- 为什么 LIKE 'A%' 能用索引，'%A' 不能？→ 前缀有序，后缀无序

### 背诵版
**覆盖避免回表，最左前缀按从左匹配，ICP 把过滤下推**。联合索引：**等值前，范围后，选择性高在前**。

---

## <a id="q6"></a>6. 事务隔离级别？幻读怎么解？

### 标准答案

| 级别 | 脏读 | 不可重复读 | 幻读 |
| --- | --- | --- | --- |
| READ UNCOMMITTED | ✅ | ✅ | ✅ |
| READ COMMITTED | ❌ | ✅ | ✅ |
| **REPEATABLE READ（默认）** | ❌ | ❌ | ❌（InnoDB 解决） |
| SERIALIZABLE | ❌ | ❌ | ❌ |

**幻读**：同事务两次查询返回不同行数（其他事务插入了符合条件的行）。

**InnoDB RR 怎么解幻读**：
- **快照读**（普通 SELECT）：MVCC + ReadView，看历史版本，不会幻读
- **当前读**（SELECT FOR UPDATE / UPDATE / DELETE）：**Next-Key Lock**（行锁 + 间隙锁）

**MySQL 默认 RR，但很多公司用 RC**（Oracle / PostgreSQL 默认 RC）。

### 易错点
- 误以为 RR 完全解决幻读（**当前读仍可能幻读**，需 Next-Key Lock）
- 误以为 RC 一定有幻读（实际单语句也用 ReadView）
- 不知道 InnoDB RR 比标准更强（间隙锁）

### 追问点
- 为什么很多公司用 RC？→ 性能好（无间隙锁，并发高）+ 主从一致性更好
- RR 完全无幻读吗？→ 快照读不幻读，当前读靠间隙锁防

### 背诵版
**默认 RR**：脏读 / 不可重复读 / 幻读全解决。**幻读靠快照读（MVCC）+ 当前读（间隙锁）**。**RC 性能好很多公司在用**。

---

## <a id="q7"></a>7. MVCC 原理？快照读 vs 当前读？

### 标准答案

**MVCC（Multi-Version Concurrency Control）= 多版本并发控制**：
- 每行有隐藏字段 `trx_id`（事务 ID）和 `roll_pointer`（指向 undo log 旧版本）
- 事务读时看 **ReadView**（已活跃事务列表）
- 根据 trx_id 和 ReadView 判断哪个版本可见

**ReadView 生成时机**：
- RC：每次 SELECT 都生成
- RR：第一次 SELECT 生成，整个事务复用

**可见性判断**：
```
trx_id < min_trx_id  → 已提交，可见
trx_id > max_trx_id  → 未来事务，不可见
trx_id 在活跃列表里 → 不可见
否则 → 可见
```

**快照读 vs 当前读**：
- 快照读：普通 SELECT，看 MVCC 版本，不加锁
- 当前读：`SELECT FOR UPDATE` / UPDATE / DELETE，**读最新版本 + 加锁**

### 易错点
- 误以为 RR 整个事务一个 ReadView（对，但 RC 每次新建）
- 误以为 MVCC 解决所有并发问题（当前读仍要锁）

### 追问点
- undo log 什么时候删？→ 没事务再用时（purge 线程）
- 长事务危害？→ undo log 不能 purge → 表空间膨胀

### 背诵版
MVCC = **trx_id + roll_pointer + ReadView**。RC 每次新 ReadView，RR 一次复用。**快照读不锁，当前读加锁读最新**。

---

## <a id="q8"></a>8. 行锁 / 间隙锁 / Next-Key Lock？

### 标准答案

| | 锁内容 |
| --- | --- |
| **行锁（Record Lock）** | 索引上的行 |
| **间隙锁（Gap Lock）** | 索引行之间的间隙 |
| **Next-Key Lock** | 行锁 + 间隙锁（左开右闭区间） |

```sql
-- 索引值: 5, 10, 15, 20
-- WHERE id = 10 FOR UPDATE
-- → Next-Key Lock 锁 (5, 10]，防止插入 6/7/8/9 和重复 10

-- WHERE id BETWEEN 10 AND 20 FOR UPDATE
-- → 锁 (5, 10] (10, 15] (15, 20] (20, +∞)（部分版本）
```

**Next-Key Lock 防幻读**：插入到锁定间隙会被阻塞。

**RC 隔离级别只有行锁，无间隙锁** → 性能好但有幻读。

### 易错点
- 误以为只锁单行（实际可能锁整个范围）
- 不知道间隙锁阻塞插入（导致死锁）
- RR 下范围查询锁太多（性能问题）

### 追问点
- 怎么避免间隙锁问题？→ 改用 RC / 缩小范围 / 加唯一索引
- 间隙锁怎么发现？→ `SHOW ENGINE INNODB STATUS` 看 LATEST DETECTED DEADLOCK

### 背诵版
**行锁锁单行，间隙锁锁间隙，Next-Key = 行锁 + 间隙锁**。**RR 才有间隙锁，RC 只行锁**。**间隙锁防幻读但易死锁**。

---

## <a id="q9"></a>9. 死锁怎么排查？

### 标准答案

**死锁**：两个事务互相等对方的锁。

```
T1: lock A → wait B
T2: lock B → wait A
→ 死锁
```

**InnoDB 自动检测 + 回滚**：
- `innodb_deadlock_detect = ON`
- 检测到 → 选 undo 量小的事务回滚

**排查工具**：
- `SHOW ENGINE INNODB STATUS` → LATEST DETECTED DEADLOCK 段
- 看两个事务持有的锁 + 等待的锁

**常见原因**：
- 加锁顺序不一致（T1 A→B，T2 B→A）
- 间隙锁交叉（两事务锁不同范围互冲）
- 长事务持锁不放
- 缺索引（行锁退化为表锁）

**预防**：
- **统一加锁顺序**（按主键）
- **缩小事务**（持锁时间短）
- **加索引**（避免锁表）
- **改 RC 减间隙锁**

### 易错点
- 误以为死锁不会发生（RR + 间隙锁极易发生）
- 不开 deadlock detect（线上卡死）
- 不看 INNODB STATUS 直接猜

### 追问点
- 怎么减少死锁？→ 短事务 + 统一加锁顺序 + 加索引
- 高并发死锁检测开销？→ 8.0 引入 latch 优化，但仍有开销，可关 detect 用超时

### 背诵版
**InnoDB 自动检测回滚**，看 `SHOW ENGINE INNODB STATUS`。**统一加锁顺序 + 缩短事务 + 加索引** 是预防三件套。

---

## <a id="q10"></a>10. redo log / undo log / binlog 区别？

### 标准答案

| | redo log | undo log | binlog |
| --- | --- | --- | --- |
| 层 | InnoDB | InnoDB | Server 层 |
| 用途 | 崩溃恢复（持久化） | 回滚 + MVCC | 主从复制 + 备份 |
| 内容 | 物理修改（页变化） | 逻辑反向（INSERT 反向 DELETE） | 逻辑变更（SQL 或行） |
| 写时机 | 事务提交前 | 事务执行时 | 事务提交时 |
| 大小 | 固定循环写 | 不固定 | 持续追加 |

**redo log**：保证 ACID 的 D（Durability）。
**undo log**：保证 ACID 的 A（Atomicity）+ MVCC。
**binlog**：跨 server 层数据复制。

**redo 是物理日志（页 X offset Y 改成 Z），binlog 是逻辑日志（row event 或 SQL）**。

### 易错点
- 混淆 redo 和 binlog（一个 InnoDB 一个 server 层）
- 不知道 undo 也是 MVCC 基础

### 追问点
- 为什么有了 redo 还要 binlog？→ binlog 跨引擎（MyISAM 也有）+ 主从复制 + 时点恢复
- redo 可以代替 binlog 吗？→ MySQL 8.0 一直有讨论，但 binlog 生态太广

### 背诵版
**redo 物理日志（崩溃恢复）/ undo 逻辑反向（回滚+MVCC）/ binlog 逻辑变更（复制+备份）**。redo 在 InnoDB，binlog 在 server 层。

---

## <a id="q11"></a>11. 两阶段提交？为什么需要？

### 标准答案

**MySQL 内部 2PC**：保证 redo log 和 binlog 一致。

```
1. InnoDB 写 redo log（prepare 状态）
2. server 层写 binlog
3. InnoDB 提交 redo log（commit 状态）
```

**如果不 2PC**：
- 先写 redo 后写 binlog → redo 已 commit 但 binlog 没写 → 主从不一致（从库少这个变更）
- 先写 binlog 后写 redo → binlog 已写但 redo 失败 → 主库回滚但从库已经执行

**崩溃恢复**：
- redo prepare + binlog 完整 → commit
- redo prepare + binlog 缺 → rollback

**关键**：redo log 中包含 XID（事务 ID），binlog 也包含，崩溃时通过 XID 匹配。

### 易错点
- 误以为 2PC 是分布式事务（这里是 MySQL 内部 redo + binlog 一致性）
- 不知道有 prepare 状态

### 追问点
- 性能影响？→ 多次 fsync（redo prepare / binlog / redo commit）
- 怎么优化？→ 组提交（group commit）批量 fsync

### 背诵版
**MySQL 2PC 保 redo + binlog 一致**：prepare → 写 binlog → commit。崩溃恢复看 XID 决定 commit/rollback。**组提交优化性能**。

---

## <a id="q12"></a>12. 主从复制原理？延迟怎么处理？

### 标准答案

**复制流程**：
```
Master                          Slave
1. 写 binlog                     IO Thread 拉 binlog → relay log
2. 提交事务                      SQL Thread 重放 relay log
```

**线程**：
- **Dump Thread（主）**：把 binlog 推给从
- **IO Thread（从）**：接收 binlog 写到 relay log
- **SQL Thread（从）**：重放 relay log

**延迟原因**：
- 主写入快，从单线程重放慢
- 大事务（一条更新百万行）
- 从机器配置低
- 网络延迟

**解决**：
- **多线程复制**（5.6+ 库级，5.7+ 行级，8.0+ writeset 真并行）
- **半同步复制**（保证至少一个从收到）
- **主从延迟监控**（Seconds_Behind_Master）
- **业务层处理**：写主读主 / 写后读主 / 强制路由

### 易错点
- 误以为从单线程是 MySQL 限制（其实多线程可配）
- 写后立即读从（可能读不到）
- Seconds_Behind_Master 不准（参考意义）

### 追问点
- 多线程复制怎么保顺序？→ writeset 检测无冲突的事务并行回放
- 真实延迟怎么测？→ 主写入时间戳 + 从读出来的差

### 背诵版
**主写 binlog → 从拉 relay log → SQL Thread 重放**。延迟用**多线程复制 + 半同步 + 业务层路由**。

---

## <a id="q13"></a>13. 半同步复制 vs 异步复制 vs 全同步？

### 标准答案

| | 异步 | 半同步 | 全同步 |
| --- | --- | --- | --- |
| 主等从 | 不等 | 等至少一个 ACK | 等所有从 |
| 性能 | 高 | 中 | 低 |
| 数据安全 | 弱 | 中 | 强 |
| 默认 | ✅ | 需开启 | 不实用 |

**异步**：主提交不等从 → 主挂可能丢数据。

**半同步（rpl_semi_sync_master_enabled）**：
- 主等至少 1 个从写入 relay log（ACK）才返回
- 超时降级为异步（rpl_semi_sync_master_timeout，默认 10s）
- 数据丢失风险大幅降低

**全同步**：所有从都确认 → 性能差，几乎不用。

**MySQL 8.0 增强半同步**：在事务真正写入 binlog 前等 ACK（更严格）。

### 易错点
- 误以为半同步 = 强一致（仍可能降级为异步）
- 误以为全同步实用（性能差）

### 追问点
- 半同步降级为异步会丢数据吗？→ 会，所以要监控降级事件
- MGR（Group Replication）和半同步关系？→ MGR 用 Paxos，更强

### 背诵版
**异步性能高但易丢，半同步至少 1 个 ACK，全同步不实用**。**半同步超时降级为异步**，MGR 用 Paxos 强一致。

---

## <a id="q14"></a>14. 慢 SQL 怎么排查和优化？

### 标准答案

**开启慢查询日志**：
```
slow_query_log = ON
long_query_time = 1.0  # 秒
```

**排查工具**：
- `slow log` + `pt-query-digest` 分析
- `SHOW PROCESSLIST` 看实时
- `EXPLAIN`（详见 Q15）
- `SHOW PROFILE` 看各阶段耗时

**优化思路**：
1. **看 EXPLAIN**（type / key / rows / Extra）
2. **看是否走索引**（type = ALL → 全表扫）
3. **看回表多少**（rows × 字段大小）
4. **看 Extra**（Using filesort / Using temporary 红灯）

**优化手段**：
- 加索引 / 改索引（覆盖 / 联合 / 顺序）
- 改 SQL（拆分 / 减 JOIN / 减子查询）
- 业务层（缓存 / 异步 / 批量）
- 分表（数据量过大）

### 易错点
- 不开 slow log（事后无数据）
- 只看 EXPLAIN 不看实际执行（基于估算）
- 加索引不看选择性（性别加索引无意义）

### 追问点
- 怎么找出影响最大的慢 SQL？→ pt-query-digest 按 total time 排序
- LIKE '%X%' 怎么优化？→ 全文索引 / Elasticsearch / Trigram 索引

### 背诵版
**slow log + pt-query-digest 找慢 SQL**。EXPLAIN 看 **type/key/rows/Extra**。**加索引 + 改 SQL + 业务层缓存 + 分表**四板斧。

---

## <a id="q15"></a>15. explain 关键字段？type / key / Extra？

### 标准答案

**关键字段**：

| 字段 | 含义 | 关注 |
| --- | --- | --- |
| **type** | 访问类型 | `ALL` 全表扫（红灯） |
| **key** | 实际用的索引 | `NULL` 没用索引（红灯） |
| **rows** | 估算扫描行数 | 越小越好 |
| **Extra** | 附加信息 | `Using filesort` / `Using temporary` 红灯 |

**type 从好到差**：
```
system > const > eq_ref > ref > range > index > ALL
```

- `const`：唯一索引等值（`WHERE id = 1`）
- `eq_ref`：JOIN 用主键
- `ref`：非唯一索引等值
- `range`：范围（`WHERE id > 100`）
- `index`：扫描整个索引（覆盖索引）
- `ALL`：全表扫（**警告**）

**Extra 红灯**：
- `Using filesort`：排序用磁盘文件 → 加索引覆盖排序
- `Using temporary`：用临时表 → 改 SQL 避免
- `Using where`：服务层过滤（不一定坏）
- `Using index`：覆盖索引（**好**）
- `Using index condition`：ICP 索引下推（好）

### 易错点
- 看 type 不看 rows（rows 大也慢）
- 误以为 index 比 ref 好（其实 ref 更优）
- 不看 Extra（filesort 是大坑）

### 追问点
- explain analyze（8.0）？→ 实际执行时间 + 估算对比
- 强制索引怎么用？→ FORCE INDEX(idx_name)

### 背诵版
关键四字段：**type / key / rows / Extra**。type 越好越快，Extra **避免 filesort/temporary**，**Using index 是覆盖索引最好**。

---

## <a id="q16"></a>16. InnoDB Buffer Pool 是什么？

### 标准答案

**Buffer Pool = InnoDB 内存中的数据页缓存**。

```
读: 先查 Buffer Pool，miss 才从磁盘读
写: 先改 Buffer Pool（脏页）+ redo log，定期刷盘
```

**关键参数**：
- `innodb_buffer_pool_size`：通常设服务器内存 50-80%
- `innodb_buffer_pool_instances`：分多个实例（减少锁竞争）

**LRU 改进**：
- 普通 LRU：被全表扫一次冲掉所有热数据
- **InnoDB 改进 LRU**：分 Young 和 Old，新页先进 Old，1s 后才晋升 Young
- 防止全表扫污染 Buffer Pool

**Change Buffer**：
- 二级索引修改先缓存（不立即读磁盘页）
- 后续读时合并
- 优化非唯一二级索引的写性能

**Adaptive Hash Index**：
- 监控热点查询，自动建内存 Hash 索引
- 进一步加速

### 易错点
- Buffer Pool 设太小（命中率低）
- 不分实例（高并发锁竞争）
- 全表扫被认为不重要（其实污染 Buffer Pool）

### 追问点
- Buffer Pool 命中率怎么算？→ `Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests`，越低越好
- 怎么预热？→ `innodb_buffer_pool_dump_at_shutdown` + load_at_startup

### 背诵版
**Buffer Pool 内存缓存数据页**，配 50-80% 内存。**改进 LRU 防全表扫污染**。**Change Buffer + AHI** 优化二级索引和热查询。

---

## <a id="q17"></a>17. Online DDL / MDL 锁？大表加字段怎么做？

### 标准答案

**MDL（Metadata Lock）**：表的元数据锁。
- DML（select/update）持 MDL 读锁
- DDL（alter table）持 MDL 写锁
- 写锁等读锁释放才能加 → 长事务卡死 DDL

**Online DDL**：
- 5.6+ 大部分 DDL 支持 ONLINE
- 仅短暂获取 MDL 写锁（开始 + 结束），中间过程允许 DML
- 拷贝数据 / 重建索引并行

**真正"无锁"加字段**：
- `ALTER TABLE t ADD COLUMN c VARCHAR(20), ALGORITHM=INSTANT`（8.0+）
- 元数据级修改，秒级完成

**大表 DDL 三种方法**：
1. **Online DDL（默认）**：MySQL 5.6+，但 8.0 之前部分操作仍要锁
2. **pt-online-schema-change**：影子表 + 触发器同步 + RENAME 切换
3. **gh-ost**（GitHub）：影子表 + binlog 同步（无触发器，更稳）

### 易错点
- 长事务持 MDL 读锁卡死 DDL（事务必须短）
- 直接 ALTER 大表导致锁表（应该用 gh-ost）
- 不看版本（8.0 INSTANT 才是真秒级）

### 追问点
- gh-ost 怎么避免触发器？→ 监听主库 binlog 同步到影子表
- INSTANT 算法限制？→ 仅末尾加字段 / 改默认值，不改类型

### 背诵版
**MDL = 元数据锁，长事务卡 DDL**。Online DDL 短暂锁，**INSTANT（8.0）秒级**。大表用 **gh-ost / pt-osc** 影子表方案。

---

## <a id="q18"></a>18. 分库分表怎么做？跨库 join 怎么解？

### 标准答案

**何时分**：
- 单表 5000 万-1 亿行
- 单库 IOPS 瓶颈

**分片键选择**：
- 高频查询字段（如 user_id）
- 写入分散（避免热点）
- 避免跨片查询

**分片策略**：
- **Hash**（user_id % N）：均匀，扩容难
- **Range**（按时间 / ID 范围）：扩容易，可能热点
- **一致性 Hash**：扩容只影响 1/N

**跨库 join 解法**：
- **避免 join**（业务层多次查询）
- **冗余字段**（订单表冗余商品名）
- **数据同步到大宽表**（数仓 / ES）
- **分布式中间件**（ShardingSphere / Vitess / MyCat）

**事务**：
- **本地事务**（同分片）
- **Saga / 事件最终一致**（跨分片）
- **TCC**（强一致需求）

**扩容难点**：
- Hash 扩容要 rehash 全部数据
- 双写迁移：新写双库 + 全量迁移历史 + 切流 + 清理

### 易错点
- 分片键选错（导致跨片查询）
- 单表小于 5000 万就分（过度设计）
- 不考虑扩容（一开始分得太细）

### 追问点
- 雪花算法怎么用？→ 64 位（时间 + 机器 ID + 序列号），全局唯一不重复
- ShardingSphere vs Vitess？→ ShardingSphere Java 生态主流，Vitess YouTube 出品 K8s 友好

### 背诵版
**分片键选高频查询 + 写入均匀**。Hash 均匀但扩容难，Range 扩容易但易热。**避免跨库 join + 冗余 + 大宽表 + 中间件**。

---

## <a id="q19"></a>19. 分布式 ID 方案对比？

### 标准答案

| 方案 | 优 | 缺 | 代表 |
| --- | --- | --- | --- |
| UUID | 全局唯一 | 无序，索引性能差 | java UUID |
| **雪花算法** | 趋势递增 + 高性能 | 时钟回拨问题 | 推特 Snowflake |
| 数据库自增 | 简单 | 单点，扩展性差 | - |
| 号段（Leaf） | 高性能 + 容灾 | 实现复杂 | 美团 Leaf |
| Redis INCR | 高性能 | 单点 | - |

**雪花算法 64 位**：
```
1 bit 符号位 + 41 bit 时间戳 + 10 bit 机器 ID + 12 bit 序列号
69 年 + 1024 机器 + 4096 ID/ms
```

**时钟回拨问题**：服务器时钟倒退 → ID 可能重复。
- 解决：拒绝服务（短回拨）/ 等待 / 用预留位 / 弃用回拨号段

**美团 Leaf**：
- 号段模式（DB 一次取一段，缓存本地）
- Snowflake 模式（基于 ZK 注册机器 ID）

### 易错点
- 用 UUID 做主键（B+ 树插入分裂多）
- 雪花算法不处理时钟回拨（线上事故）
- 数据库自增 + 分库（每库自增冲突）

### 追问点
- 雪花算法机器 ID 怎么分配？→ ZK 注册 / 配置中心 / Leaf 雪花模式
- 怎么保证号段不重复？→ DB 自增 + 行锁

### 背诵版
**雪花算法主流（64 位 = 时间+机器+序列）**。**时钟回拨**是关键坑。美团 **Leaf 号段 + Snowflake** 双模式。UUID 别做主键。

---

## <a id="q20"></a>20. 线上数据一致性怎么对账？

### 标准答案

**对账场景**：
- 缓存 vs DB
- 主 vs 从
- 主库 vs 数仓
- 订单 vs 支付
- 业务系统 vs 财务系统

**对账方法**：
1. **定时对账**：T+1 跑离线脚本对比
2. **实时对账**：MQ 消费业务事件 + 实时核对
3. **抽样对账**：随机抽 1% 实时验证
4. **关键路径对账**：每笔订单立即对账

**实现要点**：
- **唯一对账 ID**（订单号 / 业务 ID）
- **批量对比**（按时间窗口）
- **差异告警**（不一致立即通知）
- **自动修复**（小差异自动补偿，大差异人工）

**典型工具**：
- 自研对账系统（大厂多）
- DataCompare（阿里）
- 数据库 checksum

### 易错点
- 不做对账（信任系统 → 出事爆雷）
- 只做 T+1（实时性差）
- 差异不告警（堆积起来才发现）

### 追问点
- 怎么实现实时对账？→ binlog → MQ → 对账服务实时比对
- 大数据量对账性能？→ 分桶（按 ID 段）+ 并行 + 增量

### 背诵版
**对账 = 唯一 ID + 批量对比 + 差异告警 + 自动修复**。**T+1 离线 + 实时抽样 + 关键路径**三结合。**金融系统必备**。

---

## 复习建议

**面试前 1 天**：通读"背诵版"。

**面试前 1 周**：每天 3-5 题，结合 03-mysql 各篇。

**实战检验**：
- 能不能讲清楚 InnoDB B+ 树索引为什么 3-4 层撑亿级？
- 能不能完整描述 RR 怎么解决幻读（快照读 + 当前读）？
- 能不能解释 redo / undo / binlog 各自职责？
- 能不能给出大表 DDL 的多种方案？
