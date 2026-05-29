# 第五章：多线程与并发

> 8 篇 · 从线程基础到异步编程，覆盖 C++ 并发编程全栈知识。

---

## 篇目

| # | 文件 | 核心内容 |
|---|------|----------|
| 01 | [01-thread-basics.md](./01-thread-basics.md) | std::thread/pthread 创建、join/detach、参数传递陷阱、thread_local |
| 02 | [02-mutex-locks.md](./02-mutex-locks.md) | mutex/recursive_mutex/timed_mutex、自旋锁、shared_mutex(读写锁)、lock_guard/unique_lock/scoped_lock |
| 03 | [03-cv-semaphore-barrier.md](./03-cv-semaphore-barrier.md) | 条件变量(虚假唤醒)、信号量、barrier/latch |
| 04 | [04-atomic-memory-order.md](./04-atomic-memory-order.md) | atomic\<T\>、CAS(lock cmpxchg)、六种内存序、happens-before/synchronizes-with、内存屏障 |
| 05 | [05-lockfree-aba-rcu.md](./05-lockfree-aba-rcu.md) | SPSC 无锁队列、Treiber Stack、ABA 问题、Hazard Pointers、RCU |
| 06 | [06-deadlock-problems.md](./06-deadlock-problems.md) | 死锁四大条件、活锁、优先级反转(优先级继承)、惊群效应、锁优化策略 |
| 07 | [07-classic-sync-models.md](./07-classic-sync-models.md) | 生产者-消费者、读者-写者(读优先/写优先)、哲学家就餐 |
| 08 | [08-cpp-async.md](./08-cpp-async.md) | future/promise、async(launch策略)、packaged_task、shared_future、线程池设计 |

## 路线

线程基础→同步原语→无锁编程→并发问题→经典模型→异步编程。04 篇是理解 C++ 内存模型的关键。
