# errgroup:协程协同取消与首错收集

> Go 并发题最高频追问之一:**一个协程出错,其他协程怎么全部退出?**
>
> 答案是 `golang.org/x/sync/errgroup`——并发界的"WaitGroup + context cancel + 首错收集"三合一,90% 场景用它就够了。
>
> 关联:[goroutine.md](goroutine.md) / [context.md](context.md) / [channel-patterns-cookbook.md](channel-patterns-cookbook.md) §11 Cancellation。

## 〇、核心提炼(5 段式)

### 核心机制(4 条必背)

1. **`errgroup.WithContext(ctx)`** 返回一个 `*Group` + 派生 ctx,内部包了 `context.WithCancel`
2. **`g.Go(f func() error)`** 启动协程,任一返回非 nil error → 立即 `cancel(ctx)` 广播给其他协程
3. **`g.Wait()`** 阻塞等所有协程结束,**返回第一个非 nil error**(后面的 error 丢弃)
4. **子协程必须监听 `<-ctx.Done()`**——否则 cancel 信号传过去也退不出去,这是最高频踩坑

### 核心本质(必懂)

> errgroup 的本质是 **"WaitGroup + context.WithCancel + sync.Once(收第一个 err)"** 三件套的封装。
>
> - **WaitGroup** 负责"等所有协程结束"
> - **context.WithCancel** 负责"出错时广播取消"
> - **sync.Once** 负责"只记第一个 err"(避免被后续 ctx.Err() 覆盖)
>
> **它不做的事**(必须自己处理):
> - ❌ **不 recover panic**——子协程 panic 会让进程崩(用 `sourcegraph/conc` 或手动 defer recover)
> - ❌ **不收集所有 error**——只返回第一个(要全部 err 自己用 mutex + slice 收)
> - ❌ **不强制限制并发**——默认无限并发(Go 1.20+ 用 `SetLimit(n)`)
> - ❌ **不会强杀协程**——只是 cancel ctx,子协程不监听就退不出
>
> **取消的本质是"协作式"**:Go 没有强制中断协程的机制(没有 `Thread.interrupt()`),
> errgroup 发的 cancel 信号必须**靠子协程主动检查** `ctx.Done()` 才能生效。
> **不监听 ctx 的协程就是"耳聋"的协程,谁也叫不停**。

### 完整流程(面试必背)

```
1. g, ctx := errgroup.WithContext(parentCtx)
   内部:childCtx, cancel = context.WithCancel(parentCtx)

2. g.Go(f) × N 次
   每个 g.Go 内部:
     wg.Add(1)
     go func() {
         defer wg.Done()
         if err := f(); err != nil {
             errOnce.Do(func() {
                 g.err = err
                 cancel()           ← 关键:广播取消
             })
         }
     }()

3. 某协程返回 err
   → errOnce.Do 记下第一个 err + cancel
   → ctx.Done() channel 被 close
   → 所有监听 ctx 的协程感知 → 自己 return

4. g.Wait()
   → wg.Wait() 等所有协程结束
   → cancel() (兜底,防 ctx 泄漏)
   → return g.err
```

```mermaid
sequenceDiagram
    participant Main as main
    participant G as errgroup
    participant G1 as goroutine 1
    participant G2 as goroutine 2 (出错)
    participant G3 as goroutine 3

    Main->>G: WithContext(ctx) 返回 g, ctx
    Main->>G: g.Go(f1), g.Go(f2), g.Go(f3)
    G->>G1: 启动 f1
    G->>G2: 启动 f2
    G->>G3: 启动 f3
    G2-->>G: f2 返回 error
    G->>G: errOnce.Do(err=err2; cancel())
    Note over G1,G3: ctx.Done() 被 close
    G1->>G1: select ctx.Done() → return ctx.Err()
    G3->>G3: select ctx.Done() → return ctx.Err()
    Main->>G: g.Wait()
    G-->>Main: 返回 err2 (第一个错误)
```

### 4 条核心机制 - 逐点讲透

#### 1. `errgroup.WithContext` vs `errgroup.Group{}`

```go
// ✓ 推荐:出错联动取消
g, ctx := errgroup.WithContext(parentCtx)

// ⚠️ 不推荐:零值用法,没有 cancel 联动
var g errgroup.Group
// 一个出错,其他协程照跑完(只收集第一个 err 但不广播)
```

→ **几乎所有场景都用 `WithContext`**,零值版只在"明确不想取消其他协程"时用。

#### 2. `g.Go` 启动协程的源码逻辑

```go
// errgroup 源码核心(简化版)
func (g *Group) Go(f func() error) {
    g.wg.Add(1)
    go func() {
        defer g.wg.Done()
        if err := f(); err != nil {
            g.errOnce.Do(func() {
                g.err = err
                if g.cancel != nil {
                    g.cancel()   // 广播取消
                }
            })
        }
    }()
}
```

→ **没有 panic recover**——子协程 panic 进程崩,这是和 `sourcegraph/conc` 的最大区别。

