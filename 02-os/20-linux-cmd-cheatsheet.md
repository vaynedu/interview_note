# Linux 排查命令速查（CPU / 内存 / IO / 网络 / 进程）

> **场景排查在哪查**：[09-linux-troubleshooting.md](09-linux-troubleshooting.md) / [15-cpu-memory-troubleshooting.md](15-cpu-memory-troubleshooting.md) / [17-network-troubleshooting.md](17-network-troubleshooting.md) / [18-disk-io-troubleshooting.md](18-disk-io-troubleshooting.md)
>
> **本文定位**：按"命令"组织的速查表——每个命令"做什么 / 关键参数 / 输出字段解读 / 典型场景"，一页扫完。

---

## 〇、超级速查表（一页全看完）

### 〇.1 USE 方法（Brendan Gregg）

> 排查任何性能问题先问 3 个问题：**Utilization（利用率高吗）/ Saturation（饱和了吗，排队了吗）/ Errors（有错误吗）**。

| 资源 | Utilization | Saturation | Errors |
| --- | --- | --- | --- |
| **CPU** | `top` / `mpstat` 看 %usr+%sys | `top` 看 load average / `vmstat r` | `dmesg` 看 MCE |
| **内存** | `free -h` / `vmstat` | swap 用量 / `vmstat si so` | `dmesg` 看 OOM |
| **磁盘** | `iostat -x` 看 %util | `iostat` 看 await / avgqu-sz | `dmesg` / `smartctl` |
| **网络** | `sar -n DEV` / `iftop` | `ss -i` / `netstat -s` retrans | `ifconfig` errors / drops |

### 〇.2 一句话命令地图

| 维度 | 一句话命令链 |
| --- | --- |
| **CPU** | `top` → 找进程 → `top -Hp PID` → 找线程 → `perf top -p PID` → 看热点 |
| **内存** | `free -h` → `vmstat` → `ps aux --sort -rss` → `pmap PID` → 看分布 |
| **磁盘** | `iostat -xz 1` → `iotop -oP` → `pidstat -d 1` → `lsof -p PID` |
| **网络** | `ss -s` → `ss -ant` 看连接 → `netstat -s` 看错误 → `tcpdump` 抓包 |
| **进程** | `ps aux` → `pstack PID` 看栈 → `lsof -p PID` 看 fd → `strace -p PID` 看 syscall |
| **内核** | `dmesg -T` → `journalctl -k` → `/var/log/messages` |

### 〇.3 高频命令优先级（背完这 15 个够用 80%）

```
日常 5 件套（必背）:
  top      看进程 CPU / 内存全貌
  free     看内存
  vmstat   看 CPU / 内存 / IO 整体
  iostat   看磁盘
  ss       看网络连接（替代 netstat）

排查 5 件套:
  pidstat  按进程看 CPU / 内存 / IO / 网络
  pstack   看 C/C++ 线程栈
  lsof     看打开的文件 / 网络连接
  strace   看 syscall
  dmesg    看内核日志（OOM / 段错误）

性能剖析 5 件套:
  perf top         实时热点函数
  perf record/report  采样 + 离线分析
  火焰图           可视化
  sar              历史数据回溯
  tcpdump          抓包
```

---

## 一、CPU 排查命令

### 1.1 top / htop — CPU 总览

```bash
top                    # 进入交互
top -Hp PID            # 看进程内每个线程的 CPU
top -bn1               # 一次性输出（适合脚本）
top -d 1               # 1 秒刷新一次

按键:
  P    按 CPU 排序
  M    按内存排序
  c    显示完整命令
  1    展开每核 CPU
  t    切换 CPU 显示模式
```

**输出字段精解**：

```
load average: 1.50, 2.10, 3.00
              1min  5min  15min  ← 三个时间窗口的平均负载

含义:
  负载 < 核数 × 0.7   健康
  负载 = 核数           饱和
  负载 > 核数 × 2       严重过载

%Cpu(s):
  us  用户态 CPU（业务代码）
  sy  内核态 CPU（系统调用 / 内核处理）
  ni  nice 优先级
  id  空闲
  wa  IO wait（等磁盘 / 网络）   ← 高 = IO 瓶颈
  hi  硬中断
  si  软中断                     ← 高 = 网络包多
  st  虚拟化偷走的 CPU（云主机）  ← 高 = 邻居吵
```

