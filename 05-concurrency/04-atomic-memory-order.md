# 04 — 原子操作、CAS、六种内存序、C++ 内存模型

> 关联篇目：[05 无锁数据结构](./05-lockfree-aba-rcu.md) | [第二章 09 并发基础](../02-cpp11-17-features/09-concurrency-basics.md)

---

## 1. `std::atomic<T>`——原子操作的基石

### 1.1 基本操作

```cpp
std::atomic<int> x{0};

x.store(42);              // 原子写入
int v = x.load();         // 原子读取
int old = x.exchange(99); // 交换：写入 99，返回旧值 42

x.fetch_add(5);           // x += 5，返回旧值
x.fetch_sub(3);           // x -= 3
x++;                      // 等价于 fetch_add(1)

// 位运算（整数特化）
x.fetch_and(0xFF);
x.fetch_or(0x01);
x.fetch_xor(0x80);
```

### 1.2 CAS（Compare-And-Swap）——无锁编程的核心

```cpp
int expected = 10;
int desired = 20;

// compare_exchange_weak: 可能"伪失败"（即使值相等也返回 false），适合循环
// compare_exchange_strong: 只在值确实不等时才失败，性能稍差
bool success = x.compare_exchange_strong(expected, desired);
// 如果 x == expected → x = desired, return true
// 否则 → expected = x, return false
```

### 1.3 CAS 的底层实现：`lock cmpxchg`

```x86asm
; CAS(x, expected, desired) 的 x86_64 实现:
mov rax, expected    ; 期望值放入 eax
mov rbx, desired     ; 新值放入 rbx
lock cmpxchg [x], rbx  ; 如果 [x]==rax → [x]=rbx, ZF=1
                        ; 否则 → rax=[x], ZF=0
; lock 前缀保证原子性（锁总线或锁缓存行）
```

### 1.4 `compare_exchange_weak` vs `strong`

```cpp
// weak: 可能伪失败，必须放在循环中——适合 LL/SC 架构（ARM）
while (!x.compare_exchange_weak(expected, desired)) {
    // expected 已被更新为当前值
    // 重新计算 desired 或直接重试
}

// strong: 保证只有值不等时才失败——实现通常就是 weak 的循环包装
// 如果已经在外层循环中，直接用 weak（避免双重循环）
```

---

## 2. C++11 六种内存序——从松散到严格

### 2.1 全景

```
memory_order_relaxed          最弱：仅保证原子性
memory_order_consume          数据依赖排序（实际多实现为 acquire）
memory_order_acquire          读屏障：后续操作不能重排到此之前
memory_order_release          写屏障：之前操作不能重排到此之后
memory_order_acq_rel          兼具 acquire + release
memory_order_seq_cst          最强：全局顺序一致性（默认）
```

### 2.2 `relaxed`——只保证原子性

```cpp
// 两个线程各自增加计数器，不需要任何顺序保证
std::atomic<int> counter{0};

// 线程 1
counter.fetch_add(1, std::memory_order_relaxed);

// 线程 2
counter.fetch_add(1, std::memory_order_relaxed);
// 两个线程的执行顺序不确定，但最终 counter 一定是 2
```

**只适用于不需要跨线程顺序同步的场景**（如引用计数递增、统计数据累加）。

### 2.3 `acquire` / `release`——最常用的同步对

```cpp
std::atomic<bool> ready{false};
int data = 0;

// 线程 1（生产者）
data = 42;                                        // A
ready.store(true, std::memory_order_release);      // B
// ↑ release: 保证 A 在 B 之前对所有 acquire 线程可见

// 线程 2（消费者）
while (!ready.load(std::memory_order_acquire));    // C
// ↑ acquire: 保证 D 在 C 之后执行
assert(data == 42);                               // D  ← 保证成立！
```

**acquire-release 对建立 synchronizes-with 关系**：release store → acquire load 形成跨线程的 happens-before。

### 2.4 `seq_cst`——默认，全局一致

