# 微服务

> 微服务核心组件 + 实战落地：**注册中心、配置中心、网关、RPC、Service Mesh、可观测**

> 战略层（边界划分）见 09-ddd；演进路径见 08-architecture/01；本目录聚焦**组件选型 + 实战**

## ★ 总览地图

> **新人/复习者优先看** [00-microservice-map.md](00-microservice-map.md)：知识树 / 8 层能力 / 题型分级 / Mesh 与 xDS / 可观测体系 / 答题模板。

## 目录

| # | 文件 | 涵盖 |
| --- | --- | --- |
| 01 | [概述](01-overview.md) | 微服务定义 / 12-Factor / 拆分原则 / 与 SOA 对比 / 真实代价 |
| 02 | [注册中心与服务发现](02-registry-discovery.md) | ZK / Eureka / Nacos / etcd / Consul 对比 / 心跳 / CAP / 优雅下线 |
| 03 | [配置中心](03-config-center.md) | Nacos / Apollo / Consul / etcd / 长轮询 / 灰度 / 加密 |
| 04 | [API 网关](04-api-gateway.md) | Kong / APISIX / Envoy / SCGW / BFF / 鉴权 / 限流 / 协议转换 |
| 05 | [RPC 框架](05-rpc-frameworks.md) | gRPC / Thrift / Dubbo / Kitex / go-zero / Kratos / IDL / LB |
| 06 | [Service Mesh](06-service-mesh.md) | Sidecar / 数据面控制面 / Istio / Envoy / Linkerd / 性能��销 |
| 07 | [观测与治理](07-observability-governance.md) | OTel / Jaeger / Prometheus / Loki / SLO / 治理整合 |

## 跨章高频题

- 微服务 vs SOA 区别？（→ 01）
- 12-Factor 是什么？（→ 01）
- 什么时候该上微服务？真实代价？（→ 01）
- 注册中心怎么选？ZK/Eureka/Nacos/etcd/Consul？（→ 02）
- 服务发现该 CP 还是 AP？（→ 02）
- 客户端发现 vs 服务端发现？（→ 02）
- 优雅下线怎么做？（→ 02）
- 配置中心解决什么问题？（→ 03）
- Nacos vs Apollo 怎么选？（→ 03）
- 长轮询原理？（→ 03）
- 灰度配置怎么做？（→ 03）
- API 网关解决什么？BFF 是什么？（→ 04）
- Kong vs APISIX vs Envoy 怎么选？（→ 04）
- 网关怎么防绕过？（→ 04）
- RPC vs HTTP？gRPC 比 JSON 快多少？（→ 05）
- Protobuf 兼容性铁律？（→ 05）
- gRPC vs Thrift？Kitex 为什么快？（→ 05）
- 客户端 LB 算法？P2C 是什么？（→ 05）
- Service Mesh 解决什么？真实代价？（→ 06）
- Sidecar 模式怎么工作？（→ 06）
- 什么时候该上 Mesh？（→ 06）
- 可观测性三支柱？（→ 07）
- OpenTelemetry 是什么？（→ 07）
- 链路追踪原理？trace_id 怎么透传？（→ 07）
- 四大黄金信号？（→ 07）

## 设计原则

- **图文并茂**，关键概念用 Mermaid
- **每篇独立可读**，高内聚优先于去重
- **结合真实项目** `ddd_order_example` 举例
- **大厂组件选型对比**：阿里 Dubbo+Nacos / 字节 Kitex / B 站 Kratos / 好未来 go-zero
- **Go 生态为主**，Java 周边对照
- **面试题驱动**，每章 10 道高频题 + 加分点

## 与其他模块的关系

- **DDD（09-ddd）**：提供战略边界划分（限界上下文 = 微服务候选）
- **架构（08-architecture）**：演进路径、高可用、扩展性、容量规划
- **分布式（06-distributed）**：理论基础、限流熔断、分布式锁/ID
- **消息队列（05-message-queue）**：服务间异步通信
- **缓存（04-redis）**：性能优化
- **CDN（11-cdn）**：边缘加速

四大模块串成完整微服务能力树。
