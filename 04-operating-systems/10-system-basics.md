# 10 — 系统调用、/proc、Cgroups/Namespaces、CPU 亲和性、eBPF

> 关联篇目：[01 进程基础](./01-process-basics.md) | [08 中断与异常](./08-interrupts.md)

---

## 1. 用户态 vs 内核态——特权级隔离

```
Ring 0: 内核态（可执行特权指令、访问所有内存）
Ring 3: 用户态（受限）
```

用户程序不能直接访问硬件、修改页表、关中断——必须通过系统调用请求内核代劳。

### 1.1 系统调用流程

```
用户程序调用 read()
  │
  ▼
libc 包装函数
  │ mov rax, 0 (syscall number for read)
  │ 设置参数寄存器 rdi, rsi, rdx...
  │ syscall 指令 (x86_64)
  ▼
CPU 切换到 Ring 0
  │ 保存用户态寄存器
  │ 从 MSR_LSTAR 加载内核入口地址
  │ 跳转到 entry_SYSCALL_64
  ▼
内核系统调用处理
  │ 从 sys_call_table[rax] 找到 sys_read
  │ 执行 sys_read
  ▼
返回用户态
  │ sysret 指令
  ▼
read() 返回给用户程序
```

```
x86 系统调用指令演化：
int 0x80   → (32 位，最慢，触发中断)
sysenter   → (32 位快速系统调用)
syscall    → (64 位，最快)
```

---

## 2. `/proc` 文件系统——运行时内核信息窗口

```bash
# CPU 信息
cat /proc/cpuinfo

# 内存信息
cat /proc/meminfo

# 进程信息
ls /proc/<PID>/
├── cmdline       # 启动命令
├── environ       # 环境变量
├── fd/           # 打开的文件描述符
│   ├── 0 → /dev/pts/0
│   ├── 1 → /dev/pts/0
│   └── 3 → socket:[12345]
├── maps          # 内存映射（地址空间布局）
├── status        # 进程状态、内存使用
├── limits        # 资源限制
├── cgroup        # 所属 cgroup
└── task/         # 线程信息

# 系统信息
cat /proc/version       # 内核版本
cat /proc/uptime        # 运行时间
cat /proc/loadavg       # 系统负载
cat /proc/stat          # CPU 统计
cat /proc/diskstats     # 磁盘 I/O 统计
cat /proc/net/tcp       # TCP 连接状态
```

**排查问题必看**：`/proc/<pid>/maps` 看进程地址空间，`/proc/<pid>/fd` 看泄漏了哪些文件描述符。

---

## 3. Cgroups & Namespaces——容器技术的两块基石

### 3.1 Namespaces（隔离）

**让进程只能看到被允许的系统资源子集。**

| Namespace | 隔离什么 |
|-----------|----------|
| **PID** | 进程树（容器内 PID 1 看不到宿主机其他进程） |
| **NET** | 网络栈（独立网卡、IP、端口、路由表） |
| **IPC** | System V IPC、POSIX 消息队列 |
| **MNT** | 挂载点（容器有自己的 `/`） |
| **UTS** | 主机名和域名 |
| **USER** | UID/GID（容器内 root ↔ 宿主机普通用户） |
| **Cgroup** | cgroup 视图 |

```bash
# 查看进程所属 namespace
$ ls -l /proc/$$/ns/
lrwxrwxrwx ... mnt -> mnt:[4026531840]
lrwxrwxrwx ... net -> net:[4026531992]
...
```

### 3.2 Cgroups（Control Groups，限制）

**限制进程组能使用的资源上限。**

```
Cgroup v2 层级:
/sys/fs/cgroup/
├── cpu.max          # CPU 使用上限（如 "200000 1000000" = 2 核）
├── memory.max       # 内存上限
├── memory.current   # 当前使用量
├── io.max           # I/O 带宽限制
└── pids.max         # 最大进程数
```

```bash
# Docker 容器的资源限制本质上就是 cgroup
docker run --cpus=2 --memory=4g my_image

# 相当于在 cgroup 中写入:
echo "200000 1000000" > /sys/fs/cgroup/docker/<container_id>/cpu.max
echo "4294967296" > /sys/fs/cgroup/docker/<container_id>/memory.max
```

---

## 4. CPU 亲和性与 NUMA 绑定

### 4.1 CPU 亲和性

