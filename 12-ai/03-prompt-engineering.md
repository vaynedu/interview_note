# Prompt Engineering 进阶

> **Prompt 是编程 LLM 的"语言"** —— 同样的模型 + 同样的任务,好 prompt 和差 prompt 的效果能差 30-50 个百分点。做 Agent 的核心工程能力之一,不是"随便写句话",是有方法有套路的工程。
>
> 本章讲透 **从 Zero-shot 到 Reflexion 全谱系 / 结构化输出 / XML vs JSON / Prompt 版本管理 / 各家模型特点差异** —— 8 年后端做 Agent 的 prompt 工程系统方法。
>
> 前置:[01-llm-fundamentals](01-llm-fundamentals.md)(采样/幻觉/token 概念) / [02-api-basics](02-api-basics.md)(system 字段)

## 〇、核心提炼(5 段式)

### 核心机制(7 大 prompt 技术,必背)

1. **Zero-shot**:直接给指令,啥示例都不给 —— 基线
2. **Few-shot**:给几个"输入-输出"示例 —— 拉起简单任务上限
3. **Chain-of-Thought (CoT)**:让 LLM 一步步想 —— 复杂推理杀手锏
4. **Self-Consistency**:多次采样投票 —— 数学/推理任务
5. **ReAct**:Thought + Action 交替 —— Agent 底层(见 [05](05-agent-architectures.md))
6. **Reflexion**:失败自我反思重来 —— 代码/数学任务(见 [05](05-agent-architectures.md))
7. **Constitutional AI**:让 LLM 按"宪法"自我修正 —— 安全对齐

### 核心本质(必懂)

> Prompt 工程的本质是 **"用自然语言编程 LLM"**——
>
> 类比传统编程:
> - **Prompt = 代码**(告诉 LLM 做什么、怎么做)
> - **Few-shot 示例 = 测试用例**(展示期望行为)
> - **CoT = 注释 + 中间变量**(让思路显式化)
> - **结构化输出 = 类型约束**(强制格式)
> - **System prompt = 常量 / config**(全局设定)
>
> **8 年后端要摆脱的认知**:
> - ✗ "Prompt 就是提示句"(低估了工程性)
> - ✗ "写 prompt 靠感觉"(其实有系统方法)
> - ✗ "越长越好"(冗余反而稀释效果)
> - ✗ "英文效果比中文好"(2026 Claude/GPT 中文都很好)
>
> **好 prompt 的 5 大特征**:
> 1. **明确**(说清楚要什么,不含糊)
> 2. **结构化**(用 XML/Markdown 分区)
> 3. **有示例**(few-shot 引导)
> 4. **约束输出**(JSON Schema / 格式)
> 5. **可版本化**(像代码一样管理)

### 完整流程(prompt 优化闭环)

```
1. 起草 Zero-shot Prompt
   最小化:告诉 LLM 做什么

2. 跑 Golden Set 评测(见 [11](11-evaluation-and-testing.md))
   → 看基线效果

3. 迭代优化:
   ├── 加 few-shot 示例(3 个够)
   ├── 加 CoT("先分析再回答")
   ├── 结构化(XML 分区 / JSON 输出)
   ├── System prompt 加约束(风格/边界/拒答)
   └── 每次改一个变量,对比效果

4. 定版本 → 存代码 / prompt 管理平台

5. 上线后监控:
   badcase 收集 → 加入 Golden Set → 反哺下轮优化
```

### 一句话总结

> Prompt = **用自然语言编程 LLM**;
> **7 大技术**(Zero-shot/Few-shot/CoT/Self-Consistency/ReAct/Reflexion/Constitutional)+ **5 大特征**(明确/结构化/示例/约束/版本化);
> 好 prompt 和差 prompt 在同任务差 30-50pp,是 agent 效果最大的杠杆之一;
> **像写代码一样写 prompt**——版本化、评测、迭代。

---

## 一、Prompt 演进路径(全谱系)

### 1.1 演进图

