# DDD · 战略设计

> DDD 是什么 / 为什么 / 通用语言 / 限界上下文 / 子域分类 / 上下文映射 9 种模式

## 一、DDD 解决什么问题

### 1.1 软件复杂度的来源

```mermaid
flowchart TB
    Complexity[复杂度]
    Complexity --> Tech[技术复杂度<br/>数据库/网络/并发]
    Complexity --> Domain[业务复杂度<br/>★ DDD 关注这里]

    Tech --> TechSolved[已被框架/中间件大量解决]
    Domain --> DomainProblem[传统开发往往忽略]

    style Domain fill:#9f9
```

传统开发：
- 代码贴近"数据库表"
- 业务逻辑散落各处（Service / Util / Helper）
- 业务概念和代码概念脱节
- 业务变化时改动放射性大

DDD 主张：**让代码贴近业务**，让业务专家和开发者用同一种语言。

### 1.2 DDD 的两大支柱

```mermaid
flowchart LR
    DDD[DDD]
    DDD --> Str[战略设计<br/>整体方向]
    DDD --> Tac[战术设计<br/>具体落地]

    Str --> S1[通用语言]
    Str --> S2[限界上下文]
    Str --> S3[子域分类]
    Str --> S4[上下文映射]

    Tac --> T1[实体/值对象]
    Tac --> T2[聚合]
    Tac --> T3[仓储/工厂]
    Tac --> T4[领域服务/事件]

    style Str fill:#9f9
    style Tac fill:#9ff
```

- **战略设计**：宏观，划分系统、定边界、定关系（本篇）
- **战术设计**：微观，每个上下文里怎么写代码（详见 [02-tactical-building-blocks.md](02-tactical-building-blocks.md)）

## 二、通用语言（Ubiquitous Language）

### 2.1 核心思想

> **业务专家、开发、产品、测试都用同一套术语**

不是技术术语翻译给业务，而是建立一套**领域内共识的语言**，代码、文档、对话都用它。

### 2.2 反例

业务说"客户下单"，代码里：

```go
// 业务概念 → 技术名词
type UserOrderRecord struct {  // 业务说"订单", 代码叫 record
    UID     int64    // 业务说"客户", 代码叫 UID
    GoodsID int64    // 业务说"商品", 代码叫 GoodsID
    Amount  int      // 业务说"金额", 代码叫 amount (单位还不清楚)
}

func (r *UserOrderRecord) UpdateStatus(s int) {  // 业务说"取消订单", 代码叫"更新状态"
    r.Status = s
}
```

业务说"取消订单"，开发说"把 status 改成 5"。**沟通成本爆炸**。

### 2.3 正例

```go
// 业务怎么说, 代码就怎么写
type Order struct {
    OrderID    OrderID
    Customer   CustomerID
    Items      []OrderItem
    Status     OrderStatus
}

func (o *Order) Cancel(reason CancelReason) error {
    if !o.Status.CanCancel() { return ErrCannotCancel }
    o.Status = StatusCancelled
    o.AddEvent(OrderCancelled{Reason: reason})
    return nil
}
```

业务说"取消订单"，代码就 `order.Cancel()`。**直观无歧义**。

### 2.4 实践

- 整理业务术语表（Glossary）
- 团队所有沟通用术语表里的词
- 代码命名严格对齐
- 文档、API、数据库字段也用相同术语

### 2.5 避免的坑

- **同词不同义**：不同上下文里"产品"含义不同（销售 vs 物流），要靠**限界上下文**隔离
- **代码缩写**：`ord` / `usr` 隐藏业务含义，**全拼优先**
- **技术术语污染**：`UserDTO` 这种命名暴露技术细节，业务侧应叫 `Customer` / `User`

## 三、限界上下文（Bounded Context）

### 3.1 概念

> **明确的边界，边界内通用语言一致；边界外可以不同**