### 1.2 mpstat — 每核 CPU 单独看

```bash
mpstat -P ALL 1        # 每核 1 秒输出一次

典型异常:
  某核 100% 其他空闲 → 单线程瓶颈 / 中断绑核问题
  所有核均匀高    → 业务热点 / 真实负载
```

### 1.3 pidstat — 按进程看资源

```bash
pidstat 1                  # 所有进程 CPU
pidstat -u 1 -p PID        # 单进程 CPU 详情
pidstat -d 1               # 磁盘 IO
pidstat -r 1               # 内存
pidstat -w 1               # 上下文切换
pidstat -t -p PID 1        # 看线程级别（-t）
```

**关键场景**：
- 进程 CPU 飙高 + 上下文切换高 → 锁竞争 / IO 阻塞
- voluntary（主动）切换高 → 等 IO
- non-voluntary（被动）切换高 → CPU 抢占（线程过多）

### 1.4 vmstat — 系统级综合

```bash
vmstat 1                # 每秒输出
vmstat 1 10             # 输出 10 次
```

**关键字段**：

```
procs:
  r    运行 / 等运行的进程数      → r > 核数 = CPU 不够
  b    阻塞在 IO 的进程数         → b 高 = IO 瓶颈

memory:
  swpd 已用 swap                  → > 0 且增长 = 内存压力
  free 空闲内存
  buff buffer cache
  cache page cache

swap:
  si   每秒从 swap 换入            → > 0 = 内存紧张
  so   每秒换出到 swap

io:
  bi   每秒块设备读
  bo   每秒块设备写

cpu:
  us / sy / id / wa  同 top
```

### 1.5 perf top — 实时热点函数

```bash
perf top                   # 系统全局
perf top -p PID            # 指定进程
perf top -g                # 含调用栈
```

**输出**：

```
Overhead  Command  Shared Object       Symbol
  15.20%  myapp    myapp               [.] hot_function    ← CPU 热点
  10.30%  myapp    libc-2.31.so        [.] memcpy
   8.50%  myapp    [kernel]            [k] do_syscall_64
```

### 1.6 看线程栈

```bash
# Go: 发 SIGQUIT 信号让 runtime 打印所有 goroutine 栈
kill -SIGQUIT PID

# Java: jstack
jstack PID

# C/C++: pstack 或 gdb
pstack PID

# 自动监控（飙 CPU 时自动 dump 栈）
while true; do
  cpu=$(ps -p PID -o %cpu=)
  if [ ${cpu%.*} -gt 80 ]; then
    pstack PID > /tmp/stack-$(date +%s).txt
  fi
  sleep 5
done
```

---

## 二、内存排查命令

### 2.1 free — 看内存总量

```bash
free -h        # 人类可读
free -m        # MB
free -s 1      # 1 秒刷新
```

**输出**：

```
              total   used   free   shared  buff/cache  available
Mem:           16G   8.0G   1.0G    100M       7.0G        7.5G
Swap:           4G   0.5G   3.5G

字段精解:
  total       物理内存总量
  used        已用（不含 buffer/cache）
  free        完全空闲
  shared      共享内存（tmpfs）
  buff/cache  缓冲 + 缓存（系统可回收）
  available   ★ 真正可用（free + 可回收的 buff/cache）

陷阱:
  ✗ used 高不一定有问题（cache 也算 used 在某些版本）
  ✓ 看 available（应用真正能拿到的）
  ✓ Linux 设计哲学: "free memory is wasted memory"
    → 缓存满了是正常的，别慌
```

### 2.2 /proc/meminfo — 精细内存分布

```bash
cat /proc/meminfo

关键字段:
  MemTotal       总
  MemFree        完全空闲
  MemAvailable   可用 ★
  Buffers        块设备缓冲
  Cached         文件 page cache
  SwapCached     交换缓存
  Active         活跃使用中
  Inactive       不活跃（可换出）
  AnonPages      匿名页（堆 / 栈 / mmap 私有）
  PageTables     页表大小
  Slab           内核 slab 分配器
  HugePages_Total  大页数量
```

