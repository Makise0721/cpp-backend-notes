# 03 — RabbitMQ 架构：Exchange、死信队列、ack/nack

> 关联篇目：[01 选型](./01-mq-selection.md) | [04 可靠性](./04-message-reliability-ordering.md)

---

## 1. AMQP 核心模型

```
AMQP (Advanced Message Queuing Protocol):

Producer → Exchange ──(binding key/routing key)→ Queue → Consumer
               │
          Exchange 类型:
          ├─ direct:  Routing Key == Binding Key (精确匹配)
          ├─ topic:   Routing Key 匹配 Binding Key 的通配符模式
          │           例: "order.*" 匹配 "order.create", "order.pay"
          ├─ fanout:  广播到所有绑定的 Queue
          └─ headers: 根据消息 Header 属性匹配（少用）
```

```
Direct Exchange:
                  ┌─ Queue A (binding: "error")
Producer ────────→│ Exchange ─┤
  routing: "info" │           └─ Queue B (binding: "info")
   → 只进入 Queue B

Topic Exchange:
                  ┌─ Queue A (binding: "order.*")
Producer ────────→│ Exchange ─┤
  routing:        │           └─ Queue B (binding: "*.pay")
  "order.pay"     └─ → 同时进入 Queue A 和 Queue B
```

---

## 2. 消息确认机制

RabbitMQ 有两层确认：

### 2.1 Producer Confirm（生产端确认）

```
同步 Confirm:
  发送消息 → 等待 Broker 确认（阻塞）

异步 Confirm:
  发送消息 → 异步回调收到 ack/nack

事务模式：
  txSelect → publish → txCommit（吞吐量极低，不推荐）
```

### 2.2 Consumer ACK/NACK（消费端确认）

```
autoAck = true（自动确认）:
  Broker 发出消息后立即标记为已确认
  消费者挂了 → 消息丢失 ❌

autoAck = false（手动确认）:
  basicAck:     确认消费成功
  basicNack:    否定（requeue=true 重新入队）
  basicReject:  拒绝单条

建议使用 manual ack + QOS(prefetch count)
```

---

## 3. TTL + 死信队列 = 延迟消息

RabbitMQ 本身不支持延迟消息，但通过 **TTL（Time-To-Live）+ 死信队列（DLX）** 变相实现：

```
┌──────────────────────────────────────────────┐
│                                              │
│  普通 Exchange                                │
│       │                                       │
│       ▼                                       │
│  ┌─────────┐   TTL 过期   ┌─────────────┐   │
│  │ 延迟队列  │───────────→│  死信 Exchange │   │
│  │(无消费者) │  变成死信    │              │   │
│  └─────────┘              └──────┬──────┘   │
│                                  │           │
│                                  ▼           │
│                            ┌─────────┐      │
│                            │ 目标队列  │      │
│                            │(有消费者) │      │
│                            └─────────┘      │
└──────────────────────────────────────────────┘

流程:
1. 消息发到"延迟队列"，设置 TTL=30min，且该队列声明了 DLX
2. 延迟队列没有消费者 → 30min 后 TTL 到期 → 消息变成"死信"
3. 死信被路由到 DLX → 进入真正的目标队列 → 消费者处理
```

```java
// 声明延迟队列
Map<String, Object> args = new HashMap<>();
args.put("x-dead-letter-exchange", "order-exchange");  // 死信转发目标
args.put("x-message-ttl", 30 * 60 * 1000);             // TTL 30min
channel.queueDeclare("order-delay-queue", true, false, false, args);
```

---

## 4. 消息持久化与高可用

```
Queue 持久化:   durable=true → Queue 元数据写入磁盘
消息持久化:     delivery_mode=2 → 消息写入磁盘
                                          ↓
                                    二者缺一不可

镜像队列 (Mirrored Queue):
  Queue 的 Master 在一个节点，Slave 在其他节点
  Master 挂了 → 最老的 Slave 升级为 Master
  (RabbitMQ 3.8+ 推荐用 Quorum Queue 替代)
```

---

## 5. RabbitMQ vs RocketMQ——什么时候用哪个？

| 场景 | 选择 |
|------|------|
| 复杂路由（多种匹配规则） | RabbitMQ（Exchange 类型丰富） |
| 低延迟要求（微秒级） | RabbitMQ |
| 事务消息/高吞吐/大量消息回溯 | RocketMQ |
| 微服务间 RPC 调用 | RabbitMQ（直接回复 + CorrelationId） |
| 简单的异步解耦 | 两者都可以 |

---

## 小结

| 概念 | 一句话 |
|------|--------|
| Exchange | 根据 routing key 路由消息到 Queue |
| direct/topic/fanout | 精确匹配/通配符/广播 |
| manual ack | 消费者手动确认，防止消息丢失 |
| TTL + DLX | 变相实现延迟消息 |
| 镜像队列 | 多节点备份，保证高可用 |

下一篇 [04 消息可靠性、顺序消息与幂等](./04-message-reliability-ordering.md)。