```mermaid
flowchart TB
    subgraph Sales["销售上下文"]
        Product1[Product<br/>名称/价格/描述]
    end

    subgraph Inventory["库存上下文"]
        Product2[Product<br/>SKU/库存数/位置]
    end

    subgraph Logistics["物流上下文"]
        Product3[Product<br/>重量/体积/运费]
    end

    Note["同名 Product, 不同含义<br/>各自的 Product 字段不同"]

    style Sales fill:#9f9
    style Inventory fill:#9ff
    style Logistics fill:#ff9
```

**核心**：
- 一个业务概念在不同上下文里**结构和职责可以完全不同**
- 上下文边界明确（包、模块、服务）
- 边界外通过**翻译**或**事件**通信

### 3.2 为什么需要边界

#### 反例：大一统模型

```go
// 一个 User 类含所有信息
type User struct {
    ID            int64
    Name          string
    Email         string
    Address       Address       // 物流要
    PaymentInfo   Payment       // 支付要
    Preferences   Preferences   // 推荐要
    LoginHistory  []Login       // 安全要
    OrderCount    int           // 营销要
    // ... 100+ 字段
}
```

问题：
- 上帝类，谁都能改
- 不同模块的需求挤在一起
- 一个字段变更影响所有模块

#### 正例：边界内独立

```go
// auth context
type User struct { ID, Email, Password }

// profile context
type Profile struct { UserID, Name, Avatar, Preferences }

// order context (Customer 是订单视角)
type Customer struct { ID, ShippingAddress }

// recommend context
type CustomerProfile struct { UserID, Tags, History }
```

各自演化，互不打扰。

### 3.3 上下文与微服务

```mermaid
flowchart LR
    BC[限界上下文] -.通常.-> MS[微服务]

    Note["1 个上下文 = 1 个微服务<br/>(常见, 但不是必须)"]
```

**关系**：
- **理想**：一个上下文 = 一个微服务（边界清晰）
- **实际**：一个微服务可包含多个紧密相关的上下文
- **反模式**：一个上下文拆成多个微服务（边界破碎）

DDD 不强制微服务，但微服务**很需要 DDD** 来划分边界。

### 3.4 边界划分依据

```mermaid
mindmap
  root((划分边界))
    业务能力
      围绕业务功能聚合
      销售/库存/物流/支付
    通用语言一致性
      边界内术语一致
      边界外可重名不重义
    变化频率
      经常一起变的放一起
      独立变化的分开
    团队边界
      康威定律
      一个团队负责一个上下文
```

## 四、子域分类

### 4.1 三种子域

```mermaid
flowchart LR
    D[业务域 Domain]
    D --> Core[核心域<br/>Core Domain]
    D --> Support[支撑域<br/>Supporting Subdomain]
    D --> Generic[通用域<br/>Generic Subdomain]

    Core --> CoreE[核心竞争力<br/>电商: 订单/推荐]
    Support --> SupportE[业务必需但非核心<br/>电商: 仓储/物流]
    Generic --> GenericE[通用功能<br/>认证/支付/通知]

    style Core fill:#9f9
    style Support fill:#ff9
    style Generic fill:#fcc
```

### 4.2 投资策略

| 子域 | 策略 | 资源投入 |
| --- | --- | --- |
| **核心域** | 自研 + DDD 充血模型 + 资深团队 | 最多 |
| **支撑域** | 自研但简化 + 偏 CRUD | 中 |
| **通用域** | 买现成（开源/SaaS） | 最少 |

### 4.3 例子：电商

| 子域 | 类型 | 说明 |
| --- | --- | --- |
| 商品管理 | 支撑域 | 业务需要但非竞争力 |
| 订单处理 | **核心域** | 业务核心，状态机复杂 |
| 推荐系统 | **核心域** | 算法是竞争力 |
| 库存管理 | 支撑域 | 业务必需但通用 |
| 支付 | 通用域 | 接第三方（支付宝/微信） |
| 短信通知 | 通用域 | 接 SaaS |
| 用户认证 | 通用域 | 接 OAuth/IAM |
| 物流配送 | 支撑域 / 部分核心 | 看业务 |

