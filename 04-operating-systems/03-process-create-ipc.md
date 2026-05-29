# 03 — 进程创建、fork/COW/exec、僵尸/守护进程、IPC 全解

> 关联篇目：[01 进程基础](./01-process-basics.md) | [02 线程](./02-threads.md)

---

## 第一部分：进程创建

### 1. `fork()`——复制一个进程

```c
pid_t pid = fork();
if (pid == 0) {
    // 子进程：fork 返回 0
    printf("child: %d\n", getpid());
} else if (pid > 0) {
    // 父进程：fork 返回子进程 PID
    printf("parent: child PID=%d\n", pid);
} else {
    // fork 失败
}
```

**`fork` 做了什么？**
- 复制父进程的 `task_struct`
- 复制父进程的地址空间（**但现代 Linux 用 COW 优化**）
- 子进程获得父进程文件描述符的副本（指向同一打开文件表项）
- 子进程从 `fork()` 的返回点开始执行

### 1.2 写时复制（COW = Copy-On-Write）

**fork 后父子共享物理页，直到某一方尝试写入时才复制。**

```
fork 前:                       fork 后（COW 未触发）:
父进程页表 → [物理页 A]        父进程页表 → [物理页 A]（只读标记）
                             子进程页表 → [物理页 A]（只读标记）

子进程写入 → 触发缺页异常:
父进程页表 → [物理页 A]（恢复写权限）
子进程页表 → [物理页 B]（新分配的拷贝）
```

**COW 是关键性能优化**——如果子进程立刻 `exec`，地址空间根本不需要复制。

### 1.3 `vfork()`——几乎不用的历史遗留

`vfork` 保证子进程先运行，且父子共享地址空间（不 COW），子进程 `exec` 或 `_exit` 前父进程阻塞。**现代 Linux 中 `fork` + COW 已经足够高效，`vfork` 几乎无用。**

---

## 2. `exec` 系列——替换进程映像

```c
// exec 不会创建新进程，而是用新程序替换当前进程的代码和数据
execl("/bin/ls", "ls", "-l", NULL);
execv("/bin/ls", argv);
execle("/bin/ls", "ls", "-l", NULL, envp);  // 指定环境变量
execve("/bin/ls", argv, envp);              // 内核实际调用的是这个
execlp("ls", "ls", "-l", NULL);             // 从 PATH 搜索
execvp("ls", argv);                          // 同上
```

**`fork` + `exec` 是创建新进程的标准组合**（shell 就是这么启动命令的）。

---

## 3. 孤儿进程与僵尸进程

### 3.1 孤儿进程

父进程比子进程先结束 → 子进程成为"孤儿"→ **被 init (PID 1) 收养**→ init 定期 `wait` 回收。

### 3.2 僵尸进程

子进程已结束，但父进程还没调用 `wait` 回收 → 子进程变成**僵尸（Z 状态）**。PCB 还留着（占用 PID 和少量内核内存），但进程已不运行。

```bash
$ ps aux | grep Z
# Z 状态的进程就是僵尸

# 解决方法：父进程调用 wait/waitpid
```

```c
// 父进程回收子进程
int status;
wait(&status);                     // 阻塞等待任意子进程
waitpid(pid, &status, WNOHANG);    // 非阻塞等待指定子进程

// 处理 SIGCHLD 信号（子进程结束时父进程收到信号）
signal(SIGCHLD, sigchld_handler);
void sigchld_handler(int sig) {
    while (waitpid(-1, NULL, WNOHANG) > 0);  // 回收所有僵尸
}
```

### 3.3 守护进程（Daemon）

后台运行、脱离终端、以 root 或特定用户运行的长期进程。

**daemon 化六步**：

```c
// 1. fork 并让父进程退出（子进程变成孤儿）
if (fork() > 0) exit(0);

// 2. 子进程创建新会话（脱离原终端）
setsid();

// 3. 再次 fork 并让父进程退出（确保不再是会话首进程，无法获得终端）
if (fork() > 0) exit(0);

// 4. 切换工作目录到 /
chdir("/");

// 5. 重设 umask
umask(0);

// 6. 关闭从父进程继承的文件描述符
for (int i = 0; i < 1024; ++i) close(i);
// 重定向 0/1/2 到 /dev/null
```

---

## 第二部分：进程间通信（IPC）

### 4. IPC 全景图

