# 05 — 事务消息与最终一致性

> 关联篇目：[08 RocketMQ 架构](./08-rocketmq-architecture.md) | [分布式事务](../07-database-storage/09-database-advanced.md) | [幂等性](./06-idempotency-consistency.md)

---

## 1. 问题场景：本地事务 + MQ 发送的原子性

```
电商下单:
1. 数据库: INSERT INTO orders ...  (本地事务)
2. 发送 MQ: 通知物流系统发货       (外部调用)

❌ 如果步骤 1 成功、步骤 2 失败（网络超时/宕机）→ 订单创建了但物流不知道
❌ 如果先发 MQ 再写数据库 → 消息发出但 DB 写失败 → 物流空跑

需要: 1 和 2 在同一个"事务"中 —— 要么都成功，要么都失败
```

---

## 2. 方案一：RocketMQ 事务消息（原生支持）

```
RocketMQ 事务消息流程:

Producer                         Broker                     Consumer
  │                                │                           │
  │── Send half message ─────────→│ (暂存，不可见)              │
  │                                │                           │
  │── Execute local transaction ──│                           │
  │   (更新订单状态等)              │                           │
  │                                │                           │
  ├─ COMMIT ───→│ (消息变为可见) ─────────→ 推送/拉取           │
  │             │   或            │                           │
  └─ ROLLBACK ─→│ (消息被删除)    │                           │
  │                                │                           │
  │   如果 Producer 挂了没回复     │                           │
  │   ←── Check back ────────────│                           │
  │   (Broker 主动回调查事务状态)  │                           │
```

```java
// RocketMQ 事务消息生产者
TransactionMQProducer producer = new TransactionMQProducer("group");
producer.setTransactionListener(new TransactionListener() {
    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        try {
            // 执行本地事务（如更新数据库）
            orderService.createOrder((Order) arg);
            return LocalTransactionState.COMMIT_MESSAGE;
        } catch (Exception e) {
            return LocalTransactionState.ROLLBACK_MESSAGE;
        }
    }

    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        // Broker 回查：根据 msg 中的业务 ID 查本地事务状态
        String orderId = msg.getKeys();
        Order order = orderService.getById(orderId);
        return order != null
            ? LocalTransactionState.COMMIT_MESSAGE
            : LocalTransactionState.ROLLBACK_MESSAGE;
    }
});
```

---

## 3. 方案二：本地消息表（通用方案，不依赖 MQ 特性）

```
适用: 任何 MQ（Kafka/RabbitMQ 都可用）

流程:
1. 本地事务内: 执行业务 + 插入消息表（同 DB、同事务）
   INSERT INTO orders (...);
   INSERT INTO outbox (id, topic, payload, status) VALUES (...);

2. 定时任务: 扫描 outbox 表 status=PENDING → 发送 MQ → 更新 status=SENT

3. 消费端: 处理消息 + 写入去重表（保证幂等）
```

```
┌────────────────────────────────────────┐
│              服务 A                     │
│  ┌──────────┐    ┌──────────────────┐  │
│  │ 本地事务   │───→│  outbox 消息表    │  │
│  │ DB操作    │    │ (同DB、同事务保证)  │  │
│  │ +插入outbox│   │                  │  │
│  └──────────┘    └────────┬─────────┘  │
│                           │ 定时扫描     │
│                           ▼            │
│                    发 MQ → Broker      │
└────────────────────────────────────────┘
```

### 3.1 实现要点

```sql
-- outbox 表
CREATE TABLE outbox (
    id        BIGINT AUTO_INCREMENT PRIMARY KEY,
    topic     VARCHAR(100) NOT NULL,
    msg_key   VARCHAR(100),
    payload   TEXT NOT NULL,
    status    TINYINT DEFAULT 0,  -- 0=PENDING, 1=SENT
    retries   INT DEFAULT 0,
    created_at DATETIME DEFAULT NOW()
);
CREATE INDEX idx_status ON outbox(status, created_at);
```

```java
// 定时扫描 + 发送
@Scheduled(fixedDelay = 1000)
void sendPendingMessages() {
    List<Outbox> pending = outboxMapper.selectPending(100);  // 每次 100 条
    for (Outbox msg : pending) {
        try {
            kafkaTemplate.send(msg.getTopic(), msg.getMsgKey(), msg.getPayload());
            outboxMapper.updateStatus(msg.getId(), SENT);
        } catch (Exception e) {
            outboxMapper.incrementRetry(msg.getId());  // 重试次数+1
        }
    }
}
```

---

## 4. 方案三：CDC（Change Data Capture）

```
不写 outbox 表，直接监听数据库 Binlog/WAL:

MySQL Binlog → Canal/Debezium → Kafka → 下游
         (CDC 工具)
```

| 方案 | 优点 | 缺点 |
|------|------|------|
| **事务消息** (RocketMQ) | 原生、简洁 | 仅 RocketMQ 支持 |
| **本地消息表** | 通用、简单 | 需要定时任务、消息可能延迟 |
| **CDC** | 零侵入业务代码 | 运维复杂、增加中间链路 |

---

## 小结

| 概念 | 一句话 |
|------|--------|
| 问题 | 本地 DB 操作和 MQ 发送需要原子性 |
| RocketMQ 事务消息 | half 消息 + 本地事务 + 回查 |
| 本地消息表 | outbox 表（同 DB 事务）+ 定时扫描发送 |
| CDC | 监听 Binlog → Canal/Debezium → Kafka |

下一篇 [06 消息积压治理](./06-backlog-governance.md)。
