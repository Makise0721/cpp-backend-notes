# 04 — 消息队列与 Kafka 核心概念

> 📘 消息队列的更深入讨论见 [第十四章：消息队列](../../14-message-queue/README.md) — MQ 选型对比、RocketMQ/RabbitMQ 架构、事务消息、顺序消息、积压治理等。

---

## 1. 消息队列——解耦、削峰、异步

```
同步调用:                     异步（消息队列）:
  A ──req──→ B                 A ──msg──→ [MQ] ──msg──→ B
  A ←──resp─ B                 A 不等 B，继续做事

三大作用:
- 解耦:  A 不需要知道 B 的地址，只管发消息
- 削峰:  瞬时流量堆积在 MQ 中，B 按自己的速度处理
- 异步:  非关键路径操作（发邮件/写日志）异步化
```

---

## 2. Kafka 核心概念

### 2.1 Topic（主题）

消息的分类单位，类似"文件夹"。一个 Topic 可以有多个 Partition。

### 2.2 Partition（分区）——并行处理的基础

```
Topic: orders (3 个 Partition)

Partition 0: [msg0][msg1][msg2][msg3]...
Partition 1: [msg0][msg1][msg2]...
Partition 2: [msg0][msg1][msg2][msg3][msg4]...

每个 Partition 是**有序、不可变**的消息序列（append-only log）
```

**关键**：
- 同一个 Partition 内的消息**有序**
- 不同 Partition 之间**无序**
- Partition 数决定消费者的最大并行度

### 2.3 Offset——消息的唯一位置标识

```
Partition 0:
Offset:  0      1      2      3      4      5
        [msgA] [msgB] [msgC] [msgD] [msgE] [msgF]

每个 Consumer 记录自己读到哪个 offset
如果 Consumer 挂了 → 重启后从上次的 offset 继续
```

### 2.4 Consumer Group——消费者组

```
Topic: orders (3 Partition)
Consumer Group: order-processor (2 Consumer)

Consumer 1 → Partition 0, Partition 1
Consumer 2 → Partition 2

组内每个 Partition 只被一个 Consumer 消费
→ 组的并行度 ≤ Partition 数
```

```
多组独立消费:
Group A (实时处理): Consumer A1, A2 → 全部 Partition
Group B (离线分析): Consumer B1 → 全部 Partition
两组互相独立，各自有自己的 offset
```

### 2.5 Broker 与副本

```
Kafka Cluster:
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Broker 0     │  │  Broker 1     │  │  Broker 2     │
│ P0 (Leader)  │  │ P0 (Follower) │  │ P1 (Leader)  │
│ P1 (Follower)│  │ P2 (Leader)  │  │ P2 (Follower) │
└──────────────┘  └──────────────┘  └──────────────┘
```

- 每个 Partition 有一个 **Leader**（处理所有读写）+ N 个 **Follower**（同步备份）
- Leader 挂了 → Follower 中选举新 Leader
- 通过 ISR（In-Sync Replicas）保证数据一致性

---

## 3. Kafka 为什么快？

```
顺序写磁盘: 可以到 ~600 MB/s（接近磁盘物理极限）
            比随机写快 100 倍以上

Page Cache:   数据先写到 OS 的 Page Cache → 异步刷盘
              （利用了操作系统的缓存）

零拷贝:       sendfile() 直接把数据从 Page Cache 发到网卡
              （不经过用户态，不经过 CPU 拷贝）

批量压缩:     多条消息一起压缩（减少网络 + 磁盘开销）
```

---

## 4. Kafka 与其他 MQ 对比

| | Kafka | RabbitMQ | RocketMQ |
|------|:---:|:---:|:---:|
| 设计目标 | 高吞吐日志流 | 灵活路由 | Kafka-like + 事务 |
| 吞吐量 | **极高** (百万/秒) | 中 (万/秒) | 高 |
| 消息顺序 | Partition 内有序 | Queue 内有序 | 有序 |
| 消费模式 | Pull（消费者拉） | Push（MQ 推） | Pull |
| 消息回溯 | ✅ 天然支持 | ❌ | ✅ |
| 适用 | 日志/流处理/事件驱动 | 业务消息路由 | 电商交易 |

---

## 5. 消息队列常见问题

```
消息丢失？
├─ Producer: acks=all（等所有副本确认）
├─ Broker:   复制因子 ≥ 3, min.insync.replicas ≥ 2
└─ Consumer:  处理完再提交 offset

消息重复？
├─ Producer 重试 → 幂等性（enable.idempotence）
└─ Consumer: 业务层幂等 + offset 提交

消息积压？
├─ 增加 Partition 数 + Consumer 数
├─ 临时扩容 Consumer
└─ 降级：丢弃非关键消息
```

---

## 小结

| 概念 | 一句话 |
|------|--------|
| Topic | 消息分类 |
| Partition | 有序日志，并行度单位 |
| Offset | 消息在 Partition 中的位置 |
| Consumer Group | 组内分摊 Partition，组间独立 |
| ISR | 与 Leader 保持同步的副本集合 |

下一篇 [05 负载均衡与服务发现](./05-load-balance-service-discovery.md)。