### 2.3 pmap — 看进程内存分布

```bash
pmap -x PID                    # 看进程内存映射
pmap -x PID | sort -k3 -n      # 按 RSS 排序

输出:
  Address           Kbytes     RSS   Dirty Mode  Mapping
  00007f1234560000   65536   65500     500 rw---  [ anon ]   ← 堆 / 匿名映射
  00007f1234670000    1024     500     100 r-x--  libc.so    ← 代码段
  00007f1234680000    8192    8000      50 rw---  [ stack ]  ← 栈

排查思路:
  - 看是哪段地址增长（堆 / 共享库 / mmap 文件）
  - 堆增长不停 → 内存泄漏
  - mmap 文件多 → 大文件 IO
```

### 2.4 ps 按内存排序

```bash
ps aux --sort -rss | head -20      # RSS 物理内存
ps aux --sort -vsz | head -20      # VSZ 虚拟内存

字段:
  VSZ   虚拟内存大小（含 mmap 但没真用的）
  RSS   实际占用物理内存 ★ 真正关心的
  %MEM  RSS / 总内存
```

### 2.5 dmesg 看 OOM Kill

```bash
dmesg -T | grep -i "killed process"   # OOM 杀过的进程
dmesg -T | grep -i oom                # 所有 OOM 相关

典型输出:
  [Thu May 11 10:00:00 2026] Out of memory: Killed process 1234 (myapp) total-vm:8GB, anon-rss:6GB

含义:
  系统内存不足 → 内核 OOM Killer 选了 myapp 杀掉
  total-vm: 虚拟内存
  anon-rss: 实际匿名内存（堆 + 栈）

排查:
  - 看进程是否泄漏（pmap / Go pprof / Java jmap）
  - 看是否有内存限制（cgroup）
  - 看是否 mmap 大文件
```

### 2.6 sar -r 历史内存

```bash
sar -r 1 5                  # 每秒采样 5 次
sar -r -s 09:00 -e 10:00    # 看 9-10 点历史

历史数据来自: /var/log/sa/saXX
默认 10 分钟采样一次（cron 配置）
```

---

## 三、磁盘 IO 排查命令

### 3.1 iostat — 看磁盘整体

```bash
iostat -xz 1          # -x 详细 / -z 跳过空闲设备
iostat -dxm 1         # MB 单位

输出:
  Device   r/s   w/s    rkB/s   wkB/s   avgrq-sz  avgqu-sz   await  r_await  w_await  svctm   %util
  sda     100    50    4000     2000      80        2.0      20.5    15.2     30.8     5.0    85.0

字段精解（必背）:
  r/s w/s        每秒读 / 写次数（IOPS）
  rkB/s wkB/s    每秒读 / 写 KB（吞吐）
  avgrq-sz       平均请求大小（扇区）
  avgqu-sz       平均队列深度 ★ > 1 = 排队
  await          平均 IO 等待时间（ms）★ > 10ms = 慢
  svctm          平均服务时间
  %util          ★ 磁盘繁忙程度（> 80% = 接近饱和）

诊断:
  %util > 80% + await > 20ms        → 磁盘瓶颈
  %util 高但 await 低                → IOPS 高但每个请求快（正常）
  await 高但 %util 不高              → 单次 IO 慢（可能是大请求）
```

### 3.2 iotop — 按进程看 IO

```bash
iotop                    # 交互界面
iotop -oP                # 只看有 IO 的进程
iotop -bn1               # 一次性输出（脚本用）

字段:
  TID         线程 ID
  PRIO        IO 优先级
  USER
  DISK READ   读速率
  DISK WRITE  写速率
  SWAPIN      swap 换入
  IO          IO wait %
  COMMAND
```

### 3.3 pidstat -d — 按进程看 IO

```bash
pidstat -d 1                 # 所有进程
pidstat -d -p PID 1          # 单进程

输出:
  Time   UID  PID  kB_rd/s  kB_wr/s  kB_ccwr/s  iodelay  Command
   ...   0   1234   200      500       0          15      myapp

kB_rd/s    读速率
kB_wr/s    写速率
iodelay    ★ IO 等待时间（高 = IO 慢拖累进程）
```

