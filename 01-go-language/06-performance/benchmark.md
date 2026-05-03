# benchmark

> Go 内置基准测试：自动迭代 b.N 直到稳定；benchstat 对比；避免编译器优化是关键技巧

## 一、核心原理

### 1.1 基本语法

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(1, 2)
    }
}
```

跑：

```bash
go test -bench=. -benchmem
# BenchmarkAdd-8    1000000000   0.34 ns/op   0 B/op   0 allocs/op
```

输出含义：
- `-8`：GOMAXPROCS
- `1000000000`：迭代次数（b.N）
- `0.34 ns/op`：单次操作纳秒
- `0 B/op`：单次分配字节
- `0 allocs/op`：单次分配次数

### 1.2 b.N 自适应

testing 框架会**自动调整 b.N**：从 1 开始，倍增直到总耗时 ≥ benchtime（默认 1s）。所以代码里写 `for i := 0; i < b.N; i++` 是约定，不要硬编码次数。

### 1.3 关键 API

```go
b.ResetTimer()       // 重置计时器(忽略 setup 耗时)
b.StopTimer()        // 暂停
b.StartTimer()       // 恢复
b.ReportAllocs()     // 强制报告内存分配
b.SetBytes(n)        // 设置每次操作字节数, 报告 MB/s
b.Run(name, func)    // 子 benchmark
b.RunParallel(func)  // 并行测试
```

### 1.4 子 benchmark

```go
func BenchmarkSort(b *testing.B) {
    for _, n := range []int{100, 1000, 10000, 100000} {
        b.Run(fmt.Sprintf("n=%d", n), func(b *testing.B) {
            data := genData(n)
            b.ResetTimer()
            for i := 0; i < b.N; i++ {
                Sort(append([]int{}, data...))
            }
        })
    }
}
```

输出：

```
BenchmarkSort/n=100-8       100000   12000 ns/op
BenchmarkSort/n=1000-8       10000  150000 ns/op
BenchmarkSort/n=10000-8       1000 1800000 ns/op
```

跑指定子 benchmark：`go test -bench=BenchmarkSort/n=1000`

### 1.5 并行 benchmark

```go
func BenchmarkParallel(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            doWork()
        }
    })
}
```

`RunParallel` 启 GOMAXPROCS 个 g 同时跑，测**并发吞吐**。

### 1.6 避免编译器优化

```go
// 错: result 没用, 可能被 dead code elimination
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(1, 2)
    }
}
```

**修复**：用 sink 变量

```go
var sink int

func BenchmarkAdd(b *testing.B) {
    var s int
    for i := 0; i < b.N; i++ {
        s = Add(1, 2)
    }
    sink = s
}
```

或 `runtime.KeepAlive(s)`。

### 1.7 benchstat 对比

```bash
# 旧版本
git stash
go test -bench=. -count=10 > old.txt

# 新版本
git stash pop
go test -bench=. -count=10 > new.txt

# 对比
benchstat old.txt new.txt
```

输出：

```
name         old time/op    new time/op    delta
Add-8         12.0ns ± 1%    8.0ns ± 1%    -33.33%  (p=0.000 n=10+10)
```

`p=0.000` 表示统计显著（< 0.05 即可信）。

### 1.8 -benchtime 控制时长

```bash
go test -bench=. -benchtime=3s    # 跑 3 秒
go test -bench=. -benchtime=100x  # 跑 100 次
```

短代码用 `100x` / `1000x` 避免单次太快导致 b.N 巨大（数十亿次循环耗内存）。

## 二、八股速记

- benchmark 函数 `Benchmark<Name>(b *testing.B)`
- 用 `b.N` 循环，框架自动调到稳定时长（默认 1s）
- `-bench=.` 跑所有，`-benchmem` 看分配
- **避免编译器优化**：用 sink 变量或 KeepAlive
- **setup 用 b.ResetTimer**：把初始化排除
- **子 benchmark** 用 `b.Run`，多场景对比
- **并行**用 `b.RunParallel`
- **对比改前后**用 `benchstat`，要 `-count=10` 多次取统计
- `b.SetBytes(n)` 报告 MB/s（IO 类）
- 短代码用 `-benchtime=Nx` 避免 b.N 巨大

## 三、面试真题

**Q1：b.N 是怎么定的？**
框架自动调：从 1 开始，跑完看耗时；不到 benchtime（默认 1s），翻倍重跑；直到总耗时 ≥ 1s 报告结果。

所以 `b.N` 在不同 benchmark 间差异巨大：快函数 `b.N=1e9`，慢函数 `b.N=10`。

**Q2：怎么避免 dead code elimination？**

```go
var sink int  // 包级变量, 编译器不能消除

