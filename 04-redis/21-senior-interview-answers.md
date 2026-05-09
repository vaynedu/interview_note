# Redis 资深面试答题模板

> 把高频问题拆成 **"一句话 → 分层展开 → 边界 → 代价"** 四段式答题法。
>
> 本篇是面试**"开口就稳"**的压舱石。所有模板都按"资深视角"设计，能区分 P5/P6/P7。

---

## 一、为什么需要答题模板

```
同一个问题，答法差距可以是 3 个 Level：

"Redis 为什么快？"

P5: "因为单线程 + 内存"       （知其然）
P6: "内存 + IO 多路复用 + 高效结构"（知其所以然）
P7: "6 点原因 + 版本演进 + 边界 + 瓶颈在哪" （能讲清楚代价）

面试官问同一个问题，听的是你的"分层能力"。
```

### 1.1 通用答题四段式

```
① 一句话：最精炼的核心（15 字以内）
② 分层展开：2-5 个维度 / 层次
③ 边界 / 代价：什么时候失效？代价是什么？
④ 实战佐证：举一个真实场景或故障案例
```

---

## 二、Redis 为什么快（最高频）

### 2.1 一句话

> "基于内存 + 单线程避锁 + 多路复用 + 高效结构 + 网络模型优化。"

### 2.2 分层展开（6 点，背下来）

```
1. 内存 vs 磁盘
   读: 内存 < 100ns，磁盘 SSD ~100μs，差 1000x

2. 单线程避免锁竞争
   无锁 / 无上下文切换 / 无 CAS 失败
   命令执行原子

3. IO 多路复用（epoll / kqueue）
   一个线程处理万级连接
   Reactor 模式 + 非阻塞 IO

4. 高效数据结构
   SDS（O(1) 长度 + 预分配）
   listpack（紧凑连续内存）
   跳表（O(log N) + 易实现）
   dict 渐进式 rehash

5. 简单协议 RESP
   纯文本 + 二进制安全
   解析快、可 pipeline

6. 6.0+ IO 多线程
   网络读写多线程（不是命令执行）
   CPU 多核利用率提升
```

### 2.3 边界 / 代价

```
瓶颈不在 CPU 而在 IO:
  单线程可撑 10w QPS 读
  瓶颈通常是: 网络带宽 / 慢命令 / 大 key / 单分片热点

单线程的代价:
  - 一个慢命令拖住所有（KEYS * / HGETALL 大 Hash）
  - 大 value 阻塞
  - Lua 脚本长时间执行阻塞整库
```

### 2.4 实战佐证

```
我们线上 Redis Cluster 32 节点，单节点:
  - 15w QPS peak
  - P99 < 1ms
  - 主要耗时: 网络 RTT（客户端到 Redis）
  - 慢命令告警阈值: > 10ms
```

---

## 三、LRU 和 LFU：Redis 怎么做近似淘汰

### 3.1 一句话

> "Redis 的 LRU/LFU 都是**近似算法**：抽样 N 个 key，淘汰最差的。用精确双向链表代价太大。"

### 3.2 分层展开

```
为什么近似？
  精确 LRU = 全量 key 维护双向链表
  100w key 每次访问都要移动链表节点
  → 主线程不能这么干

Redis 做法（maxmemory-samples，默认 5）:
  1. 随机抽 N 个 key
  2. 算每个 key 的"空闲时间"
  3. 淘汰最旧的
  4. 加"淘汰池"（evictionPool）优化：保留历史候选

LFU（4.0+）:
  每个 key 带 8bit 计数器
  access 时有概率递增（对数递增，防止爆）
  一段时间不访问 → 衰减

LRU vs LFU 选择:
  LRU: 适合"最近访问即为热"
  LFU: 适合"长期高频即为热"（更适合缓存）
```

### 3.3 8 种淘汰策略

```
noeviction:      OOM 时报错（默认）
allkeys-lru:     所有 key LRU
allkeys-lfu:     所有 key LFU（推荐缓存场景）
allkeys-random:  所有 key 随机
volatile-lru:    只淘汰设了 TTL 的 LRU
volatile-lfu:    只淘汰设了 TTL 的 LFU
volatile-random: 只淘汰设了 TTL 的 随机
volatile-ttl:    只淘汰设了 TTL 的（最快过期的优先）
```

### 3.4 边界