将进程/线程绑定到指定的 CPU 核上，减少迁移带来的 cache 失效。

```bash
# 将进程绑定到 CPU 0,1
taskset -c 0,1 ./my_app

# 查看进程当前运行在哪些核上
taskset -p <PID>
```

```c
// 编程接口
cpu_set_t cpuset;
CPU_ZERO(&cpuset);
CPU_SET(0, &cpuset);
CPU_SET(1, &cpuset);
sched_setaffinity(0, sizeof(cpuset), &cpuset);
```

### 4.2 NUMA 绑定

```bash
# 在 NUMA 节点 0 上分配内存，在节点 0 的 CPU 上运行
numactl --cpunodebind=0 --membind=0 ./my_app

# AI 推理场景：
# GPU 0 连在 NUMA 0 → 确保 GPU 驱动和服务线程也在 NUMA 0 上
numactl -N 0 -m 0 python serve.py
```

---

## 5. eBPF——内核态可编程

**eBPF** （extended Berkeley Packet Filter）= 在**不修改内核源码、不加载内核模块**的前提下，将沙箱化的程序注入内核运行。

### 5.1 工作原理

```
用户态                   内核态
┌──────────┐          ┌─────────────┐
│ eBPF 程序 │  加载    │  验证器      │ ← 安全检查（无死循环、无越界）
│ (C/Rust)  │────────→│     ↓       │
└──────────┘          │  JIT 编译    │ ← 转成机器码
      ↑               │     ↓       │
      │ 读取数据       │ 挂载到 hook  │ ← kprobe/tracepoint/socket...
      │               │     ↓       │
      │               │ 执行 eBPF    │ ← 触发时运行
      │               └──────┬──────┘
      │                      │
      └────────── BPF maps ──┘  (共享内存，传数据到用户态)
```

### 5.2 典型应用

| 场景 | 工具 |
|------|------|
| 性能分析 | `bpftrace`、`bcc-tools`（`execsnoop`、`tcptop`） |
| 网络加速 | Cilium（eBPF 驱动的容器网络） |
| 安全监控 | Falco（运行时威胁检测） |
| 可观测性 | Pixie（零插桩的 K8s 观测平台） |

```bash
# 用 bpftrace 统计系统调用次数（一行命令，无需重启任何东西）
$ bpftrace -e 'tracepoint:syscalls:sys_enter_* { @[probe] = count(); }'

# 跟踪所有 exec 调用
$ bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%s\n", str(args->filename)); }'
```

### 5.3 eBPF 为什么重要

- 不需要重新编译内核 → 能在线上的生产环境直接使用
- 安全：验证器保证不会 crash 内核
- 高性能：JIT 编译，开销极低
- 可编程：想 hook 哪里就 hook 哪里

---

## 小结

| 概念 | 一句话 |
|------|--------|
| 系统调用 | `syscall` 指令 → Ring 0 → 内核处理 → `sysret` 返回 |
| `/proc` | 运行时内核信息，排查问题的第一站 |
| Namespaces | 资源隔离 — 容器看到什么 |
| Cgroups | 资源限制 — 容器能用多少 |
| CPU 亲和性 | 绑定进程到指定核，减少 cache miss |
| eBPF | 安全的内核态可编程，性能追踪利器 |

---

## 第四章小结

| # | 篇目 | 核心 |
|---|------|------|
| 01 | 进程基础 | PCB/状态模型/上下文切换 |
| 02 | 线程 | TCB/用户态内核态/NPTL/进程 vs 线程 |
| 03 | 进程创建与 IPC | fork/COW/exec/僵尸/守护/管道/共享内存/信号 |
| 04 | CPU 调度 | FCFS/SJF/RR/MLFQ/CFS(vruntime+红黑树) |
| 05 | 虚拟内存与分页 | 4 级页表/TLB/缺页/大页 |
| 06 | 页面置换与分配器 | Clock/伙伴系统/Slab/malloc(ptmalloc/jemalloc) |
| 07 | Cache/MESI/NUMA | Cache 层级/MESI/伪共享/numactl |
| 08 | 中断与异常 | IDT/上半部下/softirq/tasklet/workqueue |
| 09 | 文件系统 | VFS/inode/dentry/PageCache/硬软链接 |
| 10 | 系统基础 | 系统调用/proc/cgroups/namespace/CPU亲和性/eBPF |
