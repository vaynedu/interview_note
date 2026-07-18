# API 基础(Anthropic / OpenAI / SDK 实战)

> LLM 应用的第一层——**API 调用**;不懂 SDK 细节,后面的 tool use / streaming / 多模态全是空中楼阁。
>
> 本章讲透 **Anthropic / OpenAI 两大主流 SDK 的核心用法 / 流式 / 多模态 / 计费 / 重试限流 / Go + Python 对照** —— 8 年后端做 Agent 的 API 基本功。
>
> 前置:[01-llm-fundamentals](01-llm-fundamentals.md)(token / 采样 / 选型)

## 〇、核心提炼(5 段式)

### 核心机制(6 条必背)

1. **API 调用 = HTTPS POST + JSON**——SDK 只是 HTTP 客户端封装,底层随时可以裸调
2. **两种响应模式**:**同步**(等全部生成完)vs **流式**(边生成边推 SSE)
3. **多模态**:文本 + 图片 + PDF 都用同一套 messages API,只是 content block type 不同
4. **计费颗粒是 token**——input / output / cache_read / cache_creation 分开算,监控必须细分
5. **重试 + 限流是生产刚需**——Anthropic 官方 SDK 内置指数退避,但要理解自己接管
6. **每家 API 大同小异,细节陷阱多**——OpenAI arguments 是字符串、Claude tool_result 走 user role、Gemini 用 parts

### 核心本质(必懂)

> API 层的本质是**"把 LLM 变成可编程组件"**——
>
> 对 8 年后端的思维:
> - **HTTP 层不神秘**:就是 REST + JSON,调 Anthropic 和调 Stripe 没本质区别
> - **SDK 只是便利**:官方 SDK 好用但可选,任何 HTTP 客户端都能调
> - **流式复杂在解析**:SSE 格式 + 状态机 + tool_use 分片重组
> - **多模态复杂在编码**:图片 base64 / URL / 大小限制 / 支持格式
> - **重试复杂在语义**:5xx 重试 / 4xx 别重试 / rate limit 特殊处理
>
> **裸调 vs SDK vs 框架 三层**:
> - **裸调 HTTP**:自己拼 JSON + parse,极端灵活但样板代码多
> - **官方 SDK**(anthropic / openai):**推荐 90% 场景**,类型安全 + 内置重试 + 流式好用
> - **框架**(LangChain / Eino):跨模型统一,但抽象重
>
> **一句话**:能裸调是基本功,常用 SDK,复杂再上框架。

### 完整流程(一次同步调用)

```
1. 应用: 构造 messages + params(JSON)
   ↓
2. SDK: HTTP POST /v1/messages
        headers: Authorization + anthropic-version
        body: JSON
   ↓
3. 服务端: 队列 → GPU 推理 → 生成 output tokens
   ↓
4. 服务端返回 JSON:
   {
     "id": "msg_xxx",
     "type": "message",
     "role": "assistant",
     "content": [{"type": "text", "text": "..."}],
     "stop_reason": "end_turn",
     "usage": {"input_tokens": 100, "output_tokens": 50}
   }
   ↓
5. SDK: 解析 → 返回类型化对象
   ↓
6. 应用: 提取 text / 记 usage
```

### 6 条核心机制 - 逐点讲透(见下面各节)

### 一句话总结

> API 调用 = **HTTPS POST + JSON**,SDK 只是封装;
> 掌握**同步/流式/多模态/计费/重试限流/多家 API 差异**这 6 件事,agent 的所有交互都在这一层;
> **能裸调是基本功,推荐官方 SDK,复杂再上框架**——8 年后端做 Agent 的入门必修。

---

## 一、Anthropic Messages API

### 1.1 最小示例

**Python**:

```python
from anthropic import Anthropic

client = Anthropic()  # 从 ANTHROPIC_API_KEY 环境变量

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "你好"}
    ]
)

print(response.content[0].text)
print(f"消耗 token: {response.usage.input_tokens} + {response.usage.output_tokens}")
```

**Go**:

