# Agent 架构:ReAct / Plan-and-Execute / Reflexion / ToT

> Agent 的"大脑回路"——**同一套 Tool Use,不同的循环控制模式,决定 agent 能不能自主完成复杂任务**。
>
> 本章讲透 5 个经典 agent 架构:**ReAct(最基础)/ Plan-and-Execute(长任务)/ Reflexion(自反思)/ ToT(树探索)/ Self-Refine(迭代改进)**——每个都读源码级理解,配 Go/Python 手写实现。
>
> 前置:[01-llm-fundamentals](01-llm-fundamentals.md) / [04-tool-use-function-calling](04-tool-use-function-calling.md)

## 〇、核心提炼(5 段式)

### 核心机制(5 大架构,必背)

1. **ReAct(Reasoning + Acting)**:Thought → Action → Observation 循环,**agent 的祖师爷架构**,90% agent 底层都是它
2. **Plan-and-Execute**:先出完整计划,再逐步执行(可回头改计划)——**长任务、多步骤任务的首选**
3. **Reflexion**:失败时自我反思,把教训写进 prompt 下次不再犯——**试错学习式 agent**
4. **Tree of Thoughts(ToT)**:分支探索多种思路,评分+回溯——**开放性问题、创意任务**
5. **Self-Refine / Chain-of-Verification**:输出后自审并迭代改进——**质量敏感场景**

### 核心本质(必懂)

> Agent 架构的本质是**"LLM 循环的控制流"**——
> Tool Use 只解决了"agent 能调工具",但**什么时候调、调完再想什么、失败怎么办、如何知道任务完成**——这些是**架构层要解决的**。
>
> 5 大架构的**根本差异是"决策粒度"**:
> - **ReAct**:每一步都决策(一步一想)——灵活但可能绕圈
> - **Plan-and-Execute**:先规划整体,再执行(一次想完再做)——稳但可能计划过时
> - **Reflexion**:执行完复盘学习(做完再想)——适合可试错场景
> - **ToT**:同一步分叉多条路径(想多种)——适合有多个可行解的问题
> - **Self-Refine**:输出后自审(边做边审)——适合追求质量
>
> **实际生产系统**往往是**混合架构**:上层 Plan-and-Execute 拆任务,每个子任务用 ReAct 执行,失败用 Reflexion 复盘,关键输出用 Self-Refine 迭代——**没有银弹,按场景组合**。

### 完整流程(以 ReAct 为例)

```
用户: "查一下北京今天天气,然后建议穿什么"
    ↓
【Thought 1】"我需要先知道今天天气"
【Action 1】get_weather(city="北京")
【Observation 1】"晴, 15℃"
    ↓
【Thought 2】"15℃ 春秋天气,应该建议薄外套"
【Action 2】(不需要工具了,直接答)
    ↓
【Final Answer】"北京今天晴 15℃,建议穿薄外套"
```

**关键**:每一轮 LLM 都要**先输出 Thought**(reasoning),再输出 Action(tool_use),再看 Observation(tool_result)——**思考-行动-观察循环 = ReAct**。

### 5 条核心机制 - 逐点讲透(见下面各节)

### 一句话总结

> Agent 架构 = **LLM 循环的控制流**;
> ReAct 是祖师爷(90% agent 底层),Plan-Execute 适合长任务,Reflexion 适合可试错,ToT 适合开放问题,Self-Refine 追求质量;
> **生产系统混合用**——上层 Plan 拆任务,每子任务 ReAct 执行,失败 Reflexion 学教训,关键输出 Self-Refine 迭代。

---

## 一、ReAct(Reasoning + Acting)★ 最重要

> **Agent 领域最重要的论文**:*ReAct: Synergizing Reasoning and Acting in Language Models*(Yao et al. 2022)
> 现在几乎所有 agent 底层都是 ReAct 变种。

### 1.1 ReAct 的核心思想

**在 tool use 循环里,强制让 LLM 先输出"Thought"再输出"Action"**——把推理过程显式化到 prompt 里,LLM 就能做更好的决策。

```
CoT(Chain-of-Thought):Thought Thought Thought → Answer
                      (只想不动,没有工具)

Act-only:            Action Action Action → Answer
                      (只动不想,盲目试探)

ReAct:               Thought → Action → Observation
                     → Thought → Action → Observation → ... → Answer
                     (想一步做一步)
```

