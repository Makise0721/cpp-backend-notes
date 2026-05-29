# 09 — 并发基础：std::atomic、std::thread、std::mutex

> 关联篇目：[04 智能指针（shared_ptr 线程安全）](./04-smart-pointers.md) | [第一章 02 RAII（lock_guard）](../01-cpp-basics/02-raii-rule-of-five.md)

---

## 1. `std::thread`——创建和管理线程

### 1.1 基本用法

```cpp
#include <thread>
#include <iostream>

void worker(int id, const std::string& msg) {
    std::cout << "Thread " << id << ": " << msg << "\n";
}

std::thread t1(worker, 1, "hello");   // 启动线程，传递参数
std::thread t2([] {                    // lambda
    std::cout << "Lambda thread\n";
});

// 必须决定 join 还是 detach
t1.join();   // 等待 t1 完成
t2.join();   // 等待 t2 完成

// ⚠️ thread 对象析构时如果还没 join/detach → std::terminate！
```

### 1.2 `join` vs `detach`

```cpp
std::thread t(worker, 1, "hi");

t.join();    // 阻塞，等待线程完成。之后 t 不再代表任何线程
t.detach();  // 分离线程（后台运行）。t 不再代表任何线程。线程结束时自动清理
             // ⚠️ 确保 detach 的线程不访问已销毁的局部变量！

// 判断是否可 join
if (t.joinable()) {
    t.join();
}
```

### 1.3 参数传递——注意引用

```cpp
void modifier(int& x) { x *= 2; }

int value = 42;

// ❌ 错误：thread 的构造函数按值复制参数，即使函数签名是引用
// std::thread t(modifier, value);  // 编译错误！int&& 不能绑定到 int&

// ✅ 用 std::ref 传递引用
std::thread t(modifier, std::ref(value));
t.join();
std::cout << value;  // 84
```

---

## 2. `std::mutex`——互斥锁家族

### 2.1 基本互斥锁

```cpp
#include <mutex>

int counter = 0;
std::mutex mtx;

void increment(int n) {
    for (int i = 0; i < n; ++i) {
        mtx.lock();
        ++counter;
        mtx.unlock();    // ⚠️ 如果中间抛异常，永远不会 unlock → 死锁！
    }
}
```

### 2.2 RAII 锁包装器（永远用它们，不裸调 lock/unlock）

```cpp
void incrementSafe(int n) {
    for (int i = 0; i < n; ++i) {
        std::lock_guard<std::mutex> guard(mtx);  // 构造=加锁，析构=解锁
        ++counter;
    }  // guard 析构——异常安全
}

// C++17: scoped_lock——可同时锁多个 mutex
std::mutex m1, m2;
void transfer() {
    std::scoped_lock lock(m1, m2);  // 原子地锁住两个（避免死锁！）
    // 操作两个共享资源
}

// unique_lock——更灵活（可延迟锁、提前解锁、可移动）
std::unique_lock<std::mutex> lock(mtx, std::defer_lock);  // 不立即锁
// 做一些准备...
lock.lock();     // 延迟加锁
// 临界区
lock.unlock();   // 提前解锁
// 做一些清理...
lock.lock();     // 重新加锁
```

### 2.3 其他 mutex 类型

| 类型 | 特点 |
|------|------|
| `std::mutex` | 基本互斥锁，不可递归 |
| `std::recursive_mutex` | 同一线程可多次 lock（每次 lock 需对应 unlock） |
| `std::timed_mutex` | 支持 `try_lock_for()` / `try_lock_until()` |
| `std::shared_mutex` (C++17) | 读写锁：多读单写 |

### 2.4 读写锁（`shared_mutex`）

```cpp
#include <shared_mutex>

class ThreadSafeCache {
    mutable std::shared_mutex mtx_;
    std::map<int, std::string> data_;
public:
    std::string read(int key) const {
        std::shared_lock lock(mtx_);  // 共享锁——多线程可同时读
        return data_.at(key);
    }

    void write(int key, std::string val) {
        std::unique_lock lock(mtx_);  // 独占锁——写时不可读写
        data_[key] = std::move(val);
    }
};
```

