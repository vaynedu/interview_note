# MySQL 资深面试答题模板

> 把高频问题拆成 **"一句话 → 分层展开 → 边界 → 代价"** 四段式答题法。
>
> 本篇是面试**"开口就稳"**的压舱石。所有模板都按"资深视角"设计，能区分 P5/P6/P7。
>
> 与 [04-redis/21-senior-interview-answers.md](../04-redis/21-senior-interview-answers.md) 对偶。

---

## 一、四段答题法

```
① 一句话：最精炼的核心（15 字以内）
② 分层展开：2-5 个维度 / 层次
③ 边界 / 代价：什么时候失效？代价是什么？
④ 实战佐证：举一个真实场景或量化数据
```

**资深信号**：主动提代价 + 主动提边界 + 主动提演进 + 主动提失败 + 主动提量化。

---

## 二、为什么 InnoDB 用 B+ 树（最高频）

### 2.1 一句话

> "**多叉平衡 + 叶子链表**：树矮 IO 少，范围查询快，比 B 树 / 红黑树 / Hash 都更适合磁盘场景。"

### 2.2 分层展开

```
1. vs B 树
   B 树叶子节点不连接，范围查询要回溯
   B+ 只在叶子存数据，叶子双向链表 → 范围扫描连续 IO

2. vs 红黑树
   红黑树二叉，1 亿数据 27 层，每层一次 IO
   B+ 单页 16KB / 索引项 ~14B → 每页 ~1170 个分支
   3 层 = 1170³ ≈ 16 亿（覆盖业务规模）

3. vs Hash
   Hash O(1) 但不支持范围、排序、模糊
   InnoDB 的 Adaptive Hash Index 是热点优化，不是主索引

4. vs LSM Tree
   LSM 写好（顺序追加）但读放大、空间放大
   B+ 读写均衡，OLTP 场景更合适
   → TiDB / RocksDB 是 LSM，业务大写入选
```

### 2.3 边界 / 代价

```
B+ 树代价:
  - 插入随机时频繁页分裂（主键无序坑）
  - 主键过长 → 二级索引也大（每个二级索引行 = key + 主键）
  - 写入需要更新所有二级索引

主键设计原则:
  ✓ 顺序递增（自增 ID / 雪花有序）
  ✓ 短（INT 4B / BIGINT 8B，避免 UUID 36B）
  ✓ 不变（避免主键更新）
```

### 2.4 实战量化

```
单页 16KB，主键 8B + 指针 6B = 14B/项
单页 1170 项

3 层 B+ 树:
  根   1 页
  中间 1170 页
  叶子 1170² ≈ 137 万页 × 每页 ~16 行（按 1KB/行）
  ≈ 2200 万行

4 层:
  ≈ 256 亿行

→ 几乎所有业务都在 3-4 层，根 + 中间常驻 Buffer Pool
→ 一次查询 1-2 次磁盘 IO
```

---

## 三、MVCC 是什么 / Read View 怎么工作

### 3.1 一句话

> "**多版本并发控制**：写不阻塞读、读不阻塞写。靠 undo 链 + Read View 实现快照读。"

### 3.2 分层展开

```
核心数据结构:
  每行隐藏字段:
    DB_TRX_ID    最近修改事务 ID
    DB_ROLL_PTR  指向 undo log 的指针
    DB_ROW_ID    行 ID（无主键时）

  undo log 链: 一行历史版本组成链表

Read View（事务的快照）:
  m_ids        活跃事务 ID 列表
  min_trx_id   最小活跃 ID
  max_trx_id   下一个分配的 ID
  creator_trx  自己

可见性判断（对每行）:
  ① row.trx_id == creator_trx → 可见（自己改的）
  ② row.trx_id < min_trx_id   → 可见（已提交）
  ③ row.trx_id >= max_trx_id  → 不可见（未来事务）
  ④ row.trx_id ∈ m_ids        → 不可见（还活跃）
  ⑤ row.trx_id ∉ m_ids        → 可见（已提交）

  不可见 → 沿 DB_ROLL_PTR 向 undo 链回溯，找下一个版本
```

### 3.3 RC vs RR 的核心区别

```
Read Committed（RC）:
  每次 SELECT 都创建新 Read View
  → 每次都看最新已提交
  → 同事务两次 SELECT 可能不同

Repeatable Read（RR，MySQL 默认）:
  事务第一个 SELECT 创建 Read View，后续复用
  → 整个事务一致快照
  → 解决不可重复读
```

### 3.4 边界 / 代价

```
RR 完全解决幻读吗？
  ❌ 快照读不幻读（MVCC）
  ✓ 当前读（FOR UPDATE / LOCK IN SHARE MODE）靠 Next-Key 锁解决
  → "RR + 当前读 + 间隙锁" 三件套才完整防幻读

代价:
  长事务 → undo 链很长 → 回溯成本 + purge 跟不上
  → undo 表空间膨胀 → ibdata 涨爆
  → 监控 history list length，> 几十万就要查长事务
```

### 3.5 实战

