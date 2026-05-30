# 02 — RocketMQ 架构：NameServer、Broker、事务消息、延迟消息

> 关联篇目：[07 选型](./07-mq-selection.md) | [10 可靠性与顺序](./10-message-reliability-ordering.md)

---

## 1. 整体架构

```
RocketMQ 四角色:

┌──────────────┐
│  NameServer   │ ← 无状态、轻量级、多节点（非集群、彼此不通信）
│  (路由中心)    │
└──────┬───────┘
       │ 注册/发现
  ┌────┴────┐          ┌──────────────┐
  │ Broker  │←───────→│  Producer     │
  │ (存储)   │  发送消息  │              │
  │ Master  │          └──────────────┘
  │  +Slave │
  └────┬────┘          ┌──────────────┐
       │←─────────────→│  Consumer     │
       │  拉取消息      └──────────────┘
```

**与 Kafka 对比**：

| | RocketMQ | Kafka |
|------|------|------|
| 路由 | NameServer（去中心化，简单） | ZooKeeper/KRaft |
| 故障转移 | Broker Master-Slave | Partition Leader + ISR |
| 队列模型 | MessageQueue | Partition |

---

## 2. 存储设计：CommitLog + ConsumeQueue

```
RocketMQ 存储 = CommitLog(所有消息顺序写) + ConsumeQueue(按队列索引)

CommitLog:
[Msg1][Msg2][Msg3][Msg4][Msg5][Msg6]...  ← 所有Topic的消息全写同一个文件

ConsumeQueue(TopicA-Queue0):
[offset1,size1,tag1] [offset3,size3,tag3]... ← 索引指向 CommitLog

消费时:
Consumer → ConsumeQueue 读索引 → 定位 CommitLog 中的消息

优点: 所有写都是顺序写 → 极高吞吐
```

---

## 3. 消费模式

### 3.1 集群消费（默认）

```
Consumer Group 内分摊 MessageQueue:
Consumer A → Queue 0, Queue 1
Consumer B → Queue 2, Queue 3

组内每个 Queue 只被一个 Consumer 消费
消费进度保存在 Broker 端
```

### 3.2 广播消费

```
每个 Consumer 消费**所有** Queue 的全量消息:
Consumer A → Queue 0,1,2,3
Consumer B → Queue 0,1,2,3

消费进度保存在 Consumer 本地
```

---

## 4. 事务消息——分布式事务的核心

RocketMQ 是少数原生支持事务消息的 MQ。

```
流程（两阶段）:

1. Producer 发送 half(半) 消息 → Broker 暂存，对 Consumer 不可见
2. Producer 执行本地事务
3. 根据本地事务结果:
   ├─ COMMIT → Broker 将消息标记为可见 → Consumer 可消费
   └─ ROLLBACK → Broker 删除消息

4. 如果 Producer 挂了没回传结果 → Broker 主动回查(check) Producer
```

```
时序图:
Producer           Broker           Consumer
  │                  │                  │
  │── half msg ────→│                  │
  │                  │ (消息不可见)       │
  │ 执行本地事务      │                  │
  │                  │                  │
  │── COMMIT ──────→│                  │
  │                  │ (消息可见)         │
  │                  │─────────────────→│ 消费
```

---

## 5. 延迟消息

RocketMQ 支持 18 级延迟消息（不可自定义级别）：

```
1s, 5s, 10s, 30s, 1m, 2m, 3m, 4m, 5m, 6m, 7m, 8m, 9m, 10m, 20m, 30m, 1h, 2h

延迟消息的实现:
Broker 有 18 个延迟队列（SCHEDULE_TOPIC_XXXX, level 0~17）
延迟消息先写入对应的延迟队列 → 定时任务轮询检查 → 到期后写入实际 Topic
```

**场景**：下单后 30 分钟未支付自动取消、定时提醒。

---

## 小结

| 概念 | 一句话 |
|------|--------|
| NameServer | 无状态路由中心，Producer/Consumer 定时拉取路由 |
| CommitLog | 所有消息顺序写的日志文件 |
| ConsumeQueue | 按 MessageQueue 的索引，指向 CommitLog |
| 事务消息 | half 消息 + 本地事务 + 回查 |
| 延迟消息 | 18 级，定时写入目标 Topic |

下一篇 [03 RabbitMQ 架构](./03-rabbitmq-architecture.md)。