```cpp
// 所有 seq_cst 操作在所有线程看来遵循同一个全局顺序
x.store(1, std::memory_order_seq_cst);
y.store(1, std::memory_order_seq_cst);

// 不可能出现：线程 1 看到 x=1, y=0 且线程 2 看到 x=0, y=1
// 而 relaxed/acquire-release 可能出现这种"不一致"视图
```

**写起来最省心，但性能最差**（需要全局同步和额外的内存屏障）。

### 2.5 内存序性能对比（x86）

| 内存序 | x86 额外指令开销 | 说明 |
|------|:---:|------|
| relaxed | 无 | x86 本身就提供较强的顺序保证 |
| acquire | 无（load 天然 acquire） | x86 不重排 load-load |
| release | 无（store 天然 release） | x86 不重排 store-store |
| acq_rel | `mfence`（如果是 RMW） | 需要全屏障 |
| seq_cst | `mfence` | 每次 store 后都加全屏障 |

> **x86 是强内存模型**——acquire/release 在 x86 上几乎零开销。ARM/PowerPC 是弱内存模型——acquire/release 需要显式屏障指令。

---

## 3. 内存屏障（Fence）

### 3.1 编译器屏障

```cpp
// 阻止编译器重排指令，不影响 CPU 重排
asm volatile("" ::: "memory");

// C++11 标准方式
std::atomic_signal_fence(std::memory_order_acq_rel);
```

### 3.2 CPU 屏障

```cpp
// 全屏障：之前所有读写必须在之后所有读写之前完成
std::atomic_thread_fence(std::memory_order_seq_cst);

// 获取屏障：之后的读不能重排到之前
std::atomic_thread_fence(std::memory_order_acquire);

// 释放屏障：之前的写不能重排到之后
std::atomic_thread_fence(std::memory_order_release);
```

```x86asm
mfence   ; 全屏障（StoreLoad）
lfence   ; 读屏障（LoadLoad + LoadStore）
sfence   ; 写屏障（StoreStore）
```

---

## 4. C++11 内存模型核心关系

### 4.1 happens-before

**如果操作 A happens-before 操作 B，那么 A 的结果对 B 可见。**

happens-before 的传递链：
```
同一线程内：按程序顺序（sequenced-before）
跨线程：通过 synchronizes-with 建立
```

### 4.2 synchronizes-with

```
release-store  synchronizes-with  acquire-load (读到了 release 写的值)

线程 1:  x.store(1, release)
线程 2:  x.load(acquire)  ← 如果读到了 1 → synchronizes-with 建立
```

### 4.3 经典示例：消息传递

```cpp
std::atomic<bool> flag{false};
std::string message;  // 非原子

// 线程 1（发送）
message = "hello";
flag.store(true, std::memory_order_release);  // ★ release

// 线程 2（接收）
while (!flag.load(std::memory_order_acquire)); // ★ acquire
// 当跳出循环时，线程 1 在 release 之前的所有写都对线程 2 可见
assert(message == "hello");  // ✅ 保证成立
```

---

## 5. 内存序选择指南

```
仅统计计数，不关心顺序 → relaxed

生产者-消费者（一个线程写，一个线程读）:
  写方用 release，读方用 acquire

多个线程同时读写 → seq_cst（默认，最安全）

不确定用哪个 → seq_cst（默认），后续优化时再降级
```

---

## 小结

| 概念 | 一句话 |
|------|--------|
| `atomic<T>` | 原子操作，无数据竞争 |
| CAS | `compare_exchange` — 无锁编程的原子指令 |
| `relaxed` | 仅原子性，无顺序保证 |
| `acquire/release` | 跨线程同步对，最常用 |
| `seq_cst` | 全局顺序一致（默认、最安全、最慢） |
| happens-before | 内存模型的传递可见性关系 |
| synchronizes-with | release → acquire 建立的跨线程关系 |

下一篇 [05 无锁数据结构 / ABA / Hazard Pointers / RCU](./05-lockfree-aba-rcu.md)。