```go
package main

import (
    "context"
    "fmt"
    "github.com/anthropics/anthropic-sdk-go"
    "github.com/anthropics/anthropic-sdk-go/option"
)

func main() {
    client := anthropic.NewClient(option.WithAPIKey("sk-ant-..."))

    resp, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
        Model:     anthropic.F(anthropic.ModelClaudeSonnet4_6),
        MaxTokens: anthropic.F(int64(1024)),
        Messages: anthropic.F([]anthropic.MessageParam{
            anthropic.NewUserMessage(anthropic.NewTextBlock("你好")),
        }),
    })
    if err != nil { panic(err) }
    fmt.Println(resp.Content[0].Text)
}
```

### 1.2 核心参数

| 参数 | 说明 | 默认 |
| --- | --- | --- |
| **model** | 模型 id(必填) | 无 |
| **max_tokens** | 输出上限(必填) | 无 |
| **messages** | 对话消息数组(必填) | 无 |
| **system** | System prompt | 无 |
| **temperature** | 0-1,采样温度 | 1.0 |
| **top_p** | 核采样阈值 | 1.0 |
| **top_k** | 只从 top-K 采(一般不用) | 无 |
| **stop_sequences** | 停止序列数组 | 无 |
| **stream** | 是否流式 | false |
| **tools** | 工具列表 | 无 |
| **tool_choice** | 工具选择模式(auto/any/tool) | auto |
| **metadata.user_id** | 用户 ID(用于滥用检测) | 无 |

### 1.3 System Prompt

**独立字段**(不在 messages 里):

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system="你是一个资深 Go 工程师,回答简洁准确。",
    messages=[{"role": "user", "content": "goroutine 泄漏怎么排查?"}]
)
```

**多段 system + cache**:

```python
system=[
    {"type": "text", "text": "简短的 role 定义"},
    {"type": "text",
     "text": "长长的领域知识文档...",
     "cache_control": {"type": "ephemeral"}}  # 缓存
]
```

### 1.4 Messages 格式

**基础对话**:

```python
messages = [
    {"role": "user", "content": "问题 1"},
    {"role": "assistant", "content": "回答 1"},
    {"role": "user", "content": "问题 2"}
]
```

**多模态 content block**:

```python
messages = [
    {"role": "user", "content": [
        {"type": "text", "text": "这张图是什么?"},
        {"type": "image", "source": {
            "type": "base64",
            "media_type": "image/jpeg",
            "data": base64_str
        }}
    ]}
]
```

**Tool use block**:见 [04-tool-use-function-calling](04-tool-use-function-calling.md)

### 1.5 响应结构

```python
response.id            # "msg_xxx"
response.model         # "claude-sonnet-4-6-xxx"
response.role          # "assistant"
response.content       # list of blocks
response.stop_reason   # "end_turn" / "max_tokens" / "stop_sequence" / "tool_use"
response.usage.input_tokens
response.usage.output_tokens
response.usage.cache_read_input_tokens   # cache hit
response.usage.cache_creation_input_tokens
```

**stop_reason 值语义**:
- `end_turn`:自然结束
- `max_tokens`:达到 max_tokens 上限截断
- `stop_sequence`:命中 stop_sequences
- `tool_use`:LLM 想调工具(见 [04](04-tool-use-function-calling.md))

> **一句话**:Anthropic Messages API 核心 = **model + max_tokens + messages + system**;system 是独立字段(不在 messages),content 支持多种 block(text/image/tool_use/tool_result),响应必看 stop_reason 判断是"完成"还是"要继续"。

---

## 二、OpenAI Chat Completions API

### 2.1 最小示例

**Python**:

```python
from openai import OpenAI

client = OpenAI()  # 从 OPENAI_API_KEY 环境变量

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "你是助手"},
        {"role": "user", "content": "你好"}
    ]
)

print(response.choices[0].message.content)
print(f"消耗: {response.usage.prompt_tokens} + {response.usage.completion_tokens}")
```

**Go**:

```go
import "github.com/openai/openai-go"

