# CDN · 协议优化

> HTTP/1.1 → HTTP/2 → HTTP/3 / TLS 优化 / Brotli 压缩 / 分片并发 / Range / 连接复用

## 〇、多概念对比：HTTP/1.1 vs HTTP/2 vs HTTP/3（D 模板）

### 一句话定位

| 协议 | 一句话定位 |
| --- | --- |
| **HTTP/1.1**（1997）| **基于 TCP 的文本协议**，持久连接 + Pipeline（但有队头阻塞），**仍占 30% 流量** |
| **HTTP/2**（2015）| **二进制分帧 + 多路复用 + HPACK 头压缩 + 服务推送**，解决应用层队头阻塞，**仍是 TCP（有传输层队头阻塞）** |
| **HTTP/3**（2022 RFC 9114）| **基于 QUIC（UDP）**，彻底解决队头阻塞 + 0-RTT 握手 + 连接迁移，**Google / Cloudflare / 国内大厂主推** |

### 多维度对比（18 维度，必背）

| 维度 | HTTP/1.1 | HTTP/2 | HTTP/3 |
| --- | --- | --- | --- |
| **发布年份** | 1997（RFC 2068）| 2015（RFC 7540）| 2022（RFC 9114）|
| **传输层** | **TCP** | **TCP** | **QUIC（基于 UDP）** |
| **协议格式** | **文本**（ASCII）| **二进制分帧** | 二进制分帧（QUIC 帧）|
| **连接复用** | 每域名 6 个并行连接 | **单连接多路复用**（Stream）| **单连接多路复用**（独立 Stream，无 HoL）|
| **应用层队头阻塞** | ⚠️ Pipeline 有 | ✅ 解决（多 Stream）| ✅ 解决 |
| **传输层队头阻塞** | ⚠️ 存在 | ⚠️ **仍存在**（TCP 重传阻塞所有 Stream）| ✅ **彻底解决**（QUIC 每 Stream 独立）|
| **头部压缩** | ❌（每次发完整头）| **HPACK**（静态表 + 动态表 + Huffman）| **QPACK**（HPACK 改进，解决乱序）|
| **服务端推送** | ❌ | ✅ Server Push（**多数浏览器已废弃**）| ⚠️ 已移除（实际不实用）|
| **优先级** | ❌ | Stream Priority Tree | Extensible Priority |
| **握手开销** | TCP 3 次（1 RTT）+ TLS 2 RTT = **3 RTT** | TCP 1 RTT + TLS 1.3 1 RTT = **2 RTT** | **QUIC 1 RTT**（首连）/ **0 RTT**（复连）|
| **0-RTT** | ❌ | TLS 1.3 可有 | ✅ **原生支持** |
| **连接迁移**（IP 切换）| ❌（TCP 五元组绑定）| ❌ | ✅ **Connection ID**（手机切 WiFi 不断）|
| **加密** | 可选（HTTPS 才加密）| **实际上必须 HTTPS**（浏览器要求）| **强制加密**（QUIC 内置 TLS 1.3）|
| **拥塞控制** | TCP 内核（CUBIC / BBR）| TCP 内核 | **QUIC 用户态**（可定制 BBR / Reno）|
| **丢包恢复** | TCP 重传 | TCP 重传（阻塞所有 Stream）| QUIC 独立 Stream 重传 |
| **CPU 占用** | 低 | 中（HPACK + 二进制解析）| **高**（用户态 + 加密 + 包处理）|
| **运维复杂度** | 低 | 中 | 高（UDP 防火墙 / NAT 穿透）|
| **业内现状** | 30% 流量 | 50% 流量 | 20% 流量（增长中）|

### 协作时序对比（同一页面：HTML + 5 CSS + 5 JS + 10 图）