**为什么 ReAct 比 Act-only 好**:
- 强制推理 → 减少盲目工具调用
- 推理写进上下文 → 后续步骤能参考前面思路
- 失败时能反思 → 换策略而不是重复调用

**为什么 ReAct 比 CoT 好**:
- 能拉真实数据(工具)→ 减少幻觉
- 能验证假设 → 中途发现方向错能及时调整

### 1.2 ReAct 的 prompt 结构

```
你是一个助手,能使用工具。回答问题时按以下格式:

Thought: 我需要...(推理你要做什么)
Action: <tool_name>(<args>)
Observation: <工具返回的结果>
... (可以重复多次)
Thought: 我已经知道答案了
Final Answer: <最终答案>

现在开始。
问题: {user_input}
```

**现代 Tool Use API 下的 ReAct**——不需要在 prompt 里强制格式,LLM 天然会输出 tool_use,只需要在 system prompt 提示"先思考再调工具":

```
你是一个 agent。收到任务时:
1. 先思考需要哪些信息,分几步完成
2. 每次调用工具前,简要说明为什么调这个
3. 拿到工具结果后,判断是否达成目标或需要下一步
4. 达成目标后给出最终回答
```

### 1.3 ReAct 手写实现(Python)

```python
from anthropic import Anthropic

client = Anthropic()

SYSTEM_PROMPT = """你是一个能调用工具的 agent。收到任务时:
1. 先分析需要哪些信息
2. 每次调工具前简要说明为什么
3. 拿到结果后判断下一步
4. 达成目标后给出最终答复"""

def react_loop(user_input: str, tools, tool_executor, max_iter=15):
    messages = [{"role": "user", "content": user_input}]
    trajectory = []  # 记录 agent 的推理路径(用于调试/评测)

    for step in range(max_iter):
        resp = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=2048,
            temperature=0.2,     # 低温:决策要稳
            system=SYSTEM_PROMPT,
            tools=tools,
            messages=messages,
        )

        messages.append({"role": "assistant", "content": resp.content})

        # 提取 thought + tool calls
        thoughts = [b.text for b in resp.content if b.type == "text"]
        tool_uses = [b for b in resp.content if b.type == "tool_use"]

        trajectory.append({
            "step": step,
            "thought": "\n".join(thoughts),
            "tools": [(t.name, t.input) for t in tool_uses],
        })

        # 结束条件
        if resp.stop_reason == "end_turn":
            return "\n".join(thoughts), trajectory

        # 执行工具
        tool_results = []
        for t in tool_uses:
            try:
                result = tool_executor(t.name, t.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": t.id,
                    "content": str(result),
                })
            except Exception as e:
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": t.id,
                    "is_error": True,
                    "content": f"错误: {e}",
                })

        messages.append({"role": "user", "content": tool_results})

    return "达到最大轮次", trajectory
```

### 1.4 ReAct 的适用与不适用

**适用**:
- ✓ 需要工具的短-中任务(1-10 步)
- ✓ 探索性任务(不知道要几步)
- ✓ 需要基于中间结果决策
- ✓ 客服 / 助手 / 简单查询

**不适用**:
- ✗ 步骤特别多(> 20 步)→ 上下文膨胀 + 容易忘记全局目标
- ✗ 需要严格顺序 → Plan-Execute 更稳
- ✗ 每一步都很贵 → Plan-Execute 少调用

### 1.5 ReAct 的常见问题

**问题 1:绕圈(反复调同工具)**

```
Thought 1: 查一下 X → 调 tool_A → 失败
Thought 2: 再试试 X → 又调 tool_A → 又失败
Thought 3: 换个方式 → 还是 tool_A ...
```

**解决**:
- 检测**同工具同参数连续调用 3 次** → 强制中断
- 在 system prompt 加"如果同一工具连续失败 2 次,改换策略或告诉用户"

**问题 2:忘记全局目标**

长循环中 LLM 可能被中间细节带偏。

**解决**:每 5 轮把原始任务复述一遍:

```
[第 5 轮开始] 请回顾原始任务: {original_task}
             你还剩什么没做?
```

**问题 3:上下文爆炸**

10 轮之后 message 几万 token,又贵又慢。

**解决**:
- 每 N 轮做 memory 摘要(见 [06-memory-and-context](06-memory-and-context.md))
- 只保留最近 5 轮完整历史 + 早期的摘要

