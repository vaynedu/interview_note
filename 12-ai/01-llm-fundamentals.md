# LLM 基础(Agent 开发的地基)

> 做 Agent 前必须先懂 LLM 本身——**Token / 上下文窗口 / 采样 / 幻觉 / 选型 / 成本**这 6 件事,每一个都决定你的 Agent 能不能用、稳不稳、贵不贵。
>
> 本章不讲 Transformer 数学原理(那是模型团队的事),讲的是**8 年后端工程师做 Agent 必须掌握的 LLM 使用侧知识**。

## 〇、核心提炼(5 段式)

### 核心机制(6 条必背)

1. **Token 是模型输入输出的最小单位**——不是字符,不是单词,是 BPE 编码后的子词;中文比英文更耗 token
2. **上下文窗口是硬约束**——超出就报错或截断,长上下文性能衰减(中间信息容易被忽略)
3. **温度决定确定性**——`temperature=0` 近乎确定,`=1` 有创造性;Agent 场景一般 `0.3-0.7`
4. **幻觉是 LLM 的固有属性**——本质是"合理但错误的补全",Agent 里靠 Tool + RAG + 结构化输出压制
5. **模型选型是成本 vs 智商的权衡**——Opus 贵但强,Haiku 快而便宜;Agent 里常用 **model routing**(简单任务走小模型)
6. **Prompt Cache + Batch API + Prompt 精简**是三大成本优化手段,能把成本降到 1/10

### 核心本质(必懂)

> LLM 本质是 **"下一个 token 概率分布 + 采样"**——
> 给定前面的 tokens,模型输出下一个 token 的概率,用采样策略(温度/top-p)选一个,循环直到停止符或 max_tokens。
>
> 这决定了几个 Agent 开发的铁律:
> - **LLM 不"知道"事实**,只是模式匹配 → **需要 Tool 和 RAG 把外部知识拉进来**
> - **LLM 不"计算"** → 数学、代码执行必须靠 Tool
> - **LLM 输出是概率的**,同一个 prompt 每次结果可能不同 → **不要靠 LLM 做强一致性判断**
> - **LLM 会"顺着往下写"** → 结构化输出必须用 JSON Schema / XML 强约束
> - **上下文是唯一的"记忆"** → 超出窗口就"失忆",这是所有 memory 方案的起点

### 完整流程(推理一次调用)

```
1. 输入 prompt(system + messages)
   ↓
2. Tokenizer 分词(BPE)→ token IDs
   ↓
3. 通过 Transformer 前向 → 得到词表上的概率分布
   ↓
4. 采样(温度 / top-p / top-k)→ 选一个 token
   ↓
5. 追加到序列 → 回到 3
   ↓
6. 遇到 stop token 或 max_tokens → 结束
   ↓
7. 反 tokenize → 输出文本
```

**Agent 一次交互 = 多次 LLM 推理 + 多次 Tool 调用**——理解一次推理的成本是理解 Agent 成本的基础。

### 6 条核心机制 - 逐点讲透(见下面各节)

### 一句话总结

> LLM = **概率下一个 token + 采样**;
> Agent 开发的所有工程约束都源于这个本质——**不知道事实(靠 Tool/RAG)、不确定(靠温度控制 + 结构化输出)、有窗口限制(靠 Memory 管理)、有成本(靠模型路由 + cache + batch)**。

---

## 一、Token(模型的"字")

### 1.1 什么是 Token

**Token 不是字符,不是单词,是 BPE(Byte Pair Encoding)算法切分的子词单元**。

```python
# 示例(GPT-4 tokenizer)
"Hello world"        → ["Hello", " world"]           # 2 tokens
"你好,世界"          → ["你", "好", ",", "世", "界"]  # 5 tokens(每个中文约 1-2 tokens)
"tokenization"        → ["token", "ization"]          # 2 tokens(英文长词会被拆)
"tokenization 是什么"  → 混合切分                       # 中英混排
```