```mermaid
sequenceDiagram
    participant B as Browser
    participant S as Server

    Note over B,S: HTTP/1.1（6 个并行连接）
    B->>S: GET / (conn1)
    S-->>B: index.html
    par
        B->>S: GET style.css (conn1)
        B->>S: GET app.js (conn2)
        B->>S: GET 1.png (conn3)
        B->>S: GET 2.png (conn4)
        B->>S: GET 3.png (conn5)
        B->>S: GET 4.png (conn6)
    end
    Note over B,S: 队头阻塞: conn1 慢 → style.css 后的请求等待

    Note over B,S: HTTP/2（单连接多 Stream）
    B->>S: TCP + TLS 握手（1 个连接）
    par
        B->>S: GET /style.css (Stream 1)
        B->>S: GET /app.js (Stream 3)
        B->>S: GET /1.png (Stream 5)
        B->>S: GET /2.png (Stream 7)
        Note over B: 二进制分帧并发
    end
    Note over B,S: ⚠️ TCP 丢包 → 所有 Stream 都等

    Note over B,S: HTTP/3（QUIC 多 Stream 独立）
    B->>S: QUIC 1-RTT 握手
    par
        B->>S: GET /style.css (QUIC Stream 1)
        B->>S: GET /app.js (QUIC Stream 2)
        B->>S: GET /1.png (QUIC Stream 3)
    end
    Note over B,S: ✅ Stream 3 丢包 → 只影响 Stream 3
```

### 握手对比（首次连接的 RTT 开销）

```
HTTP/1.1（HTTPS）:
  Client                    Server
    │                          │
    │── TCP SYN ──────────────→│  ┐
    │←── TCP SYN+ACK ──────────│  │ TCP 1 RTT
    │── TCP ACK ──────────────→│  ┘
    │                          │
    │── ClientHello ──────────→│  ┐
    │←── ServerHello + Cert ───│  │ TLS 1.2: 2 RTT
    │── KeyExchange + Fin ────→│  │
    │←── Fin ──────────────────│  ┘
    │                          │
    │── HTTP GET ─────────────→│  ┐ 数据 1 RTT
    │←── HTTP 200 ─────────────│  ┘
                                   = 4 RTT total（首次）

HTTP/2（HTTPS + TLS 1.3）:
  TCP 1 RTT + TLS 1.3 1 RTT + HTTP 1 RTT = 3 RTT

HTTP/3（QUIC）:
  Client                    Server
    │                          │
    │── QUIC Initial + TLS ───→│  ┐
    │←── QUIC Handshake + TLS ─│  │ QUIC 1 RTT 含 TLS
    │── HTTP GET ─────────────→│  │
    │←── HTTP 200 ─────────────│  ┘
                                   = 1 RTT（首次）

HTTP/3 0-RTT（已连接过）:
  Client                    Server
    │── QUIC + Resume + HTTP ─→│  ┐
    │←── HTTP 200 ─────────────│  ┘ 0 RTT
                                   首字节: 0 RTT!
```

### TCP HoL（队头阻塞）vs QUIC 独立 Stream

```
HTTP/2 over TCP（仍有传输层 HoL）:

Stream 1: ──Packet A──Packet B──Packet C──→
Stream 2: ──Packet D──Packet E──Packet F──→
Stream 3: ──Packet G──Packet H──Packet I──→

  TCP 把所有 Stream 的包混在一起按序传:
  [A, D, G, B, E, H, C, F, I]

  Packet D 丢了 → TCP 重传 D
  → A 后面的包都阻塞（即使是 Stream 1 / 3 的）
  → 所有 Stream 都等

HTTP/3 over QUIC（每 Stream 独立）:

Stream 1: ──Packet A──Packet B──Packet C──→
Stream 2: ──Packet D──Packet E──Packet F──→
Stream 3: ──Packet G──Packet H──Packet I──→

  QUIC 每 Stream 独立编号 + 独立 ACK:
  Packet D 丢了 → 只重传 D
  → Stream 1 / 3 继续接收数据
  → 真正的"无队头阻塞"
```

### HPACK / QPACK 头压缩对比

```
HTTP/1.1 头部（明文，每次发完整）:
  GET / HTTP/1.1
  Host: example.com
  User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...
  Accept: text/html,application/xhtml+xml,...
  Accept-Language: zh-CN,zh;q=0.9
  Cookie: session_id=abc; user_id=123; ...
  → 700-1500 字节（重复率高）

HTTP/2 HPACK:
  静态表（61 个常见 header）+ 动态表（连接内可变）+ Huffman 编码
  - 第一次发完整 → 加入动态表
  - 之后只发索引（1-2 字节）
  → 压缩率 85%+
  缺陷: 动态表依赖顺序，丢包导致解码失败

HTTP/3 QPACK:
  HPACK 的改进，解决乱序问题:
  - 编码流 + 解码流分离
  - 解决 QUIC 包乱序到达的头部解码问题
  → 同等压缩率 + 适配 QUIC 乱序
```

