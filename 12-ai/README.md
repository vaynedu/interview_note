# 12-ai · LLM Agent 开发(0→1)

> **本目录定位**:从 0 到 1 学**做 Agent**(不是用 AI 编码——那部分在 [`13-engineering/05-ai-coding-claude-code.md`](../13-engineering/05-ai-coding-claude-code.md) 和 [`06-ai-coding-cookbook.md`](../13-engineering/06-ai-coding-cookbook.md))。
>
> **目标读者**:8 年后端工程师,Go 主战场,补 AI / Agent 工程能力。
> **学完能做什么**:能从 0 写一个生产级 single-agent / RAG agent / multi-agent,会评测、能上线、控成本。
>
> **代码示例**:Go(Eino / `anthropic-sdk-go` / `openai-go`)+ Python(LangGraph / LlamaIndex)双栈。
> **学习路线图**:**先读** [`00-agent-learning-roadmap.md`](00-agent-learning-roadmap.md)。

---

## 〇、为什么要重做这个目录

旧目录(00-ai-map / 01-llm-agent-principles / ...)是早期占位文,**视角偏"用 AI"而不是"做 Agent"**,已清理:
- 07/09(Claude Code 实战 + AI 编码手册)→ 迁移到 [`13-engineering/`](../13-engineering/)
- 其他旧文 → 删除(git 历史可查)

**新目录按"原理 → 单 Agent → 多 Agent → 生产化"4 阶段重组**。

---

## 一、目录(4 阶段 12 章)

> ⏳ = 待写;✅ = 已完成。先有骨架再逐章填充。

### 阶段 1:LLM 基础(必须先懂)

| # | 文件 | 内容 | 状态 |
| --- | --- | --- | --- |
| 00 | [agent-learning-roadmap](00-agent-learning-roadmap.md) | 学习路线图 + 8 周计划 + 推荐资料 | ✅ |
| 01 | [llm-fundamentals](01-llm-fundamentals.md) | Token / 上下文窗口 / 温度 / 采样 / 幻觉 / 模型选型 | ✅ |
| 02 | [api-basics](02-api-basics.md) | Anthropic / OpenAI SDK / 流式 / 多模态 / 计费 / Go+Python | ⏳ |
| 03 | [prompt-engineering](03-prompt-engineering.md) | CoT / Few-shot / ReAct / Reflexion / 结构化输出 / XML | ⏳ |

### 阶段 2:单 Agent 构建(核心)

| # | 文件 | 内容 | 状态 |
| --- | --- | --- | --- |
| 04 | [tool-use-function-calling](04-tool-use-function-calling.md) | ★ Tool Schema / 并行调用 / 错误处理 / Claude vs OpenAI | ✅ |
| 05 | [agent-architectures](05-agent-architectures.md) | ★ ReAct / Plan-and-Execute / Reflexion / ToT / 论文复盘 | ⏳ |
| 06 | [memory-and-context](06-memory-and-context.md) | 短期(messages 窗口)/ 长期(向量+KV)/ 摘要 / 滑窗 | ⏳ |
| 07 | [rag-engineering](07-rag-engineering.md) | chunking / embedding / 检索 / Rerank / GraphRAG / Self-RAG | ⏳ |
| 08 | [mcp-protocol](08-mcp-protocol.md) | ★ MCP 协议 / Server-Client / 写一个 MCP server | ⏳ |

### 阶段 3:Agent 框架与多 Agent

| # | 文件 | 内容 | 状态 |
| --- | --- | --- | --- |
| 09 | [agent-frameworks](09-agent-frameworks.md) | LangChain / LangGraph / LlamaIndex / AutoGen / CrewAI / Dify / Eino 选型 | ⏳ |
| 10 | [multi-agent-orchestration](10-multi-agent-orchestration.md) | ★ Supervisor / Hierarchical / Swarm / GroupChat / 通信协议 | ⏳ |

### 阶段 4:生产化(决定能不能上)

| # | 文件 | 内容 | 状态 |
| --- | --- | --- | --- |
| 11 | [evaluation-and-testing](11-evaluation-and-testing.md) | ★ Golden Set / LLM-as-Judge / Trajectory / Promptfoo / LangSmith | ⏳ |
| 12 | [production-engineering](12-production-engineering.md) | 成本 / 监控 / Guardrails / 安全 / 限流 / 可观测 | ⏳ |

### 配套

| # | 文件 | 内容 | 状态 |
| --- | --- | --- | --- |
| 90 | [agent-projects](90-agent-projects.md) | 实战:3 个由浅到深的 agent 项目(single / RAG / multi) | ⏳ |
| 99 | [interview-questions](99-interview-questions.md) | Agent 高频面试题(原 05 内容会重写在这里) | ⏳ |

---

## 二、知识点优先级(8 年工程师视角)

```
P0 必须吃透(决定能不能做出能用的 agent):
  ① LLM token / 上下文 / 计费 / 选型      第 01 章
  ② Tool Use / Function Calling          第 04 章
  ③ ReAct 循环 + 工具失败重试            第 05 章
  ④ RAG 4 件套(chunk/embed/检索/rerank) 第 07 章
  ⑤ MCP 协议(2025 业界标准)             第 08 章
  ⑥ 评测体系(没评测 = 玩具)             第 11 章

P1 建议掌握:
  ⑦ 至少一个框架(LangGraph 或 Eino)     第 09 章
  ⑧ Multi-agent 编排                     第 10 章
  ⑨ 生产化:成本+Guardrails+可观测        第 12 章
  ⑩ Memory 长期化                        第 06 章

P2 了解即可:
  ⑪ 经典论文:ReAct / Reflexion / Voyager / Generative Agents
  ⑫ 各家框架对比
  ⑬ 多模态 / Computer Use
```

---

## 三、和 13-engineering/ 的边界

| 主题 | 在哪 |
| --- | --- |
| **做 Agent**(本目录)| 写 LLM 调用 / Tool Use / RAG / Multi-agent / MCP server |
| **用 AI 编码** | [`13-engineering/05-ai-coding-claude-code.md`](../13-engineering/05-ai-coding-claude-code.md)(Claude Code 实战)|
| | [`13-engineering/06-ai-coding-cookbook.md`](../13-engineering/06-ai-coding-cookbook.md)(用 AI 加速 debug/重构/CR)|

→ 12-ai 是"我做一个 AI 系统";13-engineering 是"我用 AI 系统提效"。

---

## 四、推荐学习入口

**第一次来 → 从这里开始**:[`00-agent-learning-roadmap.md`](00-agent-learning-roadmap.md)

里面有:
- 4 阶段 8 周学习计划
- 每章学完的 "能不能" 验收
- 推荐资料(Anthropic Cookbook / DeepLearning.ai 课程 / 论文)
- 3 个由浅到深的实战项目设计