func BenchmarkAdd(b *testing.B) {
    var s int
    for i := 0; i < b.N; i++ {
        s = Add(1, 2)
    }
    sink = s  // 阻止编译器优化掉 Add 调用
}
```

或：

```go
runtime.KeepAlive(s)  // 显式告诉编译器"s 还要用"
```

不做这个 Go 编译器很可能把无副作用的 `Add(1,2)` 直接消除。

**Q3：怎么排除 setup 时间？**

```go
func BenchmarkSort(b *testing.B) {
    data := generateBigData()  // setup
    b.ResetTimer()              // 重置计时器
    for i := 0; i < b.N; i++ {
        Sort(append([]int{}, data...))
    }
}
```

或暂停/恢复：

```go
for i := 0; i < b.N; i++ {
    b.StopTimer()
    data := generate()  // 不计入
    b.StartTimer()
    Sort(data)
}
```

**Q4：benchmark 结果稳定吗？怎么提高可信度？**
不稳定。影响因素：
- CPU 频率（睿频/降频）
- 同机其他进程
- GC 抖动
- L1/L2 缓存

**提高可信度**：
1. **`-count=10`** 多次跑
2. **关其他程序**，专机跑
3. **CPU 固频**：`echo performance > /sys/.../scaling_governor`
4. **`benchstat`** 看 p 值（< 0.05 显著）

**Q5：怎么对比改前改后性能？**

```bash
# 1. checkout 旧版
go test -bench=. -count=10 -benchmem > old.txt

# 2. checkout 新版
go test -bench=. -count=10 -benchmem > new.txt

# 3. 对比
benchstat old.txt new.txt
```

输出：

```
name        old time/op    new time/op    delta
Foo-8        100ns ± 2%     80ns ± 1%     -20.00%  (p=0.000 n=10+10)
```

`-` 是改进，`+` 是变差。`p` 值越小越显著。

**Q6：怎么测内存分配？**

```bash
go test -bench=. -benchmem
```

输出：
```
BenchmarkX-8   1000000   1500 ns/op   240 B/op   3 allocs/op
```

- `B/op`：单次分配字节
- `allocs/op`：单次分配次数

优化目标：**减少 allocs/op**（每次分配都进 mcache + 可能 GC）。

**Q7：RunParallel 怎么用？**

```go
func BenchmarkParallelGet(b *testing.B) {
    cache := newCache()
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            cache.Get(randomKey())
        }
    })
}
```

测**并发吞吐**。多 g 同时跑，看锁竞争、共享状态影响。

不能用 `b.N`，用 `pb.Next()` 控制循环。

**Q8：测 IO 性能用什么指标？**

```go
func BenchmarkRead(b *testing.B) {
    data := make([]byte, 1<<20)  // 1MB
    b.SetBytes(int64(len(data)))  // ← 关键
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        process(data)
    }
}
```

`SetBytes` 后输出会有 `MB/s`：

```
BenchmarkRead-8   1000   1500000 ns/op   666.67 MB/s
```

更直观看吞吐。

**Q9：benchmark 慢于 1ns/op 是什么意思？**
比 1ns 还快通常意味着**编译器优化掉了**。CPU 单条指令也要 ~0.3ns，单次函数调用至少 1~2 ns。

```
BenchmarkX-8   1000000000   0.34 ns/op  ← 可疑
```

检查是不是 dead code elimination。

**Q10：benchmark 数据怎么解读？**

```
BenchmarkFoo-8   1000000   1500 ns/op   240 B/op   3 allocs/op
```

- `1000000`：跑了多少次（b.N）
- `1500 ns/op`：单次 1.5μs
- `240 B/op`：单次分配 240 字节
- `3 allocs/op`：单次分配 3 次

判断：
- 函数耗时 1.5μs → QPS 上限 ~600K（单核）
- 每次分配 3 次 → 高 QPS 下 GC 压力
- 优化方向：减分配 → 提速

## 四、手写实现

**1. 对比拼接性能：**

```go
var sink string

