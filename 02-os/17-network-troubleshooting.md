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

## 三、主动检测网络质量

> 前两章解决"出问题后怎么查"，这一章解决"平时怎么主动评估一条链路好不好"。
> 思路是先盯**四大指标**，再用对应工具量化。

### 3.1 四大核心指标

| 指标 | 含义 | 红线参考 | 对业务影响 |
| --- | --- | --- | --- |
| **延迟 RTT** | 包往返时间 | 同城 <5ms / 跨地域 <50ms / 跨国 <200ms | RPC、TLS 握手 |
| **抖动 Jitter** | 延迟标准差 | <10ms 稳 / >30ms 差 | 视频、实时音视频、游戏 |
| **丢包率** | 丢弃包占比 | <0.1% 正常 / >1% 异常 / >5% 严重 | TCP 重传、吞吐崩塌 |
| **带宽 / 吞吐** | 单位时间数据量 | 看链路标称（千兆要测到 940Mbps+）| 大文件、备份、流媒体 |

附加：**MTU**（影响分片）、**PPS**（小包场景看包率而非带宽）、**端口连通性**。

### 3.2 延迟 + 丢包：`ping` / `mtr`

```bash
# 快速看延迟稳不稳
ping -c 100 -i 0.1 8.8.8.8
# 关键字段:
#   rtt min/avg/max/mdev = 0.5/1.2/3.8/0.4 ms
#                                  ^抖动 (mdev)
#   packet loss = 0%

# mtr = ping + traceroute 合体（强烈推荐）
mtr -rwzbc 100 8.8.8.8
#   -r report 模式  -w 宽输出  -z 显示 ASN
#   -b 显示 IP    -c 100 发 100 个包
# 每跳输出: Loss% / Snt / Last / Avg / Best / Wrst / StDev
# 能定位"丢包发生在哪一跳"
```

**mtr 是跨网络排查神器**：能区分丢包是自己出口、运营商骨干，还是对端入口。

### 3.3 路径分析：`traceroute` / `tcptraceroute`

```bash
traceroute 8.8.8.8           # ICMP/UDP 默认，可能被防火墙挡
traceroute -T -p 443 host    # TCP SYN，穿透防火墙更准
tcptraceroute host 443       # 同上，更直观
```

ICMP 通不代表 TCP 通——业务端口必须用 TCP 探测。

### 3.4 带宽 / 吞吐：`iperf3`（业界标准）

```bash
# 服务端
iperf3 -s

# 客户端 TCP（多流压满带宽）
iperf3 -c <server> -t 30 -P 8
#   -t 30 持续 30 秒  -P 8 并发 8 个流

# UDP 测抖动 + 丢包（限速)
iperf3 -c <server> -u -b 100M -t 30
```

典型输出：`[SUM] 0-30 sec  3.3 GBytes  942 Mbits/sec  retr=0` ← 千兆达标。

### 3.5 TCP 层连通 + 时延：`tcping` / `hping3` / `nc`

```bash
# 测端口通 + TCP 握手延迟（最贴近业务）
tcping example.com 443

# hping3 模拟 SYN 探测
hping3 -S -p 443 -c 100 example.com

# 端口连通快速检查
nc -zv example.com 443
```

业务延迟应该看 **TCP 握手时间**，比 ICMP ping 更真实。

### 3.6 网卡层面:`ethtool` / `ip -s link`

```bash
ethtool eth0                          # 协商速率、双工模式
ethtool -S eth0 | grep -E "drop|error|fifo"
# 关键计数:
#   rx_dropped / tx_dropped: 网卡丢包
#   rx_crc_errors:          物理层错误
#   rx_fifo_errors:          ring buffer 满

ip -s link show eth0                  # 收发包/错误/丢包计数
```

网卡有 errors / drops → 物理层、驱动或 ring buffer 不足。

### 3.7 长期监控:`smokeping` / 自建 mtr

```bash
# 简易后台周期 mtr
while true; do
  mtr -rwzbc 60 target >> mtr.log
  sleep 60
done

# 生产推荐: smokeping 图形化记录丢包+延迟历史
# 或 prometheus blackbox_exporter + grafana
```

### 3.8 一键体检脚本