**关键规律**:
- 英文:平均 **1 token ≈ 0.75 单词** 或 **4 字符**
- 中文:平均 **1 token ≈ 1-2 汉字**(Claude 3+ 更友好,约 0.6 汉字/token)
- **中文比英文多花 30-50% token**——同样的知识用英文更省钱

### 1.2 各家 tokenizer 差异

| 模型 | Tokenizer | 中文效率 |
| --- | --- | --- |
| GPT-4o | o200k_base | 中文 1.5-2 字/token |
| Claude 3+ | 官方 tokenizer | **中文 0.6-1 字/token(最省)** |
| Gemini | SentencePiece | 中文 1-1.5 字/token |
| Qwen / DeepSeek | 自研,中文优化 | 中文 0.5-0.8 字/token |

→ **中文场景选 Claude / 国产模型省钱**。

### 1.3 怎么算 token

```python
# Python
import tiktoken
enc = tiktoken.encoding_for_model("gpt-4o")
tokens = enc.encode("你好,世界")
print(len(tokens))  # 5

# Anthropic 精确计费
from anthropic import Anthropic
client = Anthropic()
result = client.messages.count_tokens(
    model="claude-opus-4-8",
    messages=[{"role": "user", "content": "你好"}]
)
```

```go
// Go(tiktoken-go)
import "github.com/pkoukk/tiktoken-go"
enc, _ := tiktoken.EncodingForModel("gpt-4o")
tokens := enc.Encode("你好,世界", nil, nil)
fmt.Println(len(tokens))
```

### 1.4 面试常问

**Q:为什么要用 token 不用字符?**
- BPE 平衡了词汇表大小和序列长度
- 罕见词也能表达(拆成更小子词)
- 训练时的最小单位

**Q:同一个字符串在不同模型 token 数一样吗?**
- **不一样**,每家 tokenizer 训练语料不同
- 计费必须用对应模型的 tokenizer 精确算

> **一句话**:Token 是模型的"字",**中文比英文贵 30-50%**,Claude/国产模型对中文更友好;Agent 每一次调用都在按 token 算钱,**prompt 精简 = 直接省钱**。

---

## 二、Context Window(上下文窗口)

### 2.1 各家窗口大小(2026 主流)

| 模型 | Context Window | 输出上限 |
| --- | --- | --- |
| **Claude Opus 4.8** | **200K** | 32K |
| **Claude Sonnet 4.6** | 200K | 32K |
| **Claude Haiku 4.5** | 200K | 8K |
| **GPT-4o / 4o-mini** | 128K | 16K |
| **GPT-4.1** | 1M | 32K |
| **Gemini 2.0 Pro** | **2M** | 8K |
| **Gemini Flash** | 1M | 8K |
| **Qwen2.5-Max** | 128K | 8K |
| **DeepSeek V3** | 64K | 8K |

### 2.2 上下文窗口的三个坑

**坑 1:超窗口报错**

```python
# 输入 300K tokens 到 200K 窗口的模型
# → 报错:context length exceeded
```

对策:滑窗 + 摘要(见 [06-memory-and-context.md](06-memory-and-context.md))。

**坑 2:长上下文性能衰减(Lost in the Middle)**

**关键论文**:*Lost in the Middle: How Language Models Use Long Contexts*(Liu et al. 2023)

```
LLM 对 prompt 的注意力分布:
  开头(system + 前几段):  ★★★★★
  末尾(最新 message):     ★★★★★
  中间(30%-70% 位置):     ★★☆☆☆   ← 容易忽略
```

**实战对策**:
- 关键指令放 **prompt 开头** 或 **末尾**
- RAG 检索结果按相关性排序,**最相关的放两端**
- 长文档做 **chunking + rerank**,不要直接塞满

**坑 3:窗口越大,一次调用越贵**

```
Claude Sonnet 4.6:
  1K input = $0.003
  100K input = $0.30(百倍)
  200K input = $0.60
```

→ **不是"能塞就塞",要精算成本**。

### 2.3 输入 vs 输出窗口

**输入 + 输出 ≤ context window**:

