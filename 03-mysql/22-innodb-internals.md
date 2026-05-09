# InnoDB 源码深水区

> P7+ 面试**源码级追问**必看。Buffer Pool / Latch / Mini-Transaction / Page / B+ 树 SMO / Next-Key / Purge / Change Buffer / AHI。
>
> 与 [04-redis/17-object-encoding-internals.md](../04-redis/17-object-encoding-internals.md) / [04-redis/18-eventloop-memory-internals.md](../04-redis/18-eventloop-memory-internals.md) 对偶。
>
> 目标：**能把"MySQL 内部怎么工作"讲到源码级别**，区分 P6 和 P7。

---

## 一、为什么要懂内部

```
P6 答:
  "InnoDB 用 B+ 树，MVCC 靠 undo 实现"

P7 答:
  "InnoDB Page 16KB，一页 PAGE_LEAF 含 user records + infimum/supremum
   + page directory 稀疏索引；B+ 树插入满页触发 SMO (Structure Modification
   Operation)，SMO 要拿 index latch 的 X mode；并发场景下 MVCC 读沿
   DB_ROLL_PTR 访问 undo 链，回溯到符合 Read View 的版本..."

→ 高下立判
```

---

## 二、InnoDB 存储层次

```
           [Table Space (.ibd)]
                   │
           ┌───────┴───────┐
           │               │
       [Segment]      [Segment]   -- 每个索引一个 segment
           │
       [Extent]                   -- 1 个 Extent = 64 个 Page = 1MB
           │
       [Page 16KB]                -- 最小 IO 单位
           │
       [Row]
```

### 2.1 Page（页）

```
Page 是 InnoDB 最小 IO 单位（默认 16KB）

Page Type:
  FIL_PAGE_INDEX         B+ 树索引页（最常见）
  FIL_PAGE_UNDO_LOG      undo 页
  FIL_PAGE_TYPE_SYS      系统页
  FIL_PAGE_IBUF_FREE_LIST Change Buffer 空闲列表
  FIL_PAGE_TYPE_BLOB     溢出 BLOB

为什么 16KB?
  - 磁盘通常 4KB 扇区，16KB = 4 个扇区（原子性）
  - 够大：装得下多条记录（减少 IO）
  - 够小：单页加锁粒度不过粗
  - innodb_page_size 可配 4K/8K/16K/32K/64K（一般不改）
```

### 2.2 INDEX Page 布局（最关键）

```
┌─────────────────────────┐
│ File Header (38B)       │ Page 类型 / LSN / Checksum
├─────────────────────────┤
│ Page Header (56B)       │ 记录数 / Free Slot / Level...
├─────────────────────────┤
│ Infimum + Supremum (26B)│ 虚拟最小 / 最大记录（哨兵）
├─────────────────────────┤
│ User Records (变长)      │ 真实数据行
├─────────────────────────┤
│ Free Space              │ 剩余空间
├─────────────────────────┤
│ Page Directory (4B × N) │ 稀疏索引（slot 4-8 行一组）
├─────────────────────────┤
│ File Trailer (8B)       │ 与 File Header Checksum 对账
└─────────────────────────┘

Page Directory 作用:
  单页内二分查找
  4-8 行一个 slot，定位组后组内链表扫描
```

### 2.3 记录格式 (COMPACT)

```
┌──────────────┬──────────────┐
│ Record Header│ Column Data  │
└──────────────┴──────────────┘

Record Header (5B):
  - Next Record Pointer (2B) 单链表
  - Record Type (3 bits)    0=普通 / 1=B+ 非叶 / 2=Infimum / 3=Supremum
  - Deleted Flag (1 bit)    延迟删除标记
  - Heap Number (13 bits)   堆位置

Column Data:
  - 变长字段长度列表
  - NULL 位图
  - 隐藏列: DB_TRX_ID(6B) DB_ROLL_PTR(7B) DB_ROW_ID(6B)
  - 真实列数据
```

---

## 三、Buffer Pool 深度

