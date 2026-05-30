# 04 — 消息可靠性、顺序消息与消费幂等

> 关联篇目：[02 RocketMQ](./02-rocketmq-architecture.md) | [03 RabbitMQ](./03-rabbitmq-architecture.md) | [第十一章 幂等性设计](../11-system-design/06-idempotency-consistency.md)

---

## 1. 消息可靠性的三段论

消息从 Producer 到 Consumer 经过三段，每段都可能丢消息：

```
Producer → Broker → Consumer
  ①         ②        ③
```

### 1.1 生产端：保证消息发到 Broker

| MQ | 配置 |
|------|------|
| **Kafka** | `acks=all`(等所有 ISR 确认) + `retries > 0` + `enable.idempotence=true` |
| **RocketMQ** | 同步发送 + 重试 + `sendMsgTimeout` |
| **RabbitMQ** | Publisher Confirm(异步确认) + `mandatory=true`(消息无法路由时回调) |

```java
// Kafka 生产者可靠性配置
props.put("acks", "all");                       // 等所有副本确认
props.put("retries", Integer.MAX_VALUE);         // 无限重试
props.put("enable.idempotence", true);           // 幂等（防重试重复）
props.put("max.in.flight.requests.per.connection", 5); // 配合幂等可用
```

### 1.2 Broker 端：保证消息不丢

| MQ | 配置 |
|------|------|
| **Kafka** | `replication.factor ≥ 3` + `min.insync.replicas ≥ 2` + `unclean.leader.election.enable=false` |
| **RocketMQ** | `flushDiskType=SYNC_FLUSH`(同步刷盘) + Master-Slave 同步复制 |
| **RabbitMQ** | `durable=true` + `delivery_mode=2`(持久化) + 镜像队列/Quorum Queue |

### 1.3 消费端：保证消息被正确处理

| MQ | 配置 |
|------|------|
| **Kafka** | 关闭自动提交 → `enable.auto.commit=false` → 处理完后手动 `commitSync()` |
| **RocketMQ** | `ConsumeConcurrentlyStatus.RECONSUME_LATER`(消费失败返回重试) |
| **RabbitMQ** | `autoAck=false` → 手动 `basicAck` → 配合 QOS prefetch count |

```java
// Kafka 消费端手动提交
props.put("enable.auto.commit", "false");
while (true) {
    ConsumerRecords records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord record : records) {
        process(record);  // 处理消息
    }
    consumer.commitSync();  // 处理完再提交 offset
}
```

**注意**：先处理再提交 → **at-least-once**（可能重复消费）。如果要 exactly-once → 需要消费端幂等或依靠 Kafka 事务。

---

## 2. 顺序消息——全局有序 vs 分区有序

### 2.1 为什么有乱序？

```
多 Partition/Queue 并发消费 → 消息处理的顺序 ≠ 发送的顺序

Producer 按 A→B→C 发送到不同的 Partition:
Partition 0: [A] → Consumer 1 慢 → 最后完成
Partition 1: [B] → Consumer 2 快 → 先完成
Partition 2: [C] → Consumer 3 中 → ...

实际处理顺序: B→C→A（乱序！）
```

### 2.2 全局有序

**所有消息发到同一个 Partition/Queue**，用一个线程串行消费。简单但丧失了并行性。

### 2.3 分区有序（推荐）

**相同业务 key（如订单 ID）的消息发到同一个 Partition/Queue**。

```
Kafka:
producer.send(new ProducerRecord("topic", orderId, message));
//                                     ↑ key → hash 到固定 Partition

RocketMQ:
Message msg = new Message("topic", "tag", orderId, body.getBytes());
producer.send(msg, (mqs, msg1, arg) -> {
    long orderId = (long) arg;
    int idx = (int) (orderId % mqs.size());  // 自己选 Queue
    return mqs.get(idx);
}, orderId);
```

### 2.4 消费端保证顺序

仅靠生产端分区还不够——消费端也必须串行处理同一个 Partition 的消息：

```
Kafka: 一个 Partition 只分配给 Consumer Group 中的一个 Consumer
       → 该 Consumer 单线程消费 → 天然保序

RocketMQ:
MessageListenerOrderly → Broker 对同 Queue 消息加分布式锁
                        → 同一 Queue 串行消费
```

---

## 3. 消费幂等——at-least-once 的防线

**MQ 只能保证 at-least-once**（网络超时重试、Consumer 崩溃+重平衡）。

```java
// 幂等消费框架
void consume(Message msg) {
    String msgId = msg.getMsgId();  // 或业务唯一 ID

    // 方案 1: Redis 去重
    if (!redis.setNX("msg:" + msgId, "1", 60, TimeUnit.SECONDS)) {
        return;  // 已处理
    }

    // 方案 2: DB 唯一索引
    try {
        db.insert(new ProcessedMsg(msgId));
    } catch (DuplicateKeyException e) {
        return;  // 已处理
    }

    // 实际业务处理
    doBusiness(msg);
}
```

---

## 小结

| 问题 | 解决方案 |
|------|----------|
| 生产端丢 | ack=all + retry + 幂等 |
| Broker 丢 | 多副本 + 同步刷盘 + min.insync.replicas |
| 消费端丢 | 手动提交 offset + autoAck=false |
| 顺序消息 | 同 key → 同 Partition + MessageListenerOrderly |
| 重复消费 | 消费端幂等（Redis/DB 去重） |

下一篇 [05 事务消息与最终一致性](./05-message-transaction.md)。