```
【基础】
  Zero-shot: 直接问
      ↓ 效果不够,加示例
  Few-shot: 给几个 (input, output) 示例
      ↓ 复杂推理翻车,让它慢下来
  Chain-of-Thought: 一步步想

【提升可靠】
  Self-Consistency: 多次采样投票
  CoVe(Chain-of-Verification): 输出后自己验证

【Agent 化】
  ReAct: 思考 + 行动交替
  Plan-and-Execute: 先规划再执行
  Reflexion: 失败反思重来
  Tree of Thoughts: 分支探索

【安全对齐】
  Constitutional AI: LLM 按原则自我修正
  RLHF/DPO: 训练时对齐(不是 prompt 层)
```

前 4 类是 **纯 prompt 层技巧**,后 3 类涉及 **agent 架构层**(见 [05](05-agent-architectures.md))。

### 1.2 什么时候用哪个

| 任务 | 首选 |
| --- | --- |
| 简单指令(翻译/摘要/分类) | Zero-shot / Few-shot |
| 需要按特定格式输出 | Few-shot + JSON Schema |
| 复杂推理(数学/逻辑) | CoT / Self-Consistency |
| 事实性问答 | RAG + "承认不知道" prompt |
| 代码生成 | CoT + Few-shot + 输出格式约束 |
| 多步骤任务 | ReAct / Plan-and-Execute |
| 高质量输出 | Self-Refine / CoVe |
| 从错误学习 | Reflexion |
| 安全场景 | Constitutional AI + Guardrails |

---

## 二、Zero-shot(基础)

### 2.1 什么是 Zero-shot

**直接告诉 LLM 做什么,不给任何示例**:

```
把下面的英文翻译成中文:
{text}
```

### 2.2 什么时候够用

- ✓ LLM 训练时见过的常见任务(翻译/摘要/QA)
- ✓ 现代模型(Claude Sonnet+/GPT-4+)对常见任务 zero-shot 已经很好
- ✗ 需要特定格式 / 罕见任务 / 复杂推理

### 2.3 Zero-shot 优化技巧

**技巧 1:明确任务边界**

```
❌ 差: "总结这段"
✓ 好: "用中文,3 句话内,总结下面文本的核心观点(不复述细节)"
```

**技巧 2:明确输出格式**

```
❌ 差: "分析代码"
✓ 好: "分析下面代码,输出:
       1. 主要功能(1 句)
       2. 潜在 bug(bullets)
       3. 优化建议(bullets)"
```

**技巧 3:角色设定(role)**

```
"你是一位有 10 年经验的 Go 工程师。
分析下面代码..."
```

角色设定能激活 LLM 特定知识域,通常有 5-15% 效果提升。

> **一句话**:Zero-shot 现代模型对常见任务已很好,关键是**明确任务边界 + 输出格式 + 角色设定**;简单任务先试 zero-shot,不够再加 few-shot。

---

## 三、Few-shot(黄金比例)

### 3.1 什么是 Few-shot

**在 prompt 里给 N 个 (input, output) 示例**,LLM 会模仿示例格式:

```
把英文翻译成中文,保持语气。

英文: Hello, how are you?
中文: 你好,你怎么样?

英文: I'm fine, thanks.
中文: 我很好,谢谢。

英文: What a beautiful day!
中文:
```

### 3.2 Few-shot 的效果曲线

```
0 示例(zero-shot):   基线
1 示例:              +10-20% (有格式引导)
3 示例:              +20-30% (甜蜜点)
5 示例:              +25-35% (边际递减)
10+ 示例:            +25-35% (基本没提升,成本翻倍)
```

**3 个示例是甜蜜点**——够引导又不烧钱。

### 3.3 Few-shot 的选例技巧

**技巧 1:覆盖不同情况**

```
不要 3 个都是简单例子
要包含:典型 case + 边界 case + 拒答 case
```

**技巧 2:示例质量 > 数量**

```
❌ 差:10 个平庸例子
✓ 好:3 个精心挑选的高质量例子
```

**技巧 3:分隔清晰**

```
❌ 差:
输入:X 输出:Y 输入:A 输出:B ...

✓ 好(用 XML 分区):
<example>
<input>X</input>
<output>Y</output>
</example>
<example>
<input>A</input>
<output>B</output>
</example>
```

