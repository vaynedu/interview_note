# Go 线上问题排查 Runbook

> 这篇把 Go 语言层的线上排查串起来：CPU、内存、goroutine、GC、锁、阻塞、HTTP 和数据库连接池。面试时不要只说 pprof，要说清楚“先看什么、怎么证明、怎么修复、怎么验证”。

## 一、总流程

```text
报警 / 用户反馈
        |
        v
确认影响面：实例、接口、时间段、错误率、P99
        |
        v
看资源：CPU / 内存 / goroutine / fd / 网络 / DB 连接池
        |
        v
抓证据：pprof / trace / 日志 / metrics / 火焰图
        |
        v
止血：限流 / 降级 / 扩容 / 回滚 / 熔断
        |
        v
根因修复：代码、SQL、配置、容量、依赖
        |
        v
复盘：监控、预案、压测、演练
```

## 二、高 CPU

### 1. 先确认是不是本进程

```bash
top
pidstat -p <pid> 1
```

看：

- 用户态 CPU 高：业务计算、JSON、正则、压缩、加密、忙循环。
- 系统态 CPU 高：网络、锁、系统调用、fd、内核态开销。
- 单核打满：可能有热点 goroutine 或 GOMAXPROCS/锁竞争问题。

### 2. 抓 CPU profile

```bash
go tool pprof http://127.0.0.1:6060/debug/pprof/profile?seconds=30
```

pprof 常用命令：

```text
top
top -cum
list funcName
web
```

判断：

- `encoding/json` 高：JSON 序列化/反序列化热点。
- `regexp` 高：正则复杂或重复编译。
- `runtime.mallocgc` 高：分配太多，可能引发 GC。
- `sync.(*Mutex).Lock` 高：锁竞争。
- 业务函数高：算法复杂度或循环问题。

### 3. 常见修复

- 缓存正则编译结果。
- 降低 JSON 反射和中间对象。
- 减少热点循环里的分配。
- 拆锁、缩小临界区。
- 对异常流量限流。
- 回滚最近变更。

## 三、内存上涨

### 1. 分清是泄漏还是正常缓存

看：

- RSS 是否持续上涨。
- heap_alloc 是否持续上涨。
- GC 后 heap 是否下降。
- goroutine 数是否同步上涨。
- 缓存容量是否无界。

命令：

```bash
go tool pprof http://127.0.0.1:6060/debug/pprof/heap
```

pprof 里看：

```text
top
top -cum
list funcName
```

可切换视角：

```text
inuse_space   当前仍存活对象
alloc_space   累计分配对象
inuse_objects 当前仍存活对象数量
alloc_objects 累计分配对象数量
```

### 2. 常见根因

- 本地 map 缓存无 TTL / 无容量。
- goroutine 泄漏导致栈和引用对象不释放。
- 大 slice 被小切片引用，底层数组无法释放。
- 连接、Rows、Body 未关闭。
- 队列堆积。
- 日志或指标标签爆炸。

### 3. 常见修复

- 缓存加容量、TTL、淘汰策略。
- 大对象处理后断开引用。
- 小切片需要长期保存时 copy 出来。
- 队列限长并增加丢弃/降级策略。
- 修复 goroutine 和 fd 泄漏。

## 四、goroutine 暴涨

命令：

```bash
curl -s http://127.0.0.1:6060/debug/pprof/goroutine?debug=2
go tool pprof http://127.0.0.1:6060/debug/pprof/goroutine
```

看栈状态：

| 状态 | 常见原因 |
| --- | --- |
| `chan send` | 下游没人接、队列满 |
| `chan receive` | channel 没关闭、worker 没退出 |
| `IO wait` | HTTP/DB/Redis 无超时或下游慢 |
| `semacquire` | 锁竞争、WaitGroup 等待 |
| `select` | 等待多个事件，缺少退出条件 |

修复重点：

- 每个 goroutine 都有 context。
- channel 有关闭协议。
- 请求内异步任务有并发上限。
- HTTP/DB/Redis 都有超时。

## 五、GC 和分配压力

指标：

- `gc_pause_seconds`
- `go_memstats_heap_alloc_bytes`
- `go_memstats_heap_objects`
- `go_memstats_alloc_bytes_total`
- `go_memstats_gc_cpu_fraction`

排查：

```bash
go tool pprof http://127.0.0.1:6060/debug/pprof/allocs
go tool pprof http://127.0.0.1:6060/debug/pprof/heap
```

判断：