```
"我们线上有次报警 ibdata 涨到 200GB，
 排查 information_schema.innodb_trx 找到一个
 跑了 6 小时的事务（开发误操作）。
 KILL 后 purge 慢慢回收，但 ibdata 不会缩。
 最终重做实例 + 改成 innodb_undo_tablespaces 独立表空间。"
```

---

## 四、redo / undo / binlog 怎么协作

### 4.1 一句话

> "**redo 保崩溃恢复，undo 保事务回滚 + MVCC，binlog 保主从复制 + 数据归档。两阶段提交保 redo 与 binlog 一致。**"

### 4.2 三者职责

| 日志 | 层次 | 写入时机 | 作用 | 保留 |
| --- | --- | --- | --- | --- |
| redo log | InnoDB | 事务执行中（WAL）| Crash Recovery | 循环覆盖 |
| undo log | InnoDB | 事务执行中 | 回滚 + MVCC | 事务结束 + 无活跃 ReadView 引用 |
| binlog | Server | 事务提交时 | 复制 + 归档 + 闪回 | 按时间/大小保留 |

### 4.3 两阶段提交（必背）

```
为什么要 2PC？
  redo 与 binlog 是两个独立日志系统
  如果不协调，崩溃后可能不一致：
    - redo 提交 binlog 没写 → 主从不一致（从库少一条）
    - binlog 写了 redo 没提交 → 主从不一致（从库多一条）

流程:
  ① InnoDB prepare 阶段:
     - redo log 状态 = prepare
     - 写盘（fsync）

  ② Server 写 binlog:
     - binlog 写盘（fsync）

  ③ InnoDB commit 阶段:
     - redo log 状态 = commit
     - 不需要立刻 fsync（已经在 prepare 阶段刷盘）

崩溃恢复逻辑:
  - redo prepare + binlog 完整 → commit（认为成功）
  - redo prepare + binlog 不完整 → rollback
  - redo 只 prepare 没 binlog → rollback
```

### 4.4 组提交（Group Commit）

```
问题:
  每次事务都 fsync redo / binlog → 磁盘瓶颈

优化:
  收集多个事务一起 fsync
  binlog_group_commit_sync_delay  延迟收集
  binlog_group_commit_sync_no_delay_count  数量阈值

效果:
  TPS 从几千提升到几万
```

### 4.5 双 1 配置代价

```
sync_binlog = 1            每次 commit fsync binlog
innodb_flush_log_at_trx_commit = 1  每次 commit fsync redo

→ 不丢数据（最强保证）
→ 但 TPS 损失（有 SSD 也只有几 k）

线上权衡:
  金融:    双 1 必开
  非核心:  innodb_flush_log_at_trx_commit = 2（崩溃丢 1s）
  日志类:  sync_binlog = 100/1000（崩溃丢 N 个事务）
```

### 4.6 binlog 三种格式

```
STATEMENT:
  存 SQL 原文
  ✓ 紧凑
  ✗ 不确定函数（NOW / RAND / UUID）主从不一致
  ✗ 行级触发器复制不准

ROW（推荐）:
  存每行变更前后值
  ✓ 主从一致性强
  ✓ 支持闪回（解析 binlog 反向 SQL）
  ✗ 大事务 binlog 大

MIXED:
  默认 STATEMENT，遇到不确定就 ROW
  → 5.7 后多数生产用 ROW
```

---

## 五、隔离级别四种 + 解决什么问题

### 5.1 一句话

> "RU/RC/RR/Serializable，分别解决脏读 / 不可重复读 / 幻读 / 完全串行。MySQL 默认 RR，多数业务用 RC。"

### 5.2 对照表

| 级别 | 脏读 | 不可重复读 | 幻读 | 实现 |
| --- | --- | --- | --- | --- |
| READ UNCOMMITTED | ✓ | ✓ | ✓ | 读不加锁，读到未提交 |
| READ COMMITTED（Oracle 默认）| ✗ | ✓ | ✓ | 每次 SELECT 新 Read View |
| REPEATABLE READ（MySQL 默认）| ✗ | ✗ | 部分 | 事务首 SELECT 固定 Read View + Next-Key 锁 |
| SERIALIZABLE | ✗ | ✗ | ✗ | 所有 SELECT 加共享锁 |

### 5.3 RR vs RC 怎么选

```
RR（MySQL 默认）:
  ✓ 同事务一致快照
  ✓ Next-Key 锁防幻读（当前读）
  ✗ 间隙锁更宽 → 死锁概率高
  ✗ 主从复制要求 ROW 格式

RC（多数互联网业务）:
  ✓ 锁范围小（只锁行）
  ✓ 死锁少
  ✓ 大事务影响小
  ✗ 同事务两次读可能不同

为什么互联网常用 RC？
  - 业务一般不依赖事务内一致快照
  - 死锁更少 → 高并发更稳
  - 主从兼容性更好
```

### 5.4 RR 防幻读的真相