### 3.1 整体结构

```
Buffer Pool 是 InnoDB 核心内存区:
  size = innodb_buffer_pool_size (建议内存 50-75%)
  划分为多个 instance（减少锁竞争）
    innodb_buffer_pool_instances 默认 8
  每个 instance 独立 LRU / Free / Flush 链表

单 instance 内部:
  Chunk (128MB) → 多个 Chunk 组成 instance
```

### 3.2 三大链表

```
Free List:
  未使用的空闲 page
  新 page 从这里取

LRU List:
  缓存中的 page（含脏 + 干净）
  分 young / old 两个子区域

Flush List (Dirty List):
  脏页（被改过未刷盘）
  按 oldest modification LSN 排序
  Checkpoint 按这个顺序刷盘
```

### 3.3 LRU 的改进（young/old）

```
传统 LRU 问题:
  全表扫描读入大量冷数据
  → 把热点页挤出 LRU 头
  → Buffer Pool 污染

InnoDB 改进:
  LRU 分两段:
    young 区域 (5/8) - 真正热点
    old 区域   (3/8) - 新读入 / 全表扫候选

  innodb_old_blocks_pct = 37    # old 区域占比
  innodb_old_blocks_time = 1000 # 新 page 在 old 区域待 1s 才能进 young
```

**关键参数**：

```
innodb_old_blocks_time 防扫污染:
  同一个 page 1s 内多次访问 → 不提升到 young（扫描特征）
  > 1s 后再访问 → 提升到 young（真正热点）
```

### 3.4 脏页刷盘时机

```
1. LRU 淘汰时
   LRU 尾部 page 如果是脏页，先刷再淘汰

2. Checkpoint
   周期性把 Flush List 头部（最老）刷盘
   推进 checkpoint LSN，允许 redo log 覆盖

3. 主线程定期刷
   后台线程每秒检查脏页比例
   innodb_max_dirty_pages_pct 默认 75%

4. 强制刷盘（shutdown / redo 满）
```

### 3.5 Double Write Buffer

```
问题:
  Page 16KB，磁盘原子写是 4KB
  写 16KB 时可能只写了部分 → 半页（torn page）

解决:
  脏页先写 Double Write Buffer（共享表空间 2MB 区域）
  再写真正的数据文件
  崩溃时用 Double Write 恢复半页

代价:
  每次脏页刷盘 2 次 IO
  （但是顺序写 + 批量，影响 < 10%）

SSD 上可关闭:
  innodb_doublewrite = 0
  （风险：假设 SSD 原子写）
```

---

## 四、Latch 与 Mini-Transaction

### 4.1 Latch（闩锁）vs Lock（锁）

```
Lock（事务锁）:
  - 行锁 / 表锁 / 意向锁
  - 事务级别
  - MVCC 可见性
  - 死锁检测

Latch（闩锁）:
  - 内部数据结构的互斥（极短时间）
  - 毫秒 / 微秒级
  - 保护 Buffer Pool / B+ 树页 / 链表
  - 无死锁检测，需要按固定顺序获取

类比:
  Lock = 业务层互斥
  Latch = Java 的 synchronized（对象内部锁）
```

### 4.2 Latch 类型

```
RW Latch:
  读 S / 写 X
  支持 SX（意向写 + 允许读）

常见 Latch:
  - Buffer Pool chunk latch
  - LRU mutex
  - Page latch（每 page 一把）
  - Index latch（B+ 树根 / 非叶）
  - AHI latch
  - Log latch
```

### 4.3 Mini-Transaction (MTR)

```
MTR 是内部最小的原子操作单元:
  一次 MTR 可能修改多个 page
  写 redo log → 事务 commit 前不一定持久化
  释放 latch

一个事务包含多个 MTR:
  Transaction
    ├── MTR: UPDATE row X
    │   ├── Latch page P1
    │   ├── Modify P1
    │   ├── Write redo log for P1
    │   └── Release latch
    ├── MTR: INSERT row Y (SMO)
    │   ├── Latch page P2, P3, P4 (split)
    │   ├── Modify all
    │   ├── Write redo
    │   └── Release latches
    └── Commit: flush redo (WAL)
```