**技巧 4:动态选例(RAG 式 few-shot)**

```
预存 100 个 (input, output) 示例(向量化)
新 query → 向量检索最相似 3 个作为 few-shot
```

**优点**:每次用最相关的示例,不用固定死。

### 3.4 Few-shot 的坑

**坑 1:示例格式不一致 → LLM 学错**

```
❌ 差:
示例 1: "Answer: XX"
示例 2: "答案: XX"
示例 3: "Result: XX"
→ LLM 不知道用哪种

✓ 好:三个示例完全同格式
```

**坑 2:示例太长 → 稀释效果 + 烧钱**

```
每个示例控制在核心信息,不要塞完整对话
```

**坑 3:类别不均衡**

```
分类任务,3 个示例都是同一类
→ LLM 偏向那一类
→ 类别均衡采样
```

> **一句话**:Few-shot 是拉起简单任务效果最快的手段——**3 个高质量示例是甜蜜点**,覆盖典型+边界+拒答,XML 分区保持格式一致,复杂场景用 RAG 动态选例。

---

## 四、Chain-of-Thought(CoT,推理神器)

### 4.1 什么是 CoT

**让 LLM 一步步"想"再给答案,而不是直接给答案**:

```
问题: 张三有 5 个苹果,给了李四 2 个,又买了 3 个,现在他有几个?

❌ 无 CoT: "6"(可能错)

✓ CoT:
让我一步步算:
- 一开始:5 个
- 给了李四 2 个:5-2=3 个
- 又买了 3 个:3+3=6 个
所以现在有 6 个苹果。
```

### 4.2 CoT 的两种触发方式

**方式 1:显式提示(Zero-shot CoT)**

```
问题: ...

Let's think step by step.
```

或中文:

```
让我们一步步来分析。
先...再...最后...
```

**Kojima et al. 2022 的经典**:一句 "Let's think step by step" 就能让 GSM8K 数学题准确率从 17% 提升到 78%。

**方式 2:Few-shot CoT**

```
问题: A 有 3 个球,给 B 2 个,买 4 个,现在几个?
思考: 
  开始 3 个,给出 2 个后剩 3-2=1 个,再买 4 个后 1+4=5 个。
答案: 5

问题: X 有 10 元,买 3 元的东西后又赚 5 元,现在多少?
思考:
  开始 10 元,花 3 元后 10-3=7 元,赚 5 元后 7+5=12 元。
答案: 12

问题: {your question}
思考:
```

### 4.3 CoT 的效果场景

**大幅提升**:
- 数学计算
- 逻辑推理
- 多步骤决策
- 复杂 QA

**微弱或无提升**:
- 简单查询
- 直接检索式任务
- 短文本生成

### 4.4 CoT 的成本代价

```
CoT 让 LLM 输出 3-10x 更多 token(思考过程)
Output token 比 input 贵 3-5x

一次 CoT = 一次直接回答的 5-15x 成本

不是每个 case 都值得,复杂任务再上
```

### 4.5 现代模型的"内置 CoT"

**Claude / GPT-4+ 现代模型对复杂任务会自动 CoT**,不用手动加 "think step by step"。

**深度推理模型**(OpenAI o1/o3, DeepSeek R1):
- 内部**自动做很长 CoT**
- 输出前的"thinking tokens"都要付费
- 但准确率大幅提升

### 4.6 CoT 的进阶变体

**Zero-shot CoT**:一句 "让我一步步想"
**Few-shot CoT**:示例包含思考过程
**Auto-CoT**:自动生成 CoT 示例
**Least-to-Most**:任务分解,先易后难
**Program-of-Thoughts (PoT)**:用代码代替自然语言思考
**Self-Consistency**:多次 CoT + 投票(见 §五)

> **一句话**:CoT = **让 LLM 一步步想再答**,zero-shot 一句 "Let's think step by step" 或 few-shot 带思考示例;复杂推理神器,数学题准确率能从 17% 提到 78%,但**成本 5-15x**,简单任务不必用;现代模型对复杂任务已内置 CoT。

---

## 五、Self-Consistency(投票增强)

### 5.1 核心思想