### 4.4 识别核心域

问题：**这个能力是公司赚钱的核心吗？竞品做得不好我们能因此胜出吗？**

是 → 核心域 → 用最好的人 + DDD 充血 + 持续投入。
否 → 不是核心 → 简化 / 外购 / 通用方案。

## 五、上下文映射（Context Mapping）

### 5.1 9 种关系模式

```mermaid
flowchart TB
    Map[上下文映射 9 种]
    Map --> P[Partnership<br/>合作伙伴]
    Map --> SK[Shared Kernel<br/>共享内核]
    Map --> CS[Customer-Supplier<br/>客户-供应商]
    Map --> CF[Conformist<br/>顺从者]
    Map --> ACL[Anti-Corruption Layer<br/>防腐层]
    Map --> OHS[Open Host Service<br/>开放主机服务]
    Map --> PL[Published Language<br/>发布语言]
    Map --> SW[Separate Ways<br/>各行其道]
    Map --> BBoM[Big Ball of Mud<br/>大泥球]

    style ACL fill:#9f9
    style OHS fill:#9f9
```

### 5.2 重点模式

#### Partnership（合作伙伴）

两个上下文**互相依赖**，必须协同发布。强耦合，慎用。

#### Shared Kernel（共享内核）

```mermaid
flowchart LR
    A[上下文 A] -.共享.-> SK[(共享内核<br/>核心模型)]
    B[上下文 B] -.共享.-> SK
```

两个上下文共享一小部分领域模型（如基础类型 `Money` / `Address`）。

**优点**：避免重复
**缺点**：耦合，修改要协调

#### Customer-Supplier（客户-供应商）

```mermaid
flowchart LR
    Sup[供应商<br/>上游] -->|需要满足客户需求| Cli[客户<br/>下游]
    Cli -.提需求.-> Sup
```

上游对下游负责，下游提需求上游配合实现。

#### Conformist（顺从者）

```mermaid
flowchart LR
    Big[强势上游<br/>不想配合] -->|API/数据| Small[弱势下游<br/>被动接受]
```

下游无法影响上游，被迫"顺从"上游模型（如对接第三方支付）。

#### **防腐层（Anti-Corruption Layer, ACL）★ 重要**

```mermaid
flowchart LR
    External[外部上下文<br/>含混乱模型] --> ACL[防腐层<br/>翻译]
    ACL --> Internal[内部领域模型<br/>干净]

    style ACL fill:#9f9
```

**核心**：在边界放一层翻译器，把外部模型转成内部模型，保护内部不被污染。

**典型场景**：
- 对接遗留系统
- 对接第三方（支付/物流）
- 微服务间集成

```go
// 外部 SDK 给的是 Alipay.OrderResp (字段乱七八糟)
type AlipayACL struct {
    sdk *alipay.Client
}

func (a *AlipayACL) QueryPayment(orderID string) (*PaymentStatus, error) {
    resp, err := a.sdk.QueryOrder(context.Background(), &alipay.QueryReq{
        OutTradeNo: orderID,
    })
    if err != nil { return nil, err }

    // 翻译成内部模型
    return &PaymentStatus{
        Status: a.translateStatus(resp.TradeStatus),
        Amount: Money{Cents: resp.AmountCents},
    }, nil
}
```

业务代码用 `PaymentStatus`，**不感知 Alipay SDK**。

#### Open Host Service（OHS）+ Published Language（PL）

```mermaid
flowchart LR
    Host[主机上下文<br/>提供 OHS] -->|发布的语言<br/>OpenAPI/Proto| Cli[多个客户端]
```

主机用**标准化协议**（REST + OpenAPI / gRPC + Proto）暴露服务，多个客户端按相同协议接入。

