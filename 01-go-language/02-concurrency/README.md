# Go 并发

## 文件清单

| # | 文件 | 内容 |
| --- | --- | --- |
| 01 | [goroutine.md](goroutine.md) | goroutine 调度 / 栈 / 生命周期 / 泄漏 |
| 02 | [channel.md](channel.md) | hchan 原理 / 源码深读 / 性能选型 / 关闭协议 / 陷阱 |
| 03 | [channel-patterns-cookbook.md](channel-patterns-cookbook.md) | ★ 设计模式 + 业务场景速查（15 模式 + 5 业务场景） |
| 04 | [select.md](select.md) | 多路复用 / 随机 / nil channel / for-select-done 模式 |
| 05 | [sync-package.md](sync-package.md) | Mutex / RWMutex / WaitGroup / Once / Cond / Map / Pool / atomic |
| 06 | [context.md](context.md) | Context / 取消传播 / 超时 / 值传递 |
| 07 | [memory-model.md](memory-model.md) | happens-before / 数据竞争 / atomic / race |

## 推荐学习顺序

```
1. goroutine（GMP / 栈）
2. channel（原理 + 源码 + 性能）
3. channel-patterns-cookbook（模式速查）
4. select（多路复用）
5. sync-package（其他原语）
6. context（取消传播）
7. memory-model（happens-before）
```

## Channel 强化路径

```
channel.md  原理 + 源码 + 性能选型 + 关闭协议（深度）
    ↓
channel-patterns-cookbook.md  15 模式 + 5 业务场景（实战速查）
    ↓
99-meta/go-concurrency-100.md  100 题练习
```