### 3.4 lsof — 看打开的文件

```bash
lsof -p PID                  # 看进程打开的所有 fd
lsof /var/log/app.log        # 谁打开了这个文件
lsof -i :8080                # 谁监听 8080 端口
lsof -i tcp -nP              # 所有 TCP 连接

排查典型问题:
  - too many open files: lsof -p PID | wc -l 看是否超过 ulimit -n
  - 文件已删但磁盘空间没释放: lsof | grep deleted（进程仍持有 fd）
```

### 3.5 df / du — 容量

```bash
df -h                  # 各分区使用率
df -i                  # inode 使用率（小文件多时关键）

du -sh *               # 当前目录每项大小
du -h --max-depth=1    # 一层
du -sh /var/log/* | sort -h    # 排序

典型问题:
  df 显示满 + du 加起来没那么多 → 删除文件但被进程持有（lsof grep deleted）
  inode 满（文件多）: 即使空间够也写不了
```

### 3.6 fio — 压测磁盘

```bash
# 随机读 IOPS
fio --name=randread --rw=randread --bs=4k --numjobs=4 \
    --size=1G --runtime=60 --time_based --filename=/data/test

# 顺序写吞吐
fio --name=seqwrite --rw=write --bs=1M --numjobs=1 \
    --size=10G --filename=/data/test

# 知道磁盘上限后才能判断"现在慢是磁盘问题还是业务问题"
```

---

## 四、网络排查命令

### 4.1 ss — 看连接（替代 netstat，更快）

```bash
ss -s                    # 总览统计
ss -ant                  # 所有 TCP 连接（n=不解析名 t=TCP）
ss -antp                 # 含进程 PID（要 root）
ss -ant state established | wc -l   # ESTABLISHED 数量
ss -ant state time-wait | wc -l     # TIME_WAIT 数量
ss -ant '( sport = :8080 )'         # 看 8080 端口
ss -i                    # 含 TCP info（rtt / cwnd）

输出:
  State        Recv-Q  Send-Q  Local Address:Port    Peer Address:Port
  ESTAB        0       0       10.0.0.1:8080         10.0.0.2:54321

Recv-Q  接收队列堆积   → > 0 = 应用消费慢
Send-Q  发送队列堆积   → > 0 = 网络发不出去 / 对端不接收
```

### 4.2 netstat -s — 协议错误统计

```bash
netstat -s | grep -i retrans    # 重传
netstat -s | grep -i drop       # 丢包
netstat -s | grep -i overflow   # 队列溢出

关键指标:
  segments retransmitted    重传段数（> 1% 总段数 = 问题）
  packets dropped by netfilter  防火墙丢的
  ListenOverflows           listen 队列溢出（accept 慢）
  ListenDrops               listen 丢包
```

### 4.3 tcpdump — 抓包

```bash
# 基础
tcpdump -i eth0 -nn port 8080            # 抓 8080 端口
tcpdump -i eth0 host 10.0.0.1            # 抓某 IP
tcpdump -i eth0 tcp and port 80          # TCP 80

# 保存 + 分析
tcpdump -i eth0 -w /tmp/cap.pcap port 8080
# 用 Wireshark 打开 cap.pcap 分析

# 高级
tcpdump -i eth0 -X port 8080             # 显示包内容（含 ASCII）
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0'  # 只抓 SYN / FIN

参数:
  -i 网卡（any 抓所有）
  -nn 不解析 IP 和端口
  -c N 抓 N 个包就停
  -v / -vv / -vvv 详细程度
  -s 0 抓完整包（默认截断 68 字节）
```

### 4.4 sar -n — 网络历史

```bash
sar -n DEV 1                 # 网卡流量
sar -n TCP 1                 # TCP 连接
sar -n ETCP 1                # TCP 错误（重传等）
sar -n SOCK 1                # socket 总数

每秒采样 + 历史回溯（重要）
```

### 4.5 nstat / ip — 现代替代品

```bash
nstat                        # netstat -s 的现代版
nstat -a                     # 所有计数器
nstat -z                     # 显示 0 值
nstat -r                     # reset 计数器

ip a                         # 看 IP
ip -s link                   # 看网卡统计（含错误 / 丢包）
ip route                     # 路由表
```

