# Meta 索引

> 跨领域索引、速记题集、刷题清单和面试前快速复习入口。

## 目录

### Go 专题题集

| 文件 | 内容 |
| --- | --- |
| [senior-go-interview-25.md](senior-go-interview-25.md) | Go 高级 / 资深工程师面试题 25 题 |
| [go-concurrency-100.md](go-concurrency-100.md) | Go 并发专题 100 问 |

### 12 大领域速记题集（每篇 20 题）

| 文件 | 领域 | 重点 |
| --- | --- | --- |
| [ddd-20.md](ddd-20.md) | DDD | 战略 / 战术 / 聚合 / 架构 / CQRS / 反模式 |
| [architecture-20.md](architecture-20.md) | 架构 | 演进 / 高可用 / 高性能 / 可扩展 / 容量 / 决策 |
| [microservice-20.md](microservice-20.md) | 微服务 | 注册 / 配置 / 网关 / RPC / Mesh / 观测治理 |
| [cdn-20.md](cdn-20.md) | CDN | 架构 / 缓存 / 调度 / 协议 / 安全 / 边缘 / 场景 |
| [redis-20.md](redis-20.md) | Redis | 架构 / 数据结构 / 持久化 / 集群 / 缓存 / 锁 / 调优 |
| [mysql-20.md](mysql-20.md) | MySQL | 架构 / 索引 / 事务锁 MVCC / 日志 / 复制 / 分库分表 |
| [mq-20.md](mq-20.md) | MQ | Kafka / RabbitMQ / RocketMQ / 可靠性 / 顺序 / 重复 |
| [distributed-20.md](distributed-20.md) | 分布式 | CAP / 共识 / 事务 / 锁 / 限流熔断 / 设计模式 |
| [system-design-20.md](system-design-20.md) | 系统设计 | 短链 / 秒杀 / 弹幕 / 直播 / Feed / IM / 支付等 |
| [engineering-20.md](engineering-20.md) | 工程化 | CR / 规范 / 测试 / 可观测 / 排查 / 发布 |
| [os-20.md](os-20.md) | 操作系统 | 进程线程 / IO / 内存 / 网络 / 排查 / 性能工具 |
| [ai-20.md](ai-20.md) | AI / Agent | LLM / RAG / Agent / MCP / Skills / 工程化 |

### 全局索引

| 文件 | 内容 |
| --- | --- |
| [00-interview-question-index.md](00-interview-question-index.md) | 高频面试题全局索引（按问题跳转） |

## 使用方式

### 面试前 1 天（30 分钟过完所有"背诵版"）

按重点领域顺序通读：
1. **本职岗位领域**（如 Go 后端 → Go / MySQL / Redis / MQ / 微服务 / 架构）
2. **求职公司方向**（如 CDN / 大数据 / AI）
3. **DDD / 系统设计**（资深面试必考）
4. **OS / 工程化**（基础 + 加分）

### 面试前 1 周（深度复习）

每天 3-5 题，重点看"标准答案 + 易错点 + 追问点"。

### 面试前 1 个月

结合各模块完整知识库（01-go-language / 03-mysql / 04-redis / ...）深度学习。

## 题集格式说明

每题统一 5 段式：

```
1. 题目
2. 标准答案（含表格 / 代码 / 对比）
3. 易错点（3-5 条）
4. 追问点（2-3 条）
5. 背诵版（一句话核心，30-50 字）
```

**用法分级**：
- **通读背诵版**：30 分钟过完一篇
- **入场版（标准答案）**：考场能答出东西
- **丰富版（含追问/易错）**：应对深度追问

## 知识库与题集的关系

```
完整知识库（深度）       题集（速记）
─────────────────       ─────────────
01-go-language/    ←→   senior-go-interview-25
                        go-concurrency-100
03-mysql/          ←→   mysql-20
04-redis/          ←→   redis-20
05-message-queue/  ←→   mq-20
06-distributed/    ←→   distributed-20
07-microservice/   ←→   microservice-20
08-architecture/   ←→   architecture-20
09-ddd/            ←→   ddd-20
10-system-design/  ←→   system-design-20
11-cdn/            ←→   cdn-20
12-ai/             ←→   ai-20
13-engineering/    ←→   engineering-20
02-os/             ←→   os-20
```

**学习路径**：
- 入门：题集"背诵版"快速建立框架
- 深入：完整知识库精读
- 实战：自己项目落地 + 面试

## 后续可补

- 面经汇总
- 大厂题型清单
- 算法题单
- 简历项目模板（14-projects）
