# HTTP 协议

> HTTP 是应用层无状态协议。这一篇集中讲**方法、状态码、头部、版本演进**,把散落的 HTTP 知识聚合起来。
> HTTP/1.1 vs HTTP/2 vs HTTP/3 的深度对比看 [../11-cdn/04-protocol-optimization.md](../11-cdn/04-protocol-optimization.md)。

## 一、协议概览

### 1.1 HTTP 是什么

- **应用层协议**,跑在 TCP(HTTP/1.1、2)或 QUIC/UDP(HTTP/3)之上
- **无状态**:每次请求独立,服务端不记录上下文 → 状态靠 Cookie / Session / Token 维持
- **请求-响应模型**:客户端发请求,服务端回响应
- **文本协议**(HTTP/1.x)/ 二进制协议(HTTP/2、3)

### 1.2 报文结构

```
请求报文:                              响应报文:
GET /api/user HTTP/1.1                HTTP/1.1 200 OK
Host: example.com                     Content-Type: application/json
User-Agent: curl/7.79                 Content-Length: 27
Accept: */*                           Server: nginx
                                      
                                      {"id":1,"name":"alice"}
^─────────────^                       ^──────────────^
请求行 + 请求头 + 空行 + 请求体        状态行 + 响应头 + 空行 + 响应体
```

**核心三元素**:起始行 / 头部 / 主体,用空行分隔头和主体。

## 二、HTTP 方法(9 种标准)

### 2.1 方法总览

| 方法 | 用途 | 安全 | 幂等 | 有 Body | 缓存 |
| --- | --- | --- | --- | --- | --- |
| **GET** | 获取资源 | ✅ | ✅ | ❌(规范不建议) | ✅ |
| **POST** | 创建/提交数据 | ❌ | ❌ | ✅ | ❌(默认) |
| **PUT** | 整体替换/创建 | ❌ | ✅ | ✅ | ❌ |
| **PATCH** | 局部更新 | ❌ | ❌(规范) | ✅ | ❌ |
| **DELETE** | 删除资源 | ❌ | ✅ | 可有 | ❌ |
| **HEAD** | 只取响应头 | ✅ | ✅ | ❌ | ✅ |
| **OPTIONS** | 查询支持的方法 / CORS 预检 | ✅ | ✅ | 可有 | ❌ |
| **CONNECT** | 建立隧道(HTTPS 代理) | ❌ | ❌ | - | ❌ |
| **TRACE** | 回显请求(诊断,生产禁用) | ✅ | ✅ | ❌ | ❌ |

> 来源:**RFC 9110**(HTTP Semantics, 2022,取代了 RFC 7231)。

### 2.2 三大核心概念

**安全 (Safe)**:不改变服务端状态的方法 → GET / HEAD / OPTIONS / TRACE。
- ⚠ 注意:"安全"是规范层面"只读不改";如果你写了个 `GET /api/delete?id=1` 删数据,是**反模式**,被爬虫扫一遍就完蛋。

**幂等 (Idempotent)**:同一请求执行 N 次效果 = 执行 1 次。

| 幂等 | 不幂等 |
| --- | --- |
| GET / HEAD / OPTIONS / PUT / DELETE / TRACE | POST / PATCH / CONNECT |

易错点:
- **PUT 幂等**:`PUT /user/1 {name:"a"}` 重发 N 次,最终都是 name=a
- **DELETE 幂等**:第一次 200,后面 N 次 404,**最终状态相同**就算幂等
- **POST 不幂等**:`POST /order` 调 N 次会创建 N 个订单
- **PATCH 规范上不幂等**:`PATCH /counter {op:"+1"}` 每次都改;但**可以设计成幂等**(`PATCH {value:5}` 设值)

**可缓存 (Cacheable)**:只有 **GET / HEAD** 默认可缓存。POST 理论上可以,但实际几乎不缓存。

### 2.3 PUT vs PATCH vs POST(高频混淆 ★)

```
PUT  /user/1   {name: "alice", age: 18}   → 整体替换,其他字段会被清空
PATCH /user/1  {age: 19}                  → 只改 age,其他字段保留
POST /users    {name: "alice"}            → 创建新资源,服务端分配 ID
```