```python
# Claude Opus 4.8: 200K 窗口, 32K 输出上限
# 如果输入用了 190K, 那输出最多 10K
```

Agent 场景常见问题:tool result 太大占满窗口,输出被截断。

> **一句话**:上下文窗口是**硬约束**,而且**中间容易被忽略**;RAG 结果要排序 + rerank,长任务必须做 memory 摘要,**不是塞得进就等于用得上**。

---

## 三、采样参数(控制输出的确定性和创造性)

### 3.1 核心参数

| 参数 | 作用 | 默认 | Agent 场景推荐 |
| --- | --- | --- | --- |
| **temperature** | 概率分布锐度,0=确定,1=创造 | 1.0 | **0.3-0.7**(工具调用用低温) |
| **top_p** | 核采样,只从累积概率 P 内采 | 1.0 | 0.9(和温度二选一调) |
| **top_k** | 只从概率最高的 K 个采 | 无限 | 一般不用 |
| **max_tokens** | 输出上限 | 各家不同 | 按需设,越大越贵 |
| **stop_sequences** | 遇到就停止 | 无 | Agent 结构化输出常用 |

### 3.2 温度的实战影响

```python
# temperature = 0
"1+1=" → "2"(基本确定)

# temperature = 0.5
"写一首关于秋天的诗" → 有创造性但不失控

# temperature = 1.0
"写一首关于秋天的诗" → 高度多样,可能跑题

# temperature = 2.0(Claude 上限 1.0)
输出可能完全乱掉
```

### 3.3 Agent 场景的温度选择

| 任务 | 推荐温度 | 原因 |
| --- | --- | --- |
| **Tool Calling(工具选择)** | **0-0.3** | 需要稳定选对工具 |
| **代码生成** | 0.2-0.5 | 需要正确性 |
| **数据抽取(structured output)** | **0** | 追求确定性 |
| **对话 / 写作** | 0.7-1.0 | 需要多样性 |
| **创意生成 / brainstorming** | 0.8-1.2 | 需要发散 |

**铁律**:**Agent 内部的工具调用一定用低温**;对外的用户回复才可以调高。

### 3.4 top_p vs temperature

```
温度先起效 → 再 top_p 截断
两者一起用容易互相干扰 → 只调一个

工业界主流:调温度不调 top_p(top_p 保持默认 1.0 或 0.9)
```

> **一句话**:**Agent 内部推理和 tool 选择用 `temperature=0-0.3`**(要稳),对外的对话可以 0.7;temperature 和 top_p 二选一调,别一起动。

---

## 四、幻觉(Hallucination)

### 4.1 什么是幻觉

**幻觉 = LLM 输出了合理但错误的内容**——因为 LLM 是"模式匹配 + 概率补全",不是"真的知道"。

**典型场景**:
- 编造不存在的 API / 库 / 函数名(代码幻觉最常见)
- 引用不存在的论文 / 书籍 / 案例
- 混淆相似概念(把 A 的属性说成 B 的)
- 数字算错(LLM 不真的会数学)
- 引用错版本(不同版本 API 混用)

### 4.2 为什么会幻觉

```
根本原因:LLM 的目标函数是"最大化下一个 token 概率"
       ↓
即使不确定,也会输出最"看起来合理"的答案
       ↓
不会说"我不知道",除非训练时被专门对齐过
```

**放大幻觉的场景**:
- 训练数据没覆盖的领域(冷门 API / 内部系统)
- 时效性数据(训练截止后的新版本)
- 需要精确算术
- 需要多步推理但 chain-of-thought 没走对

### 4.3 Agent 里怎么压制幻觉

| 手段 | 原理 | 强度 |
| --- | --- | --- |
| **RAG** | 把真实文档塞进 prompt,让 LLM 基于事实回答 | ★★★★★ |
| **Tool Use** | 需要事实的问题走工具查(数据库/API) | ★★★★★ |
| **结构化输出**(JSON Schema) | 强约束格式,不给"胡编"空间 | ★★★★ |
| **"我不知道"引导** | 明确让 LLM 承认不确定 | ★★★ |
| **Chain-of-Verification**(CoVe) | 让 LLM 自己验证答案 | ★★★ |
| **降温度** | 减少发散(但不能消除) | ★★ |
| **Self-Consistency** | 多次采样投票 | ★★ |