### 缺一不可分析

| 假设 | 后果 |
| --- | --- |
| **没 HTTP/1.1** | 互联网早期协议，仍 30% 流量，老服务 / API 仍依赖 |
| **没 HTTP/2** | 单连接多路复用 + HPACK 失效，老页面回到每页几十个连接 |
| **没 HTTP/3** | 移动端弱网 / WiFi 切换 / 跨国传输性能差 |
| **没 0-RTT** | 高频请求场景（API 网关）每次握手都 1-3 RTT |
| **没 QUIC 连接迁移** | 手机切网络 → 连接重建 → 长连接（如 IM / 视频）体验差 |

### 性能数据（生产参考）

```
弱网场景（5% 丢包率，100ms RTT，1MB 页面）:

HTTP/1.1: 5-8 秒
  - 多连接竞争 + 队头阻塞 + 慢启动多次

HTTP/2: 3-5 秒
  - 单连接复用 + HPACK，但 TCP 丢包阻塞所有 Stream
  - 实际比 1.1 好 30-40%

HTTP/3: 1.5-3 秒
  - QUIC 独立 Stream，丢包不影响其他
  - 0-RTT（复连）几乎无握手
  - 比 1.1 快 2-3 倍，比 2 快 30-50%

强网场景（千兆 + 1ms RTT）:
  三者差距小（< 10%），HTTP/2 已足够

移动弱网（4G + 切 WiFi）:
  HTTP/3 连接迁移 → 不断流（IM / 视频不卡）
  HTTP/2 → 必须重建连接
```

### 怎么选（决策树）

```mermaid
flowchart TD
    Q1{业务场景?}

    Q1 -->|纯静态资源 / 老服务 / 简单 API| H1[HTTP/1.1<br/>仍可用]
    Q1 -->|现代 Web / 大量小资源| H2[HTTP/2<br/>主流]
    Q1 -->|移动端 / 弱网 / 跨国 / 视频直播| H3[HTTP/3<br/>新主流]
    Q1 -->|内网 RPC| gRPC[gRPC over HTTP/2]

    style H2 fill:#9f9
    style H3 fill:#9ff
```

**实战推荐**：

| 场景 | 推荐 | 备注 |
| --- | --- | --- |
| 国内电商 PC | **HTTP/2** | 兼容性最好 |
| 国内电商 H5 / App | **HTTP/3** 优先 + HTTP/2 兜底 | 移动弱网 |
| 跨国业务 / CDN | **HTTP/3** | 跨海洋抗丢包 |
| 视频直播 / 大文件 | **HTTP/3** | 抗弱网 |
| gRPC 内部调用 | **HTTP/2** | gRPC 标准 |
| 老 API 兼容 | **HTTP/1.1** | 别动 |

### TLS 1.2 vs TLS 1.3 对比（HTTP/3 内置）

```
TLS 1.2（HTTP/2 时代）:
  - 握手 2 RTT
  - 加密算法可选（RC4/3DES 不安全选项）
  - Renegotiation 复杂

TLS 1.3（HTTP/3 内置）:
  - 握手 1 RTT（首次）/ 0 RTT（复连）
  - 加密算法精简（AEAD only，去掉不安全选项）
  - PFS 强制（每会话独立密钥）
  - 移除 RSA Key Exchange（强制 ECDHE）
  → 更快 + 更安全
```

### 反模式

```
❌ HTTP/2 over HTTP（无 TLS）→ 浏览器不支持
❌ HTTP/3 没有 UDP 防火墙配置 → 大量企业网络阻断 UDP
❌ HTTP/2 滥用 Server Push → 浏览器已废弃（实际效果差）
❌ HTTP/3 客户端不做协议降级（Alt-Svc）→ 不兼容客户端访问失败
❌ HTTPS 用 RSA 2048 + TLS 1.2 → 慢且不安全（应升级 ECDSA + TLS 1.3）
❌ 同一域名混用 HTTP/1 和 HTTP/2 → 浪费连接（应该全升 2）
❌ 期望 HTTP/3 0-RTT 完全替代 1-RTT → 0-RTT 有重放攻击风险（敏感 API 禁用）
```