```bash
TARGET=${1:-8.8.8.8}

echo "=== 1. 连通性 + 延迟 ==="
ping -c 20 -i 0.1 $TARGET | tail -3

echo "=== 2. 路径 + 丢包 ==="
mtr -rwzbc 50 $TARGET

echo "=== 3. DNS 解析时延 ==="
dig $TARGET +stats | grep "Query time"

echo "=== 4. TCP 握手 + 首字节 ==="
for i in 1 2 3; do
  curl -o /dev/null -s -w \
    "握手:%{time_connect}s 首字节:%{time_starttransfer}s 总耗时:%{time_total}s\n" \
    https://$TARGET
done

echo "=== 5. 网卡错误统计 ==="
ip -s link show eth0 | grep -A1 "RX:\|TX:"
ethtool -S eth0 2>/dev/null | grep -E "drop|error" | grep -v ": 0$"
```

### 3.9 按指标反查工具

| 想知道什么 | 第一手工具 | 次选 |
| --- | --- | --- |
| 通不通 | `ping` / `nc -zv` | `tcping` |
| 延迟多少 | `ping` / `tcping` | `curl -w time_connect` |
| 抖动大不大 | `ping`(看 mdev) / `mtr`(StDev) | `smokeping` 长跑 |
| 丢包多少 | `mtr -rwzbc 1000` | `ping -f`(root,洪 ping) |
| 路径绕不绕 | `mtr` / `traceroute` | `tcptraceroute` |
| 带宽够不够 | `iperf3 -P 8` | `nload` / `iftop` 实时看 |
| PPS 多少 | `iperf3 -l 64 -P 16` | `sar -n DEV 1` |
| TCP 重传率 | `ss -ti` / `netstat -s` | `tcpdump` 抓包 |
| 网卡有没有错 | `ethtool -S` | `ip -s link` |

### 3.10 主动检测的思维链

```
1. ping       通不通 + 基础延迟 + 抖动
   ↓ 不通
2. mtr        哪一跳开始丢
   ↓ 通但慢
3. iperf3     是带宽问题还是延迟问题
   ↓ 带宽够
4. ss -ti     TCP 层（cwnd / rtt / retrans）
   ↓ TCP 正常
5. ethtool    网卡 / 驱动 / 物理层
   ↓ 都正常
6. 应用层    DNS / TLS / 业务逻辑
```

### 3.11 关键认知

- **ping 通 ≠ 业务通**：ICMP 通但 TCP 端口可能被 ACL 挡 → 用 `tcping`
- **延迟低 ≠ 质量好**：要看**抖动**（mdev / StDev），抖动大对 RPC / 视频致命
- **带宽达标 ≠ 业务快**：小包场景看 PPS 不看 bps，TCP 拥塞窗口才是真瓶颈
- **mtr 看"路径"，iperf3 测"管道粗细"**，互补不替代
- **单次测不算数**：网络质量看长期分布（P50 / P95 / P99 + 抖动），要持续监控

## 四、场景 1:接口偶发超时

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

## 五、场景 2：CLOSE_WAIT 很多

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

## 六、场景 3：TIME_WAIT 很多

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

## 七、场景 4:端口耗尽

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

## 八、场景 5:TCP 重传

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

## 九、Go HTTP Client 坑

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

## 十、面试表达

```text
网络超时我会先拆链路：DNS、建连、TLS、服务端处理、响应和连接池。
工具上会用 curl -w 看耗时分解，用 dig 看 DNS，用 ss 看连接状态，用 netstat/sar 看重传，用 tcpdump 抓包确认。
如果 CLOSE_WAIT 多，通常是应用没有关闭连接；TIME_WAIT 多通常是短连接或连接复用不足。
处理上要设置分层超时、连接池、keepalive、幂等重试和退避，避免慢下游拖垮上游。
```

主动评估网络质量（接入新机房 / 选 CDN 节点 / 上线前）会盯四大指标：

```text
延迟 / 抖动 / 丢包 / 带宽
- ping/mtr 看延迟+抖动+丢包，mtr 还能定位哪一跳出问题
- iperf3 测带宽（业界标准，多流压满）
- tcping 测业务端口 TCP 握手延迟，比 ICMP ping 更真实
- ethtool -S 看网卡丢包/错误（物理层）
- 长期监控用 smokeping 或 prometheus blackbox_exporter
关键认知: ping 通不等于业务通，延迟低不等于质量好（要看抖动），单点测不算数（看长期 P99）。
```