例：开放平台 API。

#### Separate Ways（各行其道）

两个上下文**完全独立**，没必要集成。

#### Big Ball of Mud（大泥球）

边界混乱、模型纠缠的反模式。隔离用 ACL。

### 5.3 上下文映射图

```mermaid
flowchart TB
    subgraph CoreUS["核心: 订单上下文 ★"]
        Order[Order BC]
    end

    subgraph Supp["支撑"]
        Stock[库存 BC]
        Logistics[物流 BC]
    end

    subgraph Generic["通用"]
        Pay[支付 BC]
        Notify[通知 BC]
        Auth[认证 BC]
    end

    External[第三方<br/>Alipay/SMS]

    Order -->|U/D 客户-供应商| Stock
    Order -->|事件| Logistics
    Order -->|ACL 防腐层| Pay
    Pay -.ACL.-> External
    Notify -.ACL.-> External

    style Order fill:#9f9
    style Pay fill:#9ff
```

## 六、战略设计实战流程

### 6.1 事件风暴（Event Storming）

业内推荐方法：

```mermaid
flowchart LR
    Step1[1 头脑风暴<br/>所有人贴所有领域事件] --> Step2[2 时间排序事件]
    Step2 --> Step3[3 找出命令<br/>触发事件的动作]
    Step3 --> Step4[4 找出聚合<br/>命令操作的对象]
    Step4 --> Step5[5 划分上下文<br/>相关聚合归类]
    Step5 --> Step6[6 上下文映射<br/>定关系]
```

**核心理念**：业务专家 + 开发 + 产品**一起在墙上贴便签**。

### 6.2 简化流程（中小项目）

```
1. 列出业务用例 (用户故事)
2. 找出关键名词 (实体候选) + 动词 (命令)
3. 按业务能力分组 → 候选上下文
4. 标记: 哪些是核心, 哪些是支撑/通用
5. 画上下文映射图
6. 每个上下文做战术设计
```

### 6.3 实战例子：电商

```
事件:
- 商品上架 / 下架
- 用户注册 / 登录
- 加入购物车
- 创建订单
- 支付完成
- 库存扣减
- 物流发货
- 收货确认
- 退货退款

按业务能力归类:
- 商品域: 上架/下架
- 用户域: 注册/登录
- 订单域: 购物车 → 订单 → 支付 → 完成 (★ 核心)
- 库存域: 扣减
- 物流域: 发货/送达
- 售后域: 退货退款

上下文映射:
- 订单 → 库存: 扣减 (Customer-Supplier 或事件)
- 订单 → 支付: ACL 防腐 (对接第三方)
- 订单 → 物流: 事件触发
- 订单 → 通知: 事件
```

## 七、典型坑

### 坑 1：把模型当成 ER 图

```
[误区] 用户表 → 订单表 → 商品表 → 这就是模型
[正确] 模型是业务行为 + 数据 + 规则的整体, 不只是数据
```

### 坑 2：边界太大

整个系统一个上下文 → 退化成传统开发，DDD 没价值。

**修复**：按业务能力 + 团队拆分。一般 3~10 个上下文。

### 坑 3：边界太小

把每个实体当一个上下文 → 沟通成本爆炸。

**修复**：相关聚合归到同一上下文。

### 坑 4：上下文边界模糊

```
"订单"上下文里塞了用户、库存、支付的逻辑 → 边界没意义
```

**修复**：严格遵守边界，跨边界用 ACL 或事件。

### 坑 5：通用语言只是口号

文档里写一套，代码里另一套。

**修复**：代码命名 / API / 数据库字段都用术语表的词。

### 坑 6：忽略子域分类

所有子域投入相同资源 → 核心域被支撑域拖累。

**修复**：识别核心域，集中资源。通用域用现成方案。

### 坑 7：DDD 用错地方

简单 CRUD 系统用 DDD → 过度设计，开发慢。