### 4.4 为什么分 MTR

```
目的:
  减小 latch 持有时间
  减小 redo log 写入粒度
  崩溃恢复时粒度清晰（每个 MTR 原子）

代价:
  一个事务多个 MTR → redo log 交错
  需要 MLOG_MULTI_REC_END 标记
```

---

## 五、B+ 树操作

### 5.1 查找

```
从根节点开始:
  ① Latch(root) S 模式
  ② 二分找子节点
  ③ Latch(child) S 模式
  ④ 释放 parent latch (latch coupling / 蟹式锁)
  ⑤ 重复直到叶
  ⑥ 叶内 Page Directory 定位组 → 组内扫描

IO 次数 = 树高
  3 层 → 最多 3 次 IO（根 + 中间通常在 Buffer Pool）
```

### 5.2 Latch Coupling（蟹式锁 / 手递手锁）

```
目的:
  避免整棵树锁住
  并发查找 / 修改

规则:
  下降时:
    先 Latch 子节点
    再释放父节点
  上升时（split）:
    反向

名字来源:
  像螃蟹走路 —— 先移动一只脚，再另一只
```

### 5.3 插入 + 页分裂 (SMO)

```
Structure Modification Operation:
  插入数据满页时触发

步骤:
  ① 乐观插入
     从根到叶只持有 S latch
     叶页如果够空间 → 直接插入 → 成功

  ② 悲观插入（页分裂）
     叶页满 → 需要 SMO
     重新从根开始，全树 X latch
     分裂：创建新页，数据分成两半
     更新父节点 key 指针
     如果父也满 → 递归分裂
     极端情况 → 树高 +1

为什么要悲观 + 重试:
  分裂影响整棵树结构
  必须排他锁 → 并发差
  大部分插入不分裂 → 乐观策略快
```

### 5.4 页合并

```
DELETE 时:
  标记删除（deleted flag），不立即回收
  purge 线程异步清理

当页利用率 < MERGE_THRESHOLD (默认 50%):
  尝试与兄弟页合并
  合并触发 SMO（类似分裂反向）

合并引起问题:
  频繁 INSERT / DELETE 交替
  → 反复分裂 + 合并
  → 性能抖动
  → OPTIMIZE TABLE 重整
```

### 5.5 顺序插入 vs 随机插入

```
顺序（自增主键）:
  新数据总是插入最右页
  不会触发中间页分裂
  IO 局部性好

随机（UUID / 无序主键）:
  插入位置随机
  中间页频繁分裂
  Buffer Pool 污染
  → 主键必须顺序
```

---

## 六、锁算法

### 6.1 锁的物理表示

```
InnoDB 锁是 bitmap:
  每个 page 一个锁位图
  记录行号 → 锁状态

lock_t 结构:
  - trx_t *trx         事务
  - lock_type (LOCK_REC / LOCK_TABLE)
  - lock_mode (LOCK_S / LOCK_X / LOCK_IS / LOCK_IX)
  - 对象 (page_no / heap_no)
  - rec_bitmap[]
```

### 6.2 锁兼容矩阵

```
           IS   IX   S    X
    IS    ✓    ✓    ✓    ✗
    IX    ✓    ✓    ✗    ✗
    S     ✓    ✗    ✓    ✗
    X     ✗    ✗    ✗    ✗

意向锁 (IS/IX):
  表级标记"某行有锁"
  加行锁前先加意向锁（内部自动）
  意向锁之间兼容
```

### 6.3 Gap Lock / Next-Key Lock 源码视角