**铁律**:**任何需要事实的场景,都不要相信 LLM,一定走 Tool 或 RAG**。

**"我不知道"引导 prompt 示例**:

```
你是一个技术助手。回答问题时:
- 如果你不确定或没有相关信息,直接说"我不知道"或"需要查询相关文档"
- 不要编造 API 名、参数名或版本号
- 涉及具体数字时,说明来源或用工具查询
```

### 4.4 幻觉的类型

| 类型 | 例子 | 缓解 |
| --- | --- | --- |
| **事实幻觉** | 编造论文/API | RAG + Tool |
| **推理幻觉** | 数学算错 | Code Interpreter Tool |
| **上下文幻觉** | 无中生有的引用 | 强调"仅基于给定文档" |
| **指令幻觉** | 忽略指令 | 结构化输出 + 强约束 |

> **一句话**:幻觉是 LLM 的**固有属性**,不是 bug;Agent 开发核心不是"消除幻觉",是**用 Tool + RAG + 结构化输出把幻觉的影响面压到能接受**。

---

## 五、模型选型(2026 主流对比)

### 5.1 Claude 家族(Anthropic)

| 模型 | 定位 | 强项 | 价格(input/output per 1M) |
| --- | --- | --- | --- |
| **Opus 4.8** | 顶级智商 | 复杂推理 / Agent 主脑 / 代码 | $15 / $75 |
| **Sonnet 4.6** | 主力 | Agent 常用 / 平衡 | $3 / $15 |
| **Haiku 4.5** | 快速便宜 | 简单任务 / 分类 / 路由 | $1 / $5 |

### 5.2 OpenAI 家族

| 模型 | 定位 | 强项 | 价格 |
| --- | --- | --- | --- |
| **GPT-4o** | 主力 | 多模态好 / 生态最广 | $2.5 / $10 |
| **GPT-4.1** | 长上下文 | 1M 窗口 | $2 / $8 |
| **GPT-4o-mini** | 便宜 | 简单任务 | $0.15 / $0.6 |
| **o1 / o3** | 深度推理 | 数学 / 复杂逻辑 | 更贵,慢 |

### 5.3 Google Gemini

| 模型 | 定位 | 强项 |
| --- | --- | --- |
| **Gemini 2.0 Pro** | 主力 | 2M 窗口 / 多模态 |
| **Gemini Flash** | 快 | 便宜 / 快 |

### 5.4 开源 / 国产

| 模型 | 定位 |
| --- | --- |
| **DeepSeek V3 / R1** | 中文 + 代码强,便宜 |
| **Qwen 2.5-Max** | 阿里,中文强 |
| **Llama 3.3** | Meta 开源,自部署首选 |
| **Kimi / 智谱 GLM-4** | 国内主流 |

### 5.5 选型矩阵

```
Agent 主脑(planner / orchestrator)
  首选: Claude Sonnet(平衡)/ Opus(复杂任务)
  次选: GPT-4o
  预算紧: DeepSeek V3

工具调用 executor
  首选: Claude Haiku / GPT-4o-mini(便宜快)

代码生成
  首选: Claude Sonnet(实测代码质量最好)
  次选: DeepSeek V3(中文注释好)

RAG 问答
  首选: Claude Sonnet(引用准确)
  预算紧: Haiku / mini

深度推理(数学/逻辑)
  首选: OpenAI o3
  次选: Claude Opus + CoT

长文档(> 200K)
  首选: Gemini 2.0 Pro(2M)
  次选: GPT-4.1(1M)

自部署 / 数据不出域
  Llama 3.3 / Qwen 2.5 / DeepSeek(开源版本)
```

### 5.6 Model Routing(生产必备)

**思路**:根据任务复杂度,让不同模型处理不同环节。

