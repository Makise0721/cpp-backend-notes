# 03 — 条件变量、信号量、屏障

> 关联篇目：[02 互斥锁](./02-mutex-locks.md) | [07 生产者消费者](./07-classic-sync-models.md)

---

## 1. 条件变量——"等某个条件成立再做"

### 1.1 基本用法

```cpp
std::mutex mtx;
std::condition_variable cv;
bool ready = false;  // 条件变量总是配一个 predicate
std::queue<int> data;

// 消费者
void consumer() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return !data.empty(); });  // 等待条件
    int item = data.front();
    data.pop();
    lock.unlock();
    process(item);
}

// 生产者
void producer(int item) {
    {
        std::lock_guard<std::mutex> lock(mtx);
        data.push(item);
    }  // 释放锁后再通知（性能更好）
    cv.notify_one();  // 唤醒一个等待线程
    // cv.notify_all(); // 唤醒所有等待线程
}
```

### 1.2 `wait` 的内部流程

```cpp
// cv.wait(lock, predicate) 等价于：
while (!predicate()) {
    wait(lock);  // 原子地：unlock + 进入等待
                 // 被唤醒后：重新 lock
}
// 循环检查是必须的——因为虚假唤醒！
```

### 1.3 虚假唤醒（Spurious Wakeup）

**条件变量可能在没有任何线程调用 `notify` 的情况下醒来**——这是 POSIX 标准和 C++ 标准都允许的（源自一些硬件/OS 的实现复杂性）。

**这就是为什么 `wait` 必须用循环包裹 predicate，而不是用 `if`**：

```cpp
// ❌ 错误——一个虚假唤醒就会出错
if (!predicate())
    cv.wait(lock);

// ✅ 正确——总是检查 predicate
cv.wait(lock, [] { return predicate(); });
// 等价于 while (!predicate()) cv.wait(lock);
```

### 1.4 `notify_one` vs `notify_all`

```cpp
// notify_one: 唤醒**一个**等待线程（不确定哪个）
// 适合：每个任务只需要一个消费者处理（任务队列）

// notify_all: 唤醒**所有**等待线程
// 适合：状态变化影响所有等待者（如"程序即将退出"）
```

---

## 2. 信号量（Semaphore）

### 2.1 POSIX 信号量

```c
#include <semaphore.h>

sem_t sem;
sem_init(&sem, 0, 5);     // 初始值 5（计数信号量）
// sem_init(&sem, 0, 1);  // 初始值 1（二元信号量，类似 mutex）

sem_wait(&sem);   // P 操作：减 1，为 0 则阻塞
// 临界区
sem_post(&sem);   // V 操作：加 1，唤醒一个等待者

sem_destroy(&sem);
```

### 2.2 二元信号量 vs 互斥锁

| | 信号量 (sem=1) | 互斥锁 |
|------|------|------|
| 谁释放 | 任何线程都可以 `post` | 只有加锁的线程可以 `unlock` |
| 所有权 | 无所有权概念 | 有所有权 |
| 递归锁 | 不支持 | 支持（`recursive_mutex`） |
| 优先级继承 | 不支持 | 某些实现支持 |

### 2.3 C++20 `std::counting_semaphore`

```cpp
#include <semaphore>

std::counting_semaphore<5> sem(5);  // 最大计数 5，初始 5

sem.acquire();  // 减 1
sem.release();  // 加 1
sem.try_acquire();  // 非阻塞尝试
```

### 2.4 信号量的经典场景

```
生产者-消费者（用两个信号量控制缓冲区大小）:
- emptySlots (初始 = BUFFER_SIZE): 空位数
- fullSlots  (初始 = 0): 已填充数
+ mutex 保护 buffer 的修改
```

---

## 3. 屏障（Barrier）——等大家都到齐再出发

### 3.1 场景

多阶段并行算法中，需要所有线程都完成了当前阶段才能进入下一阶段。

```
线程 1: Phase 1 ████████│等待│ Phase 2 ████████
线程 2: Phase 1 ████│等待│ Phase 2 ████████████
线程 3: Phase 1 ██████│等待│ Phase 2 ██████
                       ↑
                    屏障点——等所有线程到齐
```

### 3.2 POSIX 屏障

```c
pthread_barrier_t barrier;
pthread_barrier_init(&barrier, NULL, 3);  // 3 个线程

// 每个线程完成当前阶段后：
pthread_barrier_wait(&barrier);  // 阻塞直到 3 个线程都到达
// 最后一个到达的线程返回 PTHREAD_BARRIER_SERIAL_THREAD
```

### 3.3 C++20 `std::barrier`

```cpp
#include <barrier>

std::barrier sync_point(3, [] {  // 3 个线程，完成回调
    std::cout << "All threads arrived!\n";
});

void worker() {
    do_phase1();
    sync_point.arrive_and_wait();  // 等待其他线程
    do_phase2();
    sync_point.arrive_and_wait();
}
```

### 3.4 C++20 `std::latch`——一次性屏障

```cpp
std::latch done(3);  // 倒计时 = 3

void worker() {
    do_work();
    done.count_down();  // 计数减 1
}

// 主线程等待所有 worker 完成
done.wait();  // 阻塞直到计数 = 0
```

| | `latch` | `barrier` |
|------|:---:|:---:|
| 可重用 | ❌ 一次性 | ✅ 可复用 |
| 计数方向 | 减到 0 | 每个阶段重置 |
| 场景 | 等待一组任务完成 | 多阶段同步 |

---

## 小结

| 概念 | 关键 |
|------|------|
| 条件变量 | `wait(lock, predicate)` — 循环检查防止虚假唤醒 |
| `notify_one` vs `notify_all` | 只唤醒一个 vs 全部唤醒 |
| 信号量 | P(acquire)/V(release)，可用于控制并发数 |
| 屏障 | `arrive_and_wait()` 等所有线程到齐 |
| latch | 一次性倒计时，等待任务完成 |

下一篇 [04 atomic / CAS / 六种内存序 / C++ 内存模型](./04-atomic-memory-order.md)。
