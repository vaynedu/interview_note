# 工程能力

> 工程能力关注“如何稳定交付和排查线上问题”：代码评审、规范、发布、监控、压测、事故处理。

## 目录

| 文件 | 内容 |
| --- | --- |
| [00-troubleshooting-runbook.md](00-troubleshooting-runbook.md) | 线上排查 Runbook 总表：CPU、内存、DB、Redis、MQ、网络、磁盘、K8s |
| [01-code-review.md](01-code-review.md) | Code Review 方法、关注点和常见问题 |
| [02-standards.md](02-standards.md) | 工程规范、代码风格、接口约定和协作边界 |

## 高频题速览

- 线上接口突然超时怎么排查？
- 高 CPU / 高内存 / OOM 怎么排查？
- 慢 SQL、Redis 热 key、MQ 积压怎么定位？
- 你如何做 Code Review？
- 你如何保证发布质量？
- 如何设计监控和告警？
- 出事故后如何复盘？

## 复习路径

1. 先看 [00-troubleshooting-runbook.md](00-troubleshooting-runbook.md)，建立排查总路径。
2. 再看 [01-code-review.md](01-code-review.md)，掌握评审关注点。
3. 最后看 [02-standards.md](02-standards.md)，补齐工程规范表达。

## 答题原则

- 先讲流程，再讲工具。
- 先止血，再定位根因。
- 先用户影响，再系统指标。
- 任何优化都要有指标验证。
- 任何事故都要有防复发动作。