```
"快照读" 不幻读:
  普通 SELECT 走 MVCC，看的是 Read View 快照

"当前读" 靠间隙锁防幻读:
  SELECT ... FOR UPDATE
  SELECT ... LOCK IN SHARE MODE
  UPDATE / DELETE
  → InnoDB 在 RR 下用 Next-Key 锁（行锁 + 间隙锁）

例子:
  事务 A: SELECT * FROM t WHERE id BETWEEN 10 AND 20 FOR UPDATE
  → 锁住 (10, 20] 区间，包括 5 → 10 的 gap
  → 事务 B 想 INSERT id=15 会阻塞
```

---

## 六、行锁 / 间隙锁 / Next-Key 锁

### 6.1 一句话

> "**行锁锁记录、间隙锁锁区间、Next-Key = 行锁 + 左开右闭间隙**。RR 默认用 Next-Key 防幻读。"

### 6.2 三种锁对比

```
Record Lock（行锁）:
  锁单条已存在记录
  WHERE id = 5（id 是主键且存在）→ 锁这行

Gap Lock（间隙锁）:
  锁两条记录之间的"空隙"
  WHERE id = 5（id 不存在）→ 锁 (3, 7) 区间
  目的: 防止其他事务 INSERT 进这个区间（防幻读）

Next-Key Lock（临键锁）:
  Record + Gap 的左开右闭区间
  WHERE id BETWEEN 5 AND 10 → 锁 (前驱, 5], (5, 10] 等
```

### 6.3 锁退化规则

```
RC 隔离级别:
  没有间隙锁，只有 Record Lock
  → 死锁少，但有幻读

RR + 唯一索引等值匹配:
  Next-Key 退化为 Record Lock
  WHERE id = 5（id 主键存在）→ 只锁 5 这行

RR + 唯一索引等值 + 不存在:
  Next-Key 退化为 Gap Lock
  WHERE id = 5（id 主键不存在）→ 锁 (3, 7) 间隙

RR + 范围查询:
  保持 Next-Key
  WHERE id > 5 AND id < 10 → 锁 (5, 10) + 后面的 gap
```

### 6.4 死锁排查标准流程

```sql
-- ① 看死锁日志
SHOW ENGINE INNODB STATUS\G
-- 找 LATEST DETECTED DEADLOCK 段

-- ② 实时查锁
SELECT * FROM information_schema.INNODB_TRX;
SELECT * FROM performance_schema.data_locks;
SELECT * FROM performance_schema.data_lock_waits;

-- ③ 分析:
--   - 哪两个事务
--   - 各持有什么锁
--   - 各等待什么锁
--   - SQL 是什么

-- ④ 解决:
--   - 调整 SQL 顺序（统一顺序加锁）
--   - 加索引（避免行锁升级表锁）
--   - 缩短事务
--   - 改用 RC（如果业务允许）
```

### 6.5 实战死锁案例

```
案例: 两个 UPDATE 互相死锁
  事务 A: UPDATE WHERE id=1; UPDATE WHERE id=2;
  事务 B: UPDATE WHERE id=2; UPDATE WHERE id=1;

修复:
  统一按 id 升序加锁（业务层 sort 一下）
  ORDER BY id 后再 UPDATE
```

---

## 七、索引失效场景

### 7.1 一句话

> "**类型转换、函数、前缀模糊、OR、违反最左前缀、范围后字段、回表多** 都会失效。EXPLAIN 一看 type=ALL 就知道。"

### 7.2 9 大失效场景

```
1. 隐式类型转换
   WHERE phone = 13800000000  -- phone 是 varchar
   → 触发 CAST，索引失效
   ✓ WHERE phone = '13800000000'

2. 函数 / 表达式
   WHERE DATE(created_at) = '2026-05-09'
   → 索引失效
   ✓ WHERE created_at >= '2026-05-09 00:00:00'
        AND created_at <  '2026-05-10 00:00:00'

3. 前缀模糊
   WHERE name LIKE '%abc'
   → 失效
   ✓ WHERE name LIKE 'abc%'  (后缀模糊 OK)

4. OR 一边没索引
   WHERE (idx_col = 1 OR no_idx_col = 2)
   → 全表
   ✓ UNION ALL 拆开 / 都加索引

5. 违反最左前缀
   联合索引 (a, b, c)
   WHERE b = 1 AND c = 2  → 失效
   WHERE a = 1 AND c = 2  → 只用 a

6. 范围后字段
   联合索引 (a, b, c)
   WHERE a = 1 AND b > 5 AND c = 2 → c 用不上
   ✓ 改顺序 (a, c, b) 让等值在前

7. 回表太多
   WHERE name = 'A'  -- name 上有索引
   → 优化器估算回表 > 全表 1/4 → 直接全表
   ✓ 覆盖索引避免回表

8. != / NOT IN
   WHERE status != 1 → 通常失效
   ✓ 改 IN 列出可能值

9. NULL 判断不一定失效（看版本和数据分布）
   WHERE col IS NULL → 5.7+ 多数能用索引
   早期版本失效
```

### 7.3 ICP（索引下推）加分