```
锁模式:
  LOCK_REC_NOT_GAP      行锁（不含 gap）
  LOCK_GAP              间隙锁
  LOCK_ORDINARY         Next-Key（行 + gap）
  LOCK_INSERT_INTENTION 插入意向锁

判断:
  RR + 非唯一索引范围查询 → LOCK_ORDINARY
  RR + 唯一索引等值命中   → LOCK_REC_NOT_GAP (退化)
  RR + 唯一索引等值不命中 → LOCK_GAP (退化)
  RC                     → LOCK_REC_NOT_GAP (无 gap)

插入逻辑:
  插入前检查:
    ① 目标位置是否有 GAP / ORDINARY 锁？
    ② 有 → 阻塞
    ③ 无 → 加 INSERT_INTENTION 锁 → 插入
```

### 6.4 死锁检测算法

```
数据结构:
  Wait-for Graph
    节点 = 事务
    边   = 等待关系

算法:
  新事务等待 → 从其节点做 DFS
  发现环 → 死锁
  回滚权重较小的事务（undo 量少）

性能:
  超大事务量 → 死锁检测成本高
  → innodb_deadlock_detect = OFF（极端优化）
  → 靠 innodb_lock_wait_timeout 被动超时
```

---

## 七、MVCC 底层

### 7.1 undo log 分类

```
insert undo log:
  记录"这一行是新插入的"
  事务回滚: 直接删除
  事务提交: undo 即可释放（没有可见性需求）

update undo log:
  记录"修改前的版本"
  事务回滚: 用它恢复旧值
  事务提交: 不能立即删（其他事务 Read View 可能还要用）
  → purge 线程延迟清理
```

### 7.2 undo 链结构

```
行 V3 (当前)
  ├── DB_TRX_ID = T3
  └── DB_ROLL_PTR → V2
                      ├── DB_TRX_ID = T2
                      └── DB_ROLL_PTR → V1
                                          ├── DB_TRX_ID = T1
                                          └── DB_ROLL_PTR → null
```

### 7.3 可见性判断详解

```c
// 伪代码
bool visible(row_version_t *rv, read_view_t *rv_view) {
    if (rv->trx_id == rv_view->creator_trx_id) {
        return true;  // 自己改的
    }
    if (rv->trx_id < rv_view->low_limit_id &&
        rv->trx_id < rv_view->up_limit_id) {
        return true;  // 所有历史事务早已提交
    }
    if (rv->trx_id >= rv_view->low_limit_id) {
        return false;  // 未来事务
    }
    // 在活跃列表？
    if (bsearch(rv->trx_id, rv_view->ids) != NULL) {
        return false;  // 还活跃
    }
    return true;  // 已提交
}
```

### 7.4 purge 线程

```
职责:
  清理已提交事务的 undo
  清理被标记删除的索引条目（delete mark）

判断"可以清理":
  history_list_length > 0
  最老的 undo 不被任何活跃 Read View 引用

相关参数:
  innodb_purge_threads = 4       purge 线程数
  innodb_max_purge_lag = 0       无限制
  innodb_max_purge_lag_delay = 0 超过 lag 强制延迟

监控:
  SHOW ENGINE INNODB STATUS\G
  → History list length: N
  > 几十万 → 长事务 / purge 跟不上

优化长事务:
  INFORMATION_SCHEMA.INNODB_TRX
  按 TRX_STARTED 排序
  找出执行 > 1h 的事务 → KILL
```

---

## 八、Change Buffer

### 8.1 目的

```
问题:
  非唯一二级索引更新
  目标 page 不在 Buffer Pool
  → 要先读 page（随机 IO）再修改

优化:
  不立即读 page
  先把变更缓存到 Change Buffer
  之后访问时合并
  → 减少随机 IO
```

### 8.2 适用条件

```
必须:
  ✓ 二级索引（不是主键）
  ✓ 非唯一索引（唯一索引必须立即读 page 校验唯一性）

不适用:
  ✗ 主键 / 唯一索引
  ✗ 自适应哈希索引

适用操作:
  INSERT / UPDATE / DELETE 的二级索引变更
```

### 8.3 参数