> **一句话**:ReAct 是 agent 底层通用模式——**Thought → Action → Observation 循环**,现代 tool use API 已经天然支持,重点是防绕圈 + 防忘目标 + 防上下文爆炸,**90% 场景先用 ReAct**。

---

## 二、Plan-and-Execute(长任务首选)

### 2.1 核心思想

**先规划完整步骤,再逐步执行**——避免 ReAct "走一步看一步"的短视问题。

```
用户: "帮我写一个 markdown 博客并发布到 GitHub"
    ↓
【Plan 阶段】(一次 LLM 调用)
Plan:
  1. 确定主题和大纲
  2. 写正文
  3. 写 SEO metadata
  4. 转成 markdown
  5. 提交 GitHub
    ↓
【Execute 阶段】(每步一次调用 + 工具)
Step 1: 用户输入主题... → 完成
Step 2: 用 LLM 写正文 → 完成
Step 3: ...
    ↓
所有 step 完成 → 输出结果
```

### 2.2 变种:动态 Plan(Plan-and-Solve)

**问题**:纯 Plan-Execute 计划一次做完,如果计划过时(如某步失败 / 发现新信息)就会翻车。

**改进**:每完成一步后**允许修改剩余计划**——这是 LangChain LangGraph 的主推模式。

```
Plan → Execute Step 1 → 观察结果 → 需要改计划?
                                    是 → Replan → 继续
                                    否 → Execute Step 2 → ...
```

### 2.3 手写实现(Python)

```python
def plan_and_execute(user_input, tools, tool_executor, max_replan=3):
    # 1. Planning phase
    plan_prompt = f"""
    任务:{user_input}
    可用工具:{[t['name'] for t in tools]}
    
    请把任务拆分成 3-10 个可执行步骤,输出 JSON:
    {{"steps": [
        {{"id": 1, "description": "...", "expected_output": "..."}},
        ...
    ]}}
    """
    plan_resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        temperature=0.3,
        messages=[{"role": "user", "content": plan_prompt}]
    )
    plan = parse_json(plan_resp.content[0].text)
    
    # 2. Execute phase
    executed = []
    for step in plan["steps"]:
        # 用 ReAct 执行这一步
        result, trajectory = react_loop(
            user_input=f"完成这一步: {step['description']}\n上下文: {executed}",
            tools=tools,
            tool_executor=tool_executor,
            max_iter=5
        )
        executed.append({"step": step, "result": result})
        
        # Replan?(可选)
        if should_replan(result, plan):
            plan = replan(plan, executed, remaining_steps=...)
    
    # 3. Synthesize final answer
    return synthesize(executed)
```

### 2.4 适用场景

**适合**:
- ✓ 步骤明确的长任务(写文档、写代码、数据分析)
- ✓ 需要资源规划(计算成本 / 时间)
- ✓ 任务可以事前分解

**不适合**:
- ✗ 高度探索性(计划完全不可预知)
- ✗ 每步依赖前步结果(动态 branching)

### 2.5 关键工程点

**工程点 1:Plan 的粒度**

```
❌ 太粗:1-2 步 → 等于没规划
❌ 太细:30+ 步 → LLM 规划本身容易错
✓ 3-10 步是甜蜜点
```

**工程点 2:Replan 频率**

```
每步都 replan:太贵,失去规划意义
从不 replan:计划过时也硬做
折中:失败或有 surprise 时 replan
```

**工程点 3:Plan 和 Execute 用不同模型**

```
Plan:Sonnet 或 Opus(规划要有战略视野)
Execute:Haiku 或 mini(执行只需要跟着做,便宜快)
```

> **一句话**:Plan-and-Execute 适合长任务——**先出完整计划再逐步执行**,可以 Replan 应对 surprise;Plan 用大模型,Execute 用小模型省钱;粒度 3-10 步是甜蜜点。

---

## 三、Reflexion(自反思 / 失败中学习)

### 3.1 核心思想

**Reflexion 论文**:Shinn et al. 2023——*Reflexion: Language Agents with Verbal Reinforcement Learning*

**做法**:每次任务失败后,让 LLM 自己反思"我为什么失败",把反思结果写进下次的 prompt。

```
Attempt 1: 完成任务 → 失败 / 结果不佳
    ↓
Reflection: LLM 反思"为什么失败,下次怎么改"
    ↓
存到 reflection memory
    ↓
Attempt 2: 任务 + 之前的 reflection → LLM 参考着重来
    ↓
成功 or 继续反思重来
```