**同一 prompt 多次采样(高温度),投票选最多的答案**:

```
Prompt(temperature=0.7): "24+37=?"

采样 5 次:
  1. "让我算...61"
  2. "24+37=61"
  3. "24+37=61"
  4. "让我加...61"
  5. "60"(错)

投票:61 出现 4 次 → 最终答案 61
```

### 5.2 效果

**Wang et al. 2022**:GSM8K 上 CoT + Self-Consistency 比纯 CoT 提升 **17.9 个百分点**。

### 5.3 代价

**N 倍成本**:
- 5 次采样 = 5 倍 LLM 调用
- 5-10 次是常用范围
- 只在关键任务用(数学 / 高价值决策)

### 5.4 实现

```python
def self_consistency(prompt, n=5, temperature=0.7):
    answers = []
    for _ in range(n):
        resp = client.messages.create(
            model="claude-sonnet-4-6",
            temperature=temperature,
            messages=[{"role": "user", "content": prompt}]
        )
        ans = extract_answer(resp.content[0].text)
        answers.append(ans)
    
    # 投票
    from collections import Counter
    return Counter(answers).most_common(1)[0][0]
```

**关键**:
- 温度必须 > 0(否则采样都一样)
- 需要能"提取答案"(定义什么是"最终答案")
- 投票不适合开放式生成(每次都不一样)

### 5.5 适用与不适用

**适用**:
- ✓ 有唯一确定答案(数学/逻辑/是非)
- ✓ 关键决策(高价值)

**不适用**:
- ✗ 开放生成(每次都不同)
- ✗ 简单任务(浪费)
- ✗ 需要低延迟(N 次串行)

> **一句话**:Self-Consistency = **高温多采样 + 投票选最多**;适合有确定答案的任务(数学/逻辑),关键决策场景 N 倍成本换 15-20pp 准确率提升。

---

## 六、结构化输出(工程重点)

### 6.1 为什么必须结构化

**自由文本输出的问题**:
```
LLM: "北京今天天气不错,温度大约 15 度,建议穿薄外套。"

应用怎么解析?
  正则?容易错
  再调 LLM 解析?套娃
  → 应用层解析地狱
```

**结构化输出**:
```json
{
  "city": "北京",
  "temperature": 15,
  "condition": "晴",
  "advice": "穿薄外套"
}
```

**应用直接 `json.loads` 使用,不用解析**。

### 6.2 三种约束手段

**手段 1:Prompt 引导(基础)**

```
返回 JSON,格式:
{
  "city": string,
  "temperature": number,
  "condition": "晴|阴|雨|雪",
  "advice": string
}
只返回 JSON,不要其他文字。
```

**问题**:LLM 可能偶尔破格,需要 parse 失败重试。

**手段 2:XML tag**(Claude 官方推荐)

```
<answer>
<city>北京</city>
<temperature>15</temperature>
<condition>晴</condition>
<advice>穿薄外套</advice>
</answer>
```

**优点**:
- Claude 训练时优化过 XML,遵循度高
- 好从文本 extract(正则或 XML parser)
- 嵌套友好

**手段 3:JSON Schema 强约束**(工程首选)

**Anthropic tool_choice**:让 LLM 必须调一个"输出工具":

```python
tools = [{
    "name": "output_weather",
    "description": "输出天气信息",
    "input_schema": {
        "type": "object",
        "properties": {
            "city": {"type": "string"},
            "temperature": {"type": "number"},
            "condition": {"type": "string", "enum": ["晴","阴","雨","雪"]},
            "advice": {"type": "string"}
        },
        "required": ["city", "temperature", "condition", "advice"]
    }
}]

resp = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=tools,
    tool_choice={"type": "tool", "name": "output_weather"},  # 强制调这个
    messages=[...]
)

# resp.content[0].input 就是结构化的字典
```

**OpenAI Structured Outputs**(2024 8 月发布):

```python
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[...],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "weather",
            "strict": True,
            "schema": {...}
        }
    }
)
```

**strict=True** 保证 100% 符合 schema(推理时约束采样)。

### 6.3 XML vs JSON 选择

