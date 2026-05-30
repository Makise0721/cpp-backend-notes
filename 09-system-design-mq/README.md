# 第九章：系统设计与消息队列

> 12 篇 · 从 CAP 定理到消息积压治理，覆盖分布式系统核心概念和消息队列全栈知识。

### 系统设计基础 (01-06)

| # | 核心内容 |
|---|------|
| 01 | CAP 定理(CP vs AP)、一致性哈希(虚拟节点) |
| 02 | 固定窗/滑动窗/漏桶/令牌桶、熔断器三态 |
| 03 | Snowflake/号段模式、Redis SETNX/RedLock/ZK 锁 |
| 04 | MQ 三大作用、Kafka Topic/Partition/Consumer Group/ISR |
| 05 | L4 vs L7 LB、轮询/最少连接/一致性哈希、Consul/Etcd |
| 06 | 幂等四方式、强一致/最终一致/因果一致 |

### 消息队列深入 (07-12)

| # | 核心内容 |
|---|------|
| 07 | Kafka/RocketMQ/RabbitMQ/Pulsar 对比、pull vs push |
| 08 | NameServer/Broker、CommitLog+ConsumeQueue、事务消息、延迟消息 |
| 09 | Exchange 类型(direct/topic/fanout)、ack/nack、TTL+DLX |
| 10 | 三段论不丢消息、分区有序、消费幂等 |
| 11 | RocketMQ 事务消息、本地消息表(Outbox)、CDC |
| 12 | 积压处理、死信队列(DLQ)、监控告警 |

路线：分布式理论(01-02)→协调服务(03-04)→服务治理(05-06)→MQ选型与架构(07-09)→可靠性实践(10-12)
