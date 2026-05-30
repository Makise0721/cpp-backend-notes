# 03 — 分布式 ID 生成与分布式锁

---

## 1. 分布式 ID 生成——全局唯一 ID

### 1.1 需求

- **全局唯一**：不同节点不能生成相同的 ID
- **趋势递增**：数据库索引友好（B+ 树）
- **高性能**：生成速度要快

### 1.2 Snowflake（雪花算法）——Twitter 的方案

```
64 位整数:
┌─┬─────────────────────┬───────────┬──────────┐
│0│  41 bit 毫秒时间戳    │ 10 bit    │ 12 bit   │
│ │  (可用 69 年)        │ 机器 ID    │ 序列号    │
└─┴─────────────────────┴───────────┴──────────┘

每毫秒每台机器可生成 4096 个 ID
理论峰值 ~400 万 ID/秒/节点
```

```cpp
class Snowflake {
    uint64_t workerId_;
    uint64_t sequence_ = 0;
    uint64_t lastTimestamp_ = 0;
    const uint64_t epoch_ = 1700000000000;  // 起始时间戳 (2023-11-14)

public:
    Snowflake(uint64_t workerId) : workerId_(workerId & 0x3FF) {} // 10 bit

    uint64_t nextId() {
        uint64_t ts = currentMillis();
        if (ts == lastTimestamp_) {
            sequence_ = (sequence_ + 1) & 0xFFF;  // 12 bit mask
            if (sequence_ == 0) ts = waitNextMillis(lastTimestamp_);
        } else {
            sequence_ = 0;
        }
        lastTimestamp_ = ts;
        return ((ts - epoch_) << 22)        // 时间戳
             | (workerId_ << 12)            // 机器 ID
             | sequence_;                   // 序列号
    }
};
```

**Snowflake 的问题**：
- 依赖机器时钟，时钟回拨导致 ID 重复
- 解决方案：等待时钟追上、用额外 counter 标记

### 1.3 号段模式

```
从数据库批量取一段 ID（如 1000-2000），
在内存中逐个分配，用完了再取下一段。

减少数据库访问次数（取一次 = 1000 个 ID）
```

### 1.4 Redis 自增

```redis
INCRBY order_id 1  → 简单但有单点风险
# 或每个服务使用不同的步长
INCRBY order_id 2  # 服务 A
INCRBY order_id 2  # 服务 B（从 1 开始，奇数给 A，偶数给 B）
```

---

## 2. 分布式锁——跨进程的互斥

### 2.1 Redis `SETNX` + 过期时间

```redis
# 加锁（原子操作）
SET lock_key unique_value NX PX 30000
#   key     锁持有者标识   仅在不存在时设置  30s 过期

# 释放锁（Lua 脚本，原子地验证 + 删除）
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

**为什么需要 unique_value？** 防止释放别人的锁。A 的锁过期 → B 获取锁 → A 释放了 B 的锁。

### 2.2 RedLock——Redis 官方的多节点方案

```
1. 获取 N 个独立 Redis 实例上的锁（N 通常 = 5）
2. 如果在大多数实例上（N/2+1）成功获取，且总耗时 < 锁有效期 → 加锁成功
3. 否则释放所有已获取的锁
```

**争议**：Martin Kleppmann 论证 RedLock 在极端情况下不安全，但多数场景够用。

### 2.3 ZooKeeper 临时顺序节点

```
1. 每个客户端在 /lock 下创建临时顺序节点
2. 序号最小的节点获得锁
3. 其他客户端 watch 前一个节点，前一个删除 → 自己变成最小 → 获得锁
4. 客户端断开 → 临时节点自动删除 → 锁自动释放
```

| | Redis | ZooKeeper |
|------|:---:|:---:|
| 安全性 | 依赖过期时间 | 强（临时节点+watch） |
| 性能 | 高 | 中 |
| 死锁风险 | 有（过期前宕机） | 低（自动释放） |
| 实现复杂度 | 低 | 中 |

---

## 3. 分布式锁的使用模式

```java
// 伪代码
String lockValue = UUID.randomUUID().toString();
boolean locked = redis.set(lockKey, lockValue, "NX", "PX", 30000);

if (locked) {
    try {
        // 业务逻辑
    } finally {
        // Lua 脚本释放：检查 value == lockValue
        releaseLock(lockKey, lockValue);
    }
}
```

**关键点**：
- 🔑 必须设置过期时间（防止死锁）
- 🔑 释放时必须验证 ownership（防止释放别人的锁）
- 🔑 如果业务执行超过锁过期时间 → 使用看门狗（Watchdog）自动续期

---

## 小结

| 概念 | 一句话 |
|------|--------|
| Snowflake | 41bit 时间戳 + 10bit 机器 + 12bit 序列号 |
| 号段模式 | 批量取 ID 范围，内存分配 |
| Redis SETNX | 简单版本锁，注意过期 + ownership 检查 |
| RedLock | 多 Redis 节点 + 多数投票 |
| ZK 分布式锁 | 临时顺序节点 + watch，自动释放 |

下一篇 [04 消息队列与 Kafka 基础](./04-message-queue-kafka.md)。