```
采样数 = 5（默认）精度不够:
  调到 10 更接近真正 LRU，但 CPU 开销略大

LFU 有衰减参数:
  lfu-log-factor / lfu-decay-time
  默认值适合大部分场景

策略选择:
  缓存 → allkeys-lfu（推荐）
  混合 → volatile-lru（只淘汰缓存数据，业务数据保留）
  禁止丢数据 → noeviction（但业务要处理 OOM 错误）
```

---

## 四、Redis 和本地缓存怎么配

### 4.1 一句话

> "本地缓存抗超热点（ns 级）+ Redis 做中心缓存（跨实例一致）+ DB 兜底。多级缓存是性能优化的标配。"

### 4.2 分层展开

```
L0: 客户端缓存（Redis 6.0 Tracking）
L1: 进程内本地缓存（Ristretto / BigCache）
    - 延迟: ns 级
    - 容量: 100MB - 1GB
    - TTL: 短（5-30s）
L2: Redis 中心缓存
    - 延迟: sub-ms
    - 容量: GB-TB
    - TTL: 中长（5-30min）
L3: DB（source of truth）

读路径:
  L1 → L2 → L3（singleflight 合并回源）

写路径（Cache-Aside）:
  写 DB → 删 L2 → 发消息让所有实例删 L1
```

### 4.3 为什么要本地缓存

```
问题场景:
  某 key 10w QPS
  Redis Cluster 单分片打爆（12-15w QPS 极限）

解决:
  热点自动识别 → 镜像到本地
  本地 TTL 短（5-30s）控制不一致窗口
  Redis 仍然存（回源兜底）
```

### 4.4 边界 / 代价

```
本地缓存代价:
  ✗ 实例间不一致（N 份拷贝）
  ✗ 进程重启丢
  ✗ 占堆内存影响 GC
  ✗ 通知失效难（Pub/Sub / 消息 / 长轮询）

取舍:
  TTL 短 → 不一致窗口小 → 但命中率低
  TTL 长 → 命中率高 → 但业务可能看到旧数据

实战:
  动态数据（订单状态）→ 不放本地
  静态配置（商品信息）→ 本地 30s TTL OK
  超热点实时（商品详情）→ 本地 5-10s TTL + Pub/Sub 失效
```

---

## 五、缓存一致性怎么讲

### 5.1 一句话

> "**先写 DB 再删缓存**是主流，但不完美。强一致用 binlog 订阅，最终一致接受短时脏读。"

### 5.2 方案对比（核心）

| 方案 | 一致性 | 性能 | 复杂度 | 场景 |
| --- | --- | --- | --- | --- |
| 先删缓存再写 DB | 差（读覆盖脏数据）| 快 | 低 | 不推荐 |
| 先写 DB 再删缓存 | 较好 | 快 | 低 | **主流** |
| 延迟双删 | 较好 | 中 | 中 | 读写并发高 |
| binlog 订阅删缓存 | 最终一致 | 快 | 高 | 大规模场景 |
| 强一致读（读 DB）| 强 | 慢 | 低 | 金融场景 |

### 5.3 为什么"先写 DB 再删缓存"会不一致

```
时序问题:
  T1: 请求 A 读 miss → 查 DB 得 V1 → 准备写 L2
  T2: 请求 B 写 DB V2 → 删 L2
  T3: 请求 A 把 V1 写入 L2
  → L2 永久脏（V1）直到 TTL 过期

概率:
  要同时满足: 读 miss + 写在读和回填之间
  实际很罕见（读 miss 很快，写比删快）
  但不是 0
```

### 5.4 延迟双删

```go
// 写路径
write(v2)
rdb.Del(key)
time.Sleep(500 * time.Millisecond)  // 等读协程回填
rdb.Del(key)                        // 再删一次
```

**代价**：写路径慢 500ms，且仍不是强一致。

### 5.5 binlog 订阅（Canal / Debezium）

```
DB 写入 → binlog → Canal → MQ → 消费端删 Redis
优点: 解耦 + 删除可靠（重试 + 顺序）
缺点: 延迟（100ms-1s）+ 运维复杂
```

### 5.6 强一致读（兜底）

```go
// 某些路径直接读 DB，不走缓存
if needStrongConsistency {
    return db.Query(...)
}
return cache.Get(...)
```

### 5.7 边界 / 代价

```
没有"完美一致性 + 高性能"的方案
→ 业务接受多少不一致窗口？

实战经验:
  - 大部分业务: 先写 DB 再删缓存 + 较短 TTL（5-30min）
  - 高并发读写: binlog 订阅 + 本地 TTL 短
  - 金融强一致: 直接走 DB，不用缓存
  - 删缓存失败: MQ 重试（不能用 Redis 自己重试）
```

