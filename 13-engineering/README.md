# 工程化

> 资深工程师必备的**质量与协作能力**：Code Review / 规范 / 测试 / 可观测性接入

> 不涵盖 CI/CD / 研发效能 / DevOps（按需求精简）

## 目录

| # | 文件 | 涵盖 |
| --- | --- | --- |
| 01 | [Code Review](01-code-review.md) | CR 价值 / 流程 / Checklist / 反模式 / 大厂实践 |
| 02 | [规范](02-standards.md) | 编码 / 接口 / 错误码 / 命名 / 工程结构 / Git / 文档 |
| 03 | [测试策略](03-testing-strategy.md) | 金字塔 / 单元 / 集成 / E2E / Mock / Table-Driven / Fuzz |
| 04 | [可观测性接入](04-observability-integration.md) | OTel / Prometheus / Loki 完整接入 + 优雅关闭 |

## 跨章高频题

- Code Review 的价值？看什么？（→ 01）
- CR 反馈怎么分级？（→ 01）
- PR 应该多大？（→ 01）
- 为什么需要规范？（→ 02）
- Go 命名规范？（→ 02）
- 错误码怎么设计？（→ 02）
- RESTful API 设计要点？（→ 02）
- Protobuf 兼容性铁律？（→ 02）
- 测试金字塔是什么？（→ 03）
- FIRST 原则？AAA 结构？（→ 03）
- Mock vs Stub vs Fake？（→ 03）
- 覆盖率多少合理？（→ 03）
- TDD 怎么做？（→ 03）
- 怎么从零接入可观测？（→ 04）
- trace_id 怎么透传？（→ 04）
- Prometheus label 怎么设计？（→ 04）
- /healthz vs /ready？（→ 04）
- 怎么优雅关闭？（→ 04）

## 设计原则

- **图文并茂**，关键概念用 Mermaid
- **每篇独立可读**
- **结合真实项目** `ddd_order_example` 举例
- **Go 生态为主**
- **面试题驱动**，每章 10 道高频题 + 加分点
- **聚焦质量与协作**，不重复 Go 语法/微服务组件细节

## 与其他模块的关系

- **Go 工程实践（01-go-language/05）**：Go 项目布局、错误处理细节
- **DDD（09-ddd）**：领域建模规范、命名约定
- **微服务（07-microservice/07）**：观测原理、治理整合
- **架构（08-architecture/02）**：高可用、容量、SLO
