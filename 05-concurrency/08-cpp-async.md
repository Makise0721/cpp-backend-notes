# 08 — C++ 异步编程：future、promise、async、线程池

> 关联篇目：[01 线程基础](./01-thread-basics.md) | [第二章 04 智能指针](../02-cpp11-17-features/04-smart-pointers.md)

---

## 1. `std::future` + `std::promise`——异步结果的生产与消费

### 1.1 基本概念

```
promise（生产者）  ──设置值──→  future（消费者）
                                  .get() 阻塞等待值
```

```cpp
std::promise<int> prom;
std::future<int> fut = prom.get_future();

// 生产者线程
std::thread producer([&prom] {
    int result = computeExpensive();
    prom.set_value(result);  // 设置结果
    // prom.set_exception(std::make_exception_ptr(...));  // 或设置异常
});

// 消费者
int value = fut.get();  // 阻塞直到 promise 设值
// get() 只能调用一次！第二次会抛异常
```

### 1.2 生命周期

```
promise / future 共享一个"共享状态"（shared state）
- promise 设值后，状态就绪
- future.get() 消费状态
- 共享状态在 promise + future 都析构后释放
```

---

## 2. `std::async`——一行启动异步任务

```cpp
#include <future>

// 异步启动，返回 future
std::future<int> fut = std::async(std::launch::async, [] {
    return computeHeavy();
});

// 可以去做别的事...
int result = fut.get();  // 需要结果时再阻塞等待
```

### 2.1 启动策略

```cpp
// async: 强制在新线程中执行
auto f1 = std::async(std::launch::async, task);

// deferred: 延迟执行——调用 get()/wait() 时才在当前线程执行
auto f2 = std::async(std::launch::deferred, task);

// 默认 = async | deferred：由实现决定（通常 = async）
auto f3 = std::async(task);

// ⚠️ 陷阱：std::async 返回的 future 析构时会阻塞等待任务完成！
// 如果不保存 future → 临时对象析构 → 变为同步调用！
std::async(task);  // 临时 future 析构 → 阻塞等待 → 丧失了异步的意义
auto f = std::async(task);  // ✅ 保存 future
```

---

## 3. `std::packaged_task`——包装可调用对象

```cpp
// 将任务包装成 future-able 对象
std::packaged_task<int(int, int)> task([](int a, int b) {
    return a + b;
});

std::future<int> fut = task.get_future();

// 可以传给线程池或者在其他线程执行
std::thread t(std::move(task), 3, 4);
t.join();
std::cout << fut.get();  // 7
```

`packaged_task` 的作用：把任意可调用对象变成一个可以返回 `future` 的"任务包裹"——是线程池的基础构件。

---

## 4. `std::shared_future`——多线程等待同一结果

普通 `future` 只能 `get()` 一次。`shared_future` 可被多次访问、可拷贝：

```cpp
std::promise<int> prom;
std::shared_future<int> sf = prom.get_future().share();

// 多个线程都可以访问同一个结果
std::thread t1([sf] { int x = sf.get(); });  // get() 多次调用 OK
std::thread t2([sf] { int x = sf.get(); });

prom.set_value(42);
t1.join(); t2.join();
```

---

## 5. `std::future_status`——轮询 future 状态

```cpp
std::future<int> fut = std::async(task);

auto status = fut.wait_for(std::chrono::milliseconds(100));
switch (status) {
    case std::future_status::ready:      // 结果就绪
        break;
    case std::future_status::timeout:    // 超时，还没好
        break;
    case std::future_status::deferred:   // 延迟执行（launch::deferred）
        break;
}

// wait_until: 等待到指定时间点
fut.wait_until(deadline);
```

---

## 6. 线程池设计

### 6.1 核心架构

```
线程池 = 任务队列 + N 个工作线程 + 结果返回机制

                      ┌─────────────┐
         submit(task)  │  任务队列    │
         ────────────→ │ (线程安全)   │
                       └──────┬──────┘
                              │ pop
              ┌───────────────┼───────────────┐
              │               │               │
         ┌────▼────┐    ┌────▼────┐    ┌────▼────┐
         │ Worker 1│    │ Worker 2│    │ Worker N│
         └─────────┘    └─────────┘    └─────────┘
```

### 6.2 最简实现

```cpp
class SimpleThreadPool {
    std::vector<std::thread> workers_;
    std::queue<std::function<void()>> tasks_;
    std::mutex mtx_;
    std::condition_variable cv_;
    bool stop_ = false;

public:
    SimpleThreadPool(size_t n) {
        for (size_t i = 0; i < n; ++i)
            workers_.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock lock(mtx_);
                        cv_.wait(lock, [this] { return stop_ || !tasks_.empty(); });
                        if (stop_ && tasks_.empty()) return;
                        task = std::move(tasks_.front());
                        tasks_.pop();
                    }
                    task();
                }
            });
    }

    template<typename F, typename... Args>
    auto submit(F&& f, Args&&... args) -> std::future<decltype(f(args...))> {
        using ReturnType = decltype(f(args...));

        // 用 packaged_task + shared_ptr 包装，支持 future 返回
        auto task = std::make_shared<std::packaged_task<ReturnType()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...));
        auto fut = task->get_future();

        {
            std::lock_guard lock(mtx_);
            tasks_.emplace([task] { (*task)(); });
        }
        cv_.notify_one();
        return fut;
    }

    ~SimpleThreadPool() {
        { std::lock_guard lock(mtx_); stop_ = true; }
        cv_.notify_all();
        for (auto& w : workers_) w.join();
    }
};

// 用法
SimpleThreadPool pool(4);
auto fut = pool.submit([](int a, int b) { return a + b; }, 3, 4);
std::cout << fut.get();  // 7
```

---

## 7. C++ 异步组件总览

| 组件 | 角色 | 关键方法 |
|------|------|----------|
| `std::promise` | 设置值/异常 | `set_value()` / `set_exception()` |
| `std::future` | 获取值（一次性） | `get()` / `wait()` / `wait_for()` |
| `std::shared_future` | 多次获取值 | 同 future，但可拷贝+多次 `get()` |
| `std::async` | 便捷异步启动 | 返回 future，注意析构行为 |
| `std::packaged_task` | 包装可调用对象 | `get_future()` + 移动执行 |
| 线程池 | 复用线程 | submit → future |

---

## 小结

| 概念 | 一句话 |
|------|--------|
| `promise/future` | 生产者设值，消费者 `get()` 等待 |
| `async` | 便捷异步调用，注意保存返回的 future |
| `packaged_task` | 将任务变成 future-able 对象 |
| `shared_future` | 支持多次 `get()` 和多线程等待 |
| `future_status` | `ready` / `timeout` / `deferred` |
| 线程池 | 任务队列 + worker 线程 + packaged_task 返回 future |

---

## 第五章小结

| # | 篇目 | 核心 |
|---|------|------|
| 01 | 线程基础 | pthread_create/std::thread/join/detach/TLS |
| 02 | 互斥锁 | mutex/shared_mutex/自旋锁/lock_guard/unique_lock/scoped_lock |
| 03 | 条件变量/信号量/屏障 | wait 必须循环检查/虚假唤醒/semaphore/barrier/latch |
| 04 | atomic/内存序 | CAS/六种内存序/happens-before/synchronizes-with |
| 05 | 无锁数据结构 | SPSC/Treiber Stack/ABA/Hazard Pointers/RCU |
| 06 | 死锁与并发问题 | 死锁四条件/优先级继承/惊群/锁优化 |
| 07 | 经典同步模型 | 生产者消费者/读者写者/哲学家 |
| 08 | C++ 异步编程 | future/promise/async/线程池 |