client := openai.NewClient(option.WithAPIKey("sk-..."))
resp, err := client.Chat.Completions.New(ctx, openai.ChatCompletionNewParams{
    Model: openai.F(openai.ChatModelGPT4o),
    Messages: openai.F([]openai.ChatCompletionMessageParamUnion{
        openai.SystemMessage("你是助手"),
        openai.UserMessage("你好"),
    }),
})
```

### 2.2 Anthropic vs OpenAI 关键差异

| 维度 | Anthropic | OpenAI |
| --- | --- | --- |
| **端点** | `/v1/messages` | `/v1/chat/completions` |
| **max_tokens** | **必填** | 可选 |
| **system** | **独立字段** | 在 messages 里 role=system |
| **响应结构** | `content` 是 blocks 数组 | `choices[0].message.content` 字符串 |
| **usage 字段** | `input_tokens`/`output_tokens` | `prompt_tokens`/`completion_tokens` |
| **Tool 参数** | `input`(对象) | `arguments`(**JSON 字符串**) |
| **Tool result 角色** | `user` + tool_result block | `tool` |
| **stop_reason** | `end_turn`/`tool_use`/... | `finish_reason: stop/tool_calls/...` |

### 2.3 OpenAI 的 responses API(2025+ 新)

**OpenAI 2025 推的新 API**(取代 chat completions 的位置):

```python
response = client.responses.create(
    model="gpt-4o",
    input="你好",
    instructions="你是助手"  # 类似 system
)
print(response.output_text)
```

**特点**:
- 简化(不用手动管 messages)
- 内置对话状态(通过 `previous_response_id`)
- 目前和 chat completions 并存

**推荐**:新项目可以试 responses,老项目继续 chat completions。

### 2.4 兼容层(重要实用)

**很多国内 / 开源模型都兼容 OpenAI 协议**:

- DeepSeek / Qwen / Kimi / 智谱 GLM
- Ollama / vLLM 自部署

```python
# 换 base_url 就能调其他模型
client = OpenAI(
    api_key="...",
    base_url="https://api.deepseek.com/v1"
)
response = client.chat.completions.create(model="deepseek-chat", ...)
```

**Anthropic 也可以走 AWS Bedrock / Google Vertex AI**,但 API 有差异,走 SDK 更好。

> **一句话**:OpenAI Chat Completions **是业界事实标准**,大量国内/开源模型兼容协议(换 base_url 就用);和 Anthropic 关键差异 = **max_tokens 必填 / system 独立 / content blocks / tool args 字符串 vs 对象**,写代码留意别搞混。

---

## 三、流式响应(Streaming)

### 3.1 为什么必须流式

**同步等 5s**:
```
用户点提问 → 转圈 5 秒 → 出答案
体验差:感觉卡
```

**流式边推边显示**:
```
用户点提问 → 500ms 出首字 → 逐字流出
体验好:类 ChatGPT 感觉
```

**首字延迟(TTFT)**从 5s 降到 500ms,**用户感知快 10 倍**。

### 3.2 SSE 协议(底层)

**流式响应用 Server-Sent Events**——HTTP 长连接分块推事件:

```
HTTP/1.1 200 OK
Content-Type: text/event-stream

event: content_block_start
data: {"type": "content_block_start", "index": 0, ...}

event: content_block_delta
data: {"type": "content_block_delta", "delta": {"type": "text_delta", "text": "你"}}

event: content_block_delta
data: {"type": "content_block_delta", "delta": {"text": "好"}}

event: message_stop
data: {"type": "message_stop"}
```

**每个 event 是完整 JSON,一行一个**。

### 3.3 Anthropic 流式(Python)

```python
with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "写一首诗"}]
) as stream:
    # 简单模式:只取文本流
    for text in stream.text_stream:
        print(text, end="", flush=True)
    
    # 结束后获取完整 message
    final = stream.get_final_message()
    print(f"\n总 token: {final.usage.output_tokens}")