| 场景 | 推荐 |
| --- | --- |
| Claude 系,信息组织 | XML |
| 需要嵌套复杂 | XML(不用 escape) |
| 应用直接消费 | JSON Schema |
| 强约束 100% 格式 | JSON Schema strict |
| 输出既要"人可读又结构化" | XML |

### 6.4 实战:Pydantic + JSON Schema

```python
from pydantic import BaseModel
from typing import Literal

class Weather(BaseModel):
    city: str
    temperature: int
    condition: Literal["晴", "阴", "雨", "雪"]
    advice: str

# 从 Pydantic 生成 JSON Schema
schema = Weather.model_json_schema()

# 调 LLM 用 schema 约束
resp = call_with_schema(schema, messages)
# parse 回 Pydantic 对象
result = Weather.model_validate(resp)
```

**优点**:类型安全 + 自动校验 + 编译时提示。

### 6.5 结构化输出的常见坑

```
1. 只在 prompt 说 "return JSON",没用 strict mode
   → 偶尔翻车,必须重试

2. Schema 太复杂,LLM 遵循度下降
   → 拆多次调用,每次要少量字段

3. enum 值给太多
   → LLM 记不住,减少或分层

4. required 太多,LLM 缺信息时无法输出
   → 只放必需字段

5. 长字符串字段
   → LLM 生成不稳定,加 maxLength
```

> **一句话**:结构化输出是**工程刚需**——三种手段(prompt 引导 / XML / JSON Schema strict);Claude 推荐 XML + tool_choice 强约束,OpenAI 用 Structured Outputs;应用直接消费用 JSON,好从文本提取用 XML;**Pydantic + JSON Schema 是生产黄金组合**。

---

## 七、System Prompt 设计(全局设定)

### 7.1 System Prompt 是什么

**独立于对话历史的"全局设定"**——LLM 每轮都能看到,不会被压缩。