### 4.6 iftop / nload — 实时流量

```bash
iftop -i eth0                # 按 IP 看流量（哪个 IP 在大量传输）
nload eth0                   # 网卡入 / 出速率

# 适合定位"哪个连接吃带宽"
```

---

## 五、进程 / 线程命令

### 5.1 ps — 看进程

```bash
ps aux                            # 所有进程
ps -ef                            # 同上不同格式
ps -eLf                           # 含线程
ps -ef --forest                   # 树形显示
ps -p PID -o pid,ppid,cmd,%cpu,%mem  # 自定义列
ps -L -p PID                      # 看进程的所有线程

按内存排:    ps aux --sort -rss
按 CPU 排:   ps aux --sort -%cpu

字段 STAT:
  R    运行
  S    可中断睡眠（等事件）
  D    不可中断睡眠（等 IO）  ← 多 = IO 卡
  Z    僵尸进程               ← 父进程没回收
  T    停止
  +    前台进程
  s    会话首领
  l    多线程
```

### 5.2 pstack — 看线程栈（C/C++）

```bash
pstack PID         # 一次性 dump 所有线程栈

适用:
  - C/C++ 应用卡死，看每个线程在干啥
  - 死锁分析（看线程都阻塞在哪个 mutex）
  - 偶发性能问题（高 CPU 时多次 pstack 找规律）

Go 等价: kill -SIGQUIT PID 让 runtime 打印
Java 等价: jstack PID
```

### 5.3 pstree — 进程树

```bash
pstree -p PID          # 看进程关系
pstree -alpsh PID      # 详细信息

适用: 看父子关系（被谁起的 / 起了哪些子进程）
```

### 5.4 strace — 跟踪 syscall

```bash
strace -p PID                     # 跟踪运行中进程
strace -c -p PID                  # 统计各 syscall 次数 + 耗时
strace -e trace=open,read,write -p PID    # 只看特定 syscall
strace -f -p PID                  # 跟踪子进程 / 子线程
strace -tt -p PID                 # 带时间戳

典型场景:
  - 进程卡死: strace -p PID 看卡在哪个 syscall（futex 锁 / read 文件 / accept）
  - 性能慢: strace -c -p PID 看哪个 syscall 占比高
  - 行为分析: 看进程读写哪些文件 / 连哪些 IP

注意:
  strace 会让进程慢 5-50 倍，生产慎用
  → 用 perf trace 替代（开销小）
```

### 5.5 ltrace — 跟踪库函数调用

```bash
ltrace -p PID         # 看调用了哪些 libc 函数

适用: strace 看不到的（不是 syscall 的）
```

---

## 六、性能剖析（perf + 火焰图）

### 6.1 perf — Linux 性能分析瑞士军刀

```bash
# 实时热点
perf top                          # 系统全局
perf top -p PID                   # 指定进程
perf top -g                       # 含调用栈

# 采样 + 离线分析
perf record -F 99 -p PID -g -- sleep 30    # 99Hz 采样 30 秒
perf report                       # 看报告

# 统计
perf stat -p PID -- sleep 10      # 看 10 秒统计
perf stat -e cycles,instructions,cache-misses,branch-misses -p PID -- sleep 10

# 追踪（替代 strace，开销小）
perf trace -p PID

# 跟踪特定事件
perf record -e block:block_rq_issue -a    # 跟踪块 IO 请求
perf record -e syscalls:* -p PID          # 跟踪所有 syscall
```

**关键事件**：

```
cycles            CPU 周期
instructions      指令数
cache-misses      cache 未命中
branch-misses     分支预测失败
context-switches  上下文切换
page-faults       缺页
LLC-load-misses   最后一级 cache miss
```

### 6.2 火焰图（Flame Graph）

```bash
# 采样
perf record -F 99 -p PID -g -- sleep 30

# 生成火焰图
git clone https://github.com/brendangregg/FlameGraph
perf script | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl > flame.svg

# 看 flame.svg:
#   - X 轴: 各函数样本数比例（宽 = CPU 占用高）
#   - Y 轴: 调用栈深度（顶部 = 真正在跑）
#   - 看顶部"平台"（宽且高 = 热点函数）
```

