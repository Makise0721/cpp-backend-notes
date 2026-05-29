# 08 — Reactor / Proactor 模式与 muduo 架构

> 关联篇目：[07 IO 多路复用](./07-io-models.md) | [09 高性能网络](./09-high-performance.md)

---

## 1. Reactor 模式——事件驱动

**Reactor = IO 多路复用 + 事件分发**。一个事件循环等待多个 fd 的事件，就绪后回调对应的 handler。

### 1.1 单 Reactor 单线程——Redis 模型

```
┌─────────────────────────┐
│        Reactor           │
│  ┌────────────────────┐  │
│  │   epoll_wait 循环   │  │
│  │        │            │  │
│  │   ┌────┼────┐      │  │
│  │   ▼    ▼    ▼      │  │
│  │ 事件1 事件2 事件3   │  │
│  │   │    │    │      │  │
│  │   ▼    ▼    ▼      │  │
│  │ 业务处理(串行)      │  │
│  └────────────────────┘  │
└─────────────────────────┘

优点: 简单、无锁、无上下文切换
缺点: 单个 CPU 核，一个慢请求阻塞所有
```

### 1.2 单 Reactor 多线程

```
┌───────────────┐
│   Reactor      │ ← 一个事件循环线程（IO 线程）
│  epoll_wait    │
│       │        │
│   事件分发      │
│       │        │
└───────┼────────┘
        │ 放入任务队列
┌───────▼────────┐
│   线程池        │ ← 多个工作线程处理业务逻辑
│  Worker 1..N   │
└────────────────┘

优点: CPU 密集型任务不阻塞 IO
缺点: Reactor 线程本身成为瓶颈（承担所有连接的事件循环）
```

### 1.3 主从 Reactor——Nginx / Netty / muduo 模型

```
┌─────────────┐
│ MainReactor  │ ← 只负责 accept 新连接
│ (epoll)      │
└──────┬───────┘
       │ 分发连接到 SubReactor
       ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ SubReactor 1│ │ SubReactor 2│ │ SubReactor N│
│ (epoll)     │ │ (epoll)     │ │ (epoll)     │
│ 读写事件    │ │ 读写事件    │ │ 读写事件    │
└─────────────┘ └─────────────┘ └─────────────┘

MainReactor: accept + 负载均衡分发
SubReactor: 负责已建立连接的 IO 事件 + 轻量业务
线程池: 处理 CPU 密集型业务（可选）
```

**Nginx 做法**：多个 worker 进程，每个 worker 是一个独立的 epoll 循环（多进程 Reactor）。
**Netty / muduo 做法**：一个 MainReactor 线程 + N 个 SubReactor 线程。

---

## 2. Proactor 模式——真正的异步 IO

**Reactor 是"事件就绪后通知你"**（你要自己 read/write），**Proactor 是"IO 完成后通知你"**。

```
Reactor:
  epoll_wait → 可读 → read(fd) → 处理数据
  事件通知         ↑ 用户自己读

Proactor:
  aio_read → IO 完成回调 → 直接处理数据
           ↑ 内核帮你读了
```

| | Reactor | Proactor |
|------|------|------|
| 通知什么 | fd 就绪（可读/可写） | IO 已完成（数据已读好） |
| 谁执行 IO | **用户**调用 read/write | **内核**完成 |
| 实现 | epoll / kqueue | IOCP (Windows) / io_uring (Linux) |
| 适用 OS | Linux/Unix/Windows | Windows Native / Linux io_uring |

---

## 3. muduo 网络库架构（Linux 经典）

muduo = 陈硕开发的 Linux C++ 网络库，主从 Reactor + one loop per thread。

```
muduo 架构:
┌─────────────────────────────────────┐
│              EventLoop               │ ← 每个线程一个 EventLoop
│  ┌─────────────────────────────┐    │    = epoll fd + 事件循环
│  │          Channel             │    │
│  │  (每个 fd 对应一个 Channel)   │    │
│  │  fd + events + callbacks    │    │
│  └─────────────────────────────┘    │
│  ┌─────────────────────────────┐    │
│  │          Poller              │    │
│  │  epoll_wait 封装             │    │
│  └─────────────────────────────┘    │
│  ┌─────────────────────────────┐    │
│  │          TimerQueue          │    │
│  │  定时器（timerfd）            │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘

TcpServer:
  Acceptor (MainLoop) → accept → 分配 TcpConnection 到 SubLoop
  TcpConnection (SubLoop) → 处理读写事件
```

**核心原则**：
- **one loop per thread**：每个线程最多一个 EventLoop
- **线程池**：EventLoopThreadPool 管理 IO 线程
- **fd 绑定到 EventLoop**：一个连接的所有事件在同一个线程处理（无锁）

---

## 小结

| 模式 | 通知 | IO 执行 | 典型实现 |
|------|------|:---:|------|
| 单 Reactor 单线程 | fd 就绪 | 用户 | Redis |
| 单 Reactor 多线程 | fd 就绪 | 用户 | Netty(旧) |
| 主从 Reactor | fd 就绪 | 用户 | Nginx、Netty、muduo |
| Proactor | IO 完成 | 内核 | IOCP、io_uring |

下一篇 [09 零拷贝、协议设计、Protobuf、RPC、定时器](./09-high-performance.md)。