### 3.2 Reflexion 的三个组件

```
1. Actor:执行任务的 agent(可以是 ReAct)
2. Evaluator:判断结果好坏(LLM-as-Judge / 单元测试 / 规则)
3. Self-Reflector:失败时反思,产出改进建议
```

### 3.3 Reflexion 用于代码生成(经典)

**任务**:让 agent 写函数通过单元测试。

```
Attempt 1:
  写了 def fib(n): return n if n < 2 else fib(n-1) + fib(n-2)
  跑单测:test_fib(10) 超时 ← 失败

Reflection:
  "递归实现有性能问题,应该用动态规划或 memoization"
  存到 memory

Attempt 2:(memory: 递归有性能问题,用 DP)
  写了 def fib(n):
      dp = [0, 1]
      for i in range(2, n+1): dp.append(dp[i-1] + dp[i-2])
      return dp[n]
  跑单测: 通过 ← 成功
```

### 3.4 手写实现(伪代码)

```python
def reflexion_loop(task, max_attempts=5):
    memory = []  # reflection history
    
    for attempt in range(max_attempts):
        # 用 reflection memory 组装 prompt
        context = "\n".join([f"过去的教训 {i}: {r}" for i, r in enumerate(memory)])
        
        # Actor:执行任务
        result = actor_agent(f"{task}\n\n{context}")
        
        # Evaluator:评估
        verdict = evaluator(task, result)  # {"success": True/False, "score": ...}
        
        if verdict["success"]:
            return result
        
        # Self-Reflector:失败反思
        reflection_prompt = f"""
        任务: {task}
        我的尝试: {result}
        评价: {verdict}
        请反思为什么失败,下次怎么改进(1-3 句话)
        """
        reflection = client.messages.create(...)
        memory.append(reflection.text)
    
    return "达到最大尝试次数"
```

### 3.5 适用与不适用

**适合**:
- ✓ 有明确评估信号的任务(代码通过单测 / 数学题对错)
- ✓ 允许多次尝试(离线任务)
- ✓ 任务模式可以从教训中提炼

**不适合**:
- ✗ 无法快速评估的任务(创意写作)
- ✗ 每次尝试代价极高(每次 1 小时)
- ✗ 实时响应场景(用户不会等你 5 次尝试)

### 3.6 和微调的关系

**Reflexion ≠ 微调**——
- Reflexion 是**运行时 prompt 层的反思**,不改模型权重
- 微调是**改模型权重**
- Reflexion 优势:即时生效 / 无训练成本 / 可解释
- 微调优势:泛化到未见过任务

> **一句话**:Reflexion = **失败反思 → 教训写进 prompt → 再来**;适合有明确评估信号的任务(代码/数学),不适合创意或实时场景;是"运行时学习",不改模型权重。

---

## 四、Tree of Thoughts(ToT)

### 4.1 核心思想

**ToT 论文**:Yao et al. 2023——*Tree of Thoughts: Deliberate Problem Solving with Large Language Models*

**做法**:同一步骤生成**多个候选思路**,评分选优,失败可**回溯到分岔点重试**。

```
问题: 24 点游戏(用 4/5/6/8 得到 24)

Root: [4, 5, 6, 8]
  ├─ 思路 A: 4 * 6 = 24, 剩 [24, 5, 8], 需要 5-8=... ← 死胡同
  ├─ 思路 B: 8 - 6 = 2, 剩 [4, 5, 2], 5 - 2 = 3, 3 * 4 =12 ✗
  ├─ 思路 C: 8 - 5 = 3, 剩 [4, 6, 3], 3 * 4 = 12, 12 * ? ✗
  └─ 思路 D: 5 - 4 = 1, 剩 [6, 8, 1], 8 / 1 = 8, 8 * 6 =48 ✗
  
回溯 root 换新分支...
```

### 4.2 ToT 的四个关键决策

```
1. 思路分解: 一步生成几个候选?(通常 3-5)
2. 状态评估: 每个候选打分(LLM 评价 or 规则)
3. 搜索算法: BFS / DFS / Best-First?
4. 剪枝策略: 什么分支放弃?
```

### 4.3 简化实现