### 6.3 sar — 历史数据

```bash
sar -u 1 5            # CPU
sar -r 1 5            # 内存
sar -d 1 5            # 磁盘
sar -n DEV 1 5        # 网络
sar -q 1 5            # 队列 / 负载

历史回溯（凌晨故障次日排查必备）:
  sar -u -s 03:00:00 -e 04:00:00     # 看凌晨 3-4 点
  
默认 10 分钟采样一次，存 /var/log/sa/
```

---

## 七、内核与日志

### 7.1 dmesg — 内核日志

```bash
dmesg -T              # 带时间戳
dmesg -w              # 实时
dmesg -l err          # 只看 error 级别
dmesg | grep -i oom   # OOM 相关
dmesg | tail -50

关键内容:
  - OOM kill
  - 段错误 / 内核 panic
  - 硬件错误（MCE / 磁盘 IO error）
  - 网络丢包（neighbour table overflow）
  - cgroup OOM
```

### 7.2 journalctl — systemd 日志

```bash
journalctl -u myservice                # 看某服务
journalctl -k                          # 内核日志（同 dmesg）
journalctl --since "1 hour ago"
journalctl -f -u myservice             # 实时跟踪
journalctl -p err                      # 只看错误
```

### 7.3 sysctl — 内核参数

```bash
sysctl -a                         # 所有参数
sysctl net.ipv4.tcp_max_syn_backlog
sysctl -w net.ipv4.tcp_tw_reuse=1 # 临时改
# 持久化: 写 /etc/sysctl.conf

常用调优参数:
  net.core.somaxconn              listen 队列上限
  net.ipv4.tcp_max_syn_backlog    SYN 队列
  net.ipv4.tcp_tw_reuse           TIME_WAIT 复用
  net.ipv4.ip_local_port_range    本地端口范围
  vm.swappiness                   swap 倾向（0-100，DB 设 1）
  vm.overcommit_memory            内存超分策略
  fs.file-max                     系统文件句柄上限
```

---

## 八、容器场景（cgroup / Docker / K8s）

### 8.1 docker / kubectl 基础

```bash
docker stats                       # 容器资源
docker top CONTAINER               # 容器内进程
docker exec -it CONTAINER bash     # 进容器
docker logs -f CONTAINER           # 看日志

kubectl top pod                    # Pod 资源
kubectl top node                   # 节点资源
kubectl exec -it POD -- bash
kubectl logs -f POD
kubectl describe pod POD           # 看事件（OOM Kill 等）
```

### 8.2 cgroup 文件

```bash
# v1
cat /sys/fs/cgroup/cpu/docker/CONTAINER_ID/cpu.cfs_quota_us
cat /sys/fs/cgroup/memory/docker/CONTAINER_ID/memory.usage_in_bytes
cat /sys/fs/cgroup/memory/docker/CONTAINER_ID/memory.limit_in_bytes

# v2（cgroup v2）
cat /sys/fs/cgroup/myslice/cpu.max
cat /sys/fs/cgroup/myslice/memory.current

容器内陷阱:
  - top 看到的 CPU / 内存可能是宿主机的（旧版 procfs）
  - 容器内 ulimit 可能和宿主机不同
  - 看 /sys/fs/cgroup/ 才准确
```

### 8.3 nsenter — 进入容器命名空间

```bash
PID=$(docker inspect -f '{{.State.Pid}}' CONTAINER)
nsenter -t $PID -n -p ss -ant      # 容器内看连接（不进容器）
nsenter -t $PID -m ls /            # 看容器文件系统
```

---

## 九、组合套路（按问题反查）

### 套路 1：CPU 高但 top 看不出哪个进程

```bash
# 1. 看是不是软中断 / 系统调用
top                            # 看 %si / %sy 是否高
mpstat -P ALL 1                # 哪个核高

# 2. 软中断高 → 看网络
cat /proc/softirqs             # 看是哪类软中断
sar -n DEV 1                   # 网卡流量
ethtool -S eth0                # 网卡错误

# 3. sys CPU 高 → 看 syscall
perf top -e syscalls:* -ag

# 4. 用户态高 → 看进程线程
top -Hp PID                    # 看线程
perf top -p PID -g             # 看热点函数
```

