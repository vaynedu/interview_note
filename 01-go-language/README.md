# Go 语言

> 资深 Go 后端面试 — 体系笔记 + 高频题汇总

## 子目录

| 子目录 | 范围 |
| --- | --- |
| [01-syntax](01-syntax/)           | slice / map / string / struct / interface / defer / error / generics |
| [02-concurrency](02-concurrency/) | goroutine / channel / select / sync / context / memory-model |
| [03-runtime](03-runtime/)         | GMP 调度 / GC / 内存分配 / 逃逸分析 |
| [04-stdlib](04-stdlib/)           | net/http / database/sql / reflect / encoding / io |
| [05-engineering](05-engineering/) | 项目结构 / 错误处理 / 日志 / 测试 / go mod |
| [06-performance](06-performance/) | pprof / benchmark / 内存优化 / 线上排查 |
| [07-ecosystem](07-ecosystem/)     | gin / gorm / grpc / go-zero / kratos |

## 高频题 Top（待整理）

- slice 扩容机制
- map 为什么不是并发安全
- channel 底层结构与发送/接收流程
- GMP 模型与 work-stealing
- GC 三色标记 + 混合写屏障
- defer 执行顺序与闭包捕获
- interface 的 iface / eface
- context 取消传播机制

## 文件内部模板

```
一、核心原理 / 二、八股速记 / 三、面试真题 / 四、手写实现 / 五、踩坑与最佳实践
```