```
没有 ICP（5.6 前）:
  联合索引 (name, age)
  WHERE name LIKE 'A%' AND age = 20
  → Server 层拿 name LIKE 'A%' 的所有主键
  → 回表查每行 → 再过滤 age

有 ICP（5.6+）:
  → InnoDB 层在索引上同时过滤 name LIKE 'A%' AND age = 20
  → 减少回表次数

EXPLAIN Extra: Using index condition
```

---

## 八、主从延迟根因 + 解法

### 8.1 一句话

> "根因 = **大事务 / 单线程 SQL / 网络 / 从库慢**。解法 = **多线程复制 + 拆事务 + 半同步 + 只读路由强制主**。"

### 8.2 根因分析

```
1. 单线程 SQL 重放（5.7 前）
   主库并发写，从库串行回放
   → 必然落后

2. 大事务
   主库一个 5 分钟事务
   → binlog 5 分钟 + 从库回放 5 分钟
   → 延迟 5-10 分钟

3. 从库 IO 慢
   从库磁盘 / 网络 / CPU 不如主
   → 持续追不上

4. 锁等待
   从库有读 + DDL → SQL 线程等锁

5. 大查询打死从库
   从库被业务读打满 → SQL 线程没资源
```

### 8.3 解法（按版本演进）

```
MySQL 5.6: 库级并行复制
  按库分线程，单库内仍然串行

MySQL 5.7: 基于 Group Commit 并行（LOGICAL_CLOCK）
  同一组提交的事务可并发
  slave_parallel_workers = 16/32

MySQL 5.7+: WRITESET（强烈推荐）
  binlog 记录每个事务修改的行 hash
  无冲突的事务可并行（不限同组）
  → 延迟优化 5-10 倍
  binlog_transaction_dependency_tracking = WRITESET

MySQL 8.0: 默认 WRITESET，性能更强
```

### 8.4 业务侧应对

```
1. 强一致读路由主库
   写后立即读 → 强制主库
   配置: 业务标记 force_master

2. 半同步（避免数据丢）
   rpl_semi_sync_master_enabled = ON
   主库等至少 1 个从库 ACK 才返回
   → 网络抖动会退化为异步（监控告警）

3. 监控
   Seconds_Behind_Master（粗略）
   pt-heartbeat（精确）

4. 大事务拆分
   循环 LIMIT 1000 分批提交
```

### 8.5 实战

```
"我们订单库主从延迟 5 秒：
 排查 Seconds_Behind_Master 每分钟波动
 主库 SHOW PROCESSLIST 发现一个跑 30 分钟的 DELETE
 → 拆成 LIMIT 1000 循环 + sleep
 主从延迟立刻降到 100ms。

 长期方案:
   开启 WRITESET 并行复制
   slave_parallel_workers = 16
   定期扫长事务（5 分钟告警）"
```

---

## 九、慢 SQL 怎么排查 / 优化

### 9.1 一句话

> "**慢日志 → pt-query-digest 聚合 → EXPLAIN 看执行计划 → 加索引 / 改 SQL / 拆事务 → 回归压测**。"

### 9.2 标准排查流程

```bash
# 1. 慢日志
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1.0;
SET GLOBAL log_queries_not_using_indexes = ON;

# 2. pt-query-digest 聚合
pt-query-digest /var/log/mysql/slow.log

# 3. 看 Top 10 慢 SQL
#    指标: 总耗时 / 调用次数 / 平均 RT / Lock 时间

# 4. EXPLAIN 详细分析
EXPLAIN <slow_sql>;
EXPLAIN ANALYZE <slow_sql>;  # 8.0+，真实执行时间

# 5. 实时定位
SHOW FULL PROCESSLIST;
SELECT * FROM sys.processlist WHERE command != 'Sleep';
```

### 9.3 EXPLAIN 关键字段

```
type（连接类型，性能从好到差）:
  system > const > eq_ref > ref > range > index > ALL
  → 最少要 range，ALL 必须优化

key:
  实际用的索引
  NULL = 没用索引

rows:
  估算扫描行数（关键！）
  > 10w 通常要优化

Extra:
  Using index        → 覆盖索引 ✓
  Using where        → 过滤
  Using filesort     → 文件排序 ✗（要优化）
  Using temporary    → 临时表 ✗
  Using index condition → ICP ✓
```

### 9.4 优化思路

```
缺索引:
  → 加索引（注意联合索引顺序：高选择性在前）

索引选错:
  → ANALYZE TABLE 更新统计
  → FORCE INDEX(idx_name)
  → 改写 SQL 让优化器选对

回表多:
  → 覆盖索引（包含查询字段）

排序慢:
  → 索引带排序字段
  → 减少返回列

JOIN 慢:
  → 小表驱动大表
  → JOIN 字段加索引
  → 拆分多次查询（应用层 JOIN）

大事务:
  → 拆分批次
  → 短事务

深分页（LIMIT 1000000, 10）:
  → 改 WHERE id > last_id LIMIT 10（游标分页）
  → 子查询 + 主键定位
```

### 9.5 深分页优化（高频）

