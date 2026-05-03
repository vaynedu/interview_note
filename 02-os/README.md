# 操作系统

> 后端面试里的 OS 重点：进程线程、内存、虚拟内存、IO、多路复用、调度、锁、文件系统，以及它们和 Go 服务端性能的关系。

## 分类导航

| 文件 | 内容 |
| --- | --- |
| [00-os-map.md](00-os-map.md) | 操作系统知识地图：CPU、内存、IO、文件、网络、并发 |
| [01-process-thread.md](01-process-thread.md) | 进程、线程、协程：资源隔离、上下文切换、Go goroutine 对比 |
| [02-memory.md](02-memory.md) | 内存管理：栈、堆、内存分配、碎片、mmap、零拷贝基础 |
| [03-virtual-memory.md](03-virtual-memory.md) | 虚拟内存：页表、缺页中断、TLB、Swap、OOM |
| [04-io-model.md](04-io-model.md) | IO 模型：阻塞、非阻塞、同步、异步、零拷贝 |
| [05-io-multiplexing.md](05-io-multiplexing.md) | IO 多路复用：select、poll、epoll、Reactor |
| [06-scheduling.md](06-scheduling.md) | CPU 调度：时间片、优先级、上下文切换、负载与抖动 |
| [07-lock-concurrency.md](07-lock-concurrency.md) | 锁与并发：互斥锁、读写锁、自旋锁、死锁、CAS |
| [08-file-system.md](08-file-system.md) | 文件系统：inode、page cache、fsync、日志、顺序写和随机写 |
| [09-linux-troubleshooting.md](09-linux-troubleshooting.md) | Linux 线上排查：CPU、内存、IO、网络、依赖、止血 |
| [10-cpu-load.md](10-cpu-load.md) | CPU 与 Load：user/system/iowait、load、上下文切换、P99 抖动 |
| [11-memory-oom.md](11-memory-oom.md) | 内存与 OOM：RSS、Page Cache、Swap、cgroup、Go heap |
| [12-network-stack.md](12-network-stack.md) | 网络栈：TCP 状态、TIME_WAIT、CLOSE_WAIT、超时、重试 |
| [13-observability.md](13-observability.md) | 可观测性：日志、指标、链路追踪、告警、事故复盘 |
| [14-performance-tools.md](14-performance-tools.md) | 性能工具：pprof、perf、strace、火焰图、性能优化流程 |
| [15-cpu-memory-troubleshooting.md](15-cpu-memory-troubleshooting.md) | 高 CPU / 高内存排查手册：top、pidstat、pmap、strace、perf、pprof |
| [16-production-incident-cases.md](16-production-incident-cases.md) | 大厂线上事故案例：CPU、网络、数据库、备份、容灾、级联故障 |
| [17-network-troubleshooting.md](17-network-troubleshooting.md) | 网络故障排查：DNS、建连、TLS、重传、连接池、端口耗尽 |
| [18-disk-io-troubleshooting.md](18-disk-io-troubleshooting.md) | 磁盘 IO 排查：df、inode、iostat、deleted 文件、fsync、Page Cache |
| [19-container-k8s-resource.md](19-container-k8s-resource.md) | 容器/K8s 资源：OOMKilled、CPU throttling、request/limit、探针 |

## 高频题速览

### 进程线程

- 进程和线程有什么区别？
- 线程切换为什么比进程切换轻？
- 协程和线程有什么区别？
- Go goroutine 和 OS thread 是什么关系？
- 上下文切换为什么会影响性能？

### 内存

- 栈和堆有什么区别？
- 虚拟内存解决什么问题？
- 什么是缺页中断？
- TLB 是什么？
- OOM 是怎么发生的？

### IO

- 阻塞 IO、非阻塞 IO、同步 IO、异步 IO 区别？
- select、poll、epoll 有什么区别？
- epoll 为什么适合高并发连接？
- 什么是零拷贝？
- Page Cache 对数据库和文件读写有什么影响？

### 调度与并发

- CPU 调度如何影响服务延迟？
- 什么是惊群问题？
- 死锁产生的条件是什么？
- CAS 和锁有什么区别？
- 自旋锁适合什么场景？

### 文件系统

- inode 是什么？
- fsync 做了什么？
- 顺序写为什么比随机写快？
- 日志型文件系统解决什么问题？
- 为什么写文件成功不代表一定落盘？

### 线上排查

- CPU 不高但接口很慢，怎么排查？
- load 高一定是 CPU 不够吗？
- RSS 高但 Go heap 不高，可能是什么原因？
- TIME_WAIT 和 CLOSE_WAIT 分别说明什么？
- 线上 P99 抖动，你会看哪些指标？
- pprof、perf、strace 分别适合定位什么？
- 线上 CPU 飙高，你如何从进程定位到函数？
- RSS 高但 pprof heap 不高，你怎么排查？
- 你了解哪些经典线上事故？从中学到了什么？
- 配置变更、数据库误删、网络分区、云区域故障分别怎么治理？
- 接口偶发超时，你如何判断是 DNS、建连、TLS、下游还是连接池？
- 磁盘空间没满但创建文件失败，可能是什么原因？
- 宿主机有内存，为什么容器还会 OOMKilled？
- CPU 没打满但 P99 升高，为什么可能是 CPU throttling？

## 复习路径

1. 先看 [00-os-map.md](00-os-map.md)，建立 OS 知识地图。
2. 再看 [01-process-thread.md](01-process-thread.md)、[06-scheduling.md](06-scheduling.md)、[07-lock-concurrency.md](07-lock-concurrency.md)，理解并发和调度。
3. 看 [02-memory.md](02-memory.md)、[03-virtual-memory.md](03-virtual-memory.md)，理解内存和虚拟内存。
4. 看 [04-io-model.md](04-io-model.md)、[05-io-multiplexing.md](05-io-multiplexing.md)，理解服务端网络 IO。
5. 看 [08-file-system.md](08-file-system.md)，补齐文件系统、Page Cache 和落盘语义。
6. 看 [09-linux-troubleshooting.md](09-linux-troubleshooting.md)、[10-cpu-load.md](10-cpu-load.md)、[11-memory-oom.md](11-memory-oom.md)，建立线上资源排查能力。
7. 看 [12-network-stack.md](12-network-stack.md)、[13-observability.md](13-observability.md)、[14-performance-tools.md](14-performance-tools.md)，补齐网络、监控告警和性能分析能力。
8. 最后看 [15-cpu-memory-troubleshooting.md](15-cpu-memory-troubleshooting.md)，把高 CPU 和高内存排查串成线上操作手册。
9. 看 [16-production-incident-cases.md](16-production-incident-cases.md)，把大厂事故抽象成可复用的排查和治理模型。
10. 看 [17-network-troubleshooting.md](17-network-troubleshooting.md)、[18-disk-io-troubleshooting.md](18-disk-io-troubleshooting.md)、[19-container-k8s-resource.md](19-container-k8s-resource.md)，补齐网络、磁盘和容器资源的实战排查。

## 答题原则

- OS 题不要只背定义，要能联系服务端性能：延迟、吞吐、抖动、上下文切换、IO 等待。
- Go 后端回答 OS 题时，可以结合 goroutine、netpoll、GC、数据库、日志写入等场景。
- 面试高分答案要讲清：机制是什么、为什么这样设计、代价是什么、线上怎么排查。
- 个人成长要能形成固定排查路径：先看影响面和时间线，再看 CPU、内存、IO、网络、依赖和应用 profile。