---

## 六、Redis 集群故障怎么回答

### 6.1 一句话

> "主从异步复制 + Cluster 哨兵切换，有写丢窗口。业务要幂等 + 对账兜底。"

### 6.2 主要故障场景

```
1. 主节点挂
   → 从节点晋升主（failover）
   → 30 秒内可能丢最后写入（异步复制）

2. 脑裂
   → 网络分区导致两个主
   → 客户端各自写 → 合并时丢一部分
   → min-replicas-to-write + min-replicas-max-lag 缓解

3. Cluster 节点挂
   → 16384 slot 重新分配
   → 客户端收到 MOVED / ASK 重定向

4. 慢命令阻塞
   → 单节点单线程 → 其他命令排队
   → 客户端超时 → 误判节点挂

5. 内存打满
   → maxmemory-policy 触发淘汰
   → 或 OOM killer 进程挂
```

### 6.3 failover 流程

```
① Sentinel/Cluster 探测主挂（quorum 投票）
② 选新主（数据最新 + 优先级）
③ 其他从改复制新主
④ 通知客户端（MOVED / +switch-master）
⑤ 客户端重连新主

丢数据窗口:
  主挂前已 ack 但未同步到从的写
  → 主恢复后不能合并（会被覆盖）
  → 需要业务幂等 + 对账补偿
```

### 6.4 业务如何兜底

```
幂等:
  业务订单号 + DB 唯一索引
  重复请求直接返回原结果

对账:
  定时任务对比 DB vs Redis
  异常触发告警 + 自动修复

降级:
  Redis 挂 → 直接读 DB（限流保护 DB）
  极端情况 → 返回兜底数据（默认推荐）

监控:
  Redis 副本延迟（master_last_io_seconds_ago）
  Cluster 节点状态（cluster_state: ok/fail）
  客户端 MOVED / ASK 频率
```

### 6.5 实战案例讲法

```
"我们线上有次主从切换:
  T+0: 主节点 CPU 打满（一个 HGETALL 大 Hash）
  T+5s: Sentinel 判定主挂 → 切从
  T+10s: 新主上任 → 客户端重连
  T+10s-12s: 这 2s 内的写有丢失

业务侧:
  - 订单写入 DB 成功 + Redis 缓存失败
  - 缓存与 DB 暂时不一致
  - 后台对账任务 5 分钟后发现 → 修复
  - 关键路径有幂等保护，无资损

复盘:
  - maxmemory 告警阈值降到 70%
  - 禁 HGETALL，全量扫走 HSCAN
  - 副本延迟告警加到 1s"
```

---

## 七、Redis 线上问题排查

### 7.1 一句话

> "按'现象 → 分类 → 工具 → 证据 → 止血 → 根因 → 修复'7 步走。"

### 7.2 常见症状 → 分类

```
慢: 慢命令 / 大 key / 网络 / IO 阻塞
错: OOM / 连接打满 / 命令错误
丢: 主从切换 / 过期 / 淘汰
不一致: 缓存与 DB / 主从延迟
QPS 异常: 热 key / 雪崩 / 业务放大
```

### 7.3 排查工具速查

```bash
# 慢命令
SLOWLOG GET 100
SLOWLOG RESET

# 大 key
redis-cli --bigkeys
redis-cli --memkeys       # 按内存大小

# 热 key（需 LFU + 4.0+）
redis-cli --hotkeys

# 实时命令流（小心，生产慎用）
MONITOR

# 全局指标
INFO all
INFO memory
INFO stats
INFO replication
INFO clients

# 延迟统计
redis-cli --latency
redis-cli --latency-history
redis-cli --latency-dist

# 集群健康
CLUSTER INFO
CLUSTER NODES
CLUSTER COUNTKEYSINSLOT <slot>

# 客户端
CLIENT LIST
CLIENT GETNAME
```

### 7.4 经典排查模板

```
症状: 某 Redis 实例 CPU 100%

Step 1: SHOW PROCESSLIST 类命令
  → INFO clients 看连接数
  → CLIENT LIST 看有没有阻塞命令

Step 2: SLOWLOG GET 20
  → 最近慢命令
  → 重点看 KEYS / HGETALL / SMEMBERS / ZRANGE 0 -1

Step 3: redis-cli --bigkeys
  → 找大 key（可能是根因）

Step 4: 止血
  - KILL 慢 SQL 对应客户端
  - 应用层限流
  - 紧急切流到从

Step 5: 根因
  - 业务代码有 HGETALL 大 Hash？
  - 某个 key 变成热点？
  - 内存快满触发淘汰？

Step 6: 修复
  - HGETALL → HSCAN
  - 热 key → 本地缓存镜像
  - 内存 → 扩容 / 淘汰策略调整

Step 7: 复盘
  - 告警加（慢命令 / CPU / 内存）
  - 代码 Review 加规则（禁 KEYS / HGETALL）
```