### 一句话总结（D 模板专属）

> 三代 HTTP 演进的核心是 **"解决队头阻塞 + 减少 RTT"两条主线**：
> **HTTP/1.1**（TCP + 文本 + 队头阻塞）→ **HTTP/2**（TCP + 二进制分帧 + 多路复用 + HPACK，解决应用层 HoL 但 TCP HoL 仍在）→ **HTTP/3**（QUIC + UDP + 独立 Stream + 0-RTT + 连接迁移，彻底解决 HoL）。
> **缺一不可**：1.1 仍占 30% 流量（老服务 / API）/ 2.0 是主流 / 3.0 是移动端 + 弱网未来。
> **业内现状**：Google / Cloudflare / 国内 CDN（阿里 / 腾讯 / 字节）都已上 HTTP/3，移动 App 弱网场景比 HTTP/2 快 30-50%。
> **关键事实**：HTTP/3 的核心创新是 **传输层换成 QUIC**（解决 TCP HoL）+ **连接迁移**（IP 切换不断）+ **0-RTT**（API 网关首字节 0 RTT）。

---

## 一、HTTP 协议演进总览

```mermaid
flowchart LR
    H1[HTTP/1.0<br/>1996<br/>每请求一连接]
    H11[HTTP/1.1<br/>1997<br/>持久连接]
    H2[HTTP/2<br/>2015<br/>多路复用]
    H3[HTTP/3<br/>2022<br/>QUIC over UDP]

    H1 --> H11 --> H2 --> H3

    style H2 fill:#9f9
    style H3 fill:#9ff
```

| 版本 | 传输 | 复用 | 头压缩 | 关键改进 |
| --- | --- | --- | --- | --- |
| HTTP/1.0 | TCP | 无 | 无 | 基础协议 |
| HTTP/1.1 | TCP | Pipeline（实质失败） | 无 | Keep-Alive |
| HTTP/2 | TCP + TLS | 多路复用 | HPACK | 二进制分帧 |
| HTTP/3 | QUIC + UDP | 多路复用 | QPACK | 0-RTT，无队头阻塞 |

## 二、HTTP/1.1 痛点

### 2.1 队头阻塞（HOL Blocking）

```mermaid
sequenceDiagram
    participant C as 客户端
    participant S as 服务器

    Note over C,S: HTTP/1.1 单连接
    C->>S: 请求 1
    S->>C: 响应 1（慢，10s）
    Note over C: 必须等响应 1 完才能发响应 2
    C->>S: 请求 2
    S->>C: 响应 2
```

**问题**：单连接串行，慢请求堵后面。

### 2.2 浏览器解决方案：多连接

```
浏览器对同一域名最多开 6 个 TCP 连接
6 个并发 → 但仍是各自串行
```

每个连接还有 TCP 慢启动 + 三次握手 + TLS 握手开销。

### 2.3 域名分片（Domain Sharding）

```
img1.example.com  → 6 连接
img2.example.com  → 6 连接
img3.example.com  → 6 连接
合计 18 连接
```

**HTTP/1.1 时代的常见优化**，HTTP/2 后**反而是反模式**（破坏多路复用）。

## 三、HTTP/2

### 3.1 核心特性

```mermaid
mindmap
  root((HTTP/2))
    二进制分帧
      帧 Frame
      流 Stream
      消息 Message
    多路复用
      单 TCP 连接
      多 Stream 并发
      无队头阻塞 应用层
    头部压缩 HPACK
      静态表
      动态表
      Huffman 编码
    服务器推送
      Server Push
      已逐渐废弃
    流优先级
      Stream Priority
```

### 3.2 多路复用

```mermaid
sequenceDiagram
    participant C as 客户端
    participant S as 服务器

    Note over C,S: 单 TCP 连接
    par
        C->>S: Stream 1: 请求 a.css
        C->>S: Stream 3: 请求 b.js
        C->>S: Stream 5: 请求 c.png
    end
    par
        S->>C: Stream 5 帧（c.png 小，先到）
        S->>C: Stream 1 帧（a.css）
        S->>C: Stream 3 帧（b.js 大，最后）
    end
```

**核心**：多个流共享一个 TCP 连接，帧交错传输。

### 3.3 收益