```python
def tot_solve(problem, depth=4, breadth=3):
    root = {"state": problem, "history": []}
    frontier = [root]
    
    for step in range(depth):
        new_frontier = []
        for node in frontier:
            # 每个节点生成 breadth 个候选
            candidates = generate_candidates(node["state"], k=breadth)
            for c in candidates:
                score = evaluate(c, problem)
                new_frontier.append({
                    "state": c["state"],
                    "history": node["history"] + [c["step"]],
                    "score": score
                })
                if is_solution(c["state"], problem):
                    return c
        
        # 剪枝:保留分数最高的 breadth 个
        new_frontier.sort(key=lambda x: -x["score"])
        frontier = new_frontier[:breadth]
    
    return best(frontier)
```

### 4.4 适用场景

**适合**:
- ✓ 开放性推理问题(数学 / 逻辑题)
- ✓ 有多个可行解(创意 / 方案设计)
- ✓ 单路径 CoT 容易翻车的任务

**不适合**:
- ✗ 简单查询任务
- ✗ 每步都很贵(ToT 成本 = breadth * depth 倍)
- ✗ 实时场景(慢)

### 4.5 ToT 的代价

```
CoT / ReAct:  1x LLM 调用
ToT(3x3):    9-30x LLM 调用

→ 只在必要时用,不是通用架构
```

> **一句话**:ToT = **一步生成多个思路,评分+剪枝+回溯**;适合开放推理和有多解的问题,代价是 10-30 倍 LLM 调用;**不是通用架构,少用**。

---

## 五、Self-Refine / Chain-of-Verification(自审迭代)

### 5.1 Self-Refine

**思路**:LLM 输出 → LLM 自己评审 → 根据评审改进 → 循环几次。

```
Draft 1:LLM 生成初稿
    ↓
Critique:LLM 评审("哪里不好?怎么改?")
    ↓
Revise:LLM 根据评审改进
    ↓
Draft 2 → Critique → Revise → ...
```

### 5.2 Chain-of-Verification(CoVe)

**思路**:LLM 输出后,针对每个事实性断言生成**验证问题**,回答验证问题,发现矛盾则修正。

```
LLM 答: "Kubernetes 是 Google 2014 年开源的容器编排系统,
        当前最新版本是 1.30"

CoVe 生成验证问题:
  Q1: Kubernetes 由哪家公司开源?什么年份?
  Q2: Kubernetes 当前最新版本是?

LLM 回答验证问题:
  A1: Google, 2014年 ✓
  A2: 需要查(可能过时)← 用 tool 验证

发现矛盾 → 修正原答复
```

### 5.3 手写 Self-Refine

```python
def self_refine(task, max_iter=3):
    draft = client.messages.create(
        messages=[{"role": "user", "content": task}]
    ).content[0].text
    
    for i in range(max_iter):
        # Critique
        critique = client.messages.create(
            messages=[{"role": "user", "content": f"""
                任务:{task}
                当前答复:{draft}
                请指出答复中的问题(3 条以内),如果没问题回答"OK"。
            """}]
        ).content[0].text
        
        if "OK" in critique:
            return draft
        
        # Revise
        draft = client.messages.create(
            messages=[{"role": "user", "content": f"""
                任务:{task}
                原答复:{draft}
                问题:{critique}
                请根据问题改进答复。
            """}]
        ).content[0].text
    
    return draft
```

### 5.4 适用场景

**适合**:
- ✓ 质量敏感的输出(报告 / 代码 / 邮件)
- ✓ 事实性任务(避免幻觉)
- ✓ 允许几倍成本换质量

**不适合**:
- ✗ 简单查询
- ✗ 实时响应(每次多花几秒)
- ✗ 已经很稳定的任务(浪费钱)

### 5.5 代价 vs 收益

```
成本:每次输出 3-5x LLM 调用
收益:
  - 事实错误减少 30-50%(CoVe 实测)
  - 代码 bug 减少 20-30%
  - 用户体验提升(输出更精炼)

选择:关键输出场景开 Self-Refine,一般对话不开
```

> **一句话**:Self-Refine = **草稿 → 自审 → 改进循环**;CoVe = **对每个断言生成验证问题**;适合质量敏感场景,代价是 3-5 倍调用,一般对话不开。

---

## 六、混合架构(生产实践)

### 6.1 现实中不会只用一种架构

**典型的生产 agent 是分层混合**:

