# Goroutine 泄漏排查与治理

> Goroutine 很轻，但不是免费。线上泄漏常见表现是 goroutine 数持续上涨、内存上涨、延迟抖动，最后连接池、队列或调度器被拖垮。

## 一、什么是 goroutine 泄漏

```text
一个 goroutine 已经没有业务价值，却因为 channel、锁、IO、context 或定时器等原因一直无法退出。
```

典型信号：

- `runtime.NumGoroutine()` 持续上涨。
- pprof goroutine 里大量栈停在同一处。
- 内存和 fd 数缓慢上涨。
- 服务重启后恢复，运行一段时间后再次恶化。
- 请求延迟 P99 抖动，CPU 不一定很高。

## 二、常见泄漏场景

### 1. channel 发送没人接

```go
func worker(ch chan int) {
    ch <- 1 // 如果没有接收方，会永久阻塞
}
```

解决：

```go
select {
case ch <- 1:
case <-ctx.Done():
    return
}
```

### 2. channel 接收没人发

```go
func worker(ch <-chan Job) {
    for {
        job := <-ch
        handle(job)
    }
}
```

如果 channel 永远不关闭，worker 永远阻塞。

更稳妥：

```go
for {
    select {
    case job, ok := <-ch:
        if !ok {
            return
        }
        handle(job)
    case <-ctx.Done():
        return
    }
}
```

### 3. 忘记取消 context

```go
ctx, cancel := context.WithTimeout(parent, time.Second)
// 忘了 defer cancel()
```

`cancel` 不只是取消请求，也会释放定时器和父子 context 引用。

```go
ctx, cancel := context.WithTimeout(parent, time.Second)
defer cancel()
```

### 4. HTTP 请求没有超时

```go
resp, err := http.Get(url)
```

默认 client 没有整体超时，网络卡住可能导致 goroutine 堆积。

```go
client := &http.Client{Timeout: 3 * time.Second}
req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
resp, err := client.Do(req)
```

### 5. ticker 没有 Stop

```go
ticker := time.NewTicker(time.Second)
for range ticker.C {
    do()
}
```

需要退出路径：

```go
ticker := time.NewTicker(time.Second)
defer ticker.Stop()

for {
    select {
    case <-ticker.C:
        do()
    case <-ctx.Done():
        return
    }
}
```

### 6. worker pool 没有关闭协议

```text
生产者退出了，但 job channel 不关闭。
消费者还在等。
```

推荐协议：

```text
生产者负责 close(jobs)。
消费者 range jobs。
上层 context 负责整体取消。
WaitGroup 等待退出。
```

## 三、排查路径

```text
发现 goroutine 数上涨
        |
        v
抓 pprof goroutine
        |
        v
看 top 栈：chan send / chan receive / select / IO / lock
        |
        v
定位创建 goroutine 的代码路径
        |
        v
补退出条件、超时、close 协议
        |
        v
压测验证 goroutine 数回落
```

常用命令：

```bash
curl -s http://127.0.0.1:6060/debug/pprof/goroutine?debug=2
go tool pprof http://127.0.0.1:6060/debug/pprof/goroutine
```

在 pprof 里看：

```text
top
traces
list funcName
```

如果没有暴露 pprof，可以临时加监控：

```go
log.Printf("goroutines=%d", runtime.NumGoroutine())
```

## 四、pprof 栈怎么看

### 1. 卡在 channel send

```text
goroutine 12345 [chan send]:
service.(*Producer).send(...)
```

含义：发送方在等接收方。常见原因是消费者退出、队列满、下游处理太慢。

治理：

- 给发送加 `select ctx.Done()`。
- 设计缓冲区容量和降级策略。
- 关闭时先停生产，再 drain，再关闭队列。

### 2. 卡在 channel receive

```text
goroutine 12345 [chan receive]:
service.(*Worker).loop(...)
```

含义：消费者在等数据。少量正常，大量可能是 worker 没退出。

治理：

- close channel。
- 增加 ctx 退出。
- worker 数量可控，不要每个请求起长期 worker。

### 3. 卡在 IO wait

```text
goroutine 12345 [IO wait]:
net/http.(*Transport).roundTrip(...)
```

含义：网络 IO 卡住。重点检查超时、连接池、DNS、下游延迟。

## 五、线上案例

### 案例：异步通知导致 goroutine 越积越多

背景：订单支付成功后异步通知多个外部系统，每个通知起一个 goroutine。

问题：

```go
go func() {
    http.Post(callbackURL, "application/json", body)
}()
```

外部系统偶发超时，但 HTTP client 没设置超时，goroutine 长时间卡在 IO wait。高峰期订单量上来后，goroutine 从几千涨到几十万，内存上涨，调度延迟变差。

处理：

- 统一封装 HTTP client，设置总超时、连接超时、响应头超时。
- 请求绑定 `context.WithTimeout`。
- 通知改成 MQ + worker pool，限制并发。
- 失败进入重试队列，按业务幂等键去重。
- 增加 goroutine 数、队列长度、下游耗时监控。

复盘：

```text
不要在请求链路里无限制 go func。
异步不等于无边界，必须有超时、并发上限、重试和退出协议。
```

## 六、治理模板

```go
type Worker struct {
    jobs chan Job
    wg   sync.WaitGroup
}

func (w *Worker) Start(ctx context.Context, n int) {
    for i := 0; i < n; i++ {
        w.wg.Add(1)
        go func() {
            defer w.wg.Done()
            for {
                select {
                case job, ok := <-w.jobs:
                    if !ok {
                        return
                    }
                    handle(ctx, job)
                case <-ctx.Done():
                    return
                }
            }
        }()
    }
}

func (w *Worker) Stop() {
    close(w.jobs)
    w.wg.Wait()
}
```

## 七、面试真题

**Q1：线上 goroutine 数一直上涨怎么排查？**

先看监控确认趋势，再抓 goroutine pprof，看大量 goroutine 卡在哪类栈：channel send/receive、select、IO wait、锁等待。然后顺着栈定位创建点，检查是否缺少 context、超时、close、WaitGroup 和并发上限。修复后压测验证 goroutine 数能否回落。

**Q2：怎么避免 goroutine 泄漏？**

每个 goroutine 都要有清晰生命周期：谁创建、谁取消、靠什么退出、是否能等待回收。具体做法是使用 context、channel close 协议、超时、worker pool、WaitGroup，不要无限制在请求里启动后台 goroutine。

**Q3：channel 泄漏怎么解决？**

发送和接收都要考虑退出。发送用 `select { case ch <- v: case <-ctx.Done(): }`，接收判断 `ok` 或监听 context。生产者负责 close，消费者不要随便 close 别人的 channel。

## 八、面试表达

```text
我排查 goroutine 泄漏一般先看 NumGoroutine 趋势，再抓 /debug/pprof/goroutine?debug=2。
如果大量栈停在 chan send/receive，就查 channel 关闭协议；停在 IO wait，就查超时和连接池；停在 sync，就查锁竞争。
治理上我会要求每个 goroutine 都有 owner、context、退出路径和等待回收机制，异步任务必须有并发上限。
```
