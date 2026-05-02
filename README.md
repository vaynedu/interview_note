# interview_note

> 资深 Go 后端面试知识库 — 体系沉淀 + 面试备战 双视角

## 设计理念

- **高内聚优先**：每一块独立可读，宁可重复也不强行抽象
- **一个知识点 = 一个文件**：打开即看完，不跨文件跳转
- **统一模板**：原理 / 八股 / 真题 / 手写 / 踩坑 五段式

完整设计见 [`docs/specs/2026-05-02-interview-note-structure.md`](docs/specs/2026-05-02-interview-note-structure.md)。

## 目录索引

| 目录 | 内容 |
| --- | --- |
| [01-go-language](01-go-language/) | Go 语法 / 并发 / runtime / 标准库 / 工程 / 性能 / 生态 |
| [02-os](02-os/) | 操作系统：进程、内存、IO、调度 |
| [03-mysql](03-mysql/) | 存储引擎、索引、事务、锁、MVCC、调优 |
| [04-redis](04-redis/) | 数据结构、持久化、集群、缓存模式、热点 |
| [05-message-queue](05-message-queue/) | Kafka / RocketMQ / 削峰 / 顺序 / 幂等 |
| [06-distributed](06-distributed/) | CAP、一致性、分布式锁、分布式事务、限流熔断 |
| [07-microservice](07-microservice/) | RPC、注册发现、服务治理、链路追踪 |
| [08-architecture](08-architecture/) | 架构演进、高并发高可用、典型模式 |
| [09-ddd](09-ddd/) | 领域驱动设计：战略 / 战术 / 落地 |
| [10-system-design](10-system-design/) | 系统设计题：短链 / 秒杀 / Feed / IM / 支付 |
| [11-cdn](11-cdn/) | 回源、缓存策略、调度、边缘计算 |
| [12-ai](12-ai/) | AI 相关（范围待定） |
| [13-engineering](13-engineering/) | Linux / Git / 可观测性 / 压测 / CI-CD |
| [14-projects](14-projects/) | 项目复盘、简历项目、STAR 拆解 |
| [99-meta](99-meta/) | 高频题索引、刷题清单、面经、算法外链 |

## 当前进度

- [x] 顶层目录骨架
- [x] Go 语言完整骨架
- [ ] Go 语言内容填充
- [ ] 其他领域骨架与填充

## 文件内部模板

```markdown
# <知识点名>

> 一句话概括

## 一、核心原理
## 二、八股速记
## 三、面试真题
## 四、手写实现
## 五、踩坑与最佳实践
```