### 套路 2：服务卡死 / 接口超时但 CPU 不高

```bash
# 1. 看进程状态
ps aux | grep myapp            # STAT 是不是 D（IO 等待）

# 2. 看 IO
iostat -xz 1                   # %util / await
iotop -oP                      # 哪个进程在 IO

# 3. 看是不是网络
ss -s                          # TIME_WAIT / 连接数
ss -ant state syn-recv         # 半连接堆积？
netstat -s | grep -i overflow  # listen 队列溢出？

# 4. 看是不是锁
pstack PID | grep -c futex     # 多少线程在等锁
strace -p PID                  # 看 syscall

# 5. Go 应用看 goroutine
kill -SIGQUIT PID              # 打印所有 goroutine 栈
curl http://localhost:6060/debug/pprof/goroutine?debug=2
```

### 套路 3：服务突然 OOM

```bash
# 1. 看内核日志确认
dmesg -T | grep -i "killed process"
journalctl -k --since "10 minutes ago" | grep -i oom

# 2. 看是不是被 cgroup 杀的
kubectl describe pod POD | grep -i oom

# 3. 看内存分布（拿到 core dump / 历史数据）
sar -r -s 09:00 -e 10:00       # 看历史
ps aux --sort -rss | head      # 当前

# 4. 应用层定位（业务进程还活着的时候）
pmap -x PID | sort -k3 -n      # 看哪段地址增长
# Go: curl http://localhost:6060/debug/pprof/heap
# Java: jmap -dump:format=b,file=heap.bin PID
```

### 套路 4：磁盘正常但 IO 慢

```bash
# 1. iostat 看磁盘
iostat -xz 1                   # %util / await / svctm

# 2. await 高但 %util 不高 → 单次 IO 大
iotop -oP                      # 看哪个进程在大 IO
pidstat -d 1                   # 同上但持续

# 3. 看是不是 sync / fsync 频繁
strace -e trace=fsync,sync -p PID -c

# 4. 看 inode / 文件碎片
df -i                          # inode 满？
xfs_db / debugfs               # 文件系统层面
```

### 套路 5：网络通但有时丢包

```bash
# 1. 网卡层
ip -s link show eth0           # 错误 / 丢包
ethtool -S eth0 | grep -i error

# 2. 协议层
netstat -s | grep -i retrans
netstat -s | grep -i overflow
nstat -z

# 3. 防火墙 / iptables
iptables -L -n -v              # 看是否有规则丢包
dmesg | grep -i netfilter

# 4. 抓包看 TCP 状态
tcpdump -i eth0 -w /tmp/cap.pcap host TARGET_IP
# Wireshark 分析: Retransmission / Duplicate ACK / Out-of-Order
```

### 套路 6：服务起不来 / 端口被占用

```bash
# 1. 看端口
ss -antp | grep :8080
lsof -i :8080
netstat -antp | grep :8080

# 2. 看 fd 是否泄漏（too many open files）
ls /proc/PID/fd | wc -l
ulimit -n
cat /proc/PID/limits | grep "open files"

# 3. 看 inode 是否满
df -i

# 4. 看 SELinux / AppArmor
getenforce
sestatus
```

### 套路 7：定时任务 / 凌晨故障次日排查

```bash
# 1. 历史 CPU / 内存
sar -u -s 03:00 -e 04:00
sar -r -s 03:00 -e 04:00

# 2. 历史磁盘 / 网络
sar -d -s 03:00 -e 04:00
sar -n DEV -s 03:00 -e 04:00

# 3. 应用日志
journalctl -u myservice --since "2026-05-11 03:00" --until "2026-05-11 04:00"
grep "2026-05-11 03" /var/log/myapp/*.log

# 4. 内核日志
journalctl -k --since "2026-05-11 03:00" --until "2026-05-11 04:00"
```

---

## 十、关键字段速记（必背）

### 十.1 load average