```sql
-- ❌ 慢：扫 100w + 10 行
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;

-- ✅ 游标分页
SELECT * FROM orders WHERE id > 1000000 ORDER BY id LIMIT 10;

-- ✅ 子查询定位
SELECT * FROM orders
WHERE id IN (
    SELECT id FROM orders ORDER BY id LIMIT 1000000, 10
);
```

---

## 十、分库分表怎么讲

### 10.1 一句话

> "**单表 > 1000w 考虑分表，单库 QPS > 5k / 数据 > 500GB 考虑分库**。按业务键 Hash 分，配雪花 ID + 分布式事务。"

### 10.2 分库分表决策

```
先纵向拆分:
  - 按业务领域拆库（订单库 / 用户库 / 商品库）
  - 按字段冷热拆表（基础信息 + 扩展信息）

再横向拆分:
  分表: 单表 > 1000w / 单表 > 50GB
  分库: 单库 QPS > 5k / 单库 > 500GB / 单机内存装不下索引
```

### 10.3 分片键选择

```
原则:
  ① 业务最常用的查询条件
  ② 数据分布均匀
  ③ 不可变（避免数据迁移）

例子:
  订单表: user_id Hash 分（用户查订单最常见）
  日志表: 按时间分（天表 / 月表）
  消息表: 接收者 ID Hash

陷阱:
  - 选了不常用的字段 → 跨库查询多
  - 数据倾斜（按城市分，北上广深占 80%）
```

### 10.4 分片算法

```
Range（范围）:
  按时间 / ID 区间
  ✓ 易扩展（加新分片）
  ✗ 数据热点（最新分片压力大）

Hash:
  user_id % 16
  ✓ 数据均匀
  ✗ 扩容难（要重新 Hash + 数据迁移）

一致性 Hash:
  虚拟节点 + Hash 环
  ✓ 扩容只迁移部分
  ✗ 复杂度高

Range + Hash:
  日表 + 表内 Hash
  → 实战常用
```

### 10.5 分库分表的难题

```
1. 跨库 JOIN
   → 应用层 JOIN（多次查询 + 内存合并）
   → 冗余字段（反范式化）
   → 数仓离线 JOIN

2. 跨库事务
   → 本地消息表
   → TCC / Saga
   → 事务消息（RocketMQ）

3. 分布式 ID
   → 雪花算法（推荐）
   → 美团 Leaf / 百度 UidGenerator
   → 数据库自增段

4. 跨库分页
   → 改造为游标分页
   → 限制深度（最多 100 页）
   → ES 兜底排序

5. 扩容
   → 双写 + 切流（最稳）
   → 一致性 Hash（部分迁移）
   → 离线迁移 + 校验
```

### 10.6 何时用 TiDB / OceanBase 替代

```
分库分表很痛 / 数据 > 10TB / 强一致 + 海量
→ NewSQL（TiDB / OceanBase / CockroachDB）

代价:
  - 运维复杂度高（需要专人）
  - 性能 P99 通常不如分库分表 MySQL
  - 生态没 MySQL 强

实战:
  - 5-10TB → 倾向分库分表（成熟稳定）
  - > 10TB + 强一致 + 跨地域 → TiDB / OB
```

---

## 十一、Online DDL 怎么讲

### 11.1 一句话

> "**5.6+ 支持 Online DDL（ALGORITHM=INPLACE/INSTANT），但仍有 MDL 锁瓶颈。生产用 gh-ost / pt-osc。**"

### 11.2 三种算法

```
COPY（最老）:
  新建临时表 → 全量复制 → 改名
  → 全程锁表，业务停摆

INPLACE（5.6+）:
  In-place 修改原表
  ✓ 大部分操作不锁表（DML 可继续）
  ✗ 仍需短暂 MDL 写锁（前后）
  ✗ 长事务会阻塞 DDL → DDL 阻塞所有新连接

INSTANT（8.0.12+）:
  仅修改元数据（如加列默认 NULL）
  ✓ 秒级
  ✗ 仅特定操作支持
```

### 11.3 MDL 雪崩问题

```
现象:
  ALTER TABLE 卡 1 小时不动
  期间所有访问该表的请求 timeout
  连接池被打满 → 服务挂

原因:
  ① 长事务持有 MDL 读锁（普通 SELECT）
  ② DDL 拿不到 MDL 写锁，进入等待队列
  ③ 此后所有新请求拿 MDL 读锁，全部排队
  → 雪崩

排查:
  SHOW PROCESSLIST | grep "Waiting for table metadata lock"

解决:
  ① 找出阻塞事务（performance_schema.metadata_locks）
  ② KILL 阻塞事务
  ③ DDL 加超时:
     SET lock_wait_timeout = 5;
     ALTER TABLE ...

  长期: gh-ost / pt-osc
```

### 11.4 gh-ost vs pt-osc

```
pt-online-schema-change:
  原理: 创建影子表 + 触发器 + 复制数据
  ✗ 触发器消耗主库性能
  ✗ 不支持有外键的表

gh-ost (GitHub):
  原理: 通过 binlog 同步增量
  ✓ 不用触发器，对主库压力小
  ✓ 可暂停 / 限速
  ✓ 切流秒级
  → 主流推荐

流程:
  1. 创建 _table_gho 影子表
  2. 在影子表上 ALTER
  3. 全量 COPY + binlog 增量同步
  4. 切流（rename）
  5. 删旧表
```

