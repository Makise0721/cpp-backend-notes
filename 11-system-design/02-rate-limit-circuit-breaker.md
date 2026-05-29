# 02 — 限流算法与熔断器

> 关联篇目：[01 CAP/一致性哈希](./01-cap-consistent-hashing.md)

---

## 1. 限流算法

### 1.1 固定窗口——最简单但有缺陷

```
时间窗口 [0s, 1s)，允许 100 个请求

      ┌─────────┐     ┌─────────┐
      │ 窗口 0-1s│     │ 窗口 1-2s│
      │ 100 req  │     │ 100 req  │
      └─────────┘     └─────────┘

问题：窗口边界的流量突刺
  0.9s-1.0s: 100 req
  1.0s-1.1s: 100 req
  0.9s-1.1s 实际过了 200 req（窗口边界无法感知）
```

### 1.2 滑动窗口——更平滑

```
固定窗口 ×100                               滑动窗口 ✓
┌────┬────┬────┬────┐        ┌───┬───┬───┬───┬───┬───┐
│ 0.5│ 1.0│ 1.5│ 2.0│        │0.5│0.7│0.9│1.1│1.3│1.5│
└────┴────┴────┴────┘        └───┴───┴───┴───┴───┴───┘
                                   └── 窗口滑动 ──┘
                                   只看最近 1 秒的请求

实现: 记录每次请求的时间戳，查询时统计最近 1 秒内的数量
或: Redis ZSet (score=时间戳, member=request_id)
```

### 1.3 漏桶（Leaky Bucket）——恒定流速

```
        请求流入 (不限速)
           │  │  │  │  │
           ▼  ▼  ▼  ▼  ▼
        ┌──────────────┐
        │   漏桶 (队列)  │
        │  容量: 100    │ ← 满了就丢弃
        └──────┬───────┘
               │ 恒定速率流出 (如 10 req/s)
               ▼
            处理器

特点: 削峰填谷——突发流量被平滑化
适用: 需要严格恒定输出速率的场景
```

### 1.4 令牌桶（Token Bucket）——允许突发

```
        令牌以恒定速率入桶 (如 10 tokens/s)
           │  │  │  │
           ▼  ▼  ▼  ▼
        ┌──────────────┐
        │   令牌桶       │
        │  容量: 100    │ ← 桶容量 = 允许的最大突发
        └──────┬───────┘
               │ 请求到达 → 取令牌
               │ 有令牌 → 放行
               │ 无令牌 → 限流

特点: 桶容量允许一定程度的突发（只要桶里有令牌）
适用: 大多数 API 限流（允许短期突发，长期均值不超）
```

```cpp
// 令牌桶简化实现
class TokenBucket {
    double capacity_, tokens_, rate_;       // 容量、当前令牌数、速率(tokens/s)
    std::chrono::steady_clock::time_point last_;
    std::mutex mtx_;
public:
    TokenBucket(double rate, double capacity)
        : capacity_(capacity), tokens_(capacity), rate_(rate),
          last_(std::chrono::steady_clock::now()) {}

    bool tryConsume(int n = 1) {
        std::lock_guard lock(mtx_);
        auto now = std::chrono::steady_clock::now();
        double elapsed = std::chrono::duration<double>(now - last_).count();
        tokens_ = std::min(capacity_, tokens_ + elapsed * rate_);
        last_ = now;
        if (tokens_ >= n) { tokens_ -= n; return true; }
        return false;
    }
};
```

### 1.5 限流算法对比

| 算法 | 平滑 | 允许突发 | 实现复杂度 | 适用 |
|------|:---:|:---:|:---:|------|
| 固定窗口 | ❌ | 边界突刺 | 低 | 简单场景 |
| 滑动窗口 | ✅ | 控制 | 中 | API 限流 |
| 漏桶 | ✅ | ❌ | 中 | 流量整形 |
| 令牌桶 | ✅ | ✅ | 中 | **通用 API 限流首选** |

---

## 2. 熔断器（Circuit Breaker）——防止级联雪崩

### 2.1 问题

服务 A 调用服务 B，B 故障 → A 的请求大量超时 → A 的线程池耗尽 → A 也故障 → 级联雪崩。

### 2.2 三态模型

```
        ┌──────────┐
        │  CLOSED  │ ← 正常状态，请求正常通过
        │ (闭合)    │
        └────┬─────┘
             │ 失败次数 > 阈值
             ▼
        ┌──────────┐
        │   OPEN   │ ← 熔断：请求直接失败，不调用下游
        │ (断开)    │    (fail-fast，快速失败)
        └────┬─────┘
             │ 超时时间到
             ▼
        ┌──────────┐
        │ HALF_OPEN│ ← 半开：允许少量探测请求
        │ (半开)    │
        └────┬─────┘
        成功→CLOSED    失败→OPEN
```

```cpp
class CircuitBreaker {
    enum State { CLOSED, OPEN, HALF_OPEN };
    State state_ = CLOSED;
    int failureCount_ = 0, failureThreshold_, halfOpenMax_;
    std::chrono::steady_clock::time_point openTime_;
    std::chrono::milliseconds timeout_;

public:
    bool allowRequest() {
        if (state_ == CLOSED) return true;
        if (state_ == OPEN) {
            if (std::chrono::steady_clock::now() - openTime_ > timeout_) {
                state_ = HALF_OPEN;
                return true;  // 允许一个探测请求
            }
            return false;
        }
        // HALF_OPEN: 允许少量请求
        return true;
    }

    void recordSuccess() {
        if (state_ == HALF_OPEN) { state_ = CLOSED; failureCount_ = 0; }
    }

    void recordFailure() {
        failureCount_++;
        if (state_ == HALF_OPEN || failureCount_ >= failureThreshold_) {
            state_ = OPEN;
            openTime_ = std::chrono::steady_clock::now();
        }
    }
};
```

---

## 小结

| 概念 | 一句话 |
|------|--------|
| 固定窗口 | 简单但有边界突刺 |
| 滑动窗口 | 平滑，适合 API 限流 |
| 漏桶 | 恒定速率，不允许突发 |
| 令牌桶 | 允许突发，通用首选 |
| 熔断器 | CLOSED→OPEN→HALF_OPEN，防级联雪崩 |

下一篇 [03 分布式 ID 与分布式锁](./03-distributed-id-lock.md)。