| | POST | PUT | PATCH |
| --- | --- | --- | --- |
| **目的** | 创建子资源 | 创建或整体替换 | 局部更新 |
| **URL 指向** | 集合 `/users` | 具体资源 `/users/1` | 具体资源 `/users/1` |
| **幂等** | ❌ | ✅ | ❌(默认) |
| **必须带全字段** | - | ✅ 是 | ❌ 否 |

面试常考:
- 创建订单为啥用 POST 不用 PUT? → 客户端不知道新 ID,服务端分配 → POST
- 用 PUT 修改字段会有什么问题? → 漏传字段会被清成默认值

### 2.4 HEAD 的实战价值

HEAD = GET 但只返回响应头不返回 body。

典型用途:
1. **检查文件存在** + 拿大小(`Content-Length`)
2. **检查缓存是否过期**(`Last-Modified` / `ETag`)
3. **断点续传前探测**(`Accept-Ranges: bytes`)
4. **健康检查**(节省带宽)
5. **检查链接有效性**(爬虫不下载 body)

```bash
curl -I https://example.com/big.zip
# HTTP/1.1 200 OK
# Content-Length: 1073741824
# Accept-Ranges: bytes
# Last-Modified: Wed, 21 Oct 2026 07:00:00 GMT
```

### 2.5 OPTIONS 的两大场景

**场景 1:查询服务端支持的方法**

```bash
curl -X OPTIONS -i https://example.com/api/user
# HTTP/1.1 200 OK
# Allow: GET, POST, PUT, DELETE
```

**场景 2:CORS 预检请求(最常见 ★)**

跨域 + 非简单请求(自定义 header / PUT / PATCH / DELETE / JSON body)前,浏览器**自动发 OPTIONS** 询问服务端是否允许。

```http
OPTIONS /api/user HTTP/1.1
Origin: https://app.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Authorization

→ 服务端响应:
Access-Control-Allow-Origin: https://app.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization
Access-Control-Max-Age: 600   ← 600 秒内不再发 OPTIONS
```

坑:OPTIONS 失败 → 真实请求根本不会发出 → 前端看到"网络错误",后端日志里只有 OPTIONS,看不到原始请求。

### 2.6 CONNECT 的用途

**用于 HTTPS 代理建立 TCP 隧道**:

```
浏览器 → 代理: CONNECT example.com:443 HTTP/1.1
代理   → 浏览器: 200 Connection Established
                ↓
        之后透传 TLS 字节,代理看不到内容(端到端加密)
```

业务代码几乎用不到,但面试常考"为啥代理能看到 HTTP 内容但看不到 HTTPS 内容"。

### 2.7 TRACE 为什么生产禁用

TRACE 让服务端把收到的请求原样回显:

```
请求: TRACE /api HTTP/1.1
      X-Custom-Header: secret

响应: HTTP/1.1 200 OK
      Content-Type: message/http
      TRACE /api HTTP/1.1
      X-Custom-Header: secret      ← 原样返回
```

**安全风险**:**Cross-Site Tracing (XST) 攻击**,攻击者借 TRACE 偷浏览器 Cookie / Authorization(即使设了 HttpOnly)。

修复:Nginx / Apache 默认禁用,生产必关。

```nginx
if ($request_method = TRACE) { return 405; }
```

### 2.8 WebDAV 扩展方法(了解)

RFC 4918 扩展的文档协作方法,面试不太考:

| 方法 | 用途 |
| --- | --- |
| COPY / MOVE | 复制 / 移动资源 |
| MKCOL | 创建集合(目录) |
| LOCK / UNLOCK | 加锁 / 解锁 |
| PROPFIND / PROPPATCH | 查询 / 修改属性 |

场景:Nextcloud / 坚果云 / Apple iCloud 等云盘同步协议。

## 三、HTTP 状态码

### 3.1 五大类