```
innodb_change_buffer_max_size = 25  Buffer Pool 中 CB 占比上限
innodb_change_buffering = all       缓存哪些操作 (all/inserts/deletes/purges/changes)

监控:
  SHOW ENGINE INNODB STATUS\G
  → Ibuf: size, free list len, seg size
```

### 8.4 合并时机

```
① 随机读到该 page 时
② 后台线程定期合并
③ Buffer Pool 空间紧张
④ MySQL 关闭时 (如果 innodb_fast_shutdown=0)
```

---

## 九、Adaptive Hash Index (AHI)

### 9.1 原理

```
InnoDB 对热点查找自动建 Hash 索引:
  监控 B+ 树访问模式
  某 page 被频繁等值查询
  → 为该 page 的某列建 Hash
  → B+ 查找从 O(log N) 降到 O(1)

完全自动:
  无需配置 / 索引定义
  innodb_adaptive_hash_index = ON (默认)
```

### 9.2 何时有效 / 何时拖累

```
有效:
  等值查询频繁（SELECT WHERE k = 1）
  访问模式固定

拖累:
  LIKE / 范围查询（用不上 Hash）
  频繁修改（Hash 维护成本）
  AHI latch 竞争（高并发单点）

线上调优:
  一般默认开启
  出现 AHI latch 高竞争 → 关闭
  SET GLOBAL innodb_adaptive_hash_index = OFF;
```

---

## 十、redo log 深度

### 10.1 redo log 组成

```
物理层:
  innodb_log_group_home_dir/  
    ib_logfile0
    ib_logfile1
    (8.0.30+ 改为 #innodb_redo/ #ib_redoN)

组成:
  innodb_log_file_size × innodb_log_files_in_group
  默认 48MB × 2 = 96MB
  生产建议 1-4GB × 2

WAL（Write-Ahead Logging）:
  日志先写 → 数据后写
  崩溃恢复靠 redo
```

### 10.2 LSN (Log Sequence Number)

```
LSN 是单调递增的日志位点:
  - 全局唯一
  - 每次写 redo 递增
  - Page 头部也记录最后修改的 LSN

用途:
  崩溃恢复: 从 checkpoint LSN 开始 redo
  一致性快照: 备份要记录 LSN
  Double Write / 增量备份
```

### 10.3 Log Buffer

```
内存中的 redo 缓冲:
  innodb_log_buffer_size 默认 16MB
  
写入时机:
  1. 事务 commit（受 innodb_flush_log_at_trx_commit 控制）
  2. buffer 满 1/3
  3. 每秒后台刷（即使没提交）
  4. checkpoint 触发
```

### 10.4 innodb_flush_log_at_trx_commit

```
= 1: 每次 commit fsync redo（不丢数据，默认）
= 0: 每秒刷，commit 不刷（崩溃丢 1s）
= 2: 每次 commit 写系统 buffer，每秒 fsync（宿主机不挂 ≈ 1）

线上选择:
  金融:    1（双 1 必开）
  主流:    1
  日志类:  2（丢 1s 可接受）
  测试:    0
```

### 10.5 崩溃恢复流程

```
① 启动时找最新 checkpoint LSN (cp_lsn)
② 从 cp_lsn 开始扫 redo log
③ 对每个 redo record
   - 对应 page 已经是新版本 → 跳过
   - 老版本 → apply redo
④ 所有 redo 重放完 → 一致性恢复
⑤ undo: 回滚未提交事务
⑥ 两阶段提交裁决:
   - redo prepare + binlog 完整 → commit
   - redo prepare + binlog 缺 → rollback
```

---

## 十一、各版本关键演进

```
5.5:
  InnoDB 成默认引擎
  独立表空间 innodb_file_per_table

5.6:
  Online DDL 引入
  全文索引
  GTID 复制
  ICP（Index Condition Pushdown）

5.7:
  JSON 数据类型
  generated columns
  无损在线 Alter
  sys schema
  多源复制
  并行复制（LOGICAL_CLOCK）

8.0:
  字典存到 InnoDB（原生数据字典）
  INSTANT Add Column（秒级加列）
  窗口函数 / CTE
  不可见索引 / 降序索引
  资源组 / 角色
  UTF8MB4 默认字符集
  InnoDB Cluster 增强
  hash join（8.0.18+）
  binlog_transaction_dependency_tracking = WRITESET（性能大幅提升）
```

