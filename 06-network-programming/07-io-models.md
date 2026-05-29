# 07 — IO 模型：BIO、NIO、select/poll/epoll、io_uring

> 关联篇目：[08 Reactor/Proactor](./08-reactor-proactor.md) | [09 高性能网络](./09-high-performance.md)

---

## 1. IO 模型全景图

```
阻塞 IO (BIO):      线程挂起，等数据 + 等复制 ── 最简单，最浪费线程
非阻塞 IO (NIO):    线程轮询，数据未就绪立即返回 ── CPU 空转
IO 多路复用:        一个线程监控多个 fd ── 高效
信号驱动 IO:        fd 就绪时发 SIGIO ── 几乎不用
异步 IO:            内核完成所有操作后才通知 ── 真正的异步
```

---

## 2. select——最古老的多路复用

```c
fd_set readfds;
FD_ZERO(&readfds);
FD_SET(fd1, &readfds);
FD_SET(fd2, &readfds);

// select 会修改 fd_set，每次要重新设置
int ready = select(max_fd + 1, &readfds, NULL, NULL, &timeout);

if (FD_ISSET(fd1, &readfds)) { /* fd1 可读 */ }
if (FD_ISSET(fd2, &readfds)) { /* fd2 可读 */ }
```

**select 的四大缺陷**：
1. **fd 数量限制**：`FD_SETSIZE` = 1024（编译期固定）
2. **每次都要复制整个 fd_set**：从用户态到内核态，O(n)
3. **内核要遍历所有 fd** 才能找到就绪的，O(n)
4. **每次调用要重新设置 fd_set**（select 会修改它）

---

## 3. poll——select 的简单升级

```c
struct pollfd fds[2];
fds[0].fd = fd1; fds[0].events = POLLIN;
fds[1].fd = fd2; fds[1].events = POLLIN;

int ready = poll(fds, 2, timeout);
// 遍历 fds 检查 revents
```

**改进**：无 fd 数量限制（用动态数组）。但**仍要 O(n) 遍历**。

---

## 4. epoll——Linux 的终极多路复用

### 4.1 为什么 epoll 快？

| | select/poll | epoll |
|------|:---:|:---:|
| fd 注册 | 每次调用都传入全量 fd | `epoll_ctl` 一次性注册 |
| 事件获取 | O(n) 遍历所有 fd | O(1) 只返回就绪的（就绪链表） |
| 内核数据结构 | 无状态 | 红黑树 + 就绪链表 |

### 4.2 API

```c
int epfd = epoll_create1(0);

struct epoll_event ev;
ev.events = EPOLLIN;  // 可读事件
ev.data.fd = fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);
epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev);
epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);

struct epoll_event events[MAX_EVENTS];
int n = epoll_wait(epfd, events, MAX_EVENTS, timeout);
for (int i = 0; i < n; ++i) {
    if (events[i].events & EPOLLIN) {
        int ready_fd = events[i].data.fd;
        // 处理 ready_fd...
    }
}
```

### 4.3 底层数据结构

```
epoll 实例:
┌──────────────────────────────┐
│  红黑树（管理所有注册的 fd）    │ ← epoll_ctl 增/删/改
│     fd1, fd2, fd3, ...       │
├──────────────────────────────┤
│  就绪链表                     │ ← 设备驱动通过回调函数将就绪 fd 插入
│     fd2 → fd5 → ...          │
└──────────────────────────────┘

epoll_wait: 直接从就绪链表取 → O(1)
```

### 4.4 LT vs ET 触发模式

| | **LT (Level Triggered)** | **ET (Edge Triggered)** |
|------|------|------|
| 通知时机 | 只要 fd 可读，每次 `epoll_wait` 都通知 | 只在状态**变化**（不可读→可读）时通知一次 |
| 漏处理 | 不会（下次还通知） | 会（错过不再通知） |
| IO 模式 | 阻塞/非阻塞都可以 | **必须配合非阻塞 IO** |
| 读取方式 | 读一次就行 | 必须读到 EAGAIN（读干净） |
| 性能 | 稍低 | 更高（减少重复通知） |
| 默认 | ✅ 是 | |

```cpp
// ET 模式下的正确读法
ev.events = EPOLLIN | EPOLLET;  // 边缘触发
for (;;) {
    ssize_t n = read(fd, buf, sizeof(buf));
    if (n > 0) process(buf, n);
    else if (n == 0) { /* EOF */ break; }
    else if (errno == EAGAIN || errno == EWOULDBLOCK) break;  // 读完了
    else { /* 真错误 */ break; }
}
```

---

## 5. io_uring——Linux 新一代异步 IO

### 5.1 传统 AIO 的问题

传统 Linux AIO 只能用于 `O_DIRECT` 文件 IO（不支持 buffered IO 和 socket），使用受限。

### 5.2 io_uring 的设计

```
用户态和内核态共享两个环形缓冲区:

SQ (Submission Queue): 用户写 → 内核读     ← 提交 IO 请求
CQ (Completion Queue): 内核写 → 用户读     ← 获取 IO 完成结果
```

```
用户态                   内核态
┌──────────┐          ┌──────────┐
│   应用    │   SQ     │          │
│ 提交请求 ──→ [ ]──→  │  处理 IO  │
│          │           │          │
│ 获取结果 ←── [ ] ←── │          │
│          │   CQ     │          │
└──────────┘          └──────────┘
```

**优势**：
- 零/少系统调用（通过共享内存通信，不必每次 `syscall`）
- 支持任何 fd（文件、socket 都行）
- 支持批量提交和批量完成
- polled 模式下可达到极低延迟（~10μs）

---

## 6. IO 模型对比

| 模型 | 线程模型 | fd 限制 | 核心调用 | 适用 |
|------|----------|:---:|------|------|
| BIO | 1 线程/连接 | 线程数 | `read`/`write` | 简单/低并发 |
| select | 单线程 | 1024 | `select` | 已过时 |
| poll | 单线程 | 无 | `poll` | 已过时 |
| epoll | 单线程/多线程 | 无 | `epoll_wait` | ❤️ 高并发首选 |
| io_uring | 异步 | 无 | `io_uring_enter` | 极高 IOPS |

---

## 小结

| 概念 | 一句话 |
|------|--------|
| select | 1024 限制 + 每次 O(n) 轮询 → 已过时 |
| poll | 去掉 1024 限制，但仍是 O(n) |
| epoll | 红黑树 + 就绪链表，O(1) 取就绪事件 |
| LT | 默认，有数据就通知 |
| ET | 只通知一次，必须非阻塞 + 读到 EAGAIN |
| io_uring | 共享环形缓冲，零拷贝提交/完成，新一代异步 IO |

下一篇 [08 Reactor / Proactor 模式](./08-reactor-proactor.md)。