| 范围 | 类别 | 含义 |
| --- | --- | --- |
| **1xx** | Informational | 信息性(很少见) |
| **2xx** | Success | 成功 |
| **3xx** | Redirection | 重定向 |
| **4xx** | Client Error | 客户端错误 |
| **5xx** | Server Error | 服务端错误 |

### 3.2 高频状态码

| 状态码 | 含义 | 典型场景 |
| --- | --- | --- |
| **100** | Continue | 大 body 上传前的"许可" |
| **101** | Switching Protocols | WebSocket 升级 |
| **200** | OK | 成功 |
| **201** | Created | POST/PUT 创建成功(常带 Location 头) |
| **202** | Accepted | 异步任务已接收(还没完成) |
| **204** | No Content | 成功但无返回体(DELETE / PUT 常用) |
| **206** | Partial Content | 分块下载、断点续传 |
| **301** | Moved Permanently | **永久**重定向(SEO 友好,会被缓存) |
| **302** | Found | **临时**重定向 |
| **304** | Not Modified | 协商缓存命中,**最重要的 3xx** |
| **307** | Temporary Redirect | 临时重定向,**保持方法和 body** |
| **308** | Permanent Redirect | 永久重定向,**保持方法和 body** |
| **400** | Bad Request | 参数 / 格式错误 |
| **401** | Unauthorized | **未认证**(没登录) |
| **403** | Forbidden | **已认证但无权限** |
| **404** | Not Found | 资源不存在 |
| **405** | Method Not Allowed | 方法不允许(常带 Allow 头) |
| **408** | Request Timeout | 服务端等请求超时 |
| **409** | Conflict | 资源冲突(并发更新 / 重复创建) |
| **410** | Gone | 资源永久消失(比 404 强) |
| **413** | Payload Too Large | 请求体超过限制(Nginx `client_max_body_size`) |
| **414** | URI Too Long | URL 太长 |
| **415** | Unsupported Media Type | Content-Type 不支持 |
| **422** | Unprocessable Entity | 语法对但语义错(参数校验失败) |
| **429** | Too Many Requests | **限流**(常带 Retry-After 头) |
| **499** | Client Closed Request | **Nginx 私有**:客户端主动断开 |
| **500** | Internal Server Error | 服务端兜底错误 |
| **501** | Not Implemented | 方法未实现 |
| **502** | Bad Gateway | **上游返回了不合法响应**(后端挂了 / 连接被重置) |
| **503** | Service Unavailable | 服务暂时不可用(过载 / 维护) |
| **504** | Gateway Timeout | **网关等上游超时**(后端慢) |
| **505** | HTTP Version Not Supported | 协议版本不支持 |

### 3.3 易错对比

**301 vs 302 vs 307 vs 308**:

| | 永久/临时 | 保持方法 | 浏览器缓存 |
| --- | --- | --- | --- |
| 301 | 永久 | ❌ 可能改成 GET(历史遗留) | ✅ |
| 302 | 临时 | ❌ 可能改成 GET | ❌ |
| 307 | 临时 | ✅ 保持原方法和 body | ❌ |
| 308 | 永久 | ✅ 保持原方法和 body | ✅ |

实战:POST 跳转用 **307/308** 才能保留方法,用 301/302 会被改成 GET。

**401 vs 403**:
- **401 没认证**(token 过期 / 没登录) → 应该跳登录
- **403 已认证无权限**(普通用户访问管理接口) → 应该提示无权限

**502 vs 504**:
- **502** = 后端响应不合法(进程崩了 / 连接被 RST) → 看后端日志
- **504** = 网关等后端超时 → 看后端耗时

## 四、HTTP 头部

### 4.1 头部分类

| 类别 | 例子 |
| --- | --- |
| **通用头** | Date, Connection, Cache-Control, Via |
| **请求头** | Host, User-Agent, Accept, Authorization, Cookie, Referer, Origin |
| **响应头** | Server, Set-Cookie, Location, WWW-Authenticate, Content-Type |
| **实体头** | Content-Type, Content-Length, Content-Encoding, ETag, Last-Modified |

### 4.2 高频头部速查

**身份与认证**:

| 头 | 用途 |
| --- | --- |
| `Authorization: Bearer xxx` | 携带 Token |
| `WWW-Authenticate` | 401 响应里告诉客户端用什么方式认证 |
| `Cookie: k=v; k2=v2` | 客户端送的 Cookie |
| `Set-Cookie: k=v; HttpOnly; Secure; SameSite=Lax` | 服务端下发 Cookie |

**内容协商**:

| 头 | 用途 |
| --- | --- |
| `Accept: application/json` | 客户端能接受的格式 |
| `Accept-Encoding: gzip, br` | 能接受的压缩 |
| `Accept-Language: zh-CN` | 能接受的语言 |
| `Content-Type: application/json; charset=utf-8` | 主体格式 |
| `Content-Encoding: gzip` | 主体压缩方式 |

**缓存控制**(详见 [../11-cdn/02-cache-strategy.md](../11-cdn/02-cache-strategy.md)):

| 头 | 用途 |
| --- | --- |
| `Cache-Control: max-age=3600, s-maxage=7200` | 强缓存控制 |
| `Expires: <date>` | 强缓存绝对时间(已被 Cache-Control 取代) |
| `ETag: "abc123"` | 协商缓存内容指纹 |
| `If-None-Match: "abc123"` | 请求带上,服务端比对 |
| `Last-Modified` / `If-Modified-Since` | 协商缓存(基于时间) |
| `Vary: Accept-Encoding` | 按头分别缓存 |

**连接控制**:

| 头 | 用途 |
| --- | --- |
| `Connection: keep-alive` / `close` | 是否复用连接 |
| `Keep-Alive: timeout=60, max=1000` | 长连接参数 |
| `Upgrade: websocket` | 协议升级 |
| `Host: example.com` | **HTTP/1.1 必带**,虚拟主机依赖 |

**CORS 跨域**:

| 头 | 用途 |
| --- | --- |
| `Origin: https://app.com` | 请求来源 |
| `Access-Control-Allow-Origin` | 允许的源 |
| `Access-Control-Allow-Methods` | 允许的方法 |
| `Access-Control-Allow-Credentials: true` | 允许带 Cookie |
| `Access-Control-Max-Age: 600` | 预检结果缓存秒数 |

**安全相关**:

| 头 | 用途 |
| --- | --- |
| `Strict-Transport-Security` (HSTS) | 强制 HTTPS |
| `Content-Security-Policy` (CSP) | 防 XSS,限制资源来源 |
| `X-Frame-Options: DENY` | 防点击劫持 |
| `X-Content-Type-Options: nosniff` | 防 MIME 嗅探 |
| `Referrer-Policy` | 控制 Referer 泄漏 |

**真实 IP / 链路追踪**:

| 头 | 用途 |
| --- | --- |
| `X-Forwarded-For: client, proxy1, proxy2` | 代理链 |
| `X-Real-IP: client` | 真实客户端 IP |
| `X-Request-ID` / `Traceparent` | 链路追踪 |
| `Forwarded: for=client;by=proxy` | RFC 7239 标准化版本 |

## 五、HTTP 版本演进

> 深度对比看 [../11-cdn/04-protocol-optimization.md](../11-cdn/04-protocol-optimization.md),这里只做速查。

### 5.1 演进表

| 版本 | 年份 | 核心 | 性能瓶颈 |
| --- | --- | --- | --- |
| **HTTP/0.9** | 1991 | 只有 GET,只能传 HTML | - |
| **HTTP/1.0** | 1996 | 加方法、头、状态码,**每请求一连接** | 连接建立成本高 |
| **HTTP/1.1** | 1997 | **Keep-Alive、Pipelining、Host 头、Chunked** | **队头阻塞** |
| **HTTP/2** | 2015 | 二进制、多路复用、HPACK 头压缩、Server Push | **TCP 队头阻塞仍在** |
| **HTTP/3** | 2022 | 基于 **QUIC/UDP**,解决 TCP HoL,0-RTT,Connection ID | 部署成本 |

### 5.2 三个版本的本质区别