```
load = R + D 状态的进程数
  R: 正在运行
  D: 不可中断睡眠（多为等 IO）

健康线:
  负载 < 核数 × 0.7   完全健康
  负载 = 核数         接近饱和
  负载 = 核数 × 2     严重过载
  负载 = 核数 × 5+    救命级别
```

### 十.2 iostat 字段精解

```
%util:
  磁盘繁忙时间比例
  > 80% = 接近饱和
  现代 NVMe / SSD 可能 100% 但仍不慢（队列深度大）
  → 看 await 更准

await:
  平均 IO 等待时间（含队列 + 服务时间）
  HDD: 健康 < 20ms
  SSD: 健康 < 5ms
  NVMe: 健康 < 1ms

svctm:
  平均服务时间（已废弃，新内核不准）
  → 别看

avgqu-sz:
  平均队列深度
  > 1 = 排队
  > 10 = 严重排队
```

### 十.3 TIME_WAIT 大量

```
ss -ant state time-wait | wc -l

正常:
  作为客户端主动关闭一定有 TIME_WAIT（60s）
  几千个属于正常

异常（> 几万）:
  ✓ 服务端不应该有大量 TIME_WAIT
    → 检查是不是主动关闭了不该关的（短连接业务）
  ✓ 客户端 TIME_WAIT 多
    → 用连接池 / Keep-Alive
    → net.ipv4.tcp_tw_reuse=1（仅客户端有效）

注意:
  tcp_tw_recycle 已被废弃（Linux 4.12+ 移除，会导致 NAT 问题）
```

### 十.4 CLOSE_WAIT 大量

```
含义: 对端关了连接，本端还没关
  → 应用 bug（没调 close）

排查:
  ss -antp state close-wait | head
  lsof -p PID | grep CLOSE_WAIT
  
  → 一定是代码问题，不是系统问题
```

---

## 十一、常用 ulimit / 内核参数（生产调优）

```bash
# 文件句柄
ulimit -n 65535                          # 进程级
cat /proc/sys/fs/file-max                # 系统级
echo 1000000 > /proc/sys/fs/file-max

# TCP
net.core.somaxconn = 65535                # listen 队列
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 1024 65535

# 内存
vm.swappiness = 10                        # 0-100，DB 设 1
vm.overcommit_memory = 1                  # Redis 推荐
vm.max_map_count = 262144                 # ES / 大内存应用

# 网络
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_fin_timeout = 30
```

---

## 十二、一句话总结

> Linux 排查的核心是 **"USE 三问 + 命令地图 + 字段精解 + 组合套路"**：
>
> ① **USE 三问**：先判断哪个资源（CPU / 内存 / 磁盘 / 网络）的 Utilization / Saturation / Errors 异常
> ② **5+5+5 命令**：日常 5 件套（top/free/vmstat/iostat/ss）+ 排查 5 件套（pidstat/pstack/lsof/strace/dmesg）+ 剖析 5 件套（perf/sar/tcpdump/火焰图）
> ③ **关键字段**：load average / %util / await / TIME_WAIT / CLOSE_WAIT / si / so 这些字段背熟
> ④ **7 大套路**：按"现象 → 命令链"反查（CPU 高 / 服务卡死 / OOM / IO 慢 / 丢包 / 端口占用 / 凌晨故障）
>
> **避免无脑用 strace**（慢 5-50 倍）→ 优先 `perf trace`；**避免迷信 top 单一指标** → 一定看 USE 三维；**容器场景** → 看 `/sys/fs/cgroup/` 而不是宿主机 procfs。

---

## 配套阅读

- 完整排查流程：[09-linux-troubleshooting.md](09-linux-troubleshooting.md)
- 性能工具深度：[14-performance-tools.md](14-performance-tools.md)
- CPU + 内存深度：[15-cpu-memory-troubleshooting.md](15-cpu-memory-troubleshooting.md)
- 网络深度：[17-network-troubleshooting.md](17-network-troubleshooting.md)
- 磁盘 IO 深度：[18-disk-io-troubleshooting.md](18-disk-io-troubleshooting.md)
- 容器场景：[19-container-k8s-resource.md](19-container-k8s-resource.md)
- 生产事故复盘：[16-production-incident-cases.md](16-production-incident-cases.md)