```

**更细粒度控制**:

```python
with client.messages.stream(...) as stream:
    for event in stream:
        if event.type == "content_block_start":
            block_type = event.content_block.type   # text/tool_use
        elif event.type == "content_block_delta":
            delta = event.delta
            if delta.type == "text_delta":
                print(delta.text, end="", flush=True)
            elif delta.type == "input_json_delta":
                # tool_use 参数分片
                accumulate_tool_args(delta.partial_json)
        elif event.type == "message_stop":
            break
```

### 3.4 Anthropic 流式(Go)

```go
stream := client.Messages.NewStreaming(ctx, params)
for stream.Next() {
    event := stream.Current()
    switch event := event.AsAny().(type) {
    case anthropic.ContentBlockDeltaEvent:
        if delta, ok := event.Delta.AsAny().(anthropic.TextDelta); ok {
            fmt.Print(delta.Text)
        }
    }
}
if stream.Err() != nil { panic(stream.Err()) }
```

### 3.5 OpenAI 流式

```python
stream = client.chat.completions.create(
    model="gpt-4o",
    messages=[...],
    stream=True
)
for chunk in stream:
    delta = chunk.choices[0].delta
    if delta.content:
        print(delta.content, end="", flush=True)
```

### 3.6 流式 + Tool Use(踩坑重点)

**问题**:tool_use 的参数是**分片流出**的 JSON,不是完整对象。

```
第 1 片: {"city":
第 2 片: "北京"
第 3 片: }
```

**处理**:
- 累积所有 delta 的 `partial_json` 字符串
- 到 `content_block_stop` 时完整 parse
- SDK 会自动处理(用 `final_message`)

```python
with client.messages.stream(...) as stream:
    for event in stream:
        ...  # 边流边看

    # 拿完整 message,tool_use 参数已解析好
    final = stream.get_final_message()
    for block in final.content:
        if block.type == "tool_use":
            print(block.input)  # 完整字典
```

### 3.7 流式的陷阱

```
1. 前端要能处理逐字渲染(不能等全部)
2. 网络断了要处理(重连 / 显示错误)
3. Backend 中间层(比如 API Gateway)必须支持 SSE(不能强转 http/1.1)
4. Nginx / CDN 配 buffering off(不然攒着不推)
5. 长任务超时(默认 60s,要调大或用 keepalive)
```

> **一句话**:流式响应 = **SSE + 分片 delta**,SDK 已封装好,重点是 **前端逐字渲染 + 中间层不 buffer + tool_use 参数需累积 parse**;首字延迟从 5s 降到 500ms,用户体验第一优化,必做。

---

## 四、多模态(Vision / PDF)

### 4.1 Vision(图片)

**Anthropic Python**:

```python
import base64

with open("photo.jpg", "rb") as f:
    img_data = base64.standard_b64encode(f.read()).decode()

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": [
            {"type": "image", "source": {
                "type": "base64",
                "media_type": "image/jpeg",
                "data": img_data
            }},
            {"type": "text", "text": "描述这张图"}
        ]
    }]
)
```

**从 URL**(推荐,不用 base64):

```python
{"type": "image", "source": {
    "type": "url",
    "url": "https://example.com/photo.jpg"
}}
```

### 4.2 支持格式

- **图片**:JPEG / PNG / GIF / WebP
- **最大**:5MB / 张,8000 x 8000 像素
- **Anthropic**:一次最多 100 张
- **OpenAI**:一次最多 500 张

### 4.3 PDF(Claude 独占)

**Anthropic 支持直接传 PDF**(不用先转成图):

```python
with open("report.pdf", "rb") as f:
    pdf_data = base64.standard_b64encode(f.read()).decode()

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": [
            {"type": "document", "source": {
                "type": "base64",
                "media_type": "application/pdf",
                "data": pdf_data
            }},
            {"type": "text", "text": "总结这份报告"}
        ]
    }]
)
```

**限制**:
- 最大 32MB / 100 页
- 混合处理(文本 + 图片)
- 每页 ≈ 1500-3000 tokens(取决内容)

### 4.4 Vision 用法示例

```
✓ OCR:识别截图里的文字
✓ 图表理解:分析数据图表
✓ UI 分析:看截图讲功能
✓ Diagram:读架构图 / 流程图
✓ 文档 QA:图文并茂文档
```

### 4.5 多模态的成本

**图片 = 大量 token**:

```
1 张 1024x1024 图 ≈ 1600 tokens
5 张图 = 8000 tokens
+ prompt = 上万

