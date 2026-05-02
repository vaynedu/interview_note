# interview_note 目录结构设计

> 2026-05-02 · 资深 Go 后端面试知识库

## 一、设计原则

1. **高内聚优先于去重**：宁可重复，也要保证每一块内容能独立阅读，不依赖跨目录跳转
2. **双视角共存**：知识体系 + 面试视角，主目录按技术领域切分，面试视角通过领域内子目录 + 顶层 `99-meta/` 索引实现
3. **一个知识点 = 一个文件**：打开就能完整看完，不分散
4. **文件内部模板统一**：原理 / 八股 / 真题 / 手写 / 踩坑 五段式

## 二、顶层目录

```
01-go-language/         Go 语言专区
02-os/                  操作系统(进程/内存/IO/调度)
03-mysql/               MySQL(存储引擎/索引/事务/锁/MVCC/调优)
04-redis/               Redis(数据结构/持久化/集群/缓存模式)
05-message-queue/       消息队列(Kafka/RocketMQ/削峰/顺序/幂等)
06-distributed/         分布式(CAP/一致性/分布式锁/分布式事务/限流熔断)
07-microservice/        微服务(RPC/注册发现/治理/链路追踪)
08-architecture/        架构(演进/高并发高可用/典型模式)
09-ddd/                 领域驱动设计(战略/战术/落地)
10-system-design/       系统设计题(短链/秒杀/Feed/IM/支付…)
11-cdn/                 CDN(回源/缓存策略/调度/边缘)
12-ai/                  AI(范围待定)
13-engineering/         工程能力(Linux/Git/可观测性/压测/CI-CD)
14-projects/            项目实战 + 简历复盘 + STAR
99-meta/                高频题索引/刷题清单/面经/算法外链
```

## 三、Go 语言子目录（首期落地）

```
01-go-language/
├── README.md                   导航 + 高频题 Top N
├── 01-syntax/                  基础语法
│   ├── slice.md
│   ├── map.md
│   ├── string.md
│   ├── struct.md
│   ├── interface.md
│   ├── defer.md
│   ├── error.md
│   └── generics.md
├── 02-concurrency/             并发
│   ├── goroutine.md
│   ├── channel.md
│   ├── select.md
│   ├── sync-package.md
│   ├── context.md
│   └── memory-model.md
├── 03-runtime/                 运行时
│   ├── gmp-scheduler.md
│   ├── gc.md
│   ├── memory-allocator.md
│   └── escape-analysis.md
├── 04-stdlib/                  标准库
│   ├── net-http.md
│   ├── database-sql.md
│   ├── reflect.md
│   ├── encoding.md
│   └── io.md
├── 05-engineering/             工程实践
│   ├── project-layout.md
│   ├── error-handling.md
│   ├── logging.md
│   ├── testing.md
│   └── go-mod.md
├── 06-performance/             性能与排查
│   ├── pprof.md
│   ├── benchmark.md
│   ├── memory-optimization.md
│   └── troubleshooting.md
└── 07-ecosystem/               生态框架
    ├── gin.md
    ├── gorm.md
    ├── grpc.md
    ├── go-zero.md
    └── kratos.md
```

## 四、文件内部模板

每个知识点 `.md` 文件统一使用以下五段式：

```markdown
# <知识点名>

> 一句话概括 / 适用场景

## 一、核心原理
- 定义、底层数据结构、关键流程
- 必要时附源码片段或图示

## 二、八股速记
- 1 分钟能背完的版本，分点列出
- 用于面试前快速过

## 三、面试真题
- Q: 实际被问过的题
  A: 标准答案 + 加分点

## 四、手写实现
- 代码 + 关键注释
- 不适用时可省略本节

## 五、踩坑与最佳实践
- 真实事故 / 反模式 / 推荐写法
```

## 五、其他领域的同构约定

- 顶层每个领域目录下都放一个 `README.md` 作为本领域导航
- 每个领域内部按"知识点 = 一个 .md 文件"组织，文件内部沿用上述五段式模板
- 大领域可按二级目录分组（如 Go 拆 7 个二级），小领域可平铺

## 六、Meta 目录

```
99-meta/
├── high-frequency-questions.md     高频题 Top 100(跨领域汇总,带跳转)
├── interview-checklist.md          按岗位/公司分类的刷题清单
├── interview-experience.md         面经汇总
├── algorithm-links.md              算法刷题外链(LeetCode 题单等)
└── mind-map/                       各领域思维导图(可选)
```

## 七、首期落地范围

本次只生成：
1. 14 个顶层目录 + 各自 `README.md` 占位
2. `01-go-language/` 完整子目录与文件骨架（37 个文件，按模板预填章节标题）
3. 根 `README.md` 重写
4. 本设计文档
