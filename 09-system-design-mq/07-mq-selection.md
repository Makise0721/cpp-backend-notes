# 01 — 消息队列选型：Kafka / RocketMQ / RabbitMQ / Pulsar

> 关联篇目：[08 RocketMQ 架构](./08-rocketmq-architecture.md) | [09 RabbitMQ 架构](./09-rabbitmq-architecture.md) | [Kafka 基础](./04-message-queue-kafka.md)

---

## 1. 四大 MQ 概览

| | Kafka | RocketMQ | RabbitMQ | Pulsar |
|------|:---:|:---:|:---:|:---:|
| 开发方 | LinkedIn→Apache | Alibaba→Apache | Pivotal(原名) | Yahoo→Apache |
| 语言 | Java/Scala | Java | Erlang | Java |
| 吞吐量 | **极高**(百万/秒) | 高(十万/秒) | 中(万/秒) | 高 |
| 延迟 | 毫秒级 | 毫秒级 | 微秒级 | 毫秒级 |
| 消息模型 | 发布-订阅(Consumer Group) | 发布-订阅+广播 | 灵活路由 | 发布-订阅+多租户 |
| 消费模式 | Pull | Pull | Push/Pull | Pull |
| 消息回溯 | ✅(按 offset) | ✅(按时间) | ❌ | ✅(Cursor) |
| 事务消息 | ❌(需外挂) | ✅(原生) | ❌ | ❌ |
| 延迟消息 | ❌ | ✅(18 级) | ✅(死信+TTL) | ✅(delay delivery) |
| 存储 | 文件系统(分段日志) | CommitLog+ConsumeQueue | 内存/磁盘 | BookKeeper |
| 社区/生态 | ✅✅✅ | ✅✅(国内强) | ✅✅✅ | ✅ |

---

## 2. 核心模型差异

### 2.1 Kafka：日志模型

```
Kafka = 分布式日志系统

Topic → Partition(有序日志) → Consumer Group(组内分摊)
消息 = 持久化日志记录，不删（按时间/大小保留）
消费 = 消费者自己记 offset，可回溯

适用: 日志收集、流处理(kSql/Flink)、事件驱动
```

### 2.2 RocketMQ：队列模型 + 日志存储

```
RocketMQ = Kafka-like 存储 + 丰富的消息特性

Topic → MessageQueue → Consumer Group(组内分摊+广播)
存储 = CommitLog(顺序写,类似 Kafka) + ConsumeQueue(按队列索引)
额外: 事务消息、定时消息、消息轨迹

适用: 电商交易、金融支付、需要事务/延迟的场景
```

### 2.3 RabbitMQ：灵活路由模型

```
RabbitMQ = AMQP 标准实现

Producer → Exchange → (binding) → Queue → Consumer
Exchange 类型: direct(精确匹配)/topic(通配符)/fanout(广播)/headers

适用: 复杂路由、微服务间 RPC、低延迟(微秒级)
```

### 2.4 Pulsar：计算与存储分离

```
Pulsar = Broker(无状态计算层) + BookKeeper(分布式存储层)

优势: 存储计算分离→独立扩缩、多租户、跨地域复制
劣势: 运维复杂、学习成本高

适用: 大规模多租户场景、需要跨 DC 复制
```

---

## 3. Pull vs Push 消费模式

| | Pull（拉模式） | Push（推模式） |
|------|:---:|:---:|
| 谁控制速率 | **消费者** | MQ 服务端 |
| 背压处理 | 自然（消费慢→不拉） | 需流量控制（prefetch） |
| 长轮询 | ✅（Kafka/RocketMQ 原生支持） | ❌ |
| 延迟 | 略高（拉间隔） | 更低 |
| 代表 | Kafka、RocketMQ、Pulsar | RabbitMQ(原生Push) |

**为什么 Kafka/RocketMQ 选 Pull？**
- 消费者最清楚自己的处理能力
- 避免推送速率超过消费速率导致消息积压或被丢弃
- 配合长轮询（Long Polling），没有消息时挂起等待，有消息立即返回

---

## 4. 如何选型？

```
选型决策树:

需要消息回溯/流处理？
├─ 是 → Kafka
└─ 否
    ├─ 需要事务消息/延迟消息/顺序消息？
    │   └─ 是 + 国内团队 → RocketMQ
    │
    ├─ 需要复杂路由/低延迟/微秒级？
    │   └─ 是 → RabbitMQ
    │
    └─ 大规模多租户/跨 DC？
        └─ 是 → Pulsar
```

**国内互联网的典型选择**：
- 日志/埋点/流处理 → Kafka
- 交易/支付/订单消息 → RocketMQ
- 内部微服务异步通信 → RabbitMQ 或 RocketMQ
- 大型平台（千级租户） → Pulsar

---

## 小结

| 维度 | 最佳选择 |
|------|----------|
| 最高吞吐 | Kafka |
| 事务/延迟/顺序消息 | RocketMQ |
| 灵活路由/低延迟 | RabbitMQ |
| 存储计算分离/多租户 | Pulsar |

下一篇 [02 RocketMQ 架构](./02-rocketmq-architecture.md)。
