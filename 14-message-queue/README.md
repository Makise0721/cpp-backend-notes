# 第十四章：消息队列

> 6 篇 · 从选型到治理，覆盖消息队列的核心架构与工程实践。

---

## 篇目

| # | 文件 | 核心内容 |
|---|------|----------|
| 01 | [01-mq-selection.md](./01-mq-selection.md) | Kafka/RocketMQ/RabbitMQ/Pulsar 全面对比、pull vs push、选型决策树 |
| 02 | [02-rocketmq-architecture.md](./02-rocketmq-architecture.md) | NameServer/Broker、CommitLog+ConsumeQueue、事务消息、延迟消息 |
| 03 | [03-rabbitmq-architecture.md](./03-rabbitmq-architecture.md) | Exchange(direct/topic/fanout)、ack/nack、TTL+DLX 延迟消息 |
| 04 | [04-message-reliability-ordering.md](./04-message-reliability-ordering.md) | 三段论不丢消息、顺序消息（分区有序）、消费幂等 |
| 05 | [05-message-transaction.md](./05-message-transaction.md) | RocketMQ 事务消息、本地消息表(Outbox)、CDC |
| 06 | [06-backlog-governance.md](./06-backlog-governance.md) | 积压原因与处理、死信队列(DLQ)、监控告警指标 |

## 前置

Kafka 核心概念（Topic/Partition/Consumer Group/ISR）见 [第十一章 消息队列与 Kafka](../11-system-design/04-message-queue-kafka.md)。