### 7.5 四大黄金信号

```
Latency（延迟）:
  命令 P99 > 10ms 告警
  redis-cli --latency

Traffic（流量）:
  QPS 突增 3x 告警
  INFO stats instantaneous_ops_per_sec

Errors（错误）:
  OOM / connection refused 告警
  客户端超时率 > 1%

Saturation（饱和度）:
  CPU > 80% / 内存 > 80%
  连接数 > maxclients * 0.8
  副本延迟 > 1s
```

---

## 八、Redis 分布式锁怎么讲

### 8.1 一句话

> "`SET NX PX` + Lua 释放 + 看门狗续约是标准。Redlock 有理论争议，金融场景用 etcd。"

### 8.2 完整方案

```go
// 加锁
token := uuid()
ok, _ := rdb.SetNX(ctx, key, token, 10*time.Second).Result()

// 释放（Lua 保证原子）
var unlockScript = redis.NewScript(`
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
end
return 0
`)
unlockScript.Run(ctx, rdb, []string{key}, token)

// 看门狗（业务超过 TTL 前续期）
go func() {
    ticker := time.NewTicker(3 * time.Second)
    for range ticker.C {
        rdb.Expire(ctx, key, 10*time.Second)
    }
}()
```

### 8.3 Redlock 争议

```
Antirez: 5 个独立 Redis 节点多数派加锁
Martin Kleppmann 反驳:
  - GC 停顿 → 锁超时 → 另一个拿到锁 → 两个同时执行
  - 时钟漂移 → 一个节点认为超时另一个没 → 不一致

共识:
  Redlock 解决了一部分问题，但不是"绝对安全"
  对于资损场景 → 不够
```

### 8.4 Fencing Token（强烈推荐）

```go
// 锁 + 版本号
token, version := acquireLock()
doBusiness(version)

// 下游服务校验 version
if request.version < lastSeenVersion {
    reject()  // 老锁请求，拒绝
}
```

### 8.5 三方对比

| 维度 | Redis | etcd | ZooKeeper |
| --- | --- | --- | --- |
| 一致性 | 最终 | 强 (Raft) | 强 (ZAB) |
| 性能 | 10w QPS | 1w QPS | 5k QPS |
| 运维 | 简单 | 中 | 复杂 |
| 典型场景 | 弱一致互斥 | K8s / 强一致 | 老牌协调 |

### 8.6 业务兜底

```
无论哪种锁:
  1. 业务幂等（唯一订单号 + DB 唯一索引）
  2. Fencing Token
  3. 关键路径加对账
  4. 锁只作为"性能优化"，不作为"正确性保证"
```

---

## 九、热 key / 大 key 怎么讲

### 9.1 一句话

> "热 key → 本地缓存镜像 + 读打散；大 key → 拆分 + HSCAN + UNLINK。两者都要提前识别。"

### 9.2 热 key 识别 + 处理

```
识别:
  - redis-cli --hotkeys（需 LFU）
  - 业务埋点统计
  - 客户端侧滑窗计数
  - Proxy 日志分析（如 Twemproxy / Codis）

处理:
  1. 本地缓存（L1）短 TTL
  2. Key 分片: item:{hash_user % 10}
  3. 读副本（多从分担）
  4. Cluster 热点自动迁移（企业版有此能力）
```

### 9.3 大 key 识别 + 处理

```
识别:
  - redis-cli --bigkeys
  - redis-cli --memkeys
  - MEMORY USAGE key
  - RDB 分析工具（rdb-tools / redis-rdb-tools）

处理:
  删除:
    DEL → 阻塞 → 主线程卡 1s+
    UNLINK → 异步删除 → 不阻塞（推荐）

  读取:
    HGETALL → 阻塞
    HSCAN 游标分批 → 不阻塞

  写入:
    一次写 100w field → 阻塞
    分批写 + Pipeline → OK

  根治:
    拆 key: user:123:profile → user:123:profile:{field_group}
    放 OSS: 大 value 存对象存储，Redis 存引用
```