#### 3. `g.Wait()` 的返回值语义

```go
err := g.Wait()
// err == 第一个非 nil error
// err == nil 表示所有协程都成功

// 注意:不是 errors.Join,后续 err 会被丢弃
```

**要收集所有 err** 必须自己包一层:

```go
var (
    mu     sync.Mutex
    allErr []error
)
g.Go(func() error {
    if err := doWork(); err != nil {
        mu.Lock()
        allErr = append(allErr, err)
        mu.Unlock()
        return err
    }
    return nil
})
_ = g.Wait()
return errors.Join(allErr...)
```

#### 4. `SetLimit` 限并发(Go 1.20+)

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(10)   // 最多 10 个协程并发

for _, url := range thousandUrls {
    url := url
    g.Go(func() error {       // 满了会阻塞等待
        return fetch(ctx, url)
    })
}
g.Wait()
```

**`TryGo`**——尝试启动,满了立即返回 false 不阻塞:

```go
if !g.TryGo(func() error { return work() }) {
    log.Println("队列已满,丢弃任务")
}
```

### 一句话总结

> errgroup = **WaitGroup + context.WithCancel + sync.Once**,
> 任一 `g.Go` 返回 error → 内部 cancel ctx → 其他协程通过 `<-ctx.Done()` 主动退出 → `g.Wait()` 收第一个错。
> **关键铁律:子协程必须监听 ctx,IO 调用必须传 ctx**——发了取消信号没人听等于没发。
> 限并发用 `SetLimit`,处理 panic 用 `sourcegraph/conc` 或手 defer recover。

---

## 一、首选用法:`errgroup.WithContext`

```go
import "golang.org/x/sync/errgroup"

func run(ctx context.Context) error {
    g, ctx := errgroup.WithContext(ctx)

    for i := 0; i < 10; i++ {
        i := i  // Go 1.22 前必须捕获,1.22+ 可省
        g.Go(func() error {
            return doWork(ctx, i)
        })
    }
    return g.Wait()
}

func doWork(ctx context.Context, i int) error {
    select {
    case <-ctx.Done():
        return ctx.Err()  // 其他协程出错 → 主动退出
    case <-time.After(time.Second):
        if i == 3 {
            return errors.New("boom")  // 我自己出错
        }
        return nil
    }
}
```

## 二、手写版(理解原理,不推荐生产)

```go
func runManual(ctx context.Context) error {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    var wg sync.WaitGroup
    var once sync.Once
    var firstErr error

    for i := 0; i < 10; i++ {
        i := i
        wg.Add(1)
        go func() {
            defer wg.Done()
            if err := doWork(ctx, i); err != nil {
                once.Do(func() {
                    firstErr = err
                    cancel()  // 广播取消
                })
            }
        }()
    }
    wg.Wait()
    return firstErr
}
```

→ 这就是 errgroup 的源码,**写一遍就够**,生产用 errgroup。

## 三、三种控制方式对比

| 方案 | 出错广播 | panic 处理 | 限并发 | 适用 |
| --- | --- | --- | --- | --- |
| **裸 `sync.WaitGroup`** | ❌ 自己写 | ❌ 自己 recover | ❌ | 协程间无依赖,纯等待 |
| **`errgroup.Group`** ⭐ | ✓ ctx 自动 cancel | ❌ panic 仍崩进程 | ✓ `SetLimit(n)` | **90% 场景首选** |
| **`sourcegraph/conc`** | ✓ | ✓ 自动 recover 转 error | ✓ | panic 高发场景 / 复杂依赖 |

**`sourcegraph/conc` 示例**(panic-safe):

```go
import "github.com/sourcegraph/conc/pool"

p := pool.New().WithErrors().WithContext(ctx).WithMaxGoroutines(10)
for _, item := range items {
    item := item
    p.Go(func(ctx context.Context) error {
        return process(ctx, item)  // panic 自动转 error
    })
}
return p.Wait()
```

## 四、子协程必须做的事(高频踩坑)

### 4.1 死循环必须监听 ctx

```go
// ❌ 错:即使其他协程出错,这个一直跑完才退
g.Go(func() error {
    for _, x := range bigList {
        process(x)  // 不看 ctx,叫不停
    }
    return nil
})

// ✓ 对:循环里 select ctx
g.Go(func() error {
    for _, x := range bigList {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
        }
        process(x)
    }
    return nil
})
```

### 4.2 阻塞 IO 必须传 ctx

```go
// ❌ 错:HTTP 不带 ctx,不会被取消
http.Get(url)

// ✓ 对:NewRequestWithContext
req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
http.DefaultClient.Do(req)

// ✓ 对:DB / Redis 调用都用带 ctx 的版本
db.QueryContext(ctx, sql)
rdb.Get(ctx, key)
```

### 4.3 channel 操作要 select ctx

```go
// ❌ 错:阻塞写,ctx 取消也没用
ch <- v

