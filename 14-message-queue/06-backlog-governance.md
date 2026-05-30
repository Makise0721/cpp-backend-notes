# 06 — 消息积压治理

> 关联篇目：[04 可靠性](./04-message-reliability-ordering.md) | [第十一章 限流熔断](../11-system-design/02-rate-limit-circuit-breaker.md)

---

## 1. 消息积压的原因

| 原因 | 现象 | 排查方式 |
|------|------|----------|
| **消费速度 < 生产速度** | 持续的积压增长 | 监控消费 TPS vs 生产 TPS |
| **消费者故障** | 积压突增，消费 TPS骤降为 0 | 检查 Consumer 存活状态 |
| **消费逻辑慢** | 单条消息处理耗时长 | 跟踪消息处理耗时 |
| **频繁 Rebalance** | 积压波动 | 看 Consumer Group 日志 |
| **下游依赖慢** | 消费不慢但处理不完 | 看下游调用耗时 |

---

## 2. 积压处理策略

### 2.1 临时扩容 Consumer（治标）

```
增加 Consumer 实例数 → 重新分配 Partition/Queue → 分摊压力

限制: Consumer 数 ≤ Partition/Queue 数
      Kafka → 增加 Partition（需提前规划，在线扩容有噪音）
      RocketMQ → 增加 Queue（需先扩容 Broker 的读写队列）
```

### 2.2 降级：批量转移到重试 Topic（治本）

```
当积压严重、Consumer 处理不过来时:
1. 原 Consumer 临时下线
2. 新建一个 Consumer 来消费积压，只做一件事:
   → 不做业务处理，直接把消息批量转发到一个"重试 Topic"
3. 重试 Topic 配更多 Partition → 更多的 Consumer 实例去真正处理
4. 原 Topic 积压清空后切回
```

### 2.3 跳过非关键消息

```java
// 消费时根据消息类型选择性处理
void consume(Message msg) {
    if (isFull() && msg.isNotCritical()) {
        return; // 丢弃非关键消息（如日志、统计数据）
    }
    process(msg);
}
```

### 2.4 提高消费并行度

```java
// 单 Consumer 内多线程并行处理（只对分区内不要求严格顺序的场景）
ExecutorService pool = Executors.newFixedThreadPool(10);

void consumeBatch(List<Message> msgs) {
    List<CompletableFuture<Void>> futures = msgs.stream()
        .map(msg -> CompletableFuture.runAsync(() -> process(msg), pool))
        .toList();
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
}
```

---

## 3. 死信队列（DLQ）——最后的兜底

**当消息重试次数超过阈值后，不再无限重试，而是转入死信队列，人工介入处理。**

```
Kafka: 消费失败 → 发送到死信 Topic（应用层实现）
RocketMQ: 重试 16 次后 → 自动进入 %DLQ%GroupName
RabbitMQ: 消息被 NACK/reject + requeue=false → 有 DLX 配置则转发
```

```java
// Kafka 消费端死信处理
void consume(ConsumerRecord record) {
    try {
        process(record);
    } catch (Exception e) {
        // 重试超过 3 次 → 发送到死信 Topic
        if (record.headers().lastHeader("retry_count") != null
            && getRetryCount(record) >= 3) {
            deadLetterProducer.send(new ProducerRecord("dead-topic", record.value()));
        } else {
            // 发送到重试 Topic，增加 retry_count header
            ProducerRecord retryRecord = new ProducerRecord("retry-topic", record.key(), record.value());
            retryRecord.headers().add("retry_count", ...);
            retryProducer.send(retryRecord);
        }
    }
}
```

**死信消息的处理流程**：
1. 监控死信队列的数量 → 达到阈值告警
2. 人工查看死信消息原因
3. 修复 Bug 后重新投递到原 Topic

---

## 4. 监控与告警指标

| 指标 | 含义 | 告警阈值示例 |
|------|------|-------------|
| **消息积压量** | 消费落后生产多少条 | > 10000 或持续增长 |
| **消费延迟** | 消息从发送到被消费的间隔 | P99 > 5s |
| **消费 TPS** | 每秒消费条数 | 突降 50% 告警 |
| **死信速率** | 每分钟进入死信队列的消息 | > 10/min |
| **Rebalance 频率** | Consumer 重平衡次数 | > 1 次/分钟 |
| **Topic 流量** | 生产速率 (MB/s) | 接近 Broker 带宽上限 |

```bash
# Kafka 消费积压监控
kafka-consumer-groups --bootstrap-server localhost:9092 \
   --group my-group --describe

# 输出:
# TOPIC   PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# orders  0          1000           1500            500  ← 积压 500 条
# orders  1          2000           2500            500
```

---

## 5. 实战：积压应急 SOP

```
1. 确认积压范围 → 哪些 Topic/Consumer Group
2. 确认原因 → 消费逻辑慢？下游挂了？Rebalance？
3. 应急止血:
   ├─ 下游正常 → 扩容 Consumer 实例
   ├─ 消费逻辑慢 → 先降级（转重试 Topic）
   └─ 非关键消息 → 临时丢弃
4. 恢复后切回正常链路
5. 复盘 + 优化消费性能
```

---

## 小结

| 问题 | 措施 |
|------|------|
| 消费跟不上 | 扩容 Consumer、增加 Partition |
| 突发积压 | 降级转发重试 Topic |
| 坏消息反复重试 | 死信队列（DLQ）兜底 |
| 无感知积压 | 监控 Lag + 消费延迟 + 告警 |

---

## 第十四章小结

| # | 篇目 | 核心 |
|---|------|------|
| 01 | MQ 选型 | Kafka/RocketMQ/RabbitMQ/Pulsar 对比 + pull vs push |
| 02 | RocketMQ 架构 | NameServer/CommitLog/ConsumeQueue/事务消息/延迟消息 |
| 03 | RabbitMQ 架构 | Exchange 类型/ack nack/TTL+DLX延迟消息 |
| 04 | 可靠性与顺序 | 三段论不丢消息/分区有序/消费幂等 |
| 05 | 事务消息 | RocketMQ事务消息/本地消息表(Outbox)/CDC |
| 06 | 积压治理 | 扩容/降级/死信队列/监控告警 |
