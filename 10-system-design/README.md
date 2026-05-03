# 系统设计

> 面向资深 Go 后端面试的系统设计题：按场景拆解容量、数据模型、核心链路、架构取舍和线上风险。

## 分类导航

### 方法论

| 文件 | 内容 |
| --- | --- |
| [00-question-map.md](00-question-map.md) | 经典系统设计题库地图：按短链、交易、内容、IM、搜索、调度、存储归类 |
| [01-design-framework.md](01-design-framework.md) | 系统设计答题框架：澄清问题、容量估算、核心链路、瓶颈、取舍 |
| [07-interview-answer-playbook.md](07-interview-answer-playbook.md) | 系统设计面试回答方法论：开场、推进、取舍、追问、资深表达 |

### 高频业务系统

| 文件 | 内容 |
| --- | --- |
| [02-short-code-platform.md](02-short-code-platform.md) | 短码 / 短链平台：发号、跳转、缓存、统计、防滥用 |
| [03-seckill-system.md](03-seckill-system.md) | 秒杀系统：削峰、库存扣减、防超卖、异步下单、热点治理 |
| [04-realtime-barrage.md](04-realtime-barrage.md) | 实时弹幕系统：长连接、房间广播、限流、审核、消息扇出 |
| [05-live-streaming.md](05-live-streaming.md) | 直播系统：推流、转码、CDN、低延迟、弹幕互动、录制回放 |
| [06-feed-system.md](06-feed-system.md) | Feed 信息流：推拉模型、时间线、fanout、排序、冷热治理 |
| [08-im-system.md](08-im-system.md) | IM 即时通讯：长连接、可靠投递、离线消息、多端同步、群聊 |
| [09-comment-system.md](09-comment-system.md) | 评论系统：楼中楼、热评、分页、审核、删除、计数 |
| [10-like-counter-system.md](10-like-counter-system.md) | 点赞与计数系统：高频写、去重、热点计数、异步落库 |
| [11-search-system.md](11-search-system.md) | 搜索系统：ES、倒排索引、数据同步、排序、延迟一致 |
| [12-file-storage-system.md](12-file-storage-system.md) | 网盘 / 文件存储：分片上传、秒传、对象存储、权限、去重 |
| [13-payment-system.md](13-payment-system.md) | 支付系统：状态机、幂等、渠道回调、对账、补偿 |
| [14-coupon-system.md](14-coupon-system.md) | 优惠券系统：领券并发、防超领、核销、规则、过期 |
| [15-inventory-system.md](15-inventory-system.md) | 库存系统：预占、扣减、释放、防超卖、库存分桶 |

## 高频题速览

### 通用方法

- 系统设计题如何开场？先问哪些问题？
- 如何做容量估算？
- 如何从单体演进到分布式？
- 如何识别瓶颈：读、写、存储、网络、热点、队列？
- 如何体现架构取舍，而不是堆组件？

### 短码 / 短链

- 短链如何生成唯一短码？
- 302 和 301 如何选择？
- 高并发跳转如何优化？
- 短链统计如何做？
- 如何防止恶意刷短链和钓鱼链接？

### 秒杀

- 秒杀如何防超卖？
- 库存扣减放 Redis 还是 MySQL？
- 如何削峰限流？
- 下单链路如何异步化？
- 秒杀结果如何查询？

### 实时弹幕

- 弹幕如何做到实时广播？
- 一个直播间几十万人如何扇出？
- WebSocket 连接如何管理？
- 弹幕如何限流、去重、审核？
- 热门房间如何避免单点压力？

### 直播系统

- 推流、拉流、转码、CDN 的链路是什么？
- 直播低延迟如何优化？
- 直播弹幕和礼物如何设计？
- 直播回放如何生成？
- 主播断流和观众卡顿如何处理？

### Feed

- 关注流用推模式还是拉模式？
- 大 V 发动态如何处理？
- 时间线如何存储？
- 推荐排序如何接入？
- 如何处理删除、屏蔽、隐私和热点？

### IM / 评论 / 点赞

- IM 消息如何保证可靠投递？
- 离线消息和多端同步如何设计？
- 评论楼中楼和热评如何设计？
- 点赞如何去重？点赞数如何保证最终一致？
- 热点内容点赞计数如何治理？

### 搜索 / 文件

- 为什么搜索不用 MySQL，而用 ES？
- MySQL 到 ES 的数据同步如何保证？
- 搜索结果和详情不一致怎么办？
- 大文件如何分片上传和断点续传？
- 秒传和文件去重如何设计？

### 支付 / 优惠券 / 库存

- 支付回调如何保证幂等？
- 支付系统如何对账和补偿？
- 优惠券如何防超领？
- 优惠券核销和退券如何保证幂等？
- 库存如何预占、扣减、释放，如何防超卖？

### 题库与回答方法

- 经典系统设计题有哪些类型？
- 短链、秒杀、IM、Feed、搜索、网盘、支付、评论分别考什么？
- 面试现场如何开场？
- 如何把答案从“组件堆砌”讲成“架构取舍”？
- 遇到追问不会时怎么处理？

## 复习路径

1. 先看 [00-question-map.md](00-question-map.md)，建立系统设计题型地图。
2. 再看 [01-design-framework.md](01-design-framework.md)，掌握系统设计题答题套路。
3. 看 [07-interview-answer-playbook.md](07-interview-answer-playbook.md)，掌握现场回答节奏。
4. 再看 [02-short-code-platform.md](02-short-code-platform.md) 和 [03-seckill-system.md](03-seckill-system.md)，这是高频基础题。
5. 然后看 [04-realtime-barrage.md](04-realtime-barrage.md) 和 [05-live-streaming.md](05-live-streaming.md)，补齐长连接、实时消息、音视频链路。
6. 看 [06-feed-system.md](06-feed-system.md)、[08-im-system.md](08-im-system.md)、[09-comment-system.md](09-comment-system.md) 和 [10-like-counter-system.md](10-like-counter-system.md)，理解内容社区、实时通信和计数系统。
7. 看 [11-search-system.md](11-search-system.md) 和 [12-file-storage-system.md](12-file-storage-system.md)，补齐搜索和文件存储。
8. 最后看 [13-payment-system.md](13-payment-system.md)、[14-coupon-system.md](14-coupon-system.md) 和 [15-inventory-system.md](15-inventory-system.md)，补齐交易一致性、幂等和对账。

## 答题原则

- 先澄清业务目标，不要一上来画架构图。
- 先估算量级，再决定是否需要分库分表、缓存、MQ、CDN。
- 核心链路要讲数据一致性、幂等、降级和失败恢复。
- 不要堆技术名词，要讲为什么选、代价是什么、替代方案是什么。
- 高级答案要体现边界：哪些走强一致，哪些走最终一致，哪些走离线链路。