| 指标 | HTTP/1.1 | HTTP/2 |
| --- | --- | --- |
| 连接数 | 6/域名 | 1/域名 |
| 并发请求 | 6 | 100+ |
| 头部开销 | 每次重复 | HPACK 压缩 |
| 首屏时间 | 慢 | 快 30-50% |

### 3.4 HPACK 头部压缩

```
首次请求:
  :method: GET
  :path: /index.html
  user-agent: Mozilla/5.0...
  cookie: sessionid=xxx
（约 500 字节）

后续请求（大部分头部走索引）:
  61 (索引到 :method: GET)
  62 (索引到 :path: /index.html)
  63 (索引到 user-agent)
  64 (索引到 cookie)
（约 30 字节）

压缩比 90%+
```

### 3.5 HTTP/2 的"假"队头阻塞

**应用层无队头阻塞**（多 Stream 并发）

**但 TCP 层仍有队头阻塞**：
```
TCP 包丢失 → TCP 层等重传 → 所有 Stream 都阻塞
```

弱网下 HTTP/2 反而比 HTTP/1.1 慢（多连接还能容忍丢包）。

**这是 HTTP/3 要解决的核心问题**。

### 3.6 启用 HTTP/2 的前提

```
1. 必须 HTTPS（Chrome/Firefox 强制）
2. 服务器/CDN 支持
3. 浏览器支持（>= 2015 年的都支持）
```

CDN 默认开 HTTP/2。

## 四、HTTP/3 与 QUIC

### 4.1 为什么需要 HTTP/3

```mermaid
flowchart TB
    Problem[HTTP/2 痛点]
    Problem --> P1[TCP 队头阻塞<br/>丢包阻所有流]
    Problem --> P2[TCP 握手慢<br/>3-RTT TLS 1.2 / 2-RTT TLS 1.3]
    Problem --> P3[移动网络切换<br/>WiFi→4G 连接断]
    Problem --> P4[TCP 是内核协议<br/>升级慢]

    Problem --> Solution[QUIC 解决]
    style Solution fill:#9f9
```

### 4.2 QUIC 是什么

```
QUIC = Quick UDP Internet Connections
基于 UDP + 在用户态实现可靠传输 + 内置 TLS 1.3
```

```mermaid
flowchart TB
    subgraph QUIC[QUIC 协议栈]
        H3[HTTP/3]
        TLS[TLS 1.3 内置]
        QStream[流复用 + 拥塞控制]
        UDP[UDP]
    end

    style QUIC fill:#9f9
```

### 4.3 QUIC 三大优势

#### 优势 1：无队头阻塞

```mermaid
flowchart LR
    subgraph H2[HTTP/2 over TCP]
        Stream1_H2[Stream 1] -.同一 TCP.-> Block[一包丢失全阻塞]
        Stream2_H2[Stream 2] -.同一 TCP.-> Block
    end

    subgraph H3[HTTP/3 over QUIC]
        Stream1_H3[Stream 1] -->|独立| OK[只阻塞本流]
        Stream2_H3[Stream 2] -->|独立| OK2[继续传输]
    end

    style H3 fill:#9f9
```

QUIC 每个 Stream 独立可靠，单包丢失只阻塞自己。

#### 优势 2：0-RTT / 1-RTT 握手

```mermaid
sequenceDiagram
    participant C as 客户端
    participant S as 服务器

    Note over C,S: 首次连接 1-RTT
    C->>S: ClientHello + ECDHE
    S->>C: ServerHello + 证书 + 密钥
    C->>S: 加密数据

    Note over C,S: 复连 0-RTT
    C->>S: ClientHello + 加密数据（直接发）
    S->>C: 响应
```

对比：TLS 1.2 over TCP 要 3-RTT（TCP 握手 1-RTT + TLS 握手 2-RTT）。

#### 优势 3：连接迁移（Connection Migration）

```mermaid
flowchart LR
    User[用户] -->|WiFi 连接| Server
    User -.切到 4G.-> Server2[Server<br/>同一 ConnID]

    Note["TCP: 必须重连<br/>QUIC: ConnID 不变，无缝切换"]

    style User fill:#9f9
```

QUIC 用 **ConnectionID 标识连接**，IP 变了连接不断 → 移动场景体验飞跃。

### 4.4 QUIC 性能对比