func BenchmarkConcat(b *testing.B) {
    parts := []string{"hello", " ", "world", "!", " from", " go"}

    b.Run("plus", func(b *testing.B) {
        b.ReportAllocs()
        for i := 0; i < b.N; i++ {
            s := ""
            for _, p := range parts { s += p }
            sink = s
        }
    })

    b.Run("builder", func(b *testing.B) {
        b.ReportAllocs()
        for i := 0; i < b.N; i++ {
            var sb strings.Builder
            for _, p := range parts { sb.WriteString(p) }
            sink = sb.String()
        }
    })

    b.Run("join", func(b *testing.B) {
        b.ReportAllocs()
        for i := 0; i < b.N; i++ {
            sink = strings.Join(parts, "")
        }
    })
}
```

预期：plus 慢且分配多；builder 和 join 快且少分配。

**2. 测并发吞吐：**

```go
func BenchmarkCacheGet(b *testing.B) {
    c := NewCache()
    for i := 0; i < 1000; i++ { c.Set(i, i*2) }

    b.ResetTimer()
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            c.Get(rand.Intn(1000))
        }
    })
}
```

**3. 测 IO 吞吐：**

```go
func BenchmarkCopy(b *testing.B) {
    src := bytes.NewReader(make([]byte, 1<<20))  // 1MB
    var dst bytes.Buffer
    b.SetBytes(1 << 20)
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        src.Seek(0, 0)
        dst.Reset()
        io.Copy(&dst, src)
    }
}
```

**4. 多场景子 benchmark：**

```go
func BenchmarkMapVsSwitch(b *testing.B) {
    keys := []string{"a", "b", "c", "d", "e"}
    m := map[string]int{"a": 1, "b": 2, "c": 3, "d": 4, "e": 5}

    b.Run("map", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            sink2 = m[keys[i%5]]
        }
    })

    b.Run("switch", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            switch keys[i%5] {
            case "a": sink2 = 1
            case "b": sink2 = 2
            case "c": sink2 = 3
            case "d": sink2 = 4
            case "e": sink2 = 5
            }
        }
    })
}
var sink2 int
```

**5. CI 集成 benchmark 回归检测：**

```bash
#!/bin/bash
# benchmark-regression.sh

git checkout main
go test -bench=. -count=10 -benchmem > main.txt

git checkout $CURRENT_BRANCH
go test -bench=. -count=10 -benchmem > new.txt

benchstat main.txt new.txt > diff.txt

# 用 awk 检查是否有 > 5% 的 regression
if grep -E '\+[5-9]\.|\+[1-9][0-9]+\.' diff.txt; then
    echo "Performance regression detected!"
    cat diff.txt
    exit 1
fi
```

## 五、踩坑与最佳实践

### 坑 1：编译器优化掉测试

```go
for i := 0; i < b.N; i++ {
    expensiveCalc()  // 无副作用 → 可能被消除
}
```

修复：用 sink 变量。

### 坑 2：setup 没排除

```go
func BenchmarkX(b *testing.B) {
    data := loadHugeFile()  // 1s 加载, 计入了
    for i := 0; i < b.N; i++ { process(data) }
}
```

修复：`b.ResetTimer()` 在 setup 后。

### 坑 3：每次循环重新 setup

```go
for i := 0; i < b.N; i++ {
    data := generate()  // 每次重做, 不能 Reset
    process(data)
}
```

修复：StopTimer/StartTimer，或把 setup 拉出去用切片轮换。

### 坑 4：忘记 -benchmem

```bash
go test -bench=.
# 没有 B/op 信息, 看不到分配
```

修复：加 `-benchmem` 或代码里 `b.ReportAllocs()`。

### 坑 5：没多次跑直接对比

```bash
go test -bench=.   # 单次
# 改代码
go test -bench=.   # 单次, 差 5% 不知道是优化还是噪声
```

修复：`-count=10` + `benchstat`，看 p 值确认显著。

### 坑 6：benchmark 时机器有干扰

后台跑了 docker / IDE / 视频，结果剧烈波动。

修复：
- 关闭无关程序
- CPU 固频
- 多次跑取中位

### 坑 7：测试小函数 b.N 巨大

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ { Add(1,2) }
}
// b.N = 1e10, 内存爆 / 太慢
```

修复：`-benchtime=100x` 限制次数。或测一组操作。

### 坑 8：`b.Stop/StartTimer` 在循环里影响精度

```go
for i := 0; i < b.N; i++ {
    b.StopTimer()
    setup()  // setup 本身不应该太复杂
    b.StartTimer()
    work()
}
```

频繁 Stop/Start 本身有 ns 级开销，对超快函数失真。修复：只测 work，setup 用预分配数组轮换。

### 最佳实践

- **永远 `-benchmem`** 看分配
- **`-count=10` + benchstat** 对比改前后
- **避免编译器优化**：sink 变量或 KeepAlive
- **setup 排除**：`b.ResetTimer()` 在 setup 后
- **子 benchmark** 多参数对比
- **并发吞吐用 RunParallel**
- **IO 类用 SetBytes** 报告 MB/s
- **CI 集成回归检测**：基线 + diff 自动告警
- **关闭干扰**：固频 / 专机
- **不在 benchmark 里做不该做的事**（log / IO / sleep）
- **配合 pprof**：`-cpuprofile` / `-memprofile` 找瓶颈