```
用户请求
  ↓
Router(用 Haiku 分类):这是"简单FAQ" 还是 "复杂Agent任务"?
  ↓
简单 → Haiku 直接答
复杂 → Sonnet planner → 拆任务 → Haiku executor 执行
       ↓
       复杂子任务 → Opus 处理
```

**成本对比**(1M token input):

```
纯 Opus:    $15
纯 Sonnet:  $3
Router 混合(70% Haiku + 20% Sonnet + 10% Opus):
  0.7*$1 + 0.2*$3 + 0.1*$15 = $2.8
→ 比纯 Sonnet 还便宜,大部分场景不损失质量
```

> **一句话**:模型选型 = **智商 vs 成本 vs 速度的三角权衡**;生产系统必须做 **model routing**——70% 简单任务走 Haiku,20% 主流任务走 Sonnet,10% 复杂任务走 Opus,能把成本降到纯 Sonnet 的 1/2 以下,还不损失质量。

---

## 六、成本控制(生产核心)

### 6.1 计费模式

```
输入 + 输出分开计费(输出通常 3-5x 输入价)
按 token 数(不是按调用次数)
Prompt Cache 命中的 token 便宜 10x(Anthropic)
Batch API 便宜 50%
```

### 6.2 三大优化手段

**手段 1:Prompt Cache(最猛)**

```python
# Anthropic Prompt Cache
# 长 system prompt + 固定文档,标记为 cache
messages = [
    {
        "role": "system",
        "content": [{
            "type": "text",
            "text": "长长的 system prompt + 大文档...",
            "cache_control": {"type": "ephemeral"}   # 缓存 5 分钟
        }]
    },
    {"role": "user", "content": "用户提问 1"}
]

# 第一次调用:全额付费 + 写入 cache
# 5 分钟内后续调用:cache 部分只付 1/10 价格
```

**适用**:
- ✓ 长 system prompt(> 1024 tokens)
- ✓ 相同文档多次问答(RAG 场景)
- ✓ 批量处理(几分钟内多次调用)

**成本节约**:高频调用可省 **80-90%**。

**手段 2:Batch API(准实时可用)**

```python
# Anthropic Batch API / OpenAI Batch API
# 提交批量任务,24 小时内返回,费用打 5 折

client.messages.batches.create(requests=[
    {"custom_id": "1", "params": {...}},
    {"custom_id": "2", "params": {...}},
    ...
])
```

**适用**:
- ✓ 数据处理(打标 / 分类 / 摘要)
- ✓ 定期任务
- ✗ 实时响应场景

**成本节约**:**50%**。

**手段 3:Prompt 精简 + 输出精简**

```python
# ❌ 差:3000 token prompt 换 200 token 答案
"你是一位资深工程师,请务必仔细分析以下代码,
考虑代码质量、性能、可维护性、安全性...
[长长的 code 上下文]"

# ✓ 好:800 token prompt 换 200 token 答案
"分析代码质量,输出 issue 列表(JSON 格式)"
[必要 code]
```

**技巧**:
- 冗余修饰词删掉("请务必"、"仔细")
- 用 XML/JSON 结构化替代长自然语言
- 输出用 JSON Schema 约束长度
- Few-shot 示例够用就行(3 个 vs 10 个,效果差不多但成本 3 倍)

### 6.3 成本监控三件套

```
每次调用记:
  1. input_tokens / output_tokens / cache_read / cache_creation
  2. 调用来源(哪个业务 / 哪个用户)
  3. 调用类型(agent 主脑 / tool 调用 / RAG rerank)

看板:
  按业务分账 / 按模型分布 / 按小时趋势
  异常告警(单用户 / 单请求异常高)
```

**工具**:
- LangSmith / LangFuse:全链路 trace + 成本
- Helicone:LLM 网关 + 分账
- 自建:OpenTelemetry + Prometheus

### 6.4 一次调用的完整成本(示例)