监控 usage 时,image_tokens 是重灾区
```

**优化**:
- 缩图(不影响识别的话)
- 只传必要区域
- 图片描述缓存(如果重复问)

> **一句话**:多模态用同一套 messages API,只是 content block type 换成 `image`/`document`;**图片是 token 大户**(1024x1024 ≈ 1600 tokens),Claude 独占 **PDF 直传**(不用先 OCR),文档 QA / 图表分析 / UI 理解都合适。

---

## 五、计费与 Usage(生产必须懂)

### 5.1 计费颗粒

**按 token 算,input 和 output 分开定价**:

```
Claude Sonnet 4.6:
  input: $3 / 1M tokens
  output: $15 / 1M tokens
  cache_read: $0.30 / 1M tokens  ← 缓存命中的 input
  cache_creation: $3.75 / 1M tokens  ← 首次写入 cache 的 input
```

**Output 通常 3-5x input 价**——LLM 生成比理解更贵。

### 5.2 Usage 字段解读

```python
response.usage
# {
#   "input_tokens": 500,           # 本次 input(不含 cache)
#   "output_tokens": 200,          # 本次 output
#   "cache_read_input_tokens": 3000,     # cache 命中(便宜)
#   "cache_creation_input_tokens": 0,    # 首次写 cache
# }
```

**计算实际成本**:

```python
def calc_cost(usage, model="sonnet-4-6"):
    prices = {  # per 1M tokens
        "sonnet-4-6": {"in": 3, "out": 15, "cache_r": 0.30, "cache_c": 3.75}
    }
    p = prices[model]
    return (
        usage.input_tokens * p["in"] +
        usage.output_tokens * p["out"] +
        usage.cache_read_input_tokens * p["cache_r"] +
        usage.cache_creation_input_tokens * p["cache_c"]
    ) / 1_000_000
```

### 5.3 精确 token 预算(不发起真实请求)

```python
# Anthropic 官方 count_tokens
result = client.messages.count_tokens(
    model="claude-sonnet-4-6",
    messages=[{"role": "user", "content": "你好"}],
    system="你是助手"
)
print(result.input_tokens)  # 精确 token 数
```

**用途**:
- 上线前估算成本
- 超窗口预警
- 用户输入长度限制

### 5.4 计费监控(生产必做)

```python
# 每次调用后记录
def log_usage(user_id, request_id, response):
    metrics.record({
        "user_id": user_id,
        "request_id": request_id,
        "model": response.model,
        "input_tokens": response.usage.input_tokens,
        "output_tokens": response.usage.output_tokens,
        "cache_read": response.usage.cache_read_input_tokens,
        "cost": calc_cost(response.usage, response.model),
        "timestamp": time.time()
    })
