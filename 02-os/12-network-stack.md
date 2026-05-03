# 网络栈与连接问题

> 后端网络问题常见表现是超时、连接打满、重传、TIME_WAIT/CLOSE_WAIT 异常、DNS 慢、下游抖动。回答时要能把 TCP 状态和线上现象连起来。

## 一、一次请求经过什么

```mermaid
flowchart LR
    App["应用"] --> DNS["DNS 解析"]
    DNS --> Conn["TCP 建连"]
    Conn --> TLS["TLS 握手"]
    TLS --> Send["发送请求"]
    Send --> Server["服务端处理"]
    Server --> Resp["返回响应"]
    Resp --> Pool["连接复用 / 关闭"]
```

延迟可能来自：

- DNS 解析慢。
- TCP 建连慢。
- TLS 握手慢。
- 网络 RTT 高或重传。
- 服务端处理慢。
- 连接池等待。

## 二、TCP 连接状态

常见状态：

| 状态 | 含义 | 线上关注 |
| --- | --- | --- |
| ESTABLISHED | 连接已建立 | 连接数是否异常 |
| TIME_WAIT | 主动关闭方等待旧包消失 | 短连接多会堆积 |
| CLOSE_WAIT | 对端已关闭，本端未 close | 应用连接泄漏 |
| SYN_SENT | 已发起连接，等对端响应 | 下游不可达或网络慢 |
| SYN_RECV | 收到 SYN，等待握手完成 | 半连接队列压力 |

重点：

```text
TIME_WAIT 多：通常是短连接多，不一定是泄漏。
CLOSE_WAIT 多：通常是应用没有正确关闭连接，更危险。
```

## 三、三次握手和四次挥手

三次握手：

```text
Client -> SYN
Server -> SYN+ACK
Client -> ACK
```

四次挥手：

```text
主动方 -> FIN
被动方 -> ACK
被动方 -> FIN
主动方 -> ACK
```

为什么需要 TIME_WAIT：

- 确保最后一个 ACK 能被对端收到。
- 等待旧连接的延迟报文消失。

代价：

- 短连接很多时占用端口和内核资源。
- 客户端高并发短连接可能出现端口耗尽。

## 四、常用排查命令

```text
ss -antp
ss -s
netstat -s
sar -n TCP,ETCP,DEV 1
ping
traceroute
dig
tcpdump
```

看连接状态分布：

```text
ss -ant | awk '{print $1}' | sort | uniq -c
```

看某个端口连接：

```text
ss -antp sport = :8080
ss -antp dport = :3306
```

## 五、典型线上场景

### 场景 1：CLOSE_WAIT 很多

含义：

```text
对端已经关闭连接
本端应用没有 close
```

常见原因：

- HTTP response body 没有关闭。
- 数据库 rows 没有关闭。
- 异常分支遗漏 close。

Go 里典型坑：

```go
resp, err := http.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close()
```

### 场景 2：TIME_WAIT 很多

常见原因：

- 客户端短连接过多。
- 没有开启连接复用。
- HTTP client 每次创建新实例。

处理方向：

- 使用连接池。
- 复用 HTTP client。
- 调整 keepalive。
- 必要时调内核参数，但不要先靠参数掩盖应用问题。

### 场景 3：请求偶发超时

可能原因：

- 网络重传。
- 下游 P99 抖动。
- DNS 解析慢。
- 连接池打满。
- 跨机房调用。

排查：

- trace 拆分 DNS、connect、TLS、server processing。
- 看 TCP 重传。
- 看下游 P99。
- 看连接池等待和超时配置。

## 六、超时配置

线上必须有：

- 连接超时。
- 读超时。
- 写超时。
- 整体请求超时。
- 连接池等待超时。

原则：

```text
超时要分层设置
上游超时 > 下游超时
避免下游已经超时，上游还一直等
```

重试要谨慎：

- 只对幂等操作重试。
- 设置最大重试次数。
- 加退避和抖动。
- 避免雪崩时重试放大流量。

## 七、常见坑

- HTTP client 每次请求都 new，导致连接无法复用。
- 没有关闭 response body，导致连接泄漏。
- 没有超时，慢下游拖死线程和连接池。
- 盲目重试非幂等接口。
- 只看服务端耗时，不看 DNS、建连、TLS、网络 RTT。
- CLOSE_WAIT 多还以为是系统参数问题。

## 八、面试表达

```text
网络问题我会先拆请求链路：DNS、建连、TLS、发送、服务端处理、响应和连接复用。
TIME_WAIT 多通常和短连接有关，CLOSE_WAIT 多更像应用没有关闭连接。
偶发超时要看 TCP 重传、下游 P99、连接池等待和跨机房 RTT。
工程上必须设置连接超时、读写超时、整体超时，并且重试要考虑幂等和退避，避免放大故障。
```
