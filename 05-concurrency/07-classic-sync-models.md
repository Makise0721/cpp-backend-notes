# 07 — 经典同步模型：生产者-消费者、读者-写者、哲学家就餐

> 关联篇目：[03 条件变量](./03-cv-semaphore-barrier.md) | [02 互斥锁](./02-mutex-locks.md)

---

## 1. 生产者-消费者模型

### 1.1 有界队列（Bounded Buffer）

```cpp
template<typename T>
class BoundedQueue {
    std::queue<T> queue_;
    std::mutex mtx_;
    std::condition_variable notFull_, notEmpty_;
    size_t cap_;

public:
    explicit BoundedQueue(size_t cap) : cap_(cap) {}

    void produce(T item) {
        std::unique_lock lock(mtx_);
        notFull_.wait(lock, [this] { return queue_.size() < cap_; });
        queue_.push(std::move(item));
        notEmpty_.notify_one();
    }

    T consume() {
        std::unique_lock lock(mtx_);
        notEmpty_.wait(lock, [this] { return !queue_.empty(); });
        T item = std::move(queue_.front());
        queue_.pop();
        notFull_.notify_one();
        return item;
    }
};
```

### 1.2 信号量实现

```cpp
// 两个信号量替代条件变量
std::counting_semaphore<100> emptySlots(100);  // 空位数
std::counting_semaphore<100> fullSlots(0);     // 已填充数
std::mutex mtx;

void producer(T item) {
    emptySlots.acquire();   // 等待空位
    {
        std::lock_guard lock(mtx);
        queue.push(item);
    }
    fullSlots.release();    // 增加已填充数
}

void consumer() {
    fullSlots.acquire();    // 等待有数据
    T item;
    {
        std::lock_guard lock(mtx);
        item = queue.front(); queue.pop();
    }
    emptySlots.release();   // 增加空位
}
```

**有界 vs 无界**：
- 有界：用两个条件变量（`notFull` + `notEmpty`）或两个信号量
- 无界：只需一个条件变量（`notEmpty`），生产者永不阻塞

---

## 2. 读者-写者问题

### 2.1 读优先（读者优先——写者可能饿死）

```cpp
class ReadPriority {
    std::shared_mutex rwMtx_;  // 保护数据
    std::mutex readCountMtx_;
    int readers_ = 0;
public:
    void read() {
        {
            std::lock_guard lock(readCountMtx_);
            if (++readers_ == 1)
                rwMtx_.lock();        // 第一个读者锁住写锁
        }
        // 读数据
        {
            std::lock_guard lock(readCountMtx_);
            if (--readers_ == 0)
                rwMtx_.unlock();      // 最后一个读者释放写锁
        }
    }

    void write() {
        std::lock_guard lock(rwMtx_);
        // 写数据——可能饿死（读者不断来，readers 永远 > 0）
    }
};
```

### 2.2 写优先

在读者入口加一个"写者等待"标记，新来的读者发现有写者在等 → 让写者先走。

### 2.3 C++17 直接用 `shared_mutex`（公平性由实现决定）

```cpp
std::shared_mutex mtx;

void read() {
    std::shared_lock lock(mtx);  // 多读并发
}
void write() {
    std::unique_lock lock(mtx);  // 写独占
}
```

---

## 3. 哲学家就餐问题

```
五位哲学家围坐圆桌，每人面前一盘意面，相邻之间有一把叉子。
哲学家只能: 思考 → 拿左边叉子 → 拿右边叉子 → 进餐 → 放下叉子 → 思考...
问题: 如果五人同时拿起左边叉子 → 死锁！
```

### 3.1 解决方案：限制同时进餐人数

```cpp
class DiningPhilosophers {
    std::mutex forks_[5];
    std::counting_semaphore<4> limit_{4};  // 最多 4 人同时试图进餐
public:
    void eat(int id) {
        int left = id, right = (id + 1) % 5;

        limit_.acquire();            // 只有 4 人能通过 → 至少一人能拿到两把叉

        // 统一加锁顺序（奇数先左后右，偶数先右后左）也可以防死锁
        std::scoped_lock lock(forks_[left], forks_[right]);

        // 进餐...
    }  // forks 释放，limit_ 也需要释放
};
```

---

## 4. 同步模型选择速查

```
任务分发: 生产者-消费者
  ├─ 单生产单消费 → SPSC 无锁队列
  ├─ 多生产多消费 → 有界队列 + 条件变量
  └─ 线程池 → 任务队列 + 工作线程

多读少写: 读者-写者 → shared_mutex

资源分配防死锁:
  ├─ 统一加锁顺序
  ├─ scoped_lock 同时锁定
  └─ 限制并发数（如哲学家：semaphore 限流）
```

---

## 小结

| 模型 | 关键 | 防死锁手段 |
|------|------|------------|
| 生产者-消费者 | 两个条件变量/信号量控制缓冲区 | — |
| 读者-写者 | 读共享写独占 | 注意写者饥饿 |
| 哲学家 | 同时拿两把叉 | 限流 / 统一顺序 / 奇数偶数不同顺序 |

下一篇 [08 C++ 异步编程：future/promise/async/线程池](./08-cpp-async.md)。
