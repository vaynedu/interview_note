# CDN

> 你的强项领域。本目录按"原理 + 缓存 + 调度 + 协议 + 安全 + 边缘 + 场景 + 故障"组织，覆盖资深面试 + 项目实战。

## 目录

| # | 文件 | 涵盖 |
| --- | --- | --- |
| 01 | [架构与原理](01-architecture.md) | 三层结构 / 数据流 / 关键指标 / 大厂方案对比 |
| 02 | [缓存策略](02-cache-strategy.md) | Cache-Control / 协商缓存 / 缓存键 / SWR / 命中率优化 |
| 03 | [调度与回源](03-routing-dispatch.md) | GSLB / HTTP DNS / Anycast / 302 / 回源策略 / 跨网 |
| 04 | [协议优化](04-protocol-optimization.md) | HTTP/2 / HTTP/3 QUIC / TLS / Brotli / 长连接 |
| 05 | [安全与防护](05-security.md) | DDoS / WAF / 防盗链 / HTTPS / HSTS / 边缘鉴权 / Bot |
| 06 | [边缘计算](06-edge-computing.md) | Workers / Lambda@Edge / Fastly Compute / V8 / WASM |
| 07 | [典型场景](07-scenarios.md) | 点播 / 直播 / 短视频 / 大文件 / API / 图片 / 游戏 / 移动 |
| 08 | [故障案例与运维](08-troubleshooting-cases.md) | 命中率 / 回源风暴 / CC / 容量 / 监控 / 演练 |

## 跨章高频题

- CDN 是什么？解决什么问题？（→ 01）
- 三层架构 + 父层作用？（→ 01）
- 命中率怎么算？怎么优化？（→ 01 / 02）
- 强缓存 vs 协商缓存？ETag vs Last-Modified？（→ 02）
- 缓存键怎么设计？为什么默认含 query 危险？（→ 02）
- Stale-While-Revalidate 解决什么？（→ 02）
- CDN 怎么调度到最近节点？LDNS 不准怎么办？（→ 03）
- HTTP DNS / Anycast 各自适合什么？（→ 03）
- 回源风暴是什么？怎么防？（→ 03 / 08）
- HTTP/2 vs HTTP/3 核心区别？（→ 04）
- QUIC 为什么基于 UDP？三大优势？（→ 04）
- TLS 1.3 0-RTT 的安全风险？（→ 04）
- Brotli vs gzip？（→ 04）
- DDoS 攻击类型 + 防护？CC 怎么拦？（→ 05）
- 防盗链怎么做？签名 URL 实现？（→ 05）
- 边缘计算解决什么？V8 Isolate vs Lambda@Edge？（→ 06）
- HLS / DASH / WebRTC 怎么选？低延迟直播怎么做？（→ 07）
- API 能上 CDN 吗？动态加速原理？（→ 07）
- 大文件下载 + P2P 加速？（→ 07）
- 命中率骤降怎么排查？（→ 08）
- 大促容量怎么规划？故障演练做哪些？（→ 08）

## 设计原则

- **图文并茂**，关键概念用 Mermaid
- **每篇独立可读**
- **原理 + 实战 + 大厂对比 + 面试题** 四位一体
- 突出**真实运维经验**和**故障复盘**