| 方式 | 速度 | 数据量 | 需要同步？ | 适用场景 |
|------|:---:|:---:|:---:|------|
| **管道（匿名）** | 中 | 小/流式 | 否（内核同步） | 父子进程 |
| **命名管道（FIFO）** | 中 | 小/流式 | 否 | 任意进程 |
| **共享内存** | **最快** | 大 | ✅ 需信号量 | 大数据传输 |
| **消息队列** | 中 | 中 | 否（内核同步） | 结构化消息 |
| **信号量** | — | — | — | 同步（不是传数据） |
| **信号** | 快 | 极少（仅通知） | 不可靠（可能丢失） | 异步通知 |
| **套接字** | 较慢 | 大 | 否 | 跨网络 / 本地 |

### 4.2 管道（Pipe）

```c
int fd[2];
pipe(fd);  // fd[0] 读端, fd[1] 写端

if (fork() == 0) {
    close(fd[0]);              // 子进程：关闭读端
    write(fd[1], "hello", 5);  // 写数据
    close(fd[1]);
} else {
    close(fd[1]);              // 父进程：关闭写端
    char buf[10];
    read(fd[0], buf, sizeof(buf)); // 读数据
    close(fd[0]);
}
```

**半双工、只能在有亲缘关系的进程间使用**。`|` 管道符、`popen()` 都基于此。

### 4.3 命名管道（FIFO）

```bash
mkfifo mypipe
echo "hello" > mypipe &   # 一个进程写
cat mypipe                # 另一个进程读 → 输出 "hello"
```

**任意进程可用**，不要求亲缘关系。

### 4.4 共享内存——最快的 IPC

```
两个进程映射同一块物理内存:
进程 A 地址空间 ──→ [共享物理页] ←── 进程 B 地址空间

A 写入 → B 立即可见（无需内核中转，零拷贝！）
```

```c
// POSIX 共享内存
int fd = shm_open("/myshm", O_CREAT | O_RDWR, 0666);
ftruncate(fd, 4096);
void* ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
// ptr 指向共享内存，可像普通内存一样读写
```

但需要配合**信号量**或**互斥锁**来同步對共享内存的访问。

### 4.5 消息队列

不同于管道的字节流——消息队列发送的是**有边界的消息**（带类型和优先级）。

```c
// System V 消息队列
int msqid = msgget(key, IPC_CREAT | 0666);
msgsnd(msqid, &msg, sizeof(msg.text), 0);
msgrcv(msqid, &msg, sizeof(msg.text), msg_type, 0);
```

### 4.6 信号（Signal）

**异步通知机制**。内核或进程向目标进程发送信号，目标进程可以：
- 忽略（`SIGKILL` 和 `SIGSTOP` 不能忽略）
- 捕获（注册处理函数）
- 默认行为（终止 / 忽略 / 核心转储）

```c
// 可靠信号处理（推荐用 sigaction 而不是 signal）
struct sigaction sa;
sa.sa_handler = my_handler;
sigemptyset(&sa.sa_mask);
sa.sa_flags = SA_RESTART;  // 被信号打断的系统调用自动重启
sigaction(SIGINT, &sa, NULL);
```

| 信号 | 默认行为 | 说明 |
|------|----------|------|
| SIGINT (2) | 终止 | Ctrl+C |
| SIGKILL (9) | 终止 | **不可捕获、不可忽略** |
| SIGTERM (15) | 终止 | 优雅终止（可捕获） |
| SIGSTOP (19) | 暂停 | 不可捕获 |
| SIGCHLD (17) | 忽略 | 子进程状态变化 |
| SIGPIPE (13) | 终止 | 向无读端的管道写 |

### 4.7 信号量（Semaphore）

**不是传数据的，是控制并发访问的**。

```c
sem_t* sem = sem_open("/mysem", O_CREAT, 0666, 1);
sem_wait(sem);   // P 操作 → 减 1，为 0 时阻塞
// 临界区
sem_post(sem);   // V 操作 → 加 1，唤醒等待者
```

---

## 5. IPC 选择决策

```
需要传什么？
├─ 通知/事件 → 信号
├─ 小数据流（父子） → 管道
├─ 小数据流（任意进程） → 命名管道/消息队列
├─ 大数据/高频 → 共享内存 + 信号量
└─ 跨网络 → 套接字（Socket）
```

---

## 小结

| 概念 | 一句话 |
|------|--------|
| fork | 复制进程，现代 Linux 用 COW 延迟物理页复制 |
| exec | 替换当前进程的映像 |
| 僵尸进程 | 已结束但父进程未 `wait` — 泄漏 PID |
| 孤儿进程 | 父进程先结束 — init 收养 |
| 管道 | 父子进程间的字节流 IPC |
| 共享内存 | 最快的 IPC，需同步 |
| 信号 | 异步通知，`SIGKILL` 不可捕获 |

下一篇 [04 CPU 调度算法](./04-cpu-scheduling.md)。