### 11.5 实战

```
"我们 5 亿行的订单表加字段:
  直接 ALTER → 估算 6 小时 + 锁冲突
  改用 gh-ost:
    --max-load=Threads_running=50
    --critical-load=Threads_running=200
    --chunk-size=1000
    --throttle-control-replicas=slave1,slave2

  跑了 4 小时，主库压力 < 30%
  切流 1 秒，无业务感知"
```

---

## 十二、字段类型怎么选

### 12.1 一句话

> "**短小精悍**：能 INT 不 BIGINT，能 VARCHAR(50) 不 VARCHAR(255)，金额必 DECIMAL，时间用 DATETIME。"

### 12.2 高频选型

```
数字:
  TINYINT  1B  -128~127            状态枚举
  INT      4B  -21亿~21亿          普通 ID
  BIGINT   8B  ±9×10^18            订单号 / 主键
  DECIMAL(M,D)  精确              金额（绝不用 float/double）

字符串:
  CHAR(N)    定长     固定长度 (UUID / 手机号)
  VARCHAR(N) 变长 +1-2B  普通字符串
  TEXT      大文本   单独行存（影响性能）

时间:
  DATETIME   8B  '1000-9999'       推荐（无时区坑）
  TIMESTAMP  4B  '1970-2038'       有时区转换（Y2038 问题）
  → 业务推荐 DATETIME，要时区转换在应用层做

枚举:
  ENUM('a','b')   省空间但难维护
  TINYINT + 应用层映射  推荐

JSON:
  MySQL 5.7+ 支持 JSON 类型
  能查能改，但不如关系字段快
  适合: 灵活字段（用户偏好 / 配置）
  不适合: 高频查询字段
```

### 12.3 经典坑

```
1. 金额 float/double
   ❌ FLOAT 精度丢失（0.1 + 0.2 ≠ 0.3）
   ✓ DECIMAL(20, 4) 或 BIGINT 存"分"

2. 字符集 utf8 vs utf8mb4
   ❌ utf8 只支持 3 字节，存不了 emoji
   ✓ 统一 utf8mb4 + utf8mb4_0900_ai_ci

3. VARCHAR(255) 滥用
   ❌ VARCHAR(255) 占行内空间预算 → 索引行存不下
   ✓ 按业务真实长度，地址 200 / 名字 50

4. NULL 索引坑
   ❌ NULL 无法走 = 比较，COUNT(col) 跳 NULL
   ✓ NOT NULL DEFAULT '' / 0

5. TIMESTAMP 时区
   ❌ 主从时区不一致 → 数据不一致
   ✓ 统一 UTC，应用层转
```

---

## 十三、分布式事务怎么选

### 13.1 一句话

> "**强一致小事务用 XA / 2PC；跨服务最终一致用 TCC / Saga / 事务消息 / Outbox。业务幂等 + 对账兜底。**"

### 13.2 五种方案对比

| 方案 | 一致性 | 性能 | 业务侵入 | 场景 |
| --- | --- | --- | --- | --- |
| 2PC / XA | 强 | 低（阻塞）| 低 | 跨库强一致小事务 |
| TCC | 强 | 中 | **高**（Try/Confirm/Cancel）| 资金类强一致 |
| Saga | 最终 | 高 | 中（每步补偿）| 长流程业务 |
| 事务消息 | 最终 | 高 | 低 | 业务 + 通知 |
| Outbox | 最终 | 高 | 低 | 业务 + MQ 异步 |

### 13.3 选型决策

```
跨库 + 强一致 + 短事务:
  → MySQL XA（注意性能 + 协调者单点）

跨服务 + 强一致（资金）:
  → TCC（Seata / Hmily）
  → 注意空回滚 / 悬挂 / 幂等

跨服务 + 长流程（订单流转）:
  → Saga 编排
  → 每步补偿

业务 + 异步通知（下单 → 发券）:
  → 事务消息（RocketMQ）
  → 或 Outbox（业务表 + outbox 表 + 后台扫描）

实战:
  - 大部分用 Outbox（最简单可靠）
  - 资金用 TCC
  - 长流程用 Saga
  - 跨库基本不用 XA（生产很少）
```

### 13.4 业务幂等是基石

```
所有方案都要业务幂等:
  ① 唯一业务 ID（订单号）+ DB 唯一索引
  ② 状态机严格（PENDING → PAID，不能逆向）
  ③ 重复请求返回原结果

对账兜底:
  ① 实时对账（Kafka 双发 + 比对）
  ② T+1 对账（每天凌晨跑）
  ③ 人工兜底
```

---

## 十四、缓存与 DB 一致性

### 14.1 一句话

> "**先写 DB 再删缓存** 是主流，binlog 订阅做异步双删。强一致就直接读 DB。"

### 14.2 方案对比

