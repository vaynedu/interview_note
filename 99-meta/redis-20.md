# Redis 资深面试题（20 题)

> 架构 / 数据结构 / 持久化 / 主从集群 / 缓存模式 / 分布式锁 / 调优 / 场景
>
> 格式：题目 / 标准答案 / 易错点 / 追问点 / 背诵版

## 目录

1. [Redis 为什么快？](#1-redis-为什么快)
2. [Redis 单线程模型？6.0 多线程？](#2-redis-单线程模型60-多线程)
3. [String 底层 SDS 怎么实现？](#3-string-底层-sds-怎么实现)
4. [List / Hash / ZSet 底层结构？](#4-list--hash--zset-底层结构)
5. [跳表 vs 红黑树？为什么 ZSet 用跳表？](#5-跳表-vs-红黑树为什么-zset-用跳表)
6. [RDB vs AOF？](#6-rdb-vs-aof)
7. [fork + COW 怎么工作？](#7-fork--cow-怎么工作)
8. [主从复制流程？全量 vs 增量？](#8-主从复制流程全量-vs-增量)
9. [哨兵 vs Cluster 区别？](#9-哨兵-vs-cluster-区别)
10. [Redis Cluster 16384 槽位为什么？](#10-redis-cluster-16384-槽位为什么)
11. [缓存穿透 / 击穿 / 雪崩怎么解？](#11-缓存穿透--击穿--雪崩怎么解)
12. [Cache-Aside 一致性怎么保？](#12-cache-aside-一致性怎么保)
13. [Redis 分布式锁（SETNX + Redlock）？](#13-redis-分布式锁setnx--redlock)
14. [Redlock 真的安全吗？](#14-redlock-真的安全吗)
15. [淘汰 8 策略？怎么选？](#15-淘汰-8-策略怎么选)
16. [大 Key / 热 Key 怎么解决？](#16-大-key--热-key-怎么解决)
17. [BigKey 删除为什么阻塞？](#17-bigkey-删除为什么阻塞)
18. [Redis 实现限流？](#18-redis-实现限流)
19. [Redis 实现排行榜 / 计数器 / 去重？](#19-redis-实现排行榜--计数器--去重)
20. [Redis 慢查询排查？](#20-redis-慢查询排查)

---

## 1. Redis 为什么快？

### 标准答案

```
1. 内存操作（vs 磁盘 100x）
2. 单线程（避免上下文切换 + 锁竞争）
3. 多路复用（Reactor + epoll，单线程处理万级连接）
4. 高效数据结构（SDS / listpack / quicklist / skiplist / hashtable）
5. C 语言实现 + 紧凑内存布局
```

**单线程不是限制**：单核 10 万 QPS，瓶颈在网络/内存，不在 CPU。

### 易错点
- 误以为单线程慢（其实瓶颈在 IO 不在 CPU）
- 多客户端串行（其实多路复用并发处理）

### 追问点
- 多核服务器怎么用？→ 多实例（每核一个）/ Cluster
- Redis 6.0 多线程？→ 仅 IO 多线程，命令执行仍单线程

### 背诵版
快 = **内存 + 单线程 + 多路复用 + 高效结构**。瓶颈在 IO 不在 CPU。**Redis 6 多线程仅 IO**，命令仍单线程。

---

## 2. Redis 单线程模型？6.0 多线程？

### 标准答案

**Redis 6 之前**：单 Reactor 单线程
- 主线程：accept 连接 + read 命令 + 执行 + write 响应
- 优点：无锁、简单、避免上下文切换

**Redis 6.0+**：单 Reactor + IO 多线程
- 主线程：accept + 命令执行（仍单线程）
- IO 线程：读 socket / 写 socket（多线程）
- 命令执行**仍单线程，保证无锁**

**收益**：网卡带宽打满场景下提升 2-3 倍 QPS。

### 易错点
- 误以为 6.0 命令执行也并行（仍单线程）
- 默认开启 IO 多线程（实际默认关，需配置）

### 追问点
- 为什么命令执行不多线程？→ 多线程要加锁，复杂度爆炸 + 性能反而下降
- 怎么开启？→ `io-threads 4` + `io-threads-do-reads yes`

### 背诵版
6 之前**单 Reactor 单线程**，6.0 加 **IO 多线程**（仅读写 socket），命令执行仍单线程**保证无锁**。

---

## 3. String 底层 SDS 怎么实现？

### 标准答案

**SDS（Simple Dynamic String）**：

```c
struct sdshdr {
    int len;       // 已用长度
    int free;      // 剩余空间
    char buf[];    // 实际字符
};
```

vs C 字符串：
- **O(1) 获取长度**（C 是 O(n) strlen）
- **二进制安全**（C 字符串遇 \0 就停）
- **避免缓冲区溢出**（C 需手动算长度）
- **空间预分配**（小于 1MB 翻倍，大于 1MB 多分 1MB）
- **惰性释放**（缩短不立即 free）

Redis 7 用更紧凑的 sdshdr5/8/16/32/64（按长度选 header 大小）。

### 易错点
- 误以为是 C 字符串（其实是 SDS 包装）
- 不知道二进制安全（可以存图片/Protobuf）

### 追问点
- 为什么不用 C 字符串？→ 长度 O(n) + 不安全 + 不二进制
- SDS 怎么扩容？→ < 1MB 翻倍，>= 1MB 加 1MB

### 背诵版
SDS = **len + free + buf**。优势：**O(1) 长度 + 二进制安全 + 预分配 + 惰性释放**。Redis 7 紧凑 sdshdr5/8/16/32/64。

---

## 4. List / Hash / ZSet 底层结构？

### 标准答案

| 类型 | 小数据 | 大数据 |
| --- | --- | --- |
| String | int / embstr / raw（SDS） | raw |
| List | listpack（Redis 7+） | quicklist（双向链表 + listpack） |
| Hash | listpack | hashtable |
| Set | intset / listpack | hashtable |
| ZSet | listpack | skiplist + hashtable |

**listpack**：紧凑连续内存，适合小数据（替代 ziplist，解决级联更新问题）。

**quicklist**：双向链表 + listpack（每节点一个 listpack），平衡内存和性能。

**ZSet 双结构**：
- skiplist：按 score 排序，支持范围查询（`ZRANGEBYSCORE`）
- hashtable：member → score 映射，O(1) 取 score

### 易错点
- 误以为 ZSet 只用 skiplist
- 不知道 ziplist 已被 listpack 替代（Redis 7）
- 大 Hash 仍用 listpack（内存 OK 但操作慢）

### 追问点
- listpack 比 ziplist 强在哪？→ 解决级联更新（ziplist 改一个字段触发后面全部更新）
- 何时从 listpack 转 hashtable？→ 元素数 > 128 或单元素 > 64 字节（可配置）

### 背诵版
**小用 listpack（紧凑），大用 hashtable / quicklist / skiplist**。**ZSet = skiplist + hashtable** 双结构。

---

## 5. 跳表 vs 红黑树？为什么 ZSet 用跳表？

### 标准答案

| | 跳表 | 红黑树 |
| --- | --- | --- |
| 实现 | 简单（多层链表） | 复杂（旋转 + 着色） |
| 范围查询 | 高效（链表遍历） | 较弱（中序遍历） |
| 平均复杂度 | O(log n) | O(log n) |
| 内存 | 平均 1.33 指针/节点 | 颜色位 + 指针 |
| 并发 | 易（无锁实现） | 难 |

**Redis 选跳表**：
- 范围查询（`ZRANGEBYSCORE`）是高频操作
- 实现简单（作者原话："跳表代码相比红黑树容易调试得多"）
- 内存灵活（可调节因子）

### 易错点
- 误以为红黑树更快（同 O(log n)，跳表反而更适合范围查询）
- 不知道跳表概率性（多层是概率决定）

### 追问点
- 跳表层数怎么定？→ 概率（如 1/4），最多 32 层（Redis 7 改 64 层）
- ZSet 为什么不只用 hashtable？→ hashtable 不支持范围查询

### 背诵版
**跳表简单 + 范围查询高效**。同 O(log n) 但更适合 ZSet 场景。**作者原话**：跳表比红黑树好调试。

---

## 6. RDB vs AOF？

### 标准答案

| | RDB（快照） | AOF（追加日志） |
| --- | --- | --- |
| 形式 | 二进制全量 | 命令日志 |
| 触发 | 定时 / SAVE / BGSAVE | 每次写命令追加 |
| 恢复速度 | **快**（直接 load） | 慢（重放命令） |
| 数据丢失 | 较多（取决于 save 策略） | 少（appendfsync 控制） |
| 文件大小 | 小 | 大（可 BGREWRITEAOF 压缩） |
| 性能影响 | fork（COW） | 写入开销 |

**appendfsync 三档**：
- `always`：每次写命令 fsync（最安全 + 最慢）
- `everysec`：每秒 fsync（**推荐**，丢 1 秒数据）
- `no`：操作系统决定（最快 + 风险高）

**混合持久化（Redis 4.0+）**：RDB 全量 + AOF 增量，恢复快 + 数据少丢。

### 易错点
- 只用 RDB（数据丢可接受才行）
- AOF 配 always（QPS 砍半）
- 不开 BGREWRITEAOF（AOF 文件爆炸）

### 追问点
- 怎么选？→ 缓存场景 RDB / 数据库场景混合持久化
- BGREWRITEAOF 怎么做？→ fork + 重放当前数据集生成最小 AOF

### 背诵版
**RDB 快照（小快但丢数据）/ AOF 日志（大慢但少丢）**。**appendfsync everysec** 推荐。**混合持久化（RDB+AOF）** 最佳。

---

## 7. fork + COW 怎么工作？

### 标准答案

**Redis BGSAVE / BGREWRITEAOF 流程**：

```
1. fork() 子进程（copy-on-write）
2. 子进程拥有"逻辑全量内存"（实际共享父进程页）
3. 子进程顺序写 RDB 文件
4. 父进程继续处理请求
5. 父进程修改某页 → 触发 COW 复制该页
```

**COW（写时复制）**：
- fork 后父子共享物理内存（仅复制页表）
- 父进程写某页时才复制该页
- 子进程持续看快照视图

**陷阱**：
- 写入比例高时 COW 复制大量页 → **内存翻倍风险**
- fork 本身耗时（几十 ms），大实例阻塞主线程

**vm.overcommit_memory = 1** 防 fork 失败。

### 易错点
- 误以为 fork 立即复制内存（其实是 COW）
- 不调 overcommit_memory（OOM 时 fork 失败）
- 大实例不分片（fork 阻塞数百 ms）

### 追问点
- 怎么减少 fork 影响？→ 实例 < 4GB / 增大 stop-writes-on-bgsave-error 容忍
- huge page 影响？→ THP 应关闭（COW 单元从 4KB 变 2MB 严重放大）

### 背诵版
fork = **COW 写时复制**。父子共享物理页，写时复制。**写入多内存翻倍**风险。**关 THP + 实例 < 4GB**。

---

## 8. 主从复制流程？全量 vs 增量？

### 标准答案

**全量复制**：
1. 从节点 → `PSYNC ? -1`（首次）
2. 主节点 BGSAVE → 生成 RDB
3. 主节点发 RDB 到从
4. 主节点把复制期间的写命令暂存到 `replication backlog`
5. 从加载 RDB
6. 主把 backlog 中的命令发给从

**增量复制（Partial Resync）**：
- 从节点 `PSYNC <runid> <offset>`
- 主节点检查 offset 在 backlog 内 → 只发缺的部分
- offset 不在 backlog → 退化为全量

**关键参数**：
- `repl-backlog-size`：增量复制 buffer 大小（默认 1MB，断网久要加）
- `replicaof` / `slaveof`：配主从

### 易错点
- backlog 太小（断网稍久就退化全量，灾难）
- runid 误以为是 IP（其实是 master 启动时生成的 ID）
- 不监控复制延迟（从落后主太多）

### 追问点
- 复制延迟怎么监控？→ `INFO replication` 看 master_repl_offset 和 slave_repl_offset 差
- 一主多从怎么减少主压力？→ 树状复制：从可作为下一级从的 master

### 背诵版
**首次全量（RDB+backlog）**，**断连后增量（PSYNC offset）**，offset 不在 backlog 退化全量。**backlog 必须够大**。

---

## 9. 哨兵 vs Cluster 区别？

### 标准答案

| | 哨兵 Sentinel | Cluster |
| --- | --- | --- |
| 数据 | **单 master**（不分片） | **分片 16384 槽位** |
| 高可用 | 主挂自动切从 | 主挂自动切从 + 数据分片 |
| 容量 | 受单机限制 | 水平扩展 |
| 客户端 | 哨兵地址 | Cluster 客户端（智能路由） |
| 适合 | 中小数据量 + HA | 大数据量 + HA |

**哨兵作用**：监控 master 健康 → master 挂了选举新 master → 通知客户端。

**Cluster**：分片（CRC16 % 16384 → 槽位 → 节点） + 每分片一主多从 + 自动故障转移。

### 易错点
- 数据量小上 Cluster（杀鸡用牛刀）
- 数据量大用哨兵（容量不够）
- Cluster 跨槽位事务（不支持，要 Hash Tag）

### 追问点
- Hash Tag 是什么？→ `{user1}.profile` 和 `{user1}.order` 同槽位
- Cluster 怎么扩容？→ 在线迁移槽位

### 背诵版
**哨兵单 master + 自动切主**，**Cluster 分片 + 自动切主**。中小用哨兵，大数据用 Cluster。

---

## 10. Redis Cluster 16384 槽位为什么？

### 标准答案

**16384 = 2^14**。

为什么不更大？
- 集群最大 1000 节点
- 心跳包带 bitmap（位图）通知槽位归属
- **65536 槽位** → bitmap 8KB，每秒心跳 → 带宽爆
- **16384 槽位** → bitmap 2KB，可接受
- 16384 / 1000 ≈ 16 槽位/节点，足够细

为什么不更小？
- 太少分片不够细
- 难以平衡

**作者原话（antirez）**：心跳包大小是关键，1000 节点用 16384 槽位带宽 OK。

### 易错点
- 误以为 16384 是性能上限（其实是心跳包大小决定）
- 误以为可以改（不能改，写死的）

### 追问点
- 为什么不用一致性 Hash？→ 一致性 Hash 不支持精确控制槽位归属（迁移更复杂）
- 1000 节点上限怎么打破？→ 没有官方支持，业内用 Codis / Twemproxy 代理方案

### 背诵版
**16384 = 2^14**，平衡**心跳包大小**和**分片粒度**。原因：1000 节点上限 + bitmap 2KB 带宽 OK。

---

## 11. 缓存穿透 / 击穿 / 雪崩怎么解？

### 标准答案

| 问题 | 现象 | 解法 |
| --- | --- | --- |
| **穿透** | 查不存在 key 一直打 DB | 布隆过滤器 / 缓存空值（短 TTL） |
| **击穿** | 热点 key 过期 N 个并发查 DB | 互斥锁 / 永不过期 / 提前刷新 |
| **雪崩** | 大量 key 同时过期 | 过期时间随机化 / 多级缓存 / 限流 |

**布隆过滤器**：
- 优势：空间小（1 亿 key 约 100MB）
- 代价：有假阳性（说存在可能不存在），无假阴性

**互斥锁防击穿**：
```go
val, _ := redis.Get(key)
if val == nil {
    if redis.SetNX(lockKey, 1, 10*time.Second) {
        defer redis.Del(lockKey)
        val = db.Query(key)
        redis.Set(key, val, ttl)
    } else {
        time.Sleep(100*time.Millisecond)
        return cache.Get(key)
    }
}
```

### 易错点
- 不缓存空值（穿透）
- TTL 全部一样（雪崩）
- 互斥锁等待死循环（要超时）

### 追问点
- 布隆过滤器缺点？→ 假阳性（说有可能没）+ 不能删除元素
- 永不过期防击穿怎么更新？→ 后台定时刷新 / 业务方主动失效

### 背诵版
**穿透布隆/空值，击穿互斥锁/不过期，雪崩随机 TTL/多级**。**用 Cache-Aside + 三防**。

---

## 12. Cache-Aside 一致性怎么保？

### 标准答案

**Cache-Aside 标准操作**：
- 读：先查缓存，miss 查 DB 写回
- 写：**先更新 DB，再删缓存**

**为什么删而不更新？**
- 更新缓存：并发写 → 缓存可能是旧值
- 删除缓存：下次读重新加载，必然新值

**为什么先 DB 再删缓存？**
- 反过来（先删缓存再 DB）：删完后另一线程读 → 写老值进缓存
- 顺序：DB 成功 → 删缓存（删失败重试）

**仍可能不一致**：
- 时序问题（A 删缓存前 B 读到旧 DB 值并写缓存）
- 解法：**延迟双删**（先删 → DB 更新 → sleep → 再删）

### 易错点
- 先删缓存再更新 DB（并发问题）
- 不删缓存（永远脏）
- 写缓存而非删缓存（写入复杂数据时算错）

### 追问点
- 强一致怎么保？→ 加锁（性能低）/ 订阅 binlog 异步删
- 延迟双删 sleep 多久？→ 业务最大读耗时（500ms-1s）

### 背诵版
**先 DB 再删缓存**，**删而不更新**。**延迟双删**应对时序问题。强一致用 binlog 订阅。

---

## 13. Redis 分布式锁（SETNX + Redlock）？

### 标准答案

**单实例锁**：
```
SET key value NX PX 10000
  NX: 不存在才设
  PX: 毫秒级 TTL
  value: 唯一 ID（防误删别人锁）

释放: Lua 脚本（GET + DEL 原子）
  if redis.call('get', key) == value then
    return redis.call('del', key)
  else
    return 0
  end
```

**关键点**：
1. 必须有 TTL（防死锁）
2. value 唯一（防误释放）
3. 释放用 Lua（原子操作）
4. 业务执行可能超时 → 加锁续约（看门狗）

**Redlock**（多实例）：
- 5 个独立 Redis 节点
- 客户端依次申请锁（超时短）
- 多数（3/5）成功 + 总耗时 < TTL → 加锁成功
- 释放：所有节点都释放

### 易错点
- 没 TTL（业务挂死锁永久）
- value 不唯一（B 释放了 A 的锁）
- 释放不用 Lua（GET 后 DEL 之间被改）

### 追问点
- 锁续约怎么做？→ Redisson 看门狗：每 1/3 TTL 续期
- 业务超时怎么办？→ 短锁 + 续约 / 业务幂等设计

### 背诵版
`SET key value NX PX 10000`，**value 唯一 + Lua 释放**。Redisson **看门狗续约**。Redlock 多实例多数派。

---

## 14. Redlock 真的安全吗？

### 标准答案

**Redlock 设计**（Redis 作者）：5 个独立节点，多数派加锁。

**Martin Kleppmann 质疑**（《How to do distributed locking》）：
1. **时钟假设**：依赖各节点时钟同步，不可靠（GC pause / VM 暂停 / NTP 跳变）
2. **fencing token 缺失**：锁过期 + 业务慢 → 锁释放被另一进程拿到 → 两个进程都觉得自己持有锁

**业界共识**：
- 普通业务用 Redlock 够了（接受偶发不一致）
- 强一致场景用 **ZooKeeper / etcd**（Raft 共识 + fencing token）
- 关键交易用**乐观锁 + 业务幂等**双保险

### 易错点
- 误以为 Redlock = 完美（其实有理论缺陷）
- 强一致场景用 Redlock（应该用 etcd / ZK）
- 不做幂等（依赖锁绝对安全）

### 追问点
- 什么是 fencing token？→ 单调递增 ID，业务持锁时带 token，存储层校验 token 大小
- ZK 锁怎么做？→ 临时顺序节点 + watch

### 背诵版
**Redlock 普通业务够，强一致用 etcd/ZK**。理论缺陷：**时钟假设 + 缺 fencing token**。**幂等是终极保险**。

---

## 15. 淘汰 8 策略？怎么选？

### 标准答案

```
noeviction:           不淘汰，写入返错（默认）
allkeys-lru:          所有 key LRU 淘汰
allkeys-lfu:          所有 key LFU 淘汰
allkeys-random:       所有 key 随机
volatile-lru:         设了 TTL 的 LRU（仅这部分淘汰）
volatile-lfu:         设了 TTL 的 LFU
volatile-random:      设了 TTL 的随机
volatile-ttl:         设了 TTL 的 按 TTL 越短越先淘汰
```

**选择**：
- **缓存场景**（数据可丢）：`allkeys-lru` 或 `allkeys-lfu`
- **数据库场景**（数据不能丢）：`noeviction`
- **混合**（部分必保留）：volatile-* 类

**LFU vs LRU**：
- LRU：最近最少使用（按时间）
- LFU：最不经常使用（按频率），适合有真热点的场景

### 易错点
- 默认 noeviction 内存满写入失败
- 缓存场景用 noeviction（应该 allkeys-lru）
- 数据库场景用 allkeys-lru（数据丢失）

### 追问点
- LRU 实现？→ Redis 用近似 LRU（采样 5 个 key 选最旧），减少内存
- LFU 怎么做？→ 8 位 counter + 概率衰减

### 背诵版
**8 策略**：noeviction / allkeys-{lru,lfu,random} / volatile-{lru,lfu,random,ttl}。**缓存 allkeys-lru，数据库 noeviction**。

---

## 16. 大 Key / 热 Key 怎么解决？

### 标准答案

**大 Key 危害**：
- 阻塞主线程（删除 / 序列化）
- 网络流量爆
- 内存倾斜
- 主从复制慢

**解法**：
- 拆分大 Hash → 分桶 Hash（`user:{uid}:profile:{shard}`）
- 大 ZSet → 分时段
- 大 Value → 压缩 / 拆段
- **UNLINK 异步删**（替代 DEL）
- BigKey 扫描发现（`redis-cli --bigkeys`）

**热 Key 危害**：
- 单节点 QPS 打爆
- 集群分片不均

**解法**：
- 本地缓存（应用进程内 Caffeine / sync.Map）
- 多副本（写多份 `key:1` `key:2`，随机读）
- 拆分（按时间窗 / 业务维度）

### 易错点
- 用 DEL 删大 Key（阻塞）
- 不监控 BigKey（事后发现）
- 热 Key 不做本地缓存（Redis 单节点炸）

### 追问点
- 怎么发现大 Key？→ `--bigkeys` / scan + memory usage / 监控 latency
- 热 Key 怎么发现？→ `--hotkeys`（需要 LFU）/ 抽样统计

### 背诵版
**大 Key 拆分 + UNLINK 异步删 + 本地缓存**。**热 Key 本地缓存 + 多副本 + 拆分**。监控发现是前提。

---

## 17. BigKey 删除为什么阻塞？

### 标准答案

Redis 单线程，**删除大 Key 阻塞主线程**。

**示例**：
- 删除 100 万元素的 List → 主线程释放每个元素的内存 → 几百 ms
- 期间 Redis 不响应任何请求 → 业务超时

**解法**：
- **UNLINK**（Redis 4.0+）：异步删除，主线程仅断引用，**后台 BIO 线程释放**
- 大 Hash 用 HSCAN + HDEL 分批删
- 大 List 用 LTRIM 分段删
- 大 ZSet 用 ZREMRANGEBYRANK 分段删

```bash
UNLINK bigKey  # 异步，O(1) 立即返回
```

**BIO（Background IO）线程**：Redis 异步任务线程，处理 fsync / 异步删等。

### 易错点
- 用 DEL 删大 Key（阻塞业务）
- 不知道 UNLINK 的存在
- 误以为 UNLINK 也是 O(N)

### 追问点
- BIO 线程做什么？→ 异步删除 / fsync / Lazy Free
- lazy-free 配置？→ `lazyfree-lazy-eviction yes` 等让 Redis 主动 lazy free

### 背诵版
**DEL 单线程同步阻塞，UNLINK 异步 BIO**。Redis 4+ 优先用 UNLINK。配 `lazyfree-lazy-*` 让 eviction/expire 也异步。

---

## 18. Redis 实现限流？

### 标准答案

**固定窗口**（简单但有突刺）：
```lua
-- KEYS[1] = limit_key, ARGV[1] = limit, ARGV[2] = window
local cur = redis.call('incr', KEYS[1])
if cur == 1 then redis.call('expire', KEYS[1], ARGV[2]) end
if cur > tonumber(ARGV[1]) then return 0 end
return 1
```

**滑动窗口**（ZSet）：
```lua
-- 用时间戳作 score
redis.call('zadd', KEYS[1], now, now)
redis.call('zremrangebyscore', KEYS[1], 0, now - window)
local count = redis.call('zcard', KEYS[1])
if count > limit then return 0 end
return 1
```

**令牌桶**（Lua + 时间差计算）：精度高，业界主流。

**Redis-Cell 模块**：原生令牌桶，`CL.THROTTLE key max_burst tokens period`。

### 易错点
- 用多次命令实现（非原子，并发不准）→ 必须 Lua
- 固定窗口忽视边界突刺（第 1s 末和第 2s 初共 2x 流量）
- 滑动窗口 ZSet 数据不清（内存爆）

### 追问点
- 集群模式怎么做？→ key 上 Hash Tag 落同一槽位
- 限流 vs 熔断？→ 限流入口控量，熔断保护下游

### 背诵版
**固定窗口（Lua incr+expire）/ 滑动窗口（ZSet）/ 令牌桶（Lua+时间）**。集群用 **Hash Tag**。Redis-Cell 模块原生支持。

---

## 19. Redis 实现排行榜 / 计数器 / 去重？

### 标准答案

**排行榜**：ZSet
```
ZADD rank 100 user1
ZADD rank 200 user2
ZREVRANGE rank 0 9 WITHSCORES   # Top 10
ZRANK rank user1                # 用户排名
```

**计数器**：String INCR / HINCRBY
```
INCR view:article:123        # 单计数
HINCRBY user:stats:1 views 1 # 多字段
```

**去重**：
- **Set**：精确，但内存大
- **Bitmap**：超紧凑（1 亿 user 12.5MB）
- **HyperLogLog**：去重计数，固定 12KB，误差 0.81%
- **Bloom Filter**（RedisBloom 模块）：判断是否存在

**例**：每日 UV
```
PFADD uv:2026-05-05 user1 user2 ...
PFCOUNT uv:2026-05-05
```

### 易错点
- 用 SET 做亿级去重（内存爆）
- 用 INCR 做精确分布式 ID（多实例冲突，要用号段）
- HLL 当精确（有误差）

### 追问点
- HLL 误差从哪来？→ 概率算法，0.81% 标准误差
- Bitmap 怎么做？→ user_id 作 offset，SETBIT key user_id 1

### 背诵版
**排行榜 ZSet / 计数 INCR / 去重 Set/Bitmap/HLL/BF**。**HLL 12KB 亿级去重 0.81% 误差**。

---

## 20. Redis 慢查询排查？

### 标准答案

**slowlog 配置**：
```
slowlog-log-slower-than 10000   # 微秒，10ms 以上记录
slowlog-max-len 128
SLOWLOG GET 10                  # 看最近 10 条慢查询
```

**常见慢命令**：
- `KEYS *`（全表扫描，**禁用！**）
- `HGETALL` 大 Hash（O(N)）
- `SMEMBERS` 大 Set
- `ZRANGE 0 -1` 大 ZSet
- `LRANGE 0 -1` 大 List
- 大 Key 删除（DEL）
- Lua 脚本卡住

**排查工具**：
- `SLOWLOG GET`
- `redis-cli --latency-history`
- `redis-cli --bigkeys` / `--hotkeys`
- `MONITOR`（仅调试，**生产慎用**）

**优化**：
- KEYS → SCAN
- HGETALL → HMGET
- 大 List/ZSet → 分页（LRANGE start stop）
- 大 Key 拆分

### 易错点
- 生产用 KEYS *（阻塞主线程）
- 生产用 MONITOR（QPS 砍半）
- 不开 slowlog（事后没数据）

### 追问点
- 怎么用 SCAN 替代 KEYS？→ `SCAN cursor MATCH pattern COUNT 100` 增量扫描
- Lua 脚本超时？→ `lua-time-limit` 控制，超时 SHUTDOWN NOSAVE / SCRIPT KILL

### 背诵版
**slowlog 配 10ms** 监控。**禁 KEYS / HGETALL / 大范围 LRANGE/ZRANGE**。**SCAN 替代 KEYS**。生产**慎 MONITOR**。

---

## 复习建议

**面试前 1 天**：通读"背诵版"。

**面试前 1 周**：每天 3-5 题，结合 04-redis 各篇。

**实战检验**：
- 能不能讲清楚 ZSet 双结构 + 用跳表的原因？
- 能不能完整描述 Cache-Aside + 一致性方案？
- 能不能解释 Redlock 理论缺陷？
- 能不能给出大 Key / 热 Key 的发现 + 解决方案？
