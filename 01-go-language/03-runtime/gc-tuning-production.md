# Go GC 调优与线上实践

> GC 面试不要只背三色标记。资深后端更应该能回答：线上延迟抖动是不是 GC，怎么证据化定位，怎么降低分配和堆压力。

## 一、先说结论

```text
Go GC 调优优先级：
1. 减少不必要分配。
2. 控制对象存活时间。
3. 降低堆增长速度。
4. 再考虑 GOGC / GOMEMLIMIT。
```

不要一上来就调 `GOGC`。多数 GC 问题本质是分配太多、对象活得太久、缓存无边界。

## 二、GC 相关指标怎么看

核心指标：

| 指标 | 含义 | 异常信号 |
| --- | --- | --- |
| heap_alloc | 当前堆上存活对象大小 | 持续上涨可能泄漏或缓存无界 |
| heap_inuse | runtime 已向 OS 申请并使用的堆页 | 长期高位说明堆压力大 |
| heap_objects | 存活对象数量 | 小对象过多会增加扫描压力 |
| alloc_rate | 分配速率 | 越高越容易频繁 GC |
| gc_pause | STW 暂停 | P99 抖动要重点看 |
| gc_cpu_fraction | GC 占 CPU 比例 | 高说明 GC 成本明显 |
| next_gc | 下次 GC 目标堆大小 | 受 GOGC 影响 |

常用方式：

```bash
curl -s http://127.0.0.1:6060/debug/pprof/heap > heap.out
go tool pprof heap.out
```

运行时日志：

```bash
GODEBUG=gctrace=1 ./app
```

`gctrace` 适合临时定位，不建议长期直接刷业务日志。

## 三、GC 工作流程速记

```text
标记准备 STW
        |
        v
并发标记
        |
        v
标记终止 STW
        |
        v
并发清扫
```

Go 使用并发三色标记和写屏障，目标是把 STW 控制得很短。

面试重点不是背术语，而是说清：

- GC 要扫描存活对象，存活对象越多越慢。
- 分配速度越快，触发 GC 越频繁。
- 指针对象越多，扫描成本越高。
- 大量短生命周期对象会带来分配和回收压力。

## 四、GOGC 怎么理解

`GOGC` 控制下一次 GC 触发目标：

```text
下一次 GC 目标堆大小 = 上次 GC 后存活堆大小 * (1 + GOGC/100)
```

默认 `GOGC=100`，大致表示堆增长一倍后触发下一轮 GC。

| GOGC | 结果 | 取舍 |
| --- | --- | --- |
| 调小 | 更频繁 GC | 降低内存，增加 CPU |
| 调大 | 更少 GC | 降低 CPU，增加内存 |

例子：

```text
服务内存紧张：可以适当降低 GOGC，但 CPU 会升。
服务 CPU 紧张但内存充足：可以适当提高 GOGC。
```

## 五、GOMEMLIMIT 的价值

新版本 Go 支持软内存限制：

```bash
GOMEMLIMIT=2GiB ./app
```

它不是硬限制，不等于容器 OOM 保护，但可以让 runtime 在接近限制时更积极 GC。

容器里常见配置：

```text
容器 limit = 4GiB
GOMEMLIMIT = 3GiB ~ 3.5GiB
```

预留空间给：

- goroutine stack。
- runtime metadata。
- mmap。
- cgo。
- OS page cache。
- 业务非堆内存。

## 六、线上案例

### 案例 1：JSON 大对象导致 GC 压力

背景：接口返回大列表，每次请求把数据库结果组装成大 `[]map[string]any`，再 JSON 编码。

现象：

- CPU 上涨。
- pprof 显示 `encoding/json`、`mapassign`、`reflect` 分配多。
- GC 次数变多，P99 延迟抖动。

处理：

- `map[string]any` 改成明确 struct。
- 分页限制单次返回数量。
- 热点字段避免反射和重复转换。
- 大响应考虑流式编码或异步导出。

复盘：

```text
泛化结构方便开发，但会制造大量小对象和反射成本。
高频接口要用明确类型和边界控制。
```

### 案例 2：本地缓存无上限导致堆持续上涨

背景：为了减少 DB 查询，在进程内加 `map` 缓存，但没有 TTL 和容量限制。

现象：

- heap_alloc 持续上涨。
- GC 后内存不明显下降。
- heap pprof 显示大量业务对象由 cache map 持有。

处理：

- 引入 LRU / TinyLFU。
- 设置容量和 TTL。
- 大对象只缓存 ID 或摘要。
- 缓存命中率、容量、淘汰数加入监控。

## 七、优化手段

### 1. 减少分配

```go
buf := make([]byte, 0, 1024)
```

常见方法：

- 预估容量，减少 slice 扩容。
- `strings.Builder` 拼字符串。
- 避免 `fmt.Sprintf` 在热路径滥用。
- 避免 `[]byte` 和 `string` 频繁转换。
- 避免 `map[string]any` 泛化结构。

### 2. 控制对象生命周期

```text
对象越快变成不可达，GC 压力越低。
```

常见做法：

- 请求级对象不要放全局。
- 缓存必须有 TTL 和容量。
- 大对象处理完及时断开引用。
- 长生命周期对象减少指针字段。

### 3. 使用 sync.Pool 要谨慎

适合：

- 高频临时对象。
- 创建成本较高。
- 可以安全复用。

不适合：

- 对象很小。
- 生命周期复杂。
- 带业务状态容易脏读。
- 想用它做缓存。

`sync.Pool` 里的对象可能被 GC 清掉，不保证一定复用。

### 4. 调整 GOGC / GOMEMLIMIT

流程：

```text
先用 pprof 找分配源
        |
        v
优化热点分配
        |
        v
确认仍受内存/GC 影响
        |
        v
小步调整 GOGC/GOMEMLIMIT
        |
        v
压测和灰度观察 CPU、内存、延迟
```

## 八、面试真题

**Q1：线上怀疑 GC 导致延迟抖动，怎么排查？**

先看监控里的 GC pause、GC 次数、heap_alloc、alloc_rate、gc_cpu_fraction 是否和延迟抖动时间对齐。再抓 heap profile 和 alloc_space，定位主要分配来源。最后通过压测或灰度验证优化前后 GC 次数、CPU、P99 是否改善。

**Q2：GOGC 调大调小有什么影响？**

调小会更频繁 GC，内存降低但 CPU 增加；调大会减少 GC 次数，CPU 降低但内存占用增加。它是 CPU 和内存之间的取舍，不是万能优化。

**Q3：为什么 GC 后内存没有降？**

可能是对象仍然被引用，也可能是 Go runtime 没有立刻把空闲内存还给 OS，或者容器/OS 统计口径看到的是 RSS。要结合 heap profile、inuse_space、alloc_space、runtime metrics 判断。

**Q4：怎么降低 GC 压力？**

减少分配、减少指针对象、控制缓存容量、缩短对象生命周期、避免热路径反射和泛化结构。参数调优放在后面。

## 九、面试表达

```text
我不会先调 GOGC，而是先确认是不是 GC 导致问题：看 GC pause、alloc rate、heap、P99 是否时间对齐，再用 heap/alloc profile 找主要分配源。
优化上优先减少分配和控制缓存边界。GOGC 和 GOMEMLIMIT 是最后的调节旋钮，本质是在 CPU、内存和延迟之间做取舍。
```