```
HTTP/1.1:  一连接一时刻一请求(Pipelining 实际没用)
           ↓ 同域名要开多 TCP 连接(浏览器 6 个上限)

HTTP/2:    一连接多路复用(Stream),二进制分帧
           ↓ 但底层 TCP 丢一个包,所有 Stream 都卡(TCP HoL)

HTTP/3:    QUIC = UDP + TLS1.3 + 多路复用 + 流独立 ACK
           ↓ 一个 Stream 丢包不影响其他 Stream
           + 0-RTT 重连
           + Connection ID 支持 IP 切换(4G 切 WiFi 不断)
```

### 5.3 各版本握手成本

| 版本 | 建连成本 |
| --- | --- |
| HTTP/1.1 + TLS 1.2 | TCP 3 次 + TLS 2 次 = **3 RTT** |
| HTTP/2 + TLS 1.3 | TCP 3 次 + TLS 1 次 = **2 RTT** |
| HTTP/3 (QUIC) | **0-1 RTT**(首次 1 RTT,重连 0 RTT) |

## 六、实战坑

### 坑 1:GET 带 body 部分服务/代理会丢

规范没禁止,但**强烈不建议**。Elasticsearch 是著名"反例"(GET + body 查询),但要求 server 显式支持。中间件 / CDN 可能丢 body。

修复:查询用 URL 参数,有复杂参数就用 POST。

### 坑 2:DELETE 带 body 同上

部分代理会丢 DELETE 的 body。删除条件放 URL 或改用 POST。

### 坑 3:POST 不幂等导致重复创建

网络抖动 / 客户端重试 → 创建多个订单。

修复:**幂等键**(`Idempotency-Key` 头),Stripe / 支付宝标配。

```http
POST /api/orders
Idempotency-Key: 6f8a3b9e-...
```

服务端用这个 key 去重 24h 内的重复请求。

### 坑 4:301 重定向被浏览器永久缓存改不回

301 浏览器会**永久缓存**,改回原 URL 后用户还是被重定向。

修复:测试期间用 302,确定后再改 301。或让浏览器清缓存(强制刷新不一定够)。

### 坑 5:CORS 预检失败排查不到

OPTIONS 失败 → 真实请求不发 → 后端日志没记录原始请求 → 前端看到"网络错误"。

修复:DevTools Network 面板看 OPTIONS 响应;服务端务必正确处理 OPTIONS。

### 坑 6:`Host` 头错导致 SSRF

应用程序信任 Host 头生成链接 → 攻击者发请求时改 Host → 生成恶意链接(找回密码等)。

修复:服务端**白名单校验** Host,不要无脑信任。

### 坑 7:`X-Forwarded-For` 信任错误

应用直接读取 `X-Forwarded-For` 的第一个 IP 当客户端 IP → 攻击者**可任意伪造**。

修复:从**右边**取信任代理之前的第一个 IP,或用最后一跳代理设的 `X-Real-IP`。

### 坑 8:Cookie 没设 HttpOnly + Secure + SameSite

XSS 偷 Cookie / CSRF 攻击 / 中间人嗅探。

修复:三件套必带:

```http
Set-Cookie: session=xxx; HttpOnly; Secure; SameSite=Lax; Path=/
```

### 坑 9:413 Payload Too Large 上传失败

Nginx 默认 1MB,大文件上传失败。

修复:`client_max_body_size 10m;`(同时调 application 的限制)。

### 坑 10:499 Nginx 私有状态码

Nginx 看到客户端主动断开,记 499。常见于:
- 客户端超时(浏览器关 / 应用主动 cancel)
- 上游处理太慢

修复:看上游 P99,看客户端超时配置。

## 七、Go 中怎么用

