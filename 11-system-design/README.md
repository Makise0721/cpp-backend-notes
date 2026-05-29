# 第十一章：系统设计基础

> 6 篇 · 分布式系统核心概念——面试和实际架构都绕不开的基础知识。

---

## 篇目

| # | 文件 | 核心内容 |
|---|------|----------|
| 01 | [01-cap-consistent-hashing.md](./01-cap-consistent-hashing.md) | CAP 定理(CP vs AP)、一致性哈希(虚拟节点/环)、PACELC |
| 02 | [02-rate-limit-circuit-breaker.md](./02-rate-limit-circuit-breaker.md) | 固定窗口/滑动窗口/漏桶/令牌桶、熔断器(CLOSED→OPEN→HALF_OPEN) |
| 03 | [03-distributed-id-lock.md](./03-distributed-id-lock.md) | Snowflake/号段模式/Redis自增、Redis SETNX/RedLock/ZK 临时顺序节点 |
| 04 | [04-message-queue-kafka.md](./04-message-queue-kafka.md) | MQ 三大作用、Kafka Topic/Partition/Consumer Group/ISR、为什么快 |
| 05 | [05-load-balance-service-discovery.md](./05-load-balance-service-discovery.md) | L4 vs L7 LB、轮询/最少连接/一致性哈希、Consul/Etcd/Nacos |
| 06 | [06-idempotency-consistency.md](./06-idempotency-consistency.md) | 幂等四方式(唯一索引/Token/状态机/乐观锁)、强一致/最终一致/因果一致 |

## 路线

分布式理论(1)→流量控制(2)→协调服务(3)→消息系统(4)→服务治理(5-6)。