| 场景 | HTTP/2 | HTTP/3 |
| --- | --- | --- |
| 弱网（5% 丢包） | 慢 50% | 快 30% |
| 移动切网 | 重连 | 无缝 |
| 首屏 | 1-RTT TLS 1.3 | 0-RTT |
| 跨国 | 慢 | 显著快 |

### 4.5 QUIC 的代价

```
- UDP 在很多防火墙被限速 / 丢弃
- 用户态实现 → CPU 开销高
- 协议升级慢（部分中间设备不识别）
- 调试工具少
```

### 4.6 部署现状（2026）

| 环境 | 部署 |
| --- | --- |
| 浏览器 | Chrome / Firefox / Safari 默认开 |
| Cloudflare | 100% 节点支持 |
| Google / YouTube | 全量上 |
| Facebook / Meta | 全量上 |
| Cloudflare / Fastly | 默认开 |
| 阿里云 / 腾讯云 CDN | 大部分支持 |
| 国内 CDN | 视频厂商主推 |

## 五、TLS 优化

### 5.1 TLS 握手开销

```mermaid
sequenceDiagram
    participant C as 客户端
    participant S as 服务器

    Note over C,S: TLS 1.2 完整握手 (2-RTT)
    C->>S: ClientHello
    S->>C: ServerHello + 证书 + ServerKeyExchange + Done
    C->>S: ClientKeyExchange + ChangeCipherSpec + Finished
    S->>C: ChangeCipherSpec + Finished
    C->>S: 应用数据
```

```mermaid
sequenceDiagram
    participant C as 客户端
    participant S as 服务器

    Note over C,S: TLS 1.3 (1-RTT)
    C->>S: ClientHello + 密钥共享
    S->>C: ServerHello + 证书 + Finished
    C->>S: Finished + 应用数据
```

```mermaid
sequenceDiagram
    participant C as 客户端
    participant S as 服务器

    Note over C,S: TLS 1.3 0-RTT 重连
    C->>S: ClientHello + 加密数据（直接发）
    S->>C: 响应
```

### 5.2 TLS 1.2 vs TLS 1.3

| | TLS 1.2 | TLS 1.3 |
| --- | --- | --- |
| 完整握手 | 2-RTT | 1-RTT |
| 重连 | Session Resumption 1-RTT | 0-RTT |
| 加密套件 | 复杂、有弱算法 | 精简、强制前向安全 |
| 性能 | 慢 | 快 50%+ |

**强烈推荐 TLS 1.3**，CDN 默认开。

### 5.3 OCSP Stapling

```
传统 OCSP: 浏览器单独查证书吊销状态 → 慢 + 隐私泄漏
OCSP Stapling: 服务器替浏览器查好，握手时一起发
```

CDN 默认开，节省 100-300ms。

### 5.4 Session Resumption

```
首次握手 → 服务器给 Session ID / Session Ticket
重连时 → 客户端发 ID/Ticket → 跳过完整握手
```

CDN 节点共享 Session 池，提高重连命中率。

### 5.5 SNI（Server Name Indication）

```
单 IP 多证书共享：
  https://a.com → SNI: a.com → 用 a.com 证书
  https://b.com → SNI: b.com → 用 b.com 证书
```

CDN 节点必备，否则要每个域名一个 IP。

### 5.6 TLS 性能优化清单

```
□ 升级 TLS 1.3
□ 开 OCSP Stapling
□ 开 Session Resumption
□ 选高性能加密套件（AES-GCM / ChaCha20）
□ 证书链精简（不含中间证书冗余）
□ ECDSA 证书替代 RSA（更快、密钥更短）
□ HSTS 强制 HTTPS
```

## 六、压缩

### 6.1 压缩算法对比

| 算法 | 压缩比 | 压缩速度 | 解压速度 | 浏览器支持 |
| --- | --- | --- | --- | --- |
| **gzip** | 中 | 快 | 快 | 全部 |
| **Brotli** | 高（比 gzip 高 15-25%） | 慢 | 快 | 现代浏览器 |
| **zstd** | 中-高 | 极快 | 极快 | Cloudflare 实验 |

### 6.2 Brotli 优势

```
HTML/CSS/JS:
  原始 100KB → gzip 30KB → Brotli 22KB
节省 25% 带宽

文本类压缩比 Brotli > gzip
```

### 6.3 启用 Brotli

