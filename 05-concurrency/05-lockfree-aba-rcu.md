# 05 — 无锁数据结构、ABA 问题、Hazard Pointers、RCU

> 关联篇目：[04 atomic / 内存序](./04-atomic-memory-order.md) | [06 死锁与并发问题](./06-deadlock-problems.md)

---

## 1. 无锁队列——SPSC（单生产者单消费者）

最简单的无锁队列：固定大小的环形缓冲区，用两个原子索引。

```cpp
template<typename T, size_t N>
class SPSCQueue {
    std::array<T, N> buffer_;
    std::atomic<size_t> writePos_{0};  // 生产者下一次写的位置
    std::atomic<size_t> readPos_{0};   // 消费者下一次读的位置
public:
    bool tryPush(const T& item) {
        size_t w = writePos_.load(std::memory_order_relaxed);
        size_t r = readPos_.load(std::memory_order_acquire);
        if (w - r >= N) return false;  // 队列满
        buffer_[w % N] = item;
        writePos_.store(w + 1, std::memory_order_release);
        return true;
    }

    bool tryPop(T& item) {
        size_t r = readPos_.load(std::memory_order_relaxed);
        size_t w = writePos_.load(std::memory_order_acquire);
        if (r >= w) return false;  // 队列空
        item = buffer_[r % N];
        readPos_.store(r + 1, std::memory_order_release);
        return true;
    }
};
```

**为什么是 lock-free？** 生产者和消费者各自只写自己的索引，只读对方的——没有竞争写入。

---

## 2. 无锁栈（Treiber Stack）

基于单向链表 + CAS 的经典无锁栈：

```cpp
template<typename T>
class LockFreeStack {
    struct Node { T data; Node* next; };
    std::atomic<Node*> head_{nullptr};
public:
    void push(const T& val) {
        Node* node = new Node{val, nullptr};
        node->next = head_.load(std::memory_order_relaxed);
        while (!head_.compare_exchange_weak(node->next, node,
                std::memory_order_release, std::memory_order_relaxed));
        // CAS: 如果 head == node->next → head = node
        //      否则 → node->next = head（更新为最新值）→ 继续循环
    }

    bool pop(T& val) {
        Node* node = head_.load(std::memory_order_relaxed);
        while (node && !head_.compare_exchange_weak(node, node->next,
                std::memory_order_acquire, std::memory_order_relaxed));
        if (!node) return false;
        val = node->data;
        // ⚠️ 不能直接 delete node！可能有其他线程还在读它
        // 需要 Hazard Pointers 或 RCU 来安全释放
        return true;
    }
};
```

---

## 3. ABA 问题——无锁编程的经典陷阱

### 3.1 问题描述

```
线程 1: 读取 head = A → 准备 CAS
线程 2: pop A（A 被释放）
线程 2: 分配新节点，恰好复用了 A 的地址 → push(C) ← C 的地址 = A！
线程 2: push(A) → head 又是 A
线程 1: CAS(head, A→B, A) → 成功！
        但 head 实际上已经不是原来的 A 了，中间的 C 丢失了！
```

```
时间线:
Thread 1:               Thread 2:
read head = A
                        pop A (delete A)
                        new Node C (地址恰好 = A!)
                        push C   head = C
                        push A   head = A → C → ...
CAS(A→B, A) 成功 ← 但此时 head 的 next(C) 被错误地断开了！
```

### 3.2 解决方案 1：带标签的指针（Tagged Pointer）

```cpp
// 使用 128 位 CAS（double-word CAS / cmpxchg16b）
struct TaggedPtr {
    Node* ptr;
    uintptr_t tag;  // 每次修改递增的版本号
};

std::atomic<TaggedPtr> head_;  // 使用 double-width CAS

// 每次修改时 tag++ → 即使地址相同，tag 也不等 → CAS 失败
```

### 3.3 解决方案 2：Hazard Pointers

**延迟回收**：不立即删除 pop 出来的节点，而是将其放入"待回收列表"。只有当确认没有线程还在访问该节点时，才真正删除。

```
每个线程维护一个 Hazard Pointer 列表（自己正在访问的节点指针）
回收线程扫描所有线程的 Hazard Pointers → 不在任何列表中的节点才能安全删除
```

### 3.4 解决方案 3：RCU（Read-Copy-Update）

读者无锁、写者复制新版本、等待所有读者完成后回收旧版本。

```
读者: 完全无锁，仅标记进入/离开临界区
写者: 复制数据 → 修改副本 → 原子更新指针 → 等待所有读者离开 → 释放旧数据
```

RCU 在 Linux 内核中广泛使用（`rcu_read_lock` / `rcu_read_unlock` / `synchronize_rcu`）。

---

## 4. MPMC 队列——多生产者多消费者

MPMC 比 SPSC 复杂得多——需要协调多个生产者同时写、多个消费者同时读。常见实现：

| 方案 | 特点 |
|------|------|
| 每个生产者独立槽位 | 预分配槽位 → 无竞争 |
| CAS 竞争写入位 | 类似 SPSC 但 writePos 共享 → CAS 竞争 |
| **moodycamel::ConcurrentQueue** | 工业级实现，分块 + 细粒度锁 + 无锁快速路径 |

---

## 5. Lock-free vs Wait-free

| | Lock-free | Wait-free |
|------|:---:|:---:|
| 含义 | 至少一个线程能取得进展 | **所有**线程都保证在有限步内完成 |
| 实现难度 | 中 | 高 |
| 例子 | CAS 循环 | 每个线程独立的 slot |
| 是否有饥饿 | 可能（某个线程一直 CAS 失败） | 不会 |

---

## 小结

| 概念 | 一句话 |
|------|--------|
| SPSC 无锁队列 | 环形 buffer + 两个原子索引 |
| Treiber Stack | 链表 + CAS 实现无锁栈 |
| ABA 问题 | 地址复用导致 CAS 误成功 |
| Tagged Pointer | 指针 + 版本号，需要双字 CAS |
| Hazard Pointers | 延迟回收，安全释放无锁结构中的节点 |
| RCU | 读者无锁，写者复制 + 等待宽限期 |

下一篇 [06 死锁 / 活锁 / 优先级反转 / 惊群](./06-deadlock-problems.md)。