// ✓ 对
select {
case ch <- v:
case <-ctx.Done():
    return ctx.Err()
}
```

## 五、典型场景

### 场景 1:并发请求 N 个下游,任一失败全取消

```go
func fanout(ctx context.Context, urls []string) ([]Result, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make([]Result, len(urls))

    for i, url := range urls {
        i, url := i, url
        g.Go(func() error {
            r, err := fetch(ctx, url)
            if err != nil {
                return err
            }
            results[i] = r  // 不同下标,无需加锁
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

### 场景 2:Pipeline 多阶段并行

```go
func pipeline(ctx context.Context) error {
    g, ctx := errgroup.WithContext(ctx)
    in := make(chan int, 10)
    mid := make(chan int, 10)

    // 阶段 1:生产
    g.Go(func() error {
        defer close(in)
        for i := 0; i < 100; i++ {
            select {
            case in <- i:
            case <-ctx.Done():
                return ctx.Err()
            }
        }
        return nil
    })

    // 阶段 2:处理
    g.Go(func() error {
        defer close(mid)
        for v := range in {
            r, err := transform(v)
            if err != nil { return err }
            select {
            case mid <- r:
            case <-ctx.Done():
                return ctx.Err()
            }
        }
        return nil
    })

    // 阶段 3:消费
    g.Go(func() error {
        for v := range mid {
            if err := save(ctx, v); err != nil { return err }
        }
        return nil
    })

    return g.Wait()
}
```

### 场景 3:批量爬虫 + 限并发

```go
func crawl(ctx context.Context, urls []string) error {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(20)  // 最多 20 并发

    for _, url := range urls {
        url := url
        g.Go(func() error {
            return crawlOne(ctx, url)
        })
    }
    return g.Wait()
}
```

## 六、4 个常见坑

```
坑 1:循环变量捕获(Go 1.22 前)
  for i := range items {
      g.Go(func() { use(i) })  // 全跑成最后一个
  }
  → Go 1.22 前必须 i := i
  → Go 1.22+ for 循环变量每次迭代独立,可省

坑 2:用 errgroup.Group{} 零值(没 WithContext)
  → 没有 cancel 联动,一个出错其他照跑完
  → 几乎所有场景都该用 errgroup.WithContext

坑 3:g.Wait() 只返回第一个 err
  → 后续 err 全丢
  → 要全部 err 自己 mutex + slice 收集 + errors.Join

坑 4:g.Go 里 panic
  → 整个进程崩(errgroup 不 recover)
  → 关键路径加 defer recover() 或用 sourcegraph/conc
```

**panic 兜底模板**:

```go
func safeGo(g *errgroup.Group, f func() error) {
    g.Go(func() (err error) {
        defer func() {
            if r := recover(); r != nil {
                err = fmt.Errorf("panic: %v\n%s", r, debug.Stack())
            }
        }()
        return f()
    })
}
```

## 七、面试速答模板

### Q1:Go 协程间如何控制,一个出错全部退出?

```text
首选 errgroup.WithContext。

任一 g.Go 返回 error,内部 cancel ctx 广播给其他协程,
其他协程通过 select case <-ctx.Done() 主动退出,
g.Wait() 收集并返回第一个错误。

关键铁律:子协程必须监听 ctx,IO 调用必须传 ctx,
死循环里必须 select default + ctx.Done()——
取消是协作式的,不监听等于没发。

要限并发用 SetLimit(n),要处理 panic 用 sourcegraph/conc
或者每个 g.Go 加 defer recover 兜底。
```

### Q2:errgroup 源码大致怎么实现?

```text
三件套封装:
  sync.WaitGroup 等所有协程结束
  context.WithCancel 出错广播取消
  sync.Once 只记第一个 err

g.Go(f):
  wg.Add(1) → go func() { defer wg.Done(); err := f();
              if err != nil { once.Do(记 err + cancel) } }

g.Wait():
  wg.Wait() → cancel() 兜底 → return err
```

### Q3:errgroup 不能解决什么?

```text
1. 不 recover panic → 进程崩
2. 不收集所有 err,只返第一个
3. 不会强杀协程,子协程不监听 ctx 就退不出
4. 默认无限并发(1.20+ 才有 SetLimit)
```

## 八、关联阅读

```
本目录:
- goroutine.md                   goroutine 基础 + 第十章踩坑
- context.md                     ctx 传播与取消机制
- channel-patterns-cookbook.md   §11 Cancellation 取消传播
- sync-package.md                WaitGroup / Once / Mutex

跨模块:
- 99-meta/go-concurrency-100.md  Go 并发 100 题
- 99-meta/senior-go-interview-25.md  资深 Go 25 题
```

> **一句话核心(全篇精炼)**:errgroup = **WaitGroup + context.WithCancel + sync.Once 三件套**,
> 任一协程出错 → cancel ctx → 其他协程通过 `<-ctx.Done()` **主动退出** → Wait 返回第一个错;
> **取消是协作式的**——子协程必须监听 ctx、IO 必须传 ctx,否则信号发了也没人听;
> 限并发 `SetLimit`,panic 兜底用 `sourcegraph/conc` 或 `defer recover`。