```
# CDN 配置
content_encoding: brotli   # 优先 Brotli
fallback: gzip              # 不支持时降级
```

```
请求头: Accept-Encoding: br, gzip
响应头: Content-Encoding: br
```

### 6.4 不要压缩的内容

- 已压缩文件（图片 JPG/PNG/WebP / 视频 MP4 / 压缩包 zip）→ 越压越大
- 小文件 < 1KB（开销 > 收益）
- 二进制流

CDN 默认按 MIME 类型决定。

### 6.5 图片优化

```
原始 JPG → CDN 自动转 WebP (节省 25-35%) → AVIF (节省 50%)
按 Accept 头返回不同格式
```

CDN 厂商提供"图片处理"产品（阿里云 IMG / Cloudflare Image Resizing）。

## 七、连接复用

### 7.1 Keep-Alive

```
HTTP/1.1 默认开 Keep-Alive
连接复用，避免重复建连
```

```
Connection: keep-alive
Keep-Alive: timeout=60, max=100
```

### 7.2 边缘 ↔ 父层 ↔ 源站 长连接

```mermaid
flowchart LR
    User[用户] <-->|短连接| Edge[边缘]
    Edge <-.持久长连接.-> Parent[父层]
    Parent <-.持久长连接.-> Origin[源站]

    style Edge fill:#9f9
```

CDN 内部全部用长连接，避免每次回源都握手。

### 7.3 连接池管理

```
- 节点维护到上游的连接池
- 池大小: 按 QPS 估算
- 空闲超时: 60s
- 最大复用次数: 1000
```

## 八、其他协议优化

### 8.1 Range 请求

```
请求: Range: bytes=0-1048575     (前 1MB)
响应: 206 Partial Content
       Content-Range: bytes 0-1048575/10485760
```

**适用**：
- 视频拖动（按需加载）
- 断点续传
- 大文件分片下载

CDN 必须支持 Range 才能做视频。

### 8.2 分片并发下载

```
大文件 100MB
客户端分 4 片并发下载:
  线程 1: bytes=0-25MB
  线程 2: bytes=25-50MB
  线程 3: bytes=50-75MB
  线程 4: bytes=75-100MB
速度提升 3-4 倍
```

迅雷 / aria2 / 视频播放器都用这招。

### 8.3 预连接 / 预拉取

```html
<!-- DNS 预解析 -->
<link rel="dns-prefetch" href="https://cdn.example.com">

<!-- 预建连（含 TLS） -->
<link rel="preconnect" href="https://cdn.example.com">

<!-- 预加载关键资源 -->
<link rel="preload" href="/main.js" as="script">

<!-- 预拉取下个页面 -->
<link rel="prefetch" href="/next-page.html">
```

减少首屏白屏时间 100-300ms。

### 8.4 HTTP/2 Server Push（已废弃）

```
原意: 服务器主动推送资源给浏览器
实际: 浏览器缓存已有的也推 → 浪费带宽
状态: Chrome 已移除支持，逐渐废弃
替代: <link rel="preload">
```

## 九、大厂方案对比

### 9.1 协议支持

| 厂商 | HTTP/2 | HTTP/3 | TLS 1.3 | Brotli | 0-RTT |
| --- | --- | --- | --- | --- | --- |
| Cloudflare | ✅ | ✅ 全量 | ✅ | ✅ | ✅ |
| Fastly | ✅ | ✅ | ✅ | ✅ | ✅ |
| AWS CloudFront | ✅ | ✅ | ✅ | ✅ | 部分 |
| 阿里云 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 腾讯云 | ✅ | ✅ | ✅ | ✅ | ✅ |

### 9.2 优化默认开关

| 优化 | Cloudflare | 阿里云 |
| --- | --- | --- |
| HTTP/2 | 默认 | 默认 |
| HTTP/3 | 默认 | 需要开 |
| Brotli | 默认 | 需要开 |
| 0-RTT | 默认 | 需要开 |
| OCSP Stapling | 默认 | 默认 |
| Image WebP | 需付费 | 需付费 |

CDN 选型时关注**默认开关 + 升级速度**。

## 十、典型坑

### 坑 1：以为 HTTP/2 一定快

弱网下 HTTP/2 反而慢（TCP 队头阻塞）。

**修复**：弱网场景升级 HTTP/3。

### 坑 2：HTTP/2 还在做域名分片

