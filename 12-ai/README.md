# AI

> 面向程序员和资深后端面试的 AI / Agent 笔记：理解 LLM 原理、Coding Agent、Skills、MCP、研发工作流和未来趋势。

## 分类导航

### 基础与原理

| 文件 | 内容 |
| --- | --- |
| [00-ai-map.md](00-ai-map.md) | AI / LLM / Agent 知识地图：模型、RAG、工具调用、Agent、Coding Agent |
| [01-llm-agent-principles.md](01-llm-agent-principles.md) | LLM 与 Agent 核心原理：Token、上下文、工具、记忆、规划、评估 |

### 编程 Agent 与 Skills

| 文件 | 内容 |
| --- | --- |
| [02-coding-agent-comparison.md](02-coding-agent-comparison.md) | Claude Code、OpenAI Codex、Hermes Agent 等编程 Agent 对比 |
| [03-agent-skills-mcp.md](03-agent-skills-mcp.md) | Skills、MCP、AGENTS.md：如何把个人经验变成可复用能力 |
| [04-developer-ai-workflow.md](04-developer-ai-workflow.md) | 程序员如何高效使用 AI：需求拆解、代码生成、评审、测试、重构 |

### 面试与趋势

| 文件 | 内容 |
| --- | --- |
| [05-ai-interview-questions.md](05-ai-interview-questions.md) | AI 高频面试题：LLM、RAG、Agent、向量库、提示词、评估与安全 |
| [06-ai-future.md](06-ai-future.md) | AI 工程未来展望：Agentic Coding、多 Agent、长期记忆、端侧模型、组织知识库 |

## 高频题速览

### LLM 原理

- Token、上下文窗口、Embedding 分别是什么？
- 为什么 LLM 会幻觉？
- RAG 解决什么问题？不能解决什么问题？
- Function Calling / Tool Calling 的本质是什么？
- Agent 和普通 ChatBot 的区别是什么？

### Coding Agent

- Claude Code、Codex、Hermes Agent 的定位有什么不同？
- Coding Agent 为什么需要读仓库、运行命令和提交代码？
- Cloud Agent 和本地 CLI Agent 怎么选？
- 为什么 Agent 需要测试、沙箱和权限控制？
- 如何评价一个 Coding Agent 真的有用？

### Skills / MCP / 项目知识

- Skill 是什么？和 prompt 有什么区别？
- MCP 解决什么问题？
- AGENTS.md / CLAUDE.md 这类项目说明应该怎么写？
- 如何把团队编码规范、发布流程、排障经验沉淀成 Skill？
- Skill 什么时候会适得其反？

### 程序员使用 AI

- 哪些任务适合交给 AI？
- 如何让 AI 写出可维护代码？
- 如何避免 AI 乱改代码？
- 如何让 AI 做 code review？
- 如何把 AI 融入 TDD、重构、排障和文档？

### 未来趋势

- Agentic Coding 会如何改变研发流程？
- IDE Copilot、CLI Agent、Cloud Agent 会如何分工？
- 个人 Skills 和企业知识库有什么价值？
- 开源模型和闭源模型如何共存？
- 程序员应该提升哪些能力，避免被 AI 放大短板？

## 复习路径

1. 先看 [00-ai-map.md](00-ai-map.md)，建立 AI / Agent 知识地图。
2. 再看 [01-llm-agent-principles.md](01-llm-agent-principles.md)，理解 LLM、RAG、Tool、Agent 的原理边界。
3. 看 [02-coding-agent-comparison.md](02-coding-agent-comparison.md)，理解 Claude Code、Codex、Hermes 的差异。
4. 看 [03-agent-skills-mcp.md](03-agent-skills-mcp.md) 和 [04-developer-ai-workflow.md](04-developer-ai-workflow.md)，落到程序员日常使用。
5. 最后看 [05-ai-interview-questions.md](05-ai-interview-questions.md) 和 [06-ai-future.md](06-ai-future.md)，准备面试和长期能力建设。

## 答题原则

- 不把 AI 讲成玄学，要讲清输入、上下文、工具、反馈和评估。
- 不盲目吹 Agent，要讲清适用场景、失败模式和安全边界。
- 程序员使用 AI 的核心不是“让它替你写”，而是“让它在清晰约束下加速工程闭环”。
- 高级答案要体现：任务拆解、验证意识、上下文治理、知识沉淀、风险控制。