```python
system = """
你是资深 Go 工程师,回答有以下要求:
1. 用中文
2. 简洁准确,不要客套话
3. 代码用 ```go 标记
4. 不知道就说不知道,不要编造
"""
```

### 7.2 好 System Prompt 的模板

```markdown
<role>
你是一位资深的 X 工程师,擅长 Y 领域。
</role>

<style>
- 简洁准确
- 中文回答
- 用 markdown 格式
- 不客套
</style>

<constraints>
- 不知道就说不知道
- 涉及具体数字必须给来源
- 拒答涉政 / 敏感话题
</constraints>

<context>
[相关领域知识 / 用户档案 / 已知信息]
</context>

<examples>
[Few-shot 示例]
</examples>

<output_format>
[输出格式规范]
</output_format>
```

### 7.3 System Prompt 的 XML 分区

**Claude 官方推荐用 XML 分区**——比自然段落遵循度高:

```xml
<role>你是助手</role>

<style>简洁,中文,markdown</style>

<constraints>
- 不知道就说不知道
- 每个事实给出来源
</constraints>

<user_info>
- 姓名: 张三
- 偏好: 简洁回答
</user_info>
```

### 7.4 长 System Prompt + Prompt Cache

**长 system(> 1024 tokens)必须用 cache**:

```python
messages = [{"role": "user", "content": "你好"}]
system = [
    {"type": "text", "text": "role/style/constraints"},
    {"type": "text",
     "text": "领域大文档...",
     "cache_control": {"type": "ephemeral"}}   # 缓存
]
```

**效果**:5 分钟内后续调用 cache 部分只付 1/10 价格。见 [12-production-engineering §一](12-production-engineering.md)。

### 7.5 System Prompt 长度权衡

```
过短(< 100 tokens): 缺约束,LLM 自由发挥
中等(500-2000 tokens): 甜蜜点
过长(> 5000 tokens): 稀释,LLM 记不住

Anthropic 建议: system 中最重要的规则放开头 + 结尾
```

### 7.6 常见 system 反模式

```
❌ "你是超厉害的 AI,能做任何事"
   → 空话,没约束
   
❌ 上百条规则堆砌
   → LLM 记不住,忽略中间

❌ "请务必仔细地准确地..."
   → 冗余修饰词烧 token

✓ 精简 5-10 条核心规则 + few-shot 示例
```

> **一句话**:System prompt = **LLM 的全局设定**,推荐 **role + style + constraints + context + examples + output_format** 六段式,XML 分区提升遵循度;长 system 必须 Prompt Cache 省钱;甜蜜点 500-2000 tokens,规则精简放开头结尾。

---

## 八、Prompt 版本管理(工程刚需)

### 8.1 为什么要版本化

```
Prompt 是"代码":
  改错一版 → 效果掉 20%
  上线后想回滚 → 找不到旧版本
  多人协作 → 谁改的什么?
  评测对比 → 哪版最好?

必须像代码一样版本化管理
```

### 8.2 三种做法

**做法 1:Git 存 .md / .yaml 文件**

```
prompts/
  agent_planner_v1.md
  agent_planner_v2.md
  agent_planner_v3.md   ← current
  agent_executor_v1.md
```

**优点**:git diff / PR review / 回滚方便
**缺点**:改 prompt 要发版

**做法 2:Prompt 管理平台**

- **LangSmith Prompt Hub**:LangChain 生态
- **Braintrust**:SaaS
- **PromptLayer**:轻量
- **自建**:数据库 + 后台

**优点**:非工程也能改 / 灰度切换 / 无需发版
**缺点**:需接入,依赖平台

**做法 3:混合**

```
核心 prompt(agent 主脑):代码里管理(严格)
文案类 prompt(通知/邮件):平台管理(灵活)
```

### 8.3 Prompt 版本对比

```python
# 用 A/B 或影子对比
prompts = {"v1": "...", "v2": "...", "v3": "..."}

def route(user_id):
    if hash(user_id) % 3 == 0: return prompts["v1"]
    if hash(user_id) % 3 == 1: return prompts["v2"]
    return prompts["v3"]

# 收集每版本效果 → 评测报告(见 [11])
```

### 8.4 Prompt 变量化

```yaml
# prompts/summarizer.yaml
version: 2.1
system: |
  你是{role}。要求:
  1. 用{language}
  2. 长度不超过{max_length} 字
user: |
  总结这段:{content}
```

```python
template = load_prompt("summarizer.yaml")
prompt = template.render(role="资深编辑", language="中文", max_length=200, content=text)
```

### 8.5 Prompt 变更审计

**生产环境改 prompt = 改代码,必须**:
- Code review
- 灰度上线
- 有回滚方案
- 记录变更日志(谁 / 什么时候 / 改了什么)

> **一句话**:Prompt 像代码一样版本化——**核心 prompt 走 git,灵活 prompt 走 LangSmith/Braintrust 平台**;变量化用 yaml/jinja 模板,生产改 prompt 必须 code review + 灰度 + 可回滚,不能"改了就上"。

---

## 九、Constitutional AI(安全对齐)

### 9.1 核心思想

**让 LLM 按一套"宪法原则"自我批评并改写输出**——Anthropic 的对齐方法之一。

```
第一版输出 → 用"宪法"批评 → 改写 → 输出
```

### 9.2 应用层实现

```python
CONSTITUTION = """
输出必须满足:
1. 不含违法违规内容
2. 不含歧视/攻击性
3. 涉及医疗/法律建议要说"请咨询专业人士"
4. 不虚假宣传
5. 涉及未成年人特别谨慎
"""

def constitutional_call(query):
    # 第一次:普通回答
    draft = call_llm(query)
    
    # 第二次:自我批评 + 改写
    revised = call_llm(f"""
    原回答: {draft}
    
    请按以下原则评审并改写:
    {CONSTITUTION}
    
    如果原回答完全符合,输出原文;否则给改写版。
    """)
    return revised
```

### 9.3 何时用

**适用**:
- ✓ 面向 C 端(有合规风险)
- ✓ 涉及医疗/法律/金融建议
- ✓ 未成年人场景

**代价**:每次多一次 LLM 调用。

**替代**:直接用 Guardrails 分类器过滤(更快)——见 [12-production-engineering §三](12-production-engineering.md)。

---

## 十、Prompt 各家模型的特点差异

### 10.1 Claude(Anthropic)

**擅长**:
- 长上下文理解好
- **XML tag 遵循度最高**
- 代码质量高
- 拒答边界合理(不过度保守)
- 中文能力强(Claude 3+ 后)

**Prompt 建议**:
- 用 XML 分区(<role><style><examples>)
- System 独立字段
- 允许长 prompt(Prompt Cache 好用)
- Few-shot 3 个够

### 10.2 GPT(OpenAI)

**擅长**:
- 生态最广
- **Structured Outputs strict 模式**
- 多模态平衡
- 工具调用协议成熟

**Prompt 建议**:
- 用 Markdown / JSON 分区
- system 在 messages 里
- 短 prompt 更好
- 深度推理用 o1/o3

### 10.3 Gemini(Google)

**擅长**:
- 超长上下文(2M)
- 多模态(图片/视频/音频)原生

**Prompt 建议**:
- 用 Google AI Studio 试
- 长文档场景优势明显
- 相对不如 Claude 稳

### 10.4 DeepSeek / Qwen(国产)

**擅长**:
- 中文原生
- 便宜(1/5-1/10 Claude/GPT 价)
- 代码强(DeepSeek Coder)

**Prompt 建议**:
- 用中文写 prompt
- OpenAI 兼容 API,套一样
- DeepSeek R1 深度推理任务

### 10.5 跨模型 Prompt 兼容

**建议**:
- 用中性格式(Markdown / XML 都行)
- 避免特定家族 tricks
- 用 LangChain / LiteLLM 屏蔽差异

> **一句话**:各家模型 prompt 偏好不同——**Claude 爱 XML + 长上下文,GPT 爱 Markdown + Structured Outputs,Gemini 长文档强,国产便宜中文好**;跨模型 prompt 保持中性 + 用抽象层(LangChain/LiteLLM)屏蔽差异。

---

## 十一、常见坑

```
坑 1:prompt 太长(> 5000 tokens 核心指令)
  → LLM 稀释,重要规则被忽略
  → 精简 500-2000 tokens 甜蜜点

坑 2:Few-shot 示例格式不一致
  → LLM 学错
  → 完全统一格式

坑 3:说 "return JSON" 不用 strict mode
  → 偶尔翻车,需重试
  → 用 JSON Schema strict / tool_choice

坑 4:CoT 用在简单任务
  → 浪费 5-15x 成本
  → 只在复杂推理用

坑 5:System prompt 全靠自然语言
  → 遵循度差
  → XML 分区提升遵循度

坑 6:忽视各家模型偏好
  → 一套 prompt 跨模型效果参差
  → 关键场景针对模型调优

坑 7:改 prompt 不版本化
  → 出问题不知道回滚哪版
  → git / 平台管理

坑 8:说"请务必""非常""仔细"这类修饰
  → 烧 token,LLM 也不吃这套
  → 直接给规则

坑 9:一个 prompt 塞太多任务
  → LLM 干不好每一件
  → 拆多轮 / 多次调用

坑 10:上线不监控 badcase
  → 相同问题反复出现
  → badcase 入 Golden Set,反哺优化
```

## 十二、面试题速答

### Q1:Prompt 工程有哪些技巧?

```text
7 大技术:
  Zero-shot: 直接问,简单任务足够
  Few-shot: 3 个高质量示例是甜蜜点
  Chain-of-Thought: 一步步想,复杂推理神器(数学题 17%→78%)
  Self-Consistency: 多次采样投票,+17pp,N 倍成本
  ReAct: 思考+行动交替(Agent 底层)
  Reflexion: 失败反思重来(代码/数学任务)
  Constitutional: 按原则自我修正(安全对齐)

5 大特征:
  明确 / 结构化 / 有示例 / 约束输出 / 可版本化
```

### Q2:Few-shot 示例怎么选?

```text
甜蜜点:3 个高质量示例
选例原则:
  1. 覆盖典型 + 边界 + 拒答
  2. 示例质量 > 数量(3 精 > 10 平)
  3. 格式完全一致(XML 分区)
  4. 类别均衡(分类任务)

进阶: RAG 式动态选例
  预存 100 个示例向量化
  新 query → 检索最相似 3 个作为 few-shot
  每次用最相关示例
```

### Q3:CoT 什么时候用?什么时候不用?

```text
用:
  数学计算 / 逻辑推理 / 多步决策 / 复杂 QA / 代码生成
  一句 "Let's think step by step" 可以让 GSM8K 17% → 78%

不用:
  简单查询 / 直接检索 / 短文本生成
  成本 5-15x,不值

现代模型对复杂任务已内置 CoT(Claude/GPT 会自动),
深度推理模型(o1/o3, DeepSeek R1)内部自动做长 CoT,
但 thinking tokens 也要付费。
```

### Q4:结构化输出怎么保证?

```text
三种手段,从弱到强:
  1. Prompt 引导(基础): "return JSON"
     → 偶尔翻车,需重试
  2. XML tag(Claude 官方推荐):
     → 遵循度高,好 extract
  3. JSON Schema strict / tool_choice(工程首选):
     → 100% 符合 schema

Claude: tool_choice 强制调"输出 tool"
OpenAI: Structured Outputs strict=True

生产黄金组合: Pydantic 定义 → 生成 Schema → LLM strict 输出 → parse 回 Pydantic
类型安全 + 自动校验。
```

### Q5:System Prompt 怎么设计?

```text
六段式模板:
  <role> 角色定义
  <style> 输出风格
  <constraints> 约束边界(不知道就说不知道 / 拒答类型)
  <context> 相关上下文 / 用户档案
  <examples> Few-shot 示例
  <output_format> 输出格式规范

XML 分区(Claude 特别爱)

长度:
  500-2000 tokens 甜蜜点
  > 5000 稀释效果
  重要规则放开头 + 结尾

长 system 必用 Prompt Cache 省钱(见 12 生产化)
```

### Q6:Prompt 怎么版本化管理?

```text
Prompt 是"代码",必须像代码一样管:
  1. Git 存 .md / .yaml(核心 prompt)
     git diff / PR review / 回滚方便
  2. 平台管理(LangSmith Prompt Hub / Braintrust)
     非工程也能改 / 灰度 / 无需发版
  3. 混合:核心走 git,文案走平台

变量化: yaml + jinja 模板

生产变更铁律:
  Code review + 灰度 + 可回滚 + 变更审计
  不能"改了直接上"
```

## 十三、关联阅读

```
本目录:
- 01-llm-fundamentals              采样 / 幻觉 / 选型(prompt 影响这些)
- 02-api-basics                    SDK 里 system / messages 字段
- 04-tool-use-function-calling     tool_choice 强约束
- 05-agent-architectures           ReAct/Reflexion/CoT 在 agent 中的应用
- 11-evaluation-and-testing        Prompt 迭代必须评测
- 12-production-engineering        Prompt Cache 省钱

外部:
- Anthropic Prompt Engineering: docs.anthropic.com/en/docs/build-with-claude/prompt-engineering
- OpenAI Prompt Guide: platform.openai.com/docs/guides/prompt-engineering
- CoT 论文: Wei et al. 2022, "Chain-of-Thought Prompting Elicits Reasoning"
- Zero-shot CoT: Kojima et al. 2022, "Large Language Models are Zero-Shot Reasoners"
- Self-Consistency: Wang et al. 2022
- Constitutional AI: Bai et al. 2022(Anthropic)
- LangSmith Prompt Hub: smith.langchain.com/hub
```

> **一句话核心(全篇精炼)**:
> Prompt = **用自然语言编程 LLM**;
> **7 大技术**(Zero-shot/Few-shot/CoT/Self-Consistency/ReAct/Reflexion/Constitutional)+ **5 大特征**(明确/结构化/示例/约束/版本化);
> **Few-shot 3 个是甜蜜点,CoT 数学神器但 5-15x 成本,结构化输出必上 JSON Schema strict/tool_choice**;
> **System prompt 六段式 + XML 分区 + Prompt Cache**;
> Prompt 像代码一样版本化 —— git 或平台管理,变更必 review + 灰度 + 可回滚;
> 好差 prompt 在同任务差 30-50pp,是 agent 效果最大的杠杆之一。