详见 [04-redis/10-cache-consistency-design.md](../04-redis/10-cache-consistency-design.md) 和 [04-redis/21-senior-interview-answers.md 第五节](../04-redis/21-senior-interview-answers.md)。

```
方案:
1. 先写 DB 再删缓存（Cache Aside，主流）
2. 延迟双删
3. binlog 订阅删缓存（推荐大规模）
4. 强一致读 DB
5. 加分布式锁（重读重写）
```

### 14.3 关键观点

```
"为什么不更新缓存而是删缓存？"
  - 更新需要序列化（可能并发覆盖）
  - 删除是幂等的
  - 下次读时 lazy load 重建
  - 减少不必要的写（只有真正读时才回填）

"删缓存失败怎么办？"
  - 重试队列（Kafka）
  - binlog 订阅兜底
  - 加短 TTL 等过期
```

---

## 十五、Buffer Pool 怎么讲

### 15.1 一句话

> "**InnoDB 的内存缓存层**：缓存数据页 + 索引页 + change buffer + 自适应哈希。命中率决定性能。"

### 15.2 关键组件

```
1. LRU 链表（带 young/old 区域分离）
   young: 5/8 高频访问页
   old:   3/8 新读入页
   防止全表扫描淘汰所有热点

2. Free 链表
   未使用的 page

3. Flush 链表
   脏页（待刷盘）

4. Change Buffer
   非唯一索引的更新先缓存到 CB
   → 减少随机 IO
   唯一索引必须先读页校验，无法用 CB

5. Adaptive Hash Index
   高频热点自动建 Hash 索引
   把 B+ 树访问从 O(log N) 降到 O(1)
```

### 15.3 关键参数

```
innodb_buffer_pool_size:
  推荐机器内存 50-75%
  生产典型: 16-256GB

innodb_buffer_pool_instances:
  > 8GB 时分多个实例（减少锁竞争）
  默认 8

innodb_old_blocks_pct = 37:
  old 区域占比

innodb_old_blocks_time = 1000:
  新 page 在 old 区域待 1s 才能进 young
  → 防止预读 / 全表扫污染热点
```

### 15.4 命中率

```
命中率 = 1 - innodb_buffer_pool_reads / innodb_buffer_pool_read_requests

健康值: > 99%
< 95%: 内存不够 / 全表扫频繁

监控:
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_%';
```

---

## 十六、线上 DB CPU 100% 怎么排查

### 十六.1 一句话

> "**SHOW PROCESSLIST + 慢查询 + sys schema 三件套，定位慢 SQL → KILL → 加索引 / 限流。**"

### 十六.2 排查 7 步

```
Step 1: 看监控（QPS / TPS / 连接数 / IO / CPU 突增点）

Step 2: 实时查正在跑的 SQL
  SHOW FULL PROCESSLIST;
  SELECT * FROM sys.processlist WHERE command != 'Sleep' ORDER BY time DESC;

Step 3: 看锁等待
  SELECT * FROM information_schema.innodb_trx ORDER BY trx_started;
  SELECT * FROM performance_schema.data_locks;
  SELECT * FROM performance_schema.data_lock_waits;

Step 4: 慢查询 Top
  pt-query-digest /var/log/mysql/slow.log

Step 5: 止血
  - KILL <pid> 慢 SQL
  - 应用层限流 / 切流
  - 加从库分担

Step 6: 根因
  - 缺索引 → ALTER ADD INDEX
  - 大事务 → 拆分
  - 优化器选错 → FORCE INDEX / ANALYZE TABLE
  - 业务突增 → 限流

Step 7: 复盘
  - 慢查询监控
  - SQL Review 流程
  - 容量告警
```

### 16.3 KILL 的细节

```
KILL <thread_id>      → 终止连接
KILL QUERY <thread_id> → 仅终止当前查询，保连接

⚠️ KILL 不立刻生效:
  正在 fsync / 网络 IO 的事务要等
  正在回滚（KILL 后）可能比执行还久
  → 评估好再 KILL
```

---

## 十七、Go 实战相关高频问

### 17.1 database/sql 连接池

详见 [16-go-mysql-practice.md](16-go-mysql-practice.md)。

```go
db.SetMaxOpenConns(50)
db.SetMaxIdleConns(10)
db.SetConnMaxLifetime(30 * time.Minute)
db.SetConnMaxIdleTime(10 * time.Minute)
```

### 17.2 Context 超时与 KILL QUERY

```
Go 的 context.WithTimeout 触发后:
  database/sql 会调用 conn.Cancel()
  go-mysql-driver 发送 KILL QUERY

但:
  不会立刻终止 Server 端 SQL（KILL 也要等）
  → 业务超时但 DB 仍在跑 → 资源浪费

应对:
  应用层超时 << SQL 预期超时
  慢 SQL 优化（不能依赖 context 兜底）
```

### 17.3 事务里调外部服务