### 9.4 预防（代码规范）

```go
// Code Review 规则
// 1. 禁止 KEYS / HGETALL / SMEMBERS 全量
// 2. Set/Hash/List 长度限制（业务层校验）
// 3. 单 value < 10KB（大的走 OSS）
// 4. 禁用 SINTERSTORE / SUNIONSTORE（大集合操作）
```

---

## 十、缓存穿透 / 击穿 / 雪崩

### 10.1 一句话

> "穿透 = 查不存在的 key，用 Bloom / 空值缓存；击穿 = 热 key 过期，用互斥锁 / singleflight；雪崩 = 大量 key 同时过期，用 TTL 随机化。"

### 10.2 三者对比

| 问题 | 原因 | 方案 |
| --- | --- | --- |
| 穿透 | key 不存在，每次打 DB | Bloom Filter + 空值缓存（短 TTL）|
| 击穿 | 热 key 过期瞬间，大量并发回源 | 互斥锁 / singleflight / 永不过期 + 异步更新 |
| 雪崩 | 大量 key 同一时刻过期 | TTL 随机 + 多级缓存 + 熔断降级 |

### 10.3 穿透详解

```go
// Bloom Filter 过滤
if !bloom.Exists(key) {
    return nil, ErrNotFound  // 肯定不存在
}
// ... 走正常流程

// 空值缓存
if val == nil {
    rdb.Set(ctx, key, "NULL", 5*time.Minute)  // 短 TTL
}
```

### 10.4 击穿详解（singleflight）

```go
import "golang.org/x/sync/singleflight"

var sf singleflight.Group

func Get(key string) (string, error) {
    v, err, _ := sf.Do(key, func() (interface{}, error) {
        val, err := rdb.Get(ctx, key).Result()
        if err == nil {
            return val, nil
        }
        // miss → 回源
        val, err = loadFromDB(key)
        if err == nil {
            rdb.Set(ctx, key, val, 10*time.Minute)
        }
        return val, err
    })
    return v.(string), err
}
```

### 10.5 雪崩详解

```
原因:
  大批量缓存同时过期 → 并发打 DB
  或: Redis 整体挂 → 全量打 DB

方案:
  1. TTL 随机: baseTTL + rand(0, 300s)
  2. 多级缓存: 本地缓存兜底
  3. 熔断降级: DB QPS 超阈值 → 返回默认值
  4. 预热: 启动 + 定时刷新热数据
  5. 降级: Redis 挂 → 限流 + 默认推荐
```

---

## 十一、容量规划怎么讲

### 11.1 一句话

> "内存 = key 数 × 平均大小 × 1.3（overhead），实例按 4-8GB 切分，单 Cluster 不超 1000 节点。"

### 11.2 估算模板

```
数据量估算:
  100w 用户 × 每用户 10 个 key × 200B 平均
  = 200MB

加 overhead（Redis 本身 / jemalloc 碎片）:
  200MB × 1.3 = 260MB

加副本:
  260MB × 2 副本 = 520MB

加 buffer（缓冲区 / COW）:
  520MB × 1.5 = 780MB

→ 单实例 maxmemory 设 800MB
  物理内存至少 1.2GB
```

### 11.3 实例规格建议

```
单机:
  ≤ 4GB:   小型业务（QPS < 1w）
  4-16GB:  中型业务
  > 16GB:  fork 开销大，建议拆 Cluster

Cluster:
  每节点 4-8GB 为宜（fork 快）
  节点数 < 1000（gossip 协议开销）
  单 Cluster 数据量 < 5TB
```

### 11.4 连接数规划

```
maxclients = 应用实例数 × 每实例连接池大小 × 1.5

示例:
  100 个应用实例 × 200 连接池 × 1.5 = 30000
  (maxclients 默认 10000 要调)
```

---

## 十二、答题"套路"技巧

### 12.1 答题结构

```
标准结构（每题都能套）:
  1. 一句话（总）
  2. 分层展开（分）
  3. 边界 / 代价（转）
  4. 实战佐证（合）
```

### 12.2 资深信号

```
✓ 主动提代价：
  "这个方案性能好，但代价是 X"

✓ 主动提边界：
  "这个适用于 Y 场景，超过就要 Z"

✓ 主动提演进：
  "Redis 4.0 前是 X，之后引入 Y"

✓ 主动提失败：
  "这个方案我们踩过一次坑，因为 X，后来改成 Y"

✓ 主动提量化：
  "QPS 从 1k 到 10w，延迟 P99 从 50ms 到 2ms"
```