```

**关键指标**:
- 每用户每天 tokens
- 每业务功能 tokens
- 缓存命中率
- 异常高消耗(可能被攻击)

**详见** [12-production-engineering §一](12-production-engineering.md)。

### 5.5 免费额度和上限

- Anthropic:注册送少量额度,月度上限
- OpenAI:同上
- 都可以设 **spending limit**(账户级熔断)

生产项目 **应用层 + 账户级** 双重保险。

> **一句话**:LLM 计费按 token 算,input/output/cache 分开定价;必须每次调用记 usage → 分账 + 告警(见 [12 生产化](12-production-engineering.md));**cache 命中价格是原价 1/10**,高频调用一定用。

---

## 六、错误处理与重试

### 6.1 常见错误分类

| 错误 | HTTP | 处理 |
| --- | --- | --- |
| **BadRequest** | 400 | 参数错,**不重试** |
| **Authentication** | 401 | API Key 错,不重试 |
| **PermissionDenied** | 403 | 权限,不重试 |
| **NotFound** | 404 | 模型 id 错,不重试 |
| **RateLimit** | 429 | 限流,**重试**(退避 + 特殊处理) |
| **InternalServer** | 500 | 服务端错,重试 |
| **ServiceUnavailable** | 503 | 过载,重试 |
| **Overloaded** | 529 | Anthropic 过载,重试 |
| **APIConnection** | 网络 | 网络错,重试 |
| **APITimeout** | 超时 | 重试或降级 |

**铁律**:**5xx / 429 / 网络错误重试,4xx 不重试**(除 429)。

### 6.2 官方 SDK 内置重试

**Python**:

```python
client = Anthropic(
    max_retries=3,       # 默认 2 次
    timeout=60.0
)
```

**行为**:指数退避重试 5xx / 429 / 网络错误,4xx 直接抛异常。

### 6.3 自定义重试(生产推荐)

```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type
from anthropic import APIError, APIConnectionError, APITimeoutError, RateLimitError

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, min=2, max=30),
    retry=retry_if_exception_type((
        APIConnectionError, APITimeoutError,
        RateLimitError, APIError  # 5xx 会走 APIError
    )),
    reraise=True
)
def call_llm(messages):
    return client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=messages
    )
```

### 6.4 Rate Limit(429)特殊处理

**Anthropic 返回 Retry-After header**:

```python
try:
    resp = client.messages.create(...)
except RateLimitError as e:
    retry_after = e.response.headers.get("retry-after")
    time.sleep(int(retry_after) if retry_after else 5)
    # 重试
```

**用 SDK 一般自动处理**,自己实现要读 header。

### 6.5 超时设置

```python
# 长任务
client = Anthropic(timeout=300.0)  # 5 分钟

# 或单次调用覆盖
response = client.with_options(timeout=120).messages.create(...)
```

**建议**:
- 普通调用:60s
- 长输出:120-300s
- 流式:配 keepalive,底层可以更长

### 6.6 熔断(容灾)

```python
from circuitbreaker import circuit

@circuit(failure_threshold=5, recovery_timeout=60)
def call_llm(messages):
    return client.messages.create(...)
# 连续 5 次失败 → 熔断 60s
```

详见 [12-production-engineering §五](12-production-engineering.md)。

> **一句话**:错误处理铁律 = **5xx/429/网络错重试,4xx 不重试**;SDK 内置指数退避但生产推荐自己包一层 tenacity;429 读 Retry-After,长任务超时调大,失败多用熔断——**LLM API 会挂,不做兜底就是拿业务当赌注**。

---

## 七、多家 API 兼容与 Provider 抽象

### 7.1 场景:一套代码支持多家

需求:
- 主用 Claude,备用 GPT(Fallback)
- 客户 A 要 OpenAI,客户 B 要 Claude
- 本地开发 Ollama,生产 Anthropic

### 7.2 方案 1:LLM Gateway(推荐生产)

**Helicone / LiteLLM / Portkey / OpenRouter**:

```python
# LiteLLM 统一 API,支持 100+ 模型
from litellm import completion

response = completion(
    model="claude-sonnet-4-6",   # or "gpt-4o", "gemini-2.0-pro", ...
    messages=[{"role": "user", "content": "你好"}]
)
# 统一响应格式(OpenAI 兼容)
```

**优点**:
- 一个 API 调所有
- 自动 Fallback
- 成本追踪
- 缓存
- 限流

**参考** [12-production-engineering §七](12-production-engineering.md)。

### 7.3 方案 2:框架抽象(LangChain / Eino)

```python
from langchain_anthropic import ChatAnthropic
from langchain_openai import ChatOpenAI

# 换 model 一行事
llm = ChatAnthropic(model="claude-sonnet-4-6")
# llm = ChatOpenAI(model="gpt-4o")

response = llm.invoke([{"role": "user", "content": "你好"}])
```

### 7.4 方案 3:自己抽象接口

```python
class LLMProvider(ABC):
    @abstractmethod
    def chat(self, messages, **params) -> LLMResponse: ...