---

## 3. `std::atomic`——无锁的原子操作

### 3.1 为什么需要 atomic

```cpp
// ❌ 即使最简单的 ++ 也不是原子的
int counter = 0;
// counter++ 在汇编层面是三条指令：load → inc → store
// 两个线程同时执行 → 可能两个 ++ 只增加一次！

// ✅ atomic 保证原子性
std::atomic<int> counter{0};
counter++;  // 原子的读-改-写
```

### 3.2 基本操作

```cpp
std::atomic<int> x{0};

x.store(42);              // 原子写入
int val = x.load();       // 原子读取
int old = x.exchange(99); // 原子交换：写 99，返回旧值 42

int expected = 99;
bool success = x.compare_exchange_strong(expected, 100);
// 如果 x == expected → x = 100, return true
// 否则 → expected = x, return false
// CAS（Compare-And-Swap）——无锁数据结构的基础原子操作

x.fetch_add(5);           // x += 5，返回旧值
x.fetch_sub(3);           // x -= 3，返回旧值
x++;                      // 等价于 fetch_add(1)
```

### 3.3 内存序（Memory Order）——先了解默认

```cpp
// 默认使用 std::memory_order_seq_cst（顺序一致性）——最安全、最慢
x.store(42);  // 隐式 memory_order_seq_cst

// 可指定更弱的内存序以获得更好性能（高级话题）
x.store(42, std::memory_order_release);
x.load(std::memory_order_acquire);
```

对于大多数场景，默认的 `seq_cst` 足够且安全。内存序的精细控制在 AI Infra 的 lock-free queue 等场景中才需要深入学习。

### 3.4 `is_lock_free`——真的无锁吗？

```cpp
std::atomic<int> a;
std::cout << a.is_lock_free();  // 通常是 true（int 的操作硬件支持）

struct BigStruct { char data[256]; };
std::atomic<BigStruct> b;
std::cout << b.is_lock_free();  // 通常是 false（太大，用互斥锁实现）
```

**不是所有 `atomic<T>` 都是无锁的**——大的 T 会用内部的 mutex 实现原子性。

---

## 4. 线程安全速查

```cpp
// ✅ shared_ptr 的控制块是线程安全的
auto sp = std::make_shared<int>(42);
std::thread t1([sp] { auto copy = sp; });  // use_count++ 原子
std::thread t2([sp] { auto copy = sp; });  // 安全

// ✅ static 局部变量初始化是线程安全的（C++11 起）
static MyClass instance;  // 多线程同时首次访问，只构造一次

// ✅ 每个容器实例不是线程安全的，但不同实例之间安全
std::vector<int> v1, v2;
std::thread t3([&] { v1.push_back(1); });  // 不同对象——安全
std::thread t4([&] { v2.push_back(2); });  // 安全

// ❌ 同一个容器多线程并发读写——必须加锁
std::thread t5([&] { v1.push_back(3); });  // 数据竞争！
std::thread t6([&] { v1.push_back(4); });  // 必须加锁保护 v1
```

---

## 5. `std::call_once` 与 `std::once_flag`

线程安全的一次性初始化：

```cpp
std::once_flag initFlag;
std::unique_ptr<ExpensiveResource> resource;

void ensureInit() {
    std::call_once(initFlag, [] {
        resource = std::make_unique<ExpensiveResource>();
    });
    // 用 resource...
}

// 多个线程调用 ensureInit → 只初始化一次
```

---

## 小结

| 概念 | 一句话 |
|------|--------|
| `thread` | 创建线程，必须 join/detach 其中之一 |
| `mutex` + `lock_guard` | RAII 加锁，异常安全 |
| `scoped_lock` (C++17) | 同时锁多个 mutex，避免死锁 |
| `shared_mutex` | 读写锁：`shared_lock` 读共享，`unique_lock` 写独占 |
| `atomic` | 无锁原子操作：load/store/exchange/CAS |
| `call_once` | 线程安全的一次性初始化 |

下一篇 [10 C++17 语法糖大集合](./10-cpp17-sugar.md)。