### 12.3 避坑

```
❌ 死记硬背："Redis 有 5 种数据结构..."
✅ 场景化："排行榜用 ZSet，因为 ZSet 底层跳表 O(log N) 范围查询"

❌ 绝对化："Redis 就是快"
✅ 条件化："Redis 在 X 场景比 MySQL 快 100 倍，但在 Y 场景不适合"

❌ 技术炫耀："我们用了 Redlock"
✅ 取舍："我们评估了 Redlock 和 etcd 锁，因为 X 选了 Redis，Y 用 etcd"

❌ 只讲成功："我们系统很稳"
✅ 讲失败：";有一次 X 故障，后来改成 Y，现在稳了"
```

---

## 十三、高频 20 问速答索引

```
架构 / 原理:
Q1: Redis 为什么快？            → 二
Q2: 单线程怎么抗高并发？        → 二 + 17 / 18
Q3: 5 种数据结构底层？          → 17
Q4: RDB vs AOF 怎么选？         → 03
Q5: 主从同步流程？              → 04
Q6: Cluster 16384 怎么来的？    → 04

缓存:
Q7: 缓存一致性怎么保？          → 五 + 10
Q8: 缓存穿透/击穿/雪崩？        → 十
Q9: 多级缓存怎么设计？          → 四 + 11
Q10: 本地缓存选型？             → 11

分布式:
Q11: Redis 分布式锁？           → 八 + 06
Q12: Redlock 争议？             → 八
Q13: Redis vs etcd 锁？         → 八 + 19

调优:
Q14: 内存淘汰 8 策略？          → 三 + 07
Q15: 热 key / 大 key？          → 九 + 07
Q16: 慢查询排查？               → 七
Q17: Redis 内存打满？           → 七 + 09

线上问题:
Q18: 故障切换？                 → 六 + 15
Q19: 缓存雪崩应急？             → 十 + 09
Q20: 容量规划？                 → 十一

Go 实战:
Q21: 连接池怎么调？             → 20
Q22: Pipeline / Lua？           → 20
Q23: 熔断 fallback？            → 20
```

---

## 十四、终极答题清单（打印贴桌上）

```
开口前先想 3 秒:
  □ 这个问题的"一句话"是什么？
  □ 我能分几层展开？
  □ 代价 / 边界是什么？
  □ 有没有真实案例？

说话时:
  □ 先给结论（一句话）
  □ 再展开（2-5 点）
  □ 主动提代价（"这个方案的代价是..."）
  □ 主动提边界（"这个适合 X 场景，不适合 Y"）
  □ 最后举例（"我们线上..."）

避免:
  □ "我觉得..."（不自信）
  □ "应该是..."（不确定）
  □ 长篇大论不分层
  □ 只讲理论没案例
  □ 只讲成功没失败
```

---

## 十五、面试加分点

- **每题都按"一句话 + 分层 + 边界 + 案例" 四段**
- **LFU / LRU 都是近似** 算法（不是所有人都知道）
- **先写 DB 再删缓存** + 延迟双删 + binlog 订阅 三档方案
- **Redlock 争议 + Fencing Token** 标配
- **金融不用 Redis 锁**
- **热 key 本地缓存 + 大 key UNLINK + HSCAN**
- **业务幂等** 是所有 Redis 方案的兜底
- **量化数据**（QPS / 延迟 / 命中率 / 容量）
- **实战案例 + 踩坑**（STAR 格式）
- **主动提代价 + 边界**

---

## 十六、关联阅读

```
本目录:
- 00-redis-map.md             总览
- 01-architecture.md          为什么快（详细）
- 02-data-structures.md       数据结构
- 05-cache-patterns.md        缓存模式
- 06-distributed-lock.md      分布式锁
- 07-pitfalls-tuning.md       调优
- 09-production-cases.md      线上案例
- 10-cache-consistency-design.md 一致性
- 11-multi-tier-cache.md      多级缓存
- 15-cluster-failover-cases.md 集群故障
- 16-redis-cluster-operability.md 集群运维
- 17-object-encoding-internals.md 源码对象
- 18-eventloop-memory-internals.md 源码事件循环
- 19-redis-design-tradeoffs.md 设计边界
- 20-go-redis-best-practices.md Go 实战

跨模块:
- 06-distributed/04-lock.md   三方锁
- 03-mysql/00-mysql-map.md    MySQL 配合
- 99-meta/redis-20.md         速记题集
```