```go
// ❌ 严重反模式
tx, _ := db.Begin()
tx.Exec("UPDATE ...")
http.Get("https://third-party")  // 占着行锁等几秒
tx.Commit()
→ 行锁持有时间暴涨 → 死锁 / 阻塞

// ✓ 拆分
tx.Exec("UPDATE ...")
tx.Commit()
go callExternal()  // 事务外异步
```

### 17.4 rows.Close() 必须

```go
rows, _ := db.Query(...)
defer rows.Close()  // ⚠️ 漏了 → 连接泄漏

// 即使 for next 跑完也要 Close（防 panic 路径）
```

---

## 十八、面试加分点

- **B+ 树 3-4 层撑亿级** 用计算给出
- **MVCC + ReadView + 可见性 5 条规则** 完整讲
- **RR 防幻读 = 快照读 + 当前读 + Next-Key**（不是只靠 MVCC）
- **redo / binlog 两阶段提交** + 组提交 + 双 1 代价
- **WRITESET 并行复制**（5.7+ 解决主从延迟）
- **gh-ost vs pt-osc** 选 gh-ost 的原因
- **Online DDL MDL 雪崩**（长事务 + DDL 等锁 + 后续全卡）
- **死锁 information_schema + performance_schema** 完整排查
- **深分页游标化**（id > last_id）
- **Cache-Aside 是先写 DB 再删缓存**
- **金额必 DECIMAL，时间用 DATETIME**
- **utf8mb4 是必选**（utf8 只 3 字节）
- **TCC 三大坑**（空回滚 / 悬挂 / 幂等）
- **业务幂等 + 对账** 是所有分布式事务的兜底
- **Buffer Pool 命中率 > 99%**
- **量化数据 + 实战案例**

---

## 十九、答题"套路"技巧

```
开口前先想 3 秒:
  □ 这个问题的"一句话"是什么？
  □ 我能分几层展开？
  □ 代价 / 边界是什么？
  □ 有没有真实案例 / 量化数据？

说话时:
  □ 先给结论
  □ 再展开 2-5 点
  □ 主动提代价 + 边界
  □ 最后举案例 + 量化

避免:
  □ "我觉得 / 应该是"（不自信）
  □ 长篇大论不分层
  □ 只讲理论无案例
  □ 只讲成功无失败
```

### 19.1 资深信号

```
✓ 主动提演进:
  "MySQL 5.6 是 X，5.7 改成 Y，8.0 又增强 Z"

✓ 主动提代价:
  "双 1 配置不丢数据，但 TPS 损失 30%"

✓ 主动提边界:
  "Online DDL 5.6+ 支持，但不能改主键 / 外键"

✓ 主动提失败:
  "我们曾用 TIMESTAMP 跨时区导致数据错乱，后来改 DATETIME"

✓ 主动量化:
  "B+ 树 3 层撑 16 亿行 / Buffer 命中率 99.5%"
```

---

## 二十、20 问速答索引

```
原理 / 引擎:
Q1: B+ 树为什么适合？      → 二
Q2: InnoDB vs MyISAM？     → 08
Q3: 聚簇 vs 二级索引？     → 02
Q4: Buffer Pool 工作原理？ → 十五 + 13

事务 / 锁 / MVCC:
Q5: ACID 怎么保证？        → 03
Q6: MVCC + Read View？     → 三
Q7: RR vs RC？             → 五
Q8: Next-Key 锁？          → 六
Q9: 死锁排查？             → 六

日志 / 复制:
Q10: redo / undo / binlog？ → 四
Q11: 两阶段提交？           → 四
Q12: 主从延迟？             → 八

调优 / DDL:
Q13: 索引失效？             → 七
Q14: 慢 SQL 排查？          → 九
Q15: Online DDL？           → 十一
Q16: 深分页？               → 九

设计:
Q17: 分库分表？             → 十
Q18: 分布式事务？           → 十三
Q19: 字段类型？             → 十二
Q20: 缓存一致性？           → 十四

线上:
Q21: DB CPU 100%？          → 十六
Q22: 误删数据？             → 10
Q23: Go database/sql 用法？ → 十七 + 16
```

---

## 二十一、关联阅读

```
本目录:
- 00-mysql-map.md             总览地图
- 01-architecture.md          架构原理
- 02-index.md                 索引深度
- 03-transaction-lock-mvcc.md 事务并发
- 04-log.md                   日志体系
- 05-replication-ha.md        复制高可用
- 09-distributed-transaction.md 分布式事务
- 10-production-cases.md      8 个真实案例
- 11-order-system-design.md   订单系统设计
- 12-sharding.md              分库分表
- 14-explain-optimizer.md     EXPLAIN 优化器
- 15-online-ddl-mdl.md        Online DDL
- 16-go-mysql-practice.md     Go 实战（深度）
- 17-consistency-reconciliation.md 对账系统
- 19-storage-comparison.md    存储选型
- 21-mysql-design-tradeoffs.md 设计边界（待完成）

跨模块:
- 04-redis/21-senior-interview-answers.md  Redis 答题（对偶）
- 05-message-queue/00-mq-map.md            MQ 配合
- 06-distributed/03-transaction.md         分布式事务理论
- 99-meta/mysql-20.md                      速记题集
```