**修复**：业务复杂的核心域才用 DDD，简单业务直接 CRUD。

## 八、高频面试题

**Q1：DDD 解决什么问题？**

让代码贴近业务，让业务专家和开发用**通用语言**沟通，应对**复杂业务领域**。

不解决技术复杂度（那是框架的事）。

**Q2：什么时候用 DDD？什么时候不用？**

**用**：
- 业务复杂（复杂规则、状态机、流程）
- 长期演化的核心域
- 多团队协作

**不用**：
- 简单 CRUD
- 数据迁移类工作
- 小工具
- 团队没人懂 DDD（学习成本高于收益）

**Q3：什么是限界上下文？怎么划分？**

**限界上下文**：明确边界，边界内通用语言一致；边界外可同名不同义。

**划分依据**：
- 业务能力（销售/库存/物流）
- 通用语言一致性
- 变化频率（经常一起变）
- 团队边界（康威定律）

**Q4：限界上下文 = 微服务吗？**

**不一定**。
- 理想：1 上下文 = 1 微服务
- 实际：1 微服务可包含多个紧密上下文
- 反模式：1 上下文拆多个微服务

DDD 不强制微服务，但微服务**很需要 DDD**。

**Q5：通用语言（Ubiquitous Language）是什么？**

业务专家、开发、产品、测试**用同一套术语**沟通。代码命名、API、文档、数据库字段都对齐。

避免业务和技术的翻译损耗。

**Q6：核心域 / 支撑域 / 通用域怎么区分？**

| | 核心域 | 支撑域 | 通用域 |
| --- | --- | --- | --- |
| 定义 | 业务竞争力 | 业务必需但非核心 | 通用功能 |
| 例子（电商）| 订单/推荐 | 库存/物流 | 支付/认证/通知 |
| 投入 | 最多 + DDD | 中 | 最少 + 外购 |

识别核心：**做得更好能让公司赢的能力**。

**Q7：上下文映射有哪些模式？**

9 种：Partnership / Shared Kernel / Customer-Supplier / Conformist / **ACL** / OHS / PL / Separate Ways / Big Ball of Mud。

**重点 ACL（防腐层）**：边界处放翻译器，把外部模型转内部模型，保护核心不被污染。

**Q8：防腐层（ACL）是什么？什么时候用？**

边界处的翻译层，外部模型 → 内部模型。

**场景**：
- 对接遗留系统
- 对接第三方（支付/物流）
- 微服务间集成时上游不可控

业务代码看到的永远是干净的内部模型。

**Q9：DDD 和 CQRS / 事件溯源什么关系？**

- DDD 是基础（建模 + 限界上下文 + 聚合）
- CQRS 是 DDD 实现可选项（读写分离）
- 事件溯源是 CQRS 实现可选项（事件作为唯一真源）

DDD 不要求用 CQRS / ES，但 CQRS / ES 几乎都基于 DDD。

详见 [05-cqrs-eventsourcing.md](05-cqrs-eventsourcing.md)。

**Q10：事件风暴（Event Storming）是什么？**

DDD 战略设计的实践方法：业务专家 + 开发 + 产品**一起在墙上贴便签**梳理领域事件，从事件推导命令、聚合、上下文。

是建立通用语言和找出限界上下文的高效方式。

## 九、面试加分点

- 强调 DDD 的核心是**让代码贴近业务**，不是某种技术
- 通用语言是 DDD 的灵魂（不只是术语表，是真的所有沟通都用）
- 限界上下文 ≠ 微服务（但相关）
- 核心域要 DDD，通用域不需要
- ACL 是 DDD 的"边界守门员"
- 事件风暴是高效的战略设计方法
- DDD 适合**业务复杂的核心域**，不适合简单 CRUD
- 康威定律：组织结构决定上下文边界
- DDD 学习曲线陡，团队要有共识才能用
- 战略 > 战术（战略错了战术再好也救不回来）