```
场景:用 Claude Sonnet 4.6 做 Agent 主脑,一次任务

Input:
  System prompt (含 tools):     3000 tokens (缓存后 300 tokens 计费)
  历史对话:                       2000 tokens
  当前用户输入:                    100 tokens
  Tool result(RAG 检索):        4000 tokens

Output:
  Reasoning + tool call:          500 tokens

价格:
  Input(cached):  300 * $3/1M     = $0.0009
  Input(non-cached): 6100 * $3/1M = $0.0183
  Output: 500 * $15/1M            = $0.0075
  合计: $0.027 ≈ 0.19 元 / 次

一次 Agent 任务通常需要 5-10 轮 → 1-2 元 / 任务
一天 10 万次 → 10-20 万元 / 天

→ 优化空间(cache + routing + prompt 精简) → 降到 1-2 万元 / 天
```

> **一句话**:成本优化三件套 = **Prompt Cache(高频省 80%) + Batch API(定时省 50%) + Prompt/输出精简**;**必须监控每次调用的 token 分布**,否则线上一天烧几万很正常。

---

## 七、量化和推理速度(部分场景关注)

### 7.1 量化(Quantization)

**只在自部署开源模型时关心**:

| 精度 | 大小 | 质量损失 |
| --- | --- | --- |
| FP16 / BF16 | 100% | 0(标准) |
| INT8 | 50% | 微小(< 1%) |
| INT4(GPTQ/AWQ) | 25% | 小(2-5%) |
| INT2 | 12% | 显著(10%+) |

**实战**:70B 模型 FP16 需要 140GB 显存,INT4 只需 40GB(能塞进 1-2 张 A100)。

### 7.2 推理速度指标

```
TTFT (Time To First Token):首 token 延迟
  好: < 500ms   适合流式对话
  一般: 500ms-2s
  差: > 2s      不适合实时

TPS (Tokens Per Second):吐 token 速度
  Claude Sonnet:    ~ 60-80 TPS
  Claude Haiku:     ~ 100-150 TPS
  GPT-4o:           ~ 50-80 TPS
  GPT-4o-mini:      ~ 100+ TPS
  开源 70B(A100):  ~ 20-40 TPS
```

### 7.3 加速手段

- **流式(streaming)**:边生成边显示,提升感知速度
- **推测解码(speculative decoding)**:小模型草稿 + 大模型验证
- **KV Cache 复用**:多轮对话共享
- **模型路由**:简单任务走 Haiku 速度更快

---

## 八、面试题速答

### Q1:LLM 为什么会幻觉?

```text
LLM 的目标是"最大化下一个 token 概率",不是"输出真相"。
即使不确定,也会输出最可能的 token,表现为"合理但错误"。
根本原因:
  1. LLM 只是模式匹配,不真正"知道"事实
  2. 训练目标不惩罚"胡编但流畅"
  3. 训练数据不完整或过时
Agent 里的解法不是消除幻觉(做不到),
是用 Tool + RAG + 结构化输出把幻觉影响压到能接受。
```

### Q2:Agent 里 temperature 怎么设?

```text
Agent 内部推理和 tool 选择: 0-0.3(要稳定选对工具)
数据抽取(structured output): 0(要确定)
代码生成: 0.2-0.5
对外的用户对话: 0.7(需要多样性)
创意生成: 0.8-1.2
铁律: 内部逻辑用低温,外部呈现可以调高
```

### Q3:怎么给 Agent 场景选模型?

```text
三角权衡: 智商 vs 成本 vs 速度
  Agent 主脑: Sonnet(平衡)/ Opus(复杂)
  Tool executor: Haiku / mini(便宜快)
  代码生成: Sonnet(实测最好)
  长文档: Gemini 2M / GPT-4.1 1M
  自部署: Llama / Qwen / DeepSeek
生产必做 model routing: 70% Haiku + 20% Sonnet + 10% Opus
可以把成本降到纯 Sonnet 的一半以下
```

### Q4:LLM 成本怎么控?