class AnthropicProvider(LLMProvider):
    def chat(self, messages, **params):
        resp = self.client.messages.create(...)
        return LLMResponse(text=resp.content[0].text, usage=...)

class OpenAIProvider(LLMProvider):
    def chat(self, messages, **params):
        resp = self.client.chat.completions.create(...)
        return LLMResponse(text=resp.choices[0].message.content, usage=...)
```

**优点**:完全可控,不依赖框架

**代价**:自己维护适配层

### 7.5 兼容性陷阱

- **Tool use 协议不同**(Anthropic vs OpenAI)
- **Streaming event 格式不同**
- **多模态 content block 不同**
- **参数名不同**(max_tokens vs max_completion_tokens)
- **计费字段不同**(input_tokens vs prompt_tokens)

**建议**:抽象层要屏蔽这些差异。

> **一句话**:多家 API 兼容三方案 = **LLM Gateway(Helicone/LiteLLM,生产推荐)/ 框架抽象(LangChain/Eino)/ 自建 Provider 接口**;协议差异多(tool/streaming/多模态/参数名/计费),抽象层必须屏蔽这些细节。

---

## 八、常见坑

```
坑 1:OpenAI arguments 是 JSON 字符串忘了 json.loads
  → 直接当对象用报错

坑 2:Anthropic max_tokens 忘填
  → 报错,必填参数

坑 3:Anthropic system 塞进 messages
  → 用 role=system 也能跑但违反规范,推荐独立字段

坑 4:流式响应不设 flush
  → 缓冲了不推,像卡死

坑 5:Nginx / API Gateway 中间层不支持 SSE
  → 前端收不到流式,一直等
  → 配 proxy_buffering off

坑 6:图片直接传大文件
  → 5MB 超限,base64 编码后更大
  → 传 URL 或缩图

坑 7:重试所有错误
  → 4xx 也重试,浪费 quota
  → 只重试 5xx/429/网络错

坑 8:忘记 429 的 Retry-After
  → 立刻重试触发更严限流

坑 9:超时太短
  → 长任务 60s 断开,复杂 agent 常见
  → 生产设 120-300s

坑 10:usage 只记 input+output
  → 忘了 cache_read / cache_creation,成本算错

坑 11:多环境用同一 API key
  → 开发烧了生产的额度
  → 每环境独立 key + 独立 spending limit

坑 12:API key 提交到 git
  → 泄漏后果严重
  → .env + git-secrets + secret manager
```

## 九、面试题速答

### Q1:Anthropic 和 OpenAI API 主要区别?

```text
6 大差异:
  1. 端点: /v1/messages vs /v1/chat/completions
  2. max_tokens: Anthropic 必填,OpenAI 可选
  3. system: Anthropic 独立字段,OpenAI 在 messages 里 role=system
  4. 响应: Anthropic content 是 blocks 数组,OpenAI choices[0].message.content 是字符串
  5. Tool 参数: Anthropic input(对象),OpenAI arguments(字符串,需 json.loads)
  6. Tool result: Anthropic user + tool_result block,OpenAI role=tool
  7. usage: Anthropic input_tokens/output_tokens,OpenAI prompt_tokens/completion_tokens

大量国内/开源模型兼容 OpenAI 协议(DeepSeek/Qwen/Kimi/Ollama),
换 base_url 就能用,生态更广。
```

### Q2:流式响应怎么实现?

```text
底层是 SSE(Server-Sent Events),HTTP 长连接分块推 event。
SDK 已封装好,重点:
  - text_stream(简单)
  - event stream(细粒度,拿 tool_use delta)

关键工程:
  - 前端逐字渲染,不能等全部
  - 中间层(Nginx/API Gateway)必须支持 SSE,配 buffering off
  - 长任务超时调大或用 keepalive
  - Tool use 参数是分片 partial_json,累积后 parse
    (SDK 的 get_final_message 已处理)