- `allocs` 高：分配速率高。
- `heap` 高：存活对象多。
- `runtime.mallocgc` 在 CPU profile 高：分配影响 CPU。

修复：

- 明确 struct 替代 `map[string]any`。
- 预分配 slice/map。
- 控制缓存边界。
- 避免热点路径 `fmt.Sprintf`、反射、频繁转换。
- 最后再调整 `GOGC` 和 `GOMEMLIMIT`。

## 六、锁竞争和阻塞

打开 mutex/block profile：

```go
runtime.SetMutexProfileFraction(10)
runtime.SetBlockProfileRate(1)
```

抓取：

```bash
go tool pprof http://127.0.0.1:6060/debug/pprof/mutex
go tool pprof http://127.0.0.1:6060/debug/pprof/block
```

常见原因：

- 全局大锁。
- 锁内做 IO。
- 锁内 JSON 编码、数据库查询、RPC。
- channel 缓冲太小或无消费者。
- sync.Once 初始化卡住。

修复：

- 缩小临界区。
- 分片锁。
- 读多写少用 `RWMutex`，但要确认写锁饥饿风险。
- 锁内不做慢 IO。
- 用原子操作替代简单计数。

## 七、HTTP / DB 连接池打满

### HTTP

看：

- 下游耗时。
- 超时数。
- goroutine IO wait。
- fd 数。
- `MaxConnsPerHost` 是否限制。

修复：

- client 复用。
- 设置 Timeout。
- 关闭 Body。
- 连接池参数合理。
- 限流、熔断、降级。

### database/sql

看：

- `OpenConnections`
- `InUse`
- `Idle`
- `WaitCount`
- `WaitDuration`
- 慢 SQL。

修复：

- `QueryContext` 绑定超时。
- `rows.Close()`。
- 事务及时 commit/rollback。
- 优化 SQL 和索引。
- 合理配置 `SetMaxOpenConns`、`SetMaxIdleConns`、`SetConnMaxLifetime`。

## 八、trace 适合看什么

```bash
curl -s http://127.0.0.1:6060/debug/pprof/trace?seconds=5 > trace.out
go tool trace trace.out
```

适合分析：

- goroutine 调度延迟。
- 网络阻塞。
- syscall。
- GC 事件。
- 任务间依赖。

trace 信息量很大，通常在 pprof 还不够解释问题时使用。

## 九、止血手段

| 问题 | 止血 |
| --- | --- |
| CPU 打满 | 限流、扩容、关闭非核心功能、回滚 |
| 内存上涨 | 重启异常实例、降流量、关缓存开关、扩容 |
| goroutine 泄漏 | 回滚、重启、降低触发流量 |
| 下游慢 | 熔断、降级、缩短超时、隔离线程/worker |
| DB 池打满 | kill 慢 SQL、降级读、限流、扩容只读实例 |
| MQ 积压 | 扩消费者、限生产、跳过非核心消息、死信隔离 |

止血不是根因修复，但线上先恢复服务，再做完整修复。

## 十、面试真题

**Q1：Go 服务 CPU 突然升高怎么排查？**

先确认是否本进程和影响范围，再抓 CPU profile 看热点函数。结合最近发布、流量变化、下游异常和日志判断根因。如果是 JSON/正则/分配/锁竞争，分别用减少反射、缓存正则、降低分配、拆锁等方式修复。线上先限流、扩容或回滚止血。

**Q2：内存持续上涨怎么排查？**

看 RSS、heap_alloc、heap_objects、goroutine 数和 GC 后是否回落。抓 heap profile 看 inuse_space 定位谁持有对象，再看 allocs 找分配热点。常见是缓存无界、goroutine 泄漏、队列积压、大 slice 引用和资源未关闭。

**Q3：怎么证明是 goroutine 泄漏？**

goroutine 数持续上涨且不回落，pprof 里大量 goroutine 栈停在同一位置，例如 channel receive、IO wait 或 select。修复后通过压测验证 goroutine 数能随流量回落。

**Q4：pprof 和 trace 怎么选？**

CPU/内存/锁/阻塞先用 pprof，它聚合热点更直接；调度延迟、goroutine 生命周期、网络阻塞和 GC 时间线更适合 trace。

## 十一、面试表达

```text
我排查 Go 线上问题会先看影响面和资源指标，再抓 pprof 建立证据链。
CPU 看 profile 热点，内存看 heap/allocs，goroutine 看栈聚合，锁和阻塞看 mutex/block profile。
处理上先止血恢复服务，再做根因修复，最后补监控、压测和复盘，避免同类问题再发生。
```