```go
import "net/http"

// 方法常量
http.MethodGet      // "GET"
http.MethodPost     // "POST"
http.MethodPut      // "PUT"
http.MethodPatch    // "PATCH"
http.MethodDelete   // "DELETE"
http.MethodHead     // "HEAD"
http.MethodOptions  // "OPTIONS"
http.MethodConnect  // "CONNECT"
http.MethodTrace    // "TRACE"

// 状态码常量
http.StatusOK                  // 200
http.StatusCreated             // 201
http.StatusNoContent           // 204
http.StatusMovedPermanently    // 301
http.StatusFound               // 302
http.StatusNotModified         // 304
http.StatusBadRequest          // 400
http.StatusUnauthorized        // 401
http.StatusForbidden           // 403
http.StatusNotFound            // 404
http.StatusTooManyRequests     // 429
http.StatusInternalServerError // 500
http.StatusBadGateway          // 502
http.StatusServiceUnavailable  // 503
http.StatusGatewayTimeout      // 504

// 发送请求
req, _ := http.NewRequest(http.MethodPut, url, body)
req.Header.Set("Content-Type", "application/json")
req.Header.Set("Idempotency-Key", uuid.New().String())
resp, _ := http.DefaultClient.Do(req)
defer resp.Body.Close()
```

## 八、面试高频

**Q1:HTTP 方法有几种?各自特点?**

9 种(RFC 9110):GET / POST / PUT / PATCH / DELETE / HEAD / OPTIONS / CONNECT / TRACE。
按三大维度区分:**安全**(只读)、**幂等**(多次=一次)、**可缓存**(GET/HEAD)。

**Q2:PUT vs PATCH vs POST 区别?**

- POST 创建子资源(URL 指向集合,服务端分配 ID,不幂等)
- PUT 整体替换(URL 指向资源,要带全字段,幂等)
- PATCH 局部更新(只改指定字段,默认不幂等)

**Q3:幂等是什么?哪些方法幂等?**

同一请求执行 N 次效果 = 1 次。GET / HEAD / OPTIONS / PUT / DELETE 幂等;POST / PATCH 默认不幂等。
DELETE 第一次 200 后续 404,**最终状态相同就算幂等**。
POST 想幂等用 **Idempotency-Key**。

**Q4:GET 和 POST 区别?**

不只是"参数在 URL / Body":
- 安全 / 幂等 / 可缓存 性质不同
- GET 可被浏览器收藏 / 历史记录留痕
- GET 长度受 URL 限制(各浏览器 2KB-8KB)
- POST 可传二进制 / 大数据
- 浏览器后退按钮:GET 直接回放,POST 询问是否重发

**Q5:301 和 302 区别?**

301 永久(浏览器缓存),302 临时(不缓存)。
都会把 POST 改成 GET → 要保留方法用 **307/308**。

**Q6:401 和 403 区别?**

- 401 未认证(没登录)→ 跳登录页
- 403 已认证无权限 → 提示无权限

**Q7:502 和 504 区别?**

- 502 上游返回非法响应(后端崩了 / RST)
- 504 网关等上游超时(后端慢但没崩)

**Q8:OPTIONS 在 CORS 里干什么?**

跨域非简单请求前,浏览器自动发预检 OPTIONS,服务端返回 `Access-Control-Allow-*` 系列头。
通过后才发真实请求,失败则真实请求不发出。

**Q9:HTTP 1.1 → 2 → 3 怎么演进?**

- 1.1 长连接 + 队头阻塞
- 2 二进制 + 多路复用,但 TCP 层仍队头阻塞
- 3 QUIC/UDP 流独立 ACK + 0-RTT + Connection ID

**Q10:HTTP 是无状态的,登录怎么保持?**

无状态意味着每次请求独立。状态靠:
- Cookie + Session(传统)
- Token(JWT,服务端无状态)
- 客户端存储(localStorage + Bearer 头)

## 九、一句话总结

> **HTTP 9 种方法** = GET/POST/PUT/PATCH/DELETE/HEAD/OPTIONS/CONNECT/TRACE,
> 三大维度:**安全**(只读)/ **幂等**(多次=一次)/ **可缓存**(GET/HEAD);
> 重点区分:**PUT 整体替换 vs PATCH 局部更新 vs POST 创建**;
> 状态码记住 **301/302/307/308 区别**、**401/403 区别**、**502/504 区别**;
> 头部三类:**身份认证 / 缓存控制 / CORS 安全**;
> 版本演进核心解决"**队头阻塞**":1.1 应用层 → 2 解决但 TCP 层仍卡 → 3 用 QUIC 彻底解决。
