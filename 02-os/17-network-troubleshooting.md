# 网络故障排查实战

> 网络排查要把一次请求拆开：DNS、建连、TLS、发送、服务端处理、响应、连接复用。不要一上来就说“网络抖动”。

## 一、排查路径

```mermaid
flowchart LR
    Timeout["请求慢 / 超时"] --> DNS["DNS"]
    DNS --> Connect["TCP 建连"]
    Connect --> TLS["TLS"]
    TLS --> Request["请求发送"]
    Request --> Server["服务端处理"]
    Server --> Response["响应返回"]
    Response --> Pool["连接池复用"]
```

常见判断：

- DNS 慢：`dig`、`nslookup`、应用 trace 中 DNS 阶段耗时高。
- 建连慢：`curl -v`、`tcpdump`、`ss` 看 SYN_SENT。
- TLS 慢：证书、握手、CPU、跨地域 RTT。
- 服务端慢：trace 中 server processing 高。
- 网络丢包/重传：`sar -n TCP,ETCP`、`netstat -s`、`tcpdump`。
- 连接池问题：等待连接时间高、CLOSE_WAIT 多、TIME_WAIT 多。

## 二、命令速查

| 命令 | 用途 | 看什么 |
| --- | --- | --- |
| `curl -v` | 拆 HTTP 请求过程 | DNS、connect、TLS、响应码 |
| `curl -w` | 输出耗时分解 | namelookup、connect、starttransfer |
| `dig` | DNS 查询 | 解析结果、TTL、耗时 |
| `ping` | 连通性和 RTT | 延迟、丢包 |
| `mtr` / `traceroute` | 路径追踪 | 哪一跳延迟/丢包 |
| `ss -antp` | TCP 连接 | 状态、端口、进程 |
| `ss -s` | TCP 汇总 | established、timewait |
| `netstat -s` | 协议统计 | 重传、失败、reset |
| `sar -n TCP,ETCP,DEV 1` | 网络指标 | 重传、连接、网卡吞吐 |
| `tcpdump` | 抓包 | SYN、RST、重传、TLS 握手 |
| `lsof -i -p <pid>` | 进程连接 | fd、远端地址、连接泄漏 |

`curl` 耗时拆分示例：

```text
curl -o /dev/null -s -w \
"dns=%{time_namelookup} connect=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total}\n" \
https://example.com
```

含义：

- `time_namelookup`：DNS 耗时。
- `time_connect`：TCP 建连完成时间。
- `time_appconnect`：TLS 握手完成时间。
- `time_starttransfer`：首字节时间。
- `time_total`：总耗时。

## 三、场景 1：接口偶发超时

先问：

```text
是所有接口慢，还是某个下游慢？
是单实例慢，还是全机房慢？
是 P99 慢，还是平均值也慢？
是否刚发布、切流、扩容、改 DNS？
```

排查路径：

1. trace 看慢在 DNS、connect、TLS、server processing 还是 response。
2. `ss -antp` 看连接状态是否异常。
3. `netstat -s` / `sar -n TCP,ETCP 1` 看重传。
4. `curl -w` 从机器上直接访问下游。
5. 对比同机房、跨机房、不同实例。

常见根因：

- 下游 P99 抖动。
- 连接池打满。
- DNS 解析慢或缓存失效。
- TCP 重传。
- 跨机房 RTT 高。
- 上游超时时间大于下游，导致请求堆积。

## 四、场景 2：CLOSE_WAIT 很多

含义：

```text
对端已经关闭连接
本端应用没有 close
```

排查：

```text
ss -antp | grep CLOSE-WAIT
lsof -i -p <pid>
```

Go 常见原因：

- `resp.Body` 没有关闭。
- `rows.Close()` 没有调用。
- 异常分支提前 return。
- 长连接对象生命周期混乱。

修复方向：

- HTTP response body 必须 close。
- DB rows 必须 close。
- 用超时和 context 控制生命周期。
- code review 关注资源释放。

## 五、场景 3：TIME_WAIT 很多

常见原因：

- 短连接太多。
- 没有连接池。
- 每次请求 new HTTP client。
- 服务端主动关闭连接。

判断：

```text
ss -ant | awk '{print $1}' | sort | uniq -c
ss -antp | grep TIME-WAIT | wc -l
```

处理方向：

- 复用连接。
- 使用全局 HTTP client。
- 调整 keepalive。
- 降低短连接请求量。
- 必要时再评估内核参数。

不要一上来就调参数，先看应用是否没有复用连接。

## 六、场景 4：端口耗尽

表现：

- 客户端大量连接下游失败。
- 报 `cannot assign requested address`。
- TIME_WAIT 很多。
- 短时间新建连接太多。

排查：

```text
cat /proc/sys/net/ipv4/ip_local_port_range
ss -ant | wc -l
ss -ant | grep TIME-WAIT | wc -l
```

处理：

- 连接池和长连接。
- 降低并发建连。
- 增加客户端实例。
- 调整本地端口范围和 TIME_WAIT 复用策略。

## 七、场景 5：TCP 重传

现象：

- P99 抖动。
- 请求偶发超时。
- 带宽不一定打满。

排查：

```text
netstat -s | grep -i retrans
sar -n TCP,ETCP 1
tcpdump -i eth0 host <ip> and port <port>
```

可能原因：

- 网络拥塞。
- 跨机房链路质量差。
- 网卡/交换机问题。
- 下游处理慢导致窗口变小。
- 容器/宿主机网络栈压力。

## 八、Go HTTP Client 坑

推荐做法：

```go
client := &http.Client{
    Timeout: 2 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 100,
        IdleConnTimeout:     90 * time.Second,
    },
}
```

常见坑：

- 每次请求创建 `http.Client`。
- 没有设置 `Timeout`。
- 没有关闭 `resp.Body`。
- 没有读取并关闭 body，连接无法复用。
- 重试没有幂等和退避。

## 九、面试表达

```text
网络超时我会先拆链路：DNS、建连、TLS、服务端处理、响应和连接池。
工具上会用 curl -w 看耗时分解，用 dig 看 DNS，用 ss 看连接状态，用 netstat/sar 看重传，用 tcpdump 抓包确认。
如果 CLOSE_WAIT 多，通常是应用没有关闭连接；TIME_WAIT 多通常是短连接或连接复用不足。
处理上要设置分层超时、连接池、keepalive、幂等重试和退避，避免慢下游拖垮上游。
```