```
用户请求
    ↓
【L1 Router】(Haiku)
判断是简单查询还是复杂 agent 任务
    ↓
简单 → 单次 LLM 调用回答
复杂 → 进入 agent 流程
    ↓
【L2 Planner】(Sonnet)Plan-and-Execute
把任务拆成 5-10 步
    ↓
【L3 Executor】(Sonnet / Haiku)ReAct
每一步用 tool use 循环执行
    ↓
【L4 Verifier】(Sonnet)Self-Refine / CoVe
关键输出自审
    ↓
【L5 Reflector】(离线)Reflexion
失败案例收集,写入知识库供下次参考
    ↓
返回结果
```

### 6.2 Anthropic 官方 agent 模式(Building Effective Agents)

**Anthropic 2024 年发布的最佳实践**——真正上生产的 agent 一般是几个基础模式的组合:

| 模式 | 何时用 |
| --- | --- |
| **Prompt Chaining** | 顺序步骤,每步简单(先分类再生成) |
| **Routing** | 根据输入类型走不同分支 |
| **Parallelization** | 并行多个子任务(voting / sectioning) |
| **Orchestrator-Workers** | 主 agent 拆任务 + 派发给 sub-agent |
| **Evaluator-Optimizer** | 生成 + 评审循环(Self-Refine) |
| **Autonomous Agents** | 完整 ReAct 循环(最强也最难控) |

**关键原则**:
- **Start simple**:能用 workflow 就别上 agent
- **Add complexity when needed**:先用 chaining/routing,不够再上 agent
- **Measure**:每个层加了没加价值,靠评测数据说话

### 6.3 何时**不该**用 Agent

```
任务确定性强 + 不需要工具 → 直接 LLM 调用
一次完成不用循环 → 用 workflow / chaining
高频低成本场景 → agent 烧不起
强一致性要求 → agent 不稳定
```

> **一句话**:生产系统混合用——**Router(Haiku 路由)+ Plan-Execute(Sonnet 规划)+ ReAct(执行)+ Self-Refine(关键输出自审)+ Reflexion(离线学习)**;能用 workflow 就别上 agent,能用简单模式就别用复杂 agent。

---

## 七、经典 agent 项目(必看源码)

### 7.1 AutoGPT(第一波爆火)

- **架构**:自主 agent,LLM 自己给自己下任务
- **问题**:上下文爆炸 + 目标漂移 + 烧 token
- **意义**:开创了自主 agent 概念,但生产化用不了

### 7.2 BabyAGI

- **架构**:任务队列 + priority + 循环执行
- **意义**:任务分解 + 优先级思路很好

### 7.3 Voyager(Minecraft Agent)

- **架构**:ReAct + 技能库(学会的动作永久保存)
- **意义**:**Lifelong learning agent 的雏形**——学到的能力可复用

### 7.4 SWE-agent

- **架构**:专门修 GitHub Issue,自定义 shell + editor tool
- **成绩**:SWE-bench 12.5%(Claude 3.5 加持后 33%+)
- **意义**:**Coding agent 的标杆**,证明 agent 能做真实工程任务

### 7.5 Devin(Cognition Labs)

- **架构**:未公开细节,但明显是 Plan + Execute + Memory 组合
- **争议**:demo 好但生产落地质疑多
- **意义**:引爆商业 coding agent 赛道

### 7.6 Generative Agents(Stanford 小镇)

- **架构**:每个 agent 有记忆 + 反思 + 计划
- **意义**:**Multi-agent 模拟社会的雏形**

### 7.7 Anthropic 官方 SWE agent(2024)

- **架构**:ReAct + 内置 bash/text_editor tool + max_iterations
- **意义**:Claude Code 的核心架构公开

---

## 八、常见坑

```
坑 1:上来就用 ToT / Reflexion
  → 简单任务用简单架构,ReAct 能搞定绝大多数

坑 2:ReAct 死循环烧 token
  → 必须 max_iterations + 检测重复调用

坑 3:Plan-and-Execute 计划过时不 Replan
  → 出现 surprise 必须重新规划

坑 4:忘记全局目标(长任务)
  → 每 N 轮复述任务目标

坑 5:上下文爆炸(长任务)
  → memory 摘要,只保留最近 K 轮

坑 6:Executor 用 Opus 烧钱
  → Plan 用大模型,Execute 用 Haiku / mini

坑 7:Reflexion 无评估信号硬做
  → 没有明确成功/失败判定,反思等于瞎猜

坑 8:自主 agent 上生产
  → AutoGPT 类完全自主 agent 生产用不了
  → 加人工审批 + 边界限制
```

