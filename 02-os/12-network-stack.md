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

## 八、字节序(大小端)

> 网络协议是大端,服务端 CPU 几乎都是小端 —— 中间必须做转换。
> 这是底层网络通信最基础也最容易踩的坑。

### 8.1 大小端定义

**字节序**:多字节数据在内存里**字节的排列顺序**。

例:32 位整数 `0x12345678` 存到内存地址 `0x1000` 起:

```
              0x1000  0x1001  0x1002  0x1003
大端 (Big):    12      34      56      78       ← 高位在前
小端 (Little): 78      56      34      12       ← 低位在前
```

| | 大端 Big-Endian | 小端 Little-Endian |
| --- | --- | --- |
| 高位字节 | 放低地址 | 放高地址 |
| 直观性 | 和人读写一致 | 反着的 |
| 运算友好 | 差 | 好(加法从低位开始,顺读) |
| 典型代表 | **网络协议 / Java / PowerPC** | **x86 / ARM64 / RISC-V** |

口诀:**大端像人读数,小端像机器算数**。

### 8.2 网络字节序 = 大端

**RFC 1700 规定:网络协议传输用大端**(Big-Endian),又叫 **Network Byte Order (NBO)**。

涉及字段:
- **IP 头**:源 IP / 目的 IP / 总长度 / TTL
- **TCP 头**:端口号 / 序列号 / 确认号 / 窗口大小
- **UDP 头**:端口号 / 长度

```
TCP 端口 80 (0x0050) 在网线上传输:
   字节1   字节2
    00     50          ← 大端,高位先发
```

**为什么选大端**:
1. 历史路径依赖(早期网络设备 IBM/SUN/Motorola 都是大端)
2. 抓包工具按字节顺序展示就是人类读法,调试方便
3. 早期硬件可以读第一字节就预判包大小

### 8.3 服务端硬件 = 几乎全是小端

| 平台 | 字节序 | 服务端占比 |
| --- | --- | --- |
| x86 / x86_64 (Intel / AMD) | 小端 | 95%+ |
| ARM64 (鲲鹏 / 飞腾 / Graviton / Apple Silicon) | 小端(默认) | 增长中 |
| PowerPC / SPARC | 大端 | 已边缘化 |

**结论**:服务端 CPU **几乎 100% 小端**,网络传输是大端 → 应用层**必须做字节序转换**。

### 8.4 字节序转换 API

**C / Linux**:

```c
#include <arpa/inet.h>

htons()   // host to network short  (16位,如端口号)
htonl()   // host to network long   (32位,如 IP)
ntohs()   // network to host short
ntohl()   // network to host long

uint16_t port = 8080;
uint16_t net_port = htons(port);  // 发送前转大端
send(fd, &net_port, 2, 0);
```

x86 上 `htons` 实际就是字节翻转;大端机上是 no-op。

**Go**(显式指定,不依赖隐式假设):

```go
import "encoding/binary"

// 写入网络(大端)
buf := make([]byte, 4)
binary.BigEndian.PutUint32(buf, 0x12345678)
// buf = [0x12, 0x34, 0x56, 0x78]

// 从网络读取
val := binary.BigEndian.Uint32(buf)

// 本地存储 / 私有协议常用小端(性能略好)
binary.LittleEndian.PutUint32(buf, 0x12345678)
// buf = [0x78, 0x56, 0x34, 0x12]
```

### 8.5 检测当前机器字节序

```bash
# Linux 命令行
lscpu | grep "Byte Order"
# Byte Order: Little Endian
```

```go
// Go
func nativeEndian() binary.ByteOrder {
    var i uint16 = 1
    if *(*byte)(unsafe.Pointer(&i)) == 1 {
        return binary.LittleEndian
    }
    return binary.BigEndian
}
```

### 8.6 生产中的字节序坑

**坑 1:跨语言通信忘转字节序**

```
Go 服务(小端)直接 binary.LittleEndian 编码 → 发给 Java 服务
Java 默认大端解 → 数值全乱
```

修复:私有协议必须**文档显式约定字节序**,推荐用 protobuf / msgpack 等已封装好的序列化框架。

**坑 2:Redis / 数据库存二进制 key 跨架构不一致**

```go
// 小端机
binary.LittleEndian.PutUint64(key, userID)
redis.Set(key, val)
// 跨架构的大端机用同一 userID 编码出来的 key 完全不同 → miss
```

修复:统一用大端,或显式转字符串。

**坑 3:抓包看到的别按本地字节序解**

```
tcpdump 抓到 TCP 端口字段: 1F 90
错解(按小端) = 0x901F = 36895  ❌
正解(网络大端) = 0x1F90 = 8080  ✅
```

**坑 4:C 里结构体直接 send**

```c
struct Header { uint16_t port; uint32_t seq; };
send(fd, &header, sizeof(header), 0);  // ❌ 不同机器解析不一致
// 还有内存对齐 padding 问题
```

修复:每个字段单独 htons/htonl,或用序列化框架。

### 8.7 一句话总结

> **网络字节序 = 大端**(RFC 规定,TCP/IP 协议头都是);**服务端 CPU 几乎都是小端**(x86 / ARM64);
> → **应用层必须做转换**(C 用 `htons/htonl`,Go 用 `binary.BigEndian`);
> 选小端是 CPU 运算友好,选大端是协议互通和可读性。**两边各取所需,中间靠字节序转换桥接**。

## 九、面试表达

```text
网络问题我会先拆请求链路：DNS、建连、TLS、发送、服务端处理、响应和连接复用。
TIME_WAIT 多通常和短连接有关，CLOSE_WAIT 多更像应用没有关闭连接。
偶发超时要看 TCP 重传、下游 P99、连接池等待和跨机房 RTT。
工程上必须设置连接超时、读写超时、整体超时，并且重试要考虑幂等和退避，避免放大故障。
```
