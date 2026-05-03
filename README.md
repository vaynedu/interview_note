# interview_note

> 面向资深 Go 后端的知识库：用于面试备战，也用于长期工程能力沉淀。

这不是单纯八股题仓库，目标是把知识整理成三类能力：

- **能讲清原理**：知道机制是什么、为什么这么设计、代价是什么。
- **能解决问题**：遇到慢 SQL、高 CPU、OOM、消息堆积、主从延迟、线上事故时有排查路径。
- **能做架构取舍**：面对高并发、高可用、一致性、成本、复杂度，能说清为什么这样设计。

更细的学习路线和建设计划见 [00-roadmap.md](00-roadmap.md)。

## 目录地图

| 目录 | 定位 | 重点能力 |
| --- | --- | --- |
| [01-go-language](01-go-language/) | Go 语言与工程能力 | 语法、并发、runtime、GC、pprof、工程实践 |
| [02-os](02-os/) | 操作系统与线上排查 | 进程线程、内存、IO、网络、CPU、OOM、K8s 资源 |
| [03-mysql](03-mysql/) | MySQL 核心与实战 | 索引、事务、锁、日志、主从、慢 SQL、分库分表 |
| [04-redis](04-redis/) | Redis 与缓存治理 | 数据结构、持久化、集群、缓存一致性、热点、大 key |
| [05-message-queue](05-message-queue/) | 消息队列 | Kafka/RocketMQ、可靠性、顺序、重复、积压、削峰 |
| [06-distributed](06-distributed/) | 分布式系统 | CAP、Raft、分布式事务、锁、ID、限流熔断 |
| [07-microservice](07-microservice/) | 微服务治理 | RPC、注册发现、服务治理、链路追踪 |
| [08-architecture](08-architecture/) | 架构能力 | 架构演进、高可用、高并发、典型模式 |
| [09-ddd](09-ddd/) | DDD | 领域建模、聚合、领域事件、工程落地 |
| [10-system-design](10-system-design/) | 系统设计题 | 短链、秒杀、直播、Feed、IM、支付、库存 |
| [11-cdn](11-cdn/) | CDN 与边缘网络 | 缓存、调度、协议优化、安全、边缘计算、故障排查 |
| [12-ai](12-ai/) | AI 与 Agent | LLM、RAG、Coding Agent、Skills、MCP、AI 工作流 |
| [13-engineering](13-engineering/) | 工程效能 | Git、CI/CD、测试、发布、可观测性、压测 |
| [14-projects](14-projects/) | 项目复盘 | 简历项目、STAR、业务难点、线上案例 |
| [99-meta](99-meta/) | 元资料 | 高频题索引、刷题清单、面经、外链 |

## 推荐学习路径

### 路线 1：Go 后端面试主线

```text
01-go-language
  -> 02-os
  -> 03-mysql
  -> 04-redis
  -> 05-message-queue
  -> 06-distributed
  -> 10-system-design
```

适合目标：

- Go 后端中高级面试。
- 补齐基础原理和常见中间件。
- 准备系统设计题。

### 路线 2：线上排查能力

```text
02-os
  -> 03-mysql/10-production-cases.md
  -> 04-redis/07-pitfalls-tuning.md
  -> 05-message-queue/07-scenarios.md
  -> 11-cdn/08-troubleshooting-cases.md
  -> 13-engineering
```

适合目标：

- 高 CPU、高内存、慢 SQL、连接泄漏、消息积压、网络抖动。
- 形成固定排查套路。
- 积累事故复盘表达。

### 路线 3：架构与系统设计

```text
10-system-design/00-question-map.md
  -> 10-system-design/01-design-framework.md
  -> 03-mysql/11-order-system-design.md
  -> 03-mysql/12-sharding.md
  -> 06-distributed
  -> 08-architecture
```

适合目标：

- 资深后端系统设计面试。
- 训练容量估算、核心链路、瓶颈识别、架构取舍。
- 从“堆组件”升级到“讲约束和取舍”。

### 路线 4：AI 辅助研发

```text
12-ai
  -> 01-go-language/05-engineering
  -> 13-engineering
  -> 14-projects
```

适合目标：

- 理解 LLM / Agent / Skills / MCP。
- 把 AI 用到代码阅读、方案设计、测试、重构、排障和文档沉淀。
- 提升个人研发效率，而不是只会闲聊式提问。

## 面试使用方式

每个专题建议按四层复习：

```text
第一层：一句话说清
第二层：核心原理和关键图
第三层：线上坑和排查路径
第四层：面试表达模板
```

回答问题时尽量使用这个结构：

```text
背景和目标
  -> 核心机制
  -> 关键取舍
  -> 常见坑
  -> 线上排查或落地方案
```

系统设计题使用这个结构：

```text
澄清需求
  -> 容量估算
  -> 核心链路
  -> 数据模型
  -> 架构设计
  -> 瓶颈和治理
  -> 监控、降级、容灾
```

## 当前重点

已经重点整理：

- MySQL：核心原理、线上坑、分库分表、订单系统、存储选型。
- OS：基础机制、Linux 排查、高 CPU/内存、网络、磁盘、K8s、事故案例。
- System Design：高频设计题和答题方法。
- AI：LLM / Agent 原理、Coding Agent、Skills、MCP、程序员工作流。
- CDN：架构、缓存、调度、协议、安全、边缘计算、典型场景。

后续优先补强：

- Go runtime、GC、pprof、工程实践。
- Redis 与 MQ 的线上案例。
- 分布式与微服务治理的实战题。
- 项目复盘和简历表达。

## 维护原则

- **按主题拆文件**：一个文件聚焦一个专题，避免大杂烩。
- **先总览再细节**：每个目录优先有 `README.md` 和 `00-*.md` 总览。
- **图文结合**：核心流程尽量用 Mermaid 表达。
- **场景驱动**：原理必须能落到面试题、线上坑或架构取舍。
- **提交清晰**：优先使用 `docs(scope): 中文说明`。