## 九、面试题速答

### Q1:讲一下 ReAct

```text
ReAct = Reasoning + Acting,agent 领域最重要的架构。
核心是让 LLM 强制"先输出 Thought 再输出 Action",
把推理过程显式化到上下文,让 LLM 做更好的决策。

循环: Thought → Action(tool_use) → Observation(tool_result) → Thought → ...

vs CoT: 只想不动,靠训练数据回答,会幻觉
vs Act-only: 只动不想,盲目试探

现代 tool use API 天然支持 ReAct,重点是:
  - 防绕圈(检测重复调用)
  - 防忘目标(定期复述)
  - 防上下文爆炸(memory 摘要)
```

### Q2:什么时候用 Plan-Execute 而不是 ReAct?

```text
任务步骤特别多(> 10)+ 步骤明确 → Plan-Execute
  优点: 有全局视角,不会走一步看一步走偏
  代价: 计划过时要 Replan

短任务、探索性任务、每步依赖前步 → ReAct
  优点: 灵活
  代价: 长任务容易忘目标 / 绕圈

生产实践: 上层 Plan-Execute 拆任务,每子任务 ReAct 执行
```

### Q3:Reflexion 是什么?什么场景用?

```text
Reflexion = 失败反思 → 教训写进 prompt → 再试
本质是 "运行时 prompt 层的学习",不改模型权重。

组件: Actor(执行) + Evaluator(评估) + Self-Reflector(反思)

适合: 有明确评估信号的任务(代码通过单测/数学题对错)
不适合: 无法快速评估的任务 / 实时响应场景

vs 微调: Reflexion 即时生效无训练成本,但泛化不如微调
```

### Q4:Anthropic 官方推荐的 agent 模式?

```text
2024 年 "Building Effective Agents" 提出的 6 种:
  1. Prompt Chaining(顺序步骤)
  2. Routing(根据输入分流)
  3. Parallelization(并行子任务)
  4. Orchestrator-Workers(主 agent + sub-agent)
  5. Evaluator-Optimizer(生成 + 评审循环)
  6. Autonomous Agents(完整 ReAct)

核心原则:
  - Start simple(能用 workflow 就别上 agent)
  - Add complexity when needed
  - Measure everything(靠评测数据说话)
```

### Q5:自主 agent(AutoGPT 类)生产能用吗?

```text
不能直接用,主要问题:
  1. 上下文爆炸(长任务几万 token)
  2. 目标漂移(走着走着忘了任务)
  3. 烧 token(自主任务分解容易失控)
  4. 不可控(可能做危险操作)

生产化改造:
  - 加 max_iterations 上限
  - 关键操作人工审批
  - Sandbox 执行
  - Memory 摘要防爆炸
  - 定期复述目标
  - 用 Plan-Execute 收敛而非纯自主

Anthropic 官方也说: agent 强大但成本高,能用 workflow 就用 workflow
```

## 十、关联阅读

```
本目录:
- 04-tool-use-function-calling      Tool Use(ReAct 的基础)
- 06-memory-and-context             Memory(解决 agent 上下文爆炸)
- 09-agent-frameworks               LangGraph / Eino 落地
- 10-multi-agent-orchestration      多 agent 编排
- 11-evaluation-and-testing         Trajectory 评测

外部:
- ReAct 论文: arxiv.org/abs/2210.03629
- Reflexion 论文: arxiv.org/abs/2303.11366
- Tree of Thoughts: arxiv.org/abs/2305.10601
- Building Effective Agents (Anthropic): anthropic.com/research/building-effective-agents
```

> **一句话核心(全篇精炼)**:
> Agent 架构 = **LLM 循环的控制流**;
> **ReAct(祖师爷,一步一想)+ Plan-Execute(长任务,一次想完)+ Reflexion(失败反思)+ ToT(多路径探索)+ Self-Refine(输出自审)**;
> 生产系统混合用——**Router 分流 → Plan 拆任务 → ReAct 执行 → Self-Refine 关键输出 → Reflexion 离线学习**;
> **能用 workflow 就别上 agent,能用简单 agent 就别上自主 agent**——这是 8 年后端做 Agent 最重要的边界感。
