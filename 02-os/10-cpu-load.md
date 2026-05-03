# CPU、Load 与上下文切换

> CPU 问题不只是“CPU 使用率高”。后端更常见的是 load 高、上下文切换高、锁竞争、系统调用多，导致 P99 抖动。

## 一、CPU 使用率怎么看

常见分类：

| 指标 | 含义 | 常见原因 |
| --- | --- | --- |
| user | 用户态 CPU | 业务计算、序列化、正则、压缩 |
| system | 内核态 CPU | 系统调用、网络包处理、文件 IO |
| iowait | 等待 IO | 磁盘慢、网络文件系统、块设备拥塞 |
| steal | 被宿主机抢占 | 云主机资源争抢 |
| idle | 空闲 | CPU 没被用满 |

经验判断：

- `user` 高：看业务热点函数。
- `system` 高：看系统调用、网络、锁、内核态开销。
- `iowait` 高：看磁盘和存储。
- CPU 不高但 load 高：可能是 IO 等待或大量任务排队。

## 二、Load Average 是什么

load 表示一段时间内处于：

- 正在运行。
- 等待 CPU。
- 不可中断 IO 等待。

的任务数量。

```text
load 不是 CPU 使用率
load 高也不一定 CPU 打满
```

粗略判断：

```text
8 核机器：
load < 8    通常可接受
load = 8    接近满负载
load >> 8   有明显排队或 IO 等待
```

但要结合业务和历史基线看，不能只靠固定阈值。

## 三、上下文切换

上下文切换包括：

- 进程/线程切换。
- 用户态/内核态切换。
- 中断处理。

代价：

- 保存和恢复寄存器、栈、调度状态。
- CPU cache/TLB 失效。
- 调度器开销增加。

常见来源：

- 线程数过多。
- 锁竞争严重。
- 频繁系统调用。
- 网络包很多。
- goroutine 大量阻塞和唤醒。

```mermaid
flowchart LR
    Req["请求"] --> Lock["竞争锁"]
    Lock --> Wait["线程等待"]
    Wait --> Switch["上下文切换增加"]
    Switch --> Cache["CPU cache 失效"]
    Cache --> P99["P99 抖动"]
```

## 四、常用排查命令

```text
top
top -H -p <pid>
vmstat 1
pidstat -u -w 1
perf top
perf record / perf report
```

重点看：

- 哪个进程 CPU 高。
- 哪个线程 CPU 高。
- `cs` 上下文切换是否异常。
- `r` 运行队列是否过长。
- 热点在业务函数、系统调用还是 runtime。

Go 服务可以结合：

```text
go tool pprof http://host/debug/pprof/profile
go tool pprof http://host/debug/pprof/goroutine
go tool trace
```

## 五、典型场景

### 场景 1：CPU 被 JSON 序列化打满

现象：

- user CPU 高。
- pprof 里 `encoding/json` 占比高。
- 接口 P99 上升。

处理：

- 减少返回字段。
- 分页和限制返回量。
- 缓存热点响应。
- 必要时替换更高性能序列化库。

### 场景 2：system CPU 高

可能原因：

- 短连接太多，频繁建连。
- 日志写太频繁。
- 小包太多，网络中断多。
- epoll、futex、read/write 系统调用频繁。

处理方向：

- 使用连接池和长连接。
- 批量写日志。
- 降低无效轮询。
- 查 `strace` 和 `perf`。

### 场景 3：load 高但 CPU 不高

可能原因：

- 大量线程卡在磁盘 IO。
- 数据库连接池等待。
- 网络存储慢。
- 进程进入 D 状态。

处理方向：

- 看 `iostat -x 1` 的 await/util。
- 看 `vmstat 1` 的 `b` 列。
- 看慢 SQL、下游依赖和磁盘。

## 六、常见坑

- 只看 CPU 使用率，不看 load 和上下文切换。
- load 高就认为 CPU 不够。
- 忽略 system CPU，高频系统调用也会拖垮服务。
- 线程池开太大，导致切换开销和内存占用上升。
- 只看平均延迟，不看 P99 抖动。

## 七、面试表达

```text
CPU 问题我会拆成 user、system、iowait、steal 来看。
如果 user 高，重点看业务热点函数；如果 system 高，重点看系统调用、网络和锁；如果 iowait 高，说明可能卡在磁盘或存储。
load average 表示运行和等待的任务数，不等于 CPU 使用率，load 高但 CPU 不高时，我会重点怀疑 IO 等待或大量任务排队。
线上会结合 top、vmstat、pidstat、perf 和 pprof 来定位。
```
