# 02 — 互斥锁、读写锁与 RAII 锁包装

> 关联篇目：[01 线程基础](./01-thread-basics.md) | [03 条件变量](./03-cv-semaphore-barrier.md)

---

## 1. 互斥锁家族

### 1.1 `std::mutex`——基本互斥锁

```cpp
std::mutex mtx;
int counter = 0;

void increment() {
    mtx.lock();
    ++counter;       // 临界区
    mtx.unlock();    // ⚠️ 如果中间抛异常，unlock 不会执行！
}
```

### 1.2 `std::recursive_mutex`——同一线程可多次加锁

```cpp
std::recursive_mutex rmtx;

void recursive(int n) {
    rmtx.lock();
    if (n > 0) recursive(n - 1);  // 同一线程再次 lock → 不会死锁
    rmtx.unlock();  // 每次 lock 必须对应一次 unlock
}
```

### 1.3 `std::timed_mutex`——带超时的锁

```cpp
std::timed_mutex tmtx;

if (tmtx.try_lock_for(std::chrono::milliseconds(100))) {
    // 100ms 内获取到锁
    tmtx.unlock();
} else {
    // 超时，去做别的事
}

// 也可用绝对时间点
if (tmtx.try_lock_until(deadline)) { /* ... */ }
```

---

## 2. 互斥锁 vs 自旋锁

| | 互斥锁 (`std::mutex`) | 自旋锁 (`std::atomic_flag`) |
|------|------|------|
| 等待方式 | **睡眠等待**（让出 CPU） | **忙等待**（循环检查） |
| 内核参与 | ✅（`futex` 系统调用） | ❌ 纯用户态 |
| CPU 开销（等待时） | 低（CPU 可跑其他任务） | 高（空转消耗 CPU） |
| 上下文切换 | 有 | 无 |
| 适合场景 | 临界区较长（> 几微秒） | 临界区极短（几指令） |
| 实现 | 基于 `futex` | `lock cmpxchg` 循环 |

```cpp
// 自旋锁简单实现
class SpinLock {
    std::atomic_flag flag_ = ATOMIC_FLAG_INIT;
public:
    void lock()   { while (flag_.test_and_set(std::memory_order_acquire)); }
    void unlock() { flag_.clear(std::memory_order_release); }
};

// 使用场景：保护几行赋值或指针交换
SpinLock sl;
sl.lock();
queue_->push(item);  // 极短的临界区
sl.unlock();
```

---

## 3. 读写锁——读共享，写独占

```cpp
#include <shared_mutex>

class ThreadSafeCache {
    mutable std::shared_mutex mtx_;
    std::map<int, std::string> data_;
public:
    // 读操作——共享锁（多个读线程可并发）
    std::string read(int key) const {
        std::shared_lock lock(mtx_);   // 共享锁
        return data_.at(key);
    }

    // 写操作——独占锁
    void write(int key, std::string val) {
        std::unique_lock lock(mtx_);   // 独占锁
        data_[key] = std::move(val);
    }
};
```

| 操作 | 锁类型 | 并发性 |
|------|------|--------|
| 读-读 | `shared_lock` | ✅ 多读并发 |
| 读-写 | `shared_lock` vs `unique_lock` | ❌ 互斥 |
| 写-写 | `unique_lock` | ❌ 互斥 |

---

## 4. RAII 锁包装——永远不要裸调 lock/unlock

### 4.1 `std::lock_guard`——最简单的 RAII 锁

```cpp
void safeIncrement() {
    std::lock_guard<std::mutex> guard(mtx);
    ++counter;
    // guard 析构自动 unlock
}   // 即使抛异常也 unlock
```

### 4.2 `std::unique_lock`——更灵活

```cpp
std::unique_lock<std::mutex> lock(mtx, std::defer_lock);  // 不立即加锁
// 做一些不需要锁的准备工作...
lock.lock();      // 显式加锁
// 临界区
lock.unlock();    // 提前解锁
// 做一些不需要锁的事...
lock.lock();      // 再次加锁

// unique_lock 可以转移所有权（移动语义）
auto lock2 = std::move(lock);  // lock 不再持有 mutex

// 条件变量必须配合 unique_lock
cv.wait(lock, [] { return ready; });
```

### 4.3 `std::scoped_lock`（C++17）——同时锁多个 mutex

```cpp
std::mutex m1, m2;

// ❌ 危险：可能死锁
void transfer_bad() {
    m1.lock();
    m2.lock();   // 如果另一个线程先锁 m2 再锁 m1 → 死锁
    // ...
}

// ✅ scoped_lock 使用免死锁算法
void transfer_good() {
    std::scoped_lock lock(m1, m2);  // 原子地同时锁定，不会死锁
    // 临界区
}
```

### 4.4 `std::lock`——同时锁多个 mutex 的底层函数

```cpp
// scoped_lock 内部就是用 std::lock 实现的
std::lock(m1, m2);  // 免死锁地同时锁定
std::lock_guard<std::mutex> g1(m1, std::adopt_lock);  // 接管已锁的 mutex
std::lock_guard<std::mutex> g2(m2, std::adopt_lock);
```

**免死锁原理**：`std::lock` 用 `try_lock` + 回退重试的策略，保证不会死锁。如果某个锁获取失败，释放所有已获取的锁，统一重试。

### 4.5 RAII 锁对比

| | `lock_guard` | `unique_lock` | `scoped_lock` |
|------|:---:|:---:|:---:|
| C++ 版本 | C++11 | C++11 | C++17 |
| 延迟加锁 | ❌ | ✅ `defer_lock` | ❌ |
| 提前解锁 | ❌ | ✅ `.unlock()` | ❌ |
| 可移动 | ❌ | ✅ | ❌ |
| 配合条件变量 | ❌ | ✅ | ❌ |
| 多锁同时锁 | ❌ | ❌ | ✅ |
| 开销 | 最小 | 稍大（有状态） | 最小 |

---

## 小结

| 概念 | 一句话 |
|------|--------|
| `mutex` | 基本互斥锁，睡眠等待 |
| `recursive_mutex` | 同线程可重复 lock |
| `timed_mutex` | 支持超时的 lock |
| 自旋锁 | 忙等待，适合极短临界区 |
| `shared_mutex` | 读写锁，读共享写独占 |
| `lock_guard` | 最简单的 RAII 锁 |
| `unique_lock` | 灵活 RAII，配合条件变量 |
| `scoped_lock` | C++17 多锁免死锁 |

下一篇 [03 条件变量 / 信号量 / 屏障](./03-cv-semaphore-barrier.md)。