---

## 十二、高频 Q/A（面试速答）

### Q1: 为什么 InnoDB 用 B+ 树不用 B 树 / 红黑树 / Hash？

见 [20-mysql-senior-answers.md 第二节](20-mysql-senior-answers.md)。**本篇补充**：

```
红黑树：树高 O(log2 N)，层数多
B+ 树：O(log N)，底数可以是上千（单页 1000+ 项）
→ 同样数据量，B+ 树层数是红黑树的 1/10
→ 磁盘 IO 少 10 倍
```

### Q2: Page 为什么 16KB？

见第 2.1 节。**关键**：磁盘原子 4KB + 减少 IO + 锁粒度。

### Q3: 查询一次要几次 IO？

```
标准答案:
  3 层 B+ 树 + 根 / 中间常驻 Buffer Pool
  → 实际 1-2 次 IO

边界:
  主键查询:       1-2 次
  二级索引 + 回表: 2-3 次
  覆盖索引:       1-2 次（无回表）
```

### Q4: 什么是 SMO？什么时候发生？

见第 5.3 节。

### Q5: 为什么顺序主键比随机主键快？

见第 5.5 节。

### Q6: MVCC 怎么实现的？

见第七章 + [20 第三节](20-mysql-senior-answers.md)。

### Q7: Change Buffer 是什么？

见第八章。

### Q8: Latch 和 Lock 区别？

见第 4.1 节。

### Q9: 什么是 Mini-Transaction？

见第 4.3 节。

### Q10: redo log 崩溃恢复流程？

见第 10.5 节。

---

## 十三、面试加分点

- **Page = 16KB + Infimum/Supremum + Page Directory**
- **Latch vs Lock** 清晰区分
- **Mini-Transaction** 作为最小原子单元
- **Buffer Pool young/old + old_blocks_time** 防扫污染
- **Double Write Buffer** 解半页问题
- **SMO 乐观 / 悲观两种路径**
- **Latch Coupling 蟹式锁** 并发下降
- **Next-Key 锁 4 种模式**（ORDINARY / GAP / NOT_GAP / INSERT_INTENTION）
- **undo 分 insert / update 两类** + purge 延迟清理
- **history list length** 监控长事务
- **Change Buffer 只对非唯一二级索引**
- **AHI 自适应 Hash** 自动建
- **LSN + checkpoint + WAL** 崩溃恢复
- **8.0 INSTANT + WRITESET** 新特性

---

## 十四、推荐阅读

```
必读:
  《MySQL 技术内幕：InnoDB 存储引擎》姜承尧
  《高性能 MySQL》第 4 版（第 4-6 章）
  MySQL 8.0 官方文档（InnoDB Architecture）

进阶:
  MySQL 源码 (storage/innobase/)
  何登成《InnoDB 存储引擎》
  Percona 博客
  WL (Worklog) - MySQL 改进提案

实战:
  SHOW ENGINE INNODB STATUS\G
  INFORMATION_SCHEMA.INNODB_*
  PERFORMANCE_SCHEMA.*
  sys schema
```

---

## 十五、关联阅读

```
本目录:
- 01-architecture.md         架构总览
- 02-index.md                索引原理
- 03-transaction-lock-mvcc.md 事务并发
- 04-log.md                  redo/undo/binlog
- 13-innodb-buffer-pool.md   Buffer Pool
- 14-explain-optimizer.md    EXPLAIN
- 20-mysql-senior-answers.md 答题模板

跨模块:
- 04-redis/17-object-encoding-internals.md  Redis 源码（对偶）
- 04-redis/18-eventloop-memory-internals.md Redis 源码（对偶）
```