效果: 首字延迟从 5s 降到 500ms,用户感知快 10 倍,必做。
```

### Q3:LLM 调用怎么处理错误?

```text
分类处理:
  4xx(bad request/auth/permission): 不重试,直接抛
  429(rate limit): 重试 + 读 Retry-After header
  5xx / 网络错误: 重试 + 指数退避

SDK 内置基础重试(max_retries=2),生产推荐 tenacity 自己包:
  - stop_after_attempt(5)
  - wait_exponential(min=2, max=30)
  - retry_if_exception_type(APIConnectionError, RateLimitError, ...)

配套:
  超时: 60-300s 按场景
  熔断: 连续失败熔断 60s(见 12 生产化)
  Fallback: LLM Gateway 自动切模型
```

### Q4:多模态怎么用?

```text
同一套 messages API,content 里换 block type:
  文本: {"type": "text", "text": "..."}
  图片: {"type": "image", "source": {"type": "base64"|"url", ...}}
  PDF(Claude 独占): {"type": "document", "source": {...}}

限制:
  图片: 5MB/张, 8000x8000px, JPEG/PNG/GIF/WebP
  PDF: 32MB/100页

成本注意: 
  1 张 1024x1024 图 ≈ 1600 tokens,是 token 大户
  优化: 缩图 + 只传必要区域 + 结果缓存
```

### Q5:一套代码调多家 LLM 怎么做?

```text
三方案:
  1. LLM Gateway(生产推荐):
     Helicone / LiteLLM / Portkey / OpenRouter
     统一 API + 自动 Fallback + 成本追踪 + 缓存

  2. 框架抽象:
     LangChain ChatAnthropic vs ChatOpenAI
     换类就行

  3. 自建 Provider 接口:
     完全可控,自己写适配层

要屏蔽的差异:
  Tool use 协议 / Streaming 格式 / 多模态 block / 参数名 / 计费字段

选型:
  纯业务应用: LLM Gateway
  已用 LangChain/Eino: 框架抽象
  基础设施团队: 自建
```

### Q6:如何精确控制 LLM 成本?

```text
监控:
  每次调用记 usage(input/output/cache_read/cache_creation)
  按 user_id / feature / model 分账
  告警(单用户异常高消耗)

预估:
  调用前 count_tokens 精确算 token
  超阈值拒绝或换小模型

控制:
  上线前设账户 spending limit(硬熔断)
  应用层限流(用户/IP/token)
  Prompt Cache 高频调用(省 80%)
  Model Routing(简单走 Haiku)

铁律:
  一个 bug 让 prompt 循环 → 一天烧几十万
  必须双重保险: 应用层限流 + 账户 spending limit
```

## 十、关联阅读

```
本目录:
- 01-llm-fundamentals              token / 采样 / 选型 / 成本(本章前置)
- 03-prompt-engineering            Prompt 进阶(本章下一步)
- 04-tool-use-function-calling     Tool Use 协议详细(本章的进阶)
- 09-agent-frameworks              框架封装 SDK
- 12-production-engineering        生产化(重试/限流/成本详细)

外部:
- Anthropic API 文档: docs.anthropic.com
- OpenAI API 文档: platform.openai.com/docs
- Anthropic Python SDK: github.com/anthropics/anthropic-sdk-python
- Anthropic Go SDK: github.com/anthropics/anthropic-sdk-go
- OpenAI Python SDK: github.com/openai/openai-python
- LiteLLM: docs.litellm.ai
- Helicone: helicone.ai
```

> **一句话核心(全篇精炼)**:
> LLM API 调用 = **HTTPS POST + JSON**,SDK 只是封装;
> 核心 6 件事:**同步/流式(SSE)/多模态(image/PDF)/计费(token)/错误重试(5xx+429)/多家 API 兼容**;
> **Anthropic 和 OpenAI 差异细节多**(max_tokens/system/tool args/usage 字段);
> 生产必备:**LLM Gateway 屏蔽差异 + tenacity 重试 + 精确 usage 监控 + 账户 spending limit**;
> 能裸调是基本功,推荐官方 SDK,复杂上框架——8 年后端做 Agent 的 API 入门必修。
