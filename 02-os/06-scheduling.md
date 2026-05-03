# CPU 调度

> CPU 调度影响服务的延迟、吞吐和抖动。后端排查性能问题时，不能只看 CPU 使用率，还要看上下文切换和运行队列。

## 一、调度器做什么

调度器决定：

- 哪个任务运行。
- 运行多久。
- 什么时候切换。
- 如何处理优先级。

目标：

- 公平。
- 响应快。
- 吞吐高。
- 避免饥饿。

## 二、时间片

时间片是任务一次能运行的时间。

时间片太短：

- 响应快。
- 上下文切换多。

时间片太长：

- 上下文切换少。
- 交互响应变差。

## 三、上下文切换成本

上下文切换会带来：

- 保存和恢复寄存器。
- 调度器开销。
- CPU cache 失效。
- TLB 影响。

```mermaid
flowchart LR
    Run["任务运行"] --> Switch["上下文切换"]
    Switch --> Cache["Cache / TLB 局部性下降"]
    Cache --> Latency["延迟抖动"]
```

## 四、CPU 密集 vs IO 密集

CPU 密集：

- 长时间占用 CPU。
- 适合限制并发。
- 需要避免把机器打满。

IO 密集：

- 大量等待 IO。
- 适合事件驱动和异步。
- Go goroutine 比线程更适合大量 IO 并发。

## 五、负载和运行队列

Load Average 表示运行中和不可中断等待的任务数量。

CPU 使用率不高但 load 高，可能是：

- IO 等待。
- 磁盘阻塞。
- 不可中断 sleep。

排查要看：

- CPU 使用率。
- run queue。
- context switch。
- iowait。
- 线程数。

## 六、Go 调度联系

Go runtime 有自己的调度器，但最终仍运行在 OS thread 上。

影响：

- CPU 密集 goroutine 会占用 P。
- 阻塞系统调用可能占用 M。
- runtime 会尽量调度其他 G 继续运行。

排查：

- goroutine 数量。
- pprof CPU。
- block profile。
- mutex profile。
- runtime trace。

## 七、常见坑

- CPU 使用率低就认为系统不忙。
- load 高但忽略 iowait。
- goroutine 很多但实际都阻塞。
- CPU 密集任务和在线请求混部。
- 不限制后台任务并发，导致延迟抖动。

## 八、面试表达

```text
CPU 调度负责决定哪个任务运行以及运行多久。
上下文切换会带来寄存器保存、调度器开销和 cache/TLB 局部性下降，所以切换过多会导致延迟抖动。
排查服务性能时不能只看 CPU 使用率，还要看 load、运行队列、上下文切换和 iowait。
Go 有自己的 GMP 调度器，但最终 goroutine 还是要落到 OS thread 上执行。
```