`img1.cdn.com` `img2.cdn.com` 反而破坏多路复用 → 用单域名。

### 坑 3：HTTP/3 没回退

UDP 被 ISP 限速 → 全挂。

**修复**：必须保留 HTTP/2 over TCP 作为回退。

### 坑 4：TLS 1.3 0-RTT 重放攻击

0-RTT 数据可被重放 → 不能用于非幂等操作（POST 订单）。

**修复**：0-RTT 只允许 GET 等幂等请求。

### 坑 5：Brotli 压缩了图片

CSS 压缩好，图片压完更大。

**修复**：按 MIME 类型决定。

### 坑 6：证书链冗余

证书链含多余中间证书 → 握手包大 → 慢。

**修复**：精简证书链。

### 坑 7：长连接没复用

连接池太小 / 超时太短 → 每次都新建。

**修复**：调大连接池 + 延长 idle 超时。

## 十一、面试高频题

**Q1：HTTP/2 vs HTTP/1.1 的核心区别？**

| | HTTP/1.1 | HTTP/2 |
| --- | --- | --- |
| 传输 | 文本 | 二进制 |
| 复用 | 无（pipeline 失败） | 多路复用 |
| 头部 | 重复发送 | HPACK 压缩 |
| 连接 | 每域名 6 个 | 1 个 |

**Q2：HTTP/2 的多路复用怎么实现？**

二进制分帧：
- 多个 Stream 共享一个 TCP 连接
- 帧带 Stream ID，可交错传输
- 应用层无队头阻塞（但 TCP 层仍有）

**Q3：HTTP/3 解决了什么？**

HTTP/2 的痛点：
- TCP 队头阻塞 → QUIC 每流独立
- TCP+TLS 多次握手 → QUIC 1-RTT/0-RTT
- 移动切网重连 → QUIC ConnID 不变

**Q4：QUIC 为什么基于 UDP？**

- TCP 是内核协议，升级慢
- UDP 在用户态可灵活实现可靠传输
- 跳出 TCP 队头阻塞

代价：UDP 被 ISP 限速、CPU 开销高。

**Q5：TLS 1.3 vs TLS 1.2？**

- 1.3 完整握手 1-RTT（1.2 是 2-RTT）
- 1.3 重连 0-RTT
- 1.3 加密套件精简、强制前向安全

**Q6：0-RTT 的安全风险？**

可被重放攻击 → 不能用于非幂等操作（POST）。

CDN 一般只对幂等 GET 启用。

**Q7：Brotli vs gzip？**

Brotli 压缩比比 gzip 高 15-25%（文本场景），解压速度相当。

CDN 现在标配 Brotli + gzip 双方案。

**Q8：CDN 怎么优化首屏？**

- HTTP/2 / HTTP/3 多路复用
- TLS 1.3 + 0-RTT
- Brotli 压缩
- 长连接复用
- 预连接 / 预拉取
- 图片转 WebP / AVIF

**Q9：移动场景 CDN 优化重点？**

- HTTP DNS（避免 DNS 劫持）
- HTTP/3（弱网友好 + 切网无缝）
- TLS 1.3 0-RTT
- Brotli

**Q10：域名分片在 HTTP/2 时代是好是坏？**

**坏**。HTTP/1.1 时代用来突破 6 连接限制；HTTP/2 单连接多路复用，分片反而：
- 多次 TLS 握手
- 破坏多路复用
- 浪费连接

## 十二、面试加分点

- HTTP/2 应用层无队头阻塞，**TCP 层仍有**
- HTTP/3 = **QUIC + UDP**，每流独立 + 0-RTT + 连接迁移
- **TLS 1.3 0-RTT 不能用于非幂等**（重放攻击）
- **Brotli 比 gzip 强 15-25%**（文本场景）
- HTTP/2 **域名分片是反模式**
- CDN 内部全部**长连接复用**（边缘 ↔ 父层 ↔ 源站）
- 移动 App 必上 **HTTP DNS + HTTP/3**
- **Server Push 已废弃**，用 `<link rel="preload">` 替代
- **图片转 WebP/AVIF** 节省 25-50% 带宽
- **HTTP/3 必须保留 HTTP/2 回退**（UDP 不通时）
- 大厂默认开关比堆砌功能更重要（看 CDN 文档）