```text
三大手段:
  1. Prompt Cache(高频调用省 80-90%)
  2. Batch API(定时任务省 50%)
  3. Prompt/输出精简(冗余修饰词删掉,输出用 JSON 强约束)
监控:
  每次调用记 input/output/cache token,按业务/模型分账,异常告警
```

### Q5:上下文窗口越大越好吗?

```text
不是。三个坑:
  1. 长上下文性能衰减(Lost in the Middle,中间容易被忽略)
  2. 每次调用成本 = O(input tokens),塞满巨贵
  3. 长任务应该用 Memory 摘要 + RAG 检索,不是硬塞
关键指令放开头/末尾,RAG 结果按相关性排序,长文档 chunking + rerank。
```

## 九、常见坑

```
坑 1:以为 token 数 = 字符数
  → 中文一个字 1-2 tokens,英文 4 字符 1 token
  → 精算成本必须用对应模型 tokenizer

坑 2:同一段代码在 Claude 和 GPT 上 token 数不同
  → 必须用各自的 tokenizer

坑 3:Agent 内部工具调用用 temperature=1
  → 工具选择不稳定,行为漂移
  → 必须用低温

坑 4:相信 LLM 的算术输出
  → 5 位数字乘法 LLM 也可能算错
  → 走 Code Interpreter Tool 或计算器 Tool

坑 5:靠 prompt 消除幻觉
  → 只能减轻,不能消除
  → 事实类必须走 RAG / Tool

坑 6:大模型比小模型一定强
  → 简单任务 Haiku 和 Opus 差不多,但成本差 15 倍
  → 该省则省

坑 7:塞满 200K 窗口
  → 中间信息容易被忽略 + 贵
  → 用 RAG + rerank + memory 压缩

坑 8:不监控 token 消耗
  → 一个 bug 让 prompt 循环 → 一天烧几十万
  → 必须限流 + 告警
```

## 十、Go / Python 快速上手

### Python(Anthropic SDK)

```python
from anthropic import Anthropic

client = Anthropic()

# 基础调用
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    temperature=0.3,
    system="你是一个助手",
    messages=[{"role": "user", "content": "你好"}]
)
print(response.content[0].text)

# 流式
with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "写首诗"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

### Go(anthropic-sdk-go)

```go
import (
    "github.com/anthropics/anthropic-sdk-go"
    "github.com/anthropics/anthropic-sdk-go/option"
)

client := anthropic.NewClient(option.WithAPIKey(os.Getenv("ANTHROPIC_API_KEY")))

resp, err := client.Messages.New(ctx, anthropic.MessageNewParams{
    Model:     anthropic.F(anthropic.ModelClaudeSonnet4_6),
    MaxTokens: anthropic.F(int64(1024)),
    System: anthropic.F([]anthropic.TextBlockParam{
        anthropic.NewTextBlock("你是一个助手"),
    }),
    Messages: anthropic.F([]anthropic.MessageParam{
        anthropic.NewUserMessage(anthropic.NewTextBlock("你好")),
    }),
})
if err != nil { log.Fatal(err) }
fmt.Println(resp.Content[0].Text)
```

## 十一、关联阅读

- [00-agent-learning-roadmap](00-agent-learning-roadmap.md) 8 周学习计划
- [02-api-basics](02-api-basics.md) SDK 详细使用
- [03-prompt-engineering](03-prompt-engineering.md) prompt 进阶
- [04-tool-use-function-calling](04-tool-use-function-calling.md) Agent 工具调用
- [12-production-engineering](12-production-engineering.md) 成本 / 监控 / 安全

> **一句话核心(全篇精炼)**:
> LLM = **概率下一个 token + 采样**;
> Agent 开发所有工程约束都源于此——
> **不知道事实靠 Tool/RAG,不确定靠低温+JSON Schema,窗口有限靠 Memory,成本高靠 Cache+Routing+精简**;
> 中文比英文贵 30-50%,长上下文中间容易被忽略,幻觉是本质不是 bug——
> **8 年后端做 Agent 的第一课,是把 LLM 当成一个"聪明但不可信的实习生"来用**。
