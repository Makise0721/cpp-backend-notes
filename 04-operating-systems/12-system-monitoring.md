# 02 — 系统级监控：top、htop、pidstat、vmstat、iostat、sar、free

> 关联篇目：[11 perf](./11-perf-flamegraph.md) | [16 网络调试](./16-network-debugging.md)

---

## 1. `top` / `htop`——进程级实时监控

```bash
top           # 基础版
htop          # 彩色增强版（需安装）

# top 关键列:
PID   USER  PR  NI  VIRT   RES   SHR  S  %CPU  %MEM   TIME+  COMMAND
1234  root  20   0  2.3g  1.1g  10m  R  95.0  14.2   5:23   my_app

PR:   优先级
VIRT: 虚拟内存
RES:  物理内存（驻留内存）
SHR:  共享内存
S:    状态 (R=运行, S=睡眠, D=不可中断, Z=僵尸)
```

```bash
# 只看某进程的所有线程
top -H -p <PID>

# 按 CPU 排序 (默认)
top -o %CPU
# 按内存排序
top -o %MEM
```

---

## 2. `pidstat`——进程级别的历史数据

```bash
# 每 1 秒采样一次，持续输出
pidstat 1

# 输出:
   UID   PID  %usr %system  %CPU  CPU  minflt/s  majflt/s  VSZ   RSS  Command
     0  1234  4.00   1.00  5.00    0      1000        0   2.3g  1.1g my_app

# 关键指标:
%usr:    用户态 CPU
%system: 内核态 CPU（system 太高 → 系统调用过多？）
minflt/s: 每秒 minor page faults（COW 等）
majflt/s: 每秒 major page faults（磁盘 I/O） ← 高 = 内存不够
```

```bash
# 查看 IO 统计
pidstat -d 1    # 每进程磁盘读写
# 查看上下文切换
pidstat -w 1    # cswch/s 自愿切换, nvcswch/s 非自愿切换
```

---

## 3. `vmstat`——虚拟内存统计

```bash
vmstat 1 5   # 每秒 1 次，共 5 次

procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0  1.5g   300m   2.0g    0    0    10    30  500  800  4  1 95  0  0
```

| 列 | 含义 | 关注点 |
|------|------|------|
| **r** | 运行队列长度 | > CPU 核数 → CPU 瓶颈 |
| **b** | 阻塞进程数 | > 0 → IO 瓶颈 |
| **si/so** | swap in/out | > 0 → 内存不够，在换页 |
| **bi/bo** | 块设备读写 | 高 → 磁盘 IO 瓶颈 |
| **us/sy/id/wa** | CPU 占比 | wa 高 → IO 等待 |

---

## 4. `iostat`——磁盘 I/O 统计

```bash
iostat -x 1    # 扩展信息，每秒更新

Device  r/s  w/s  rkB/s  wkB/s  await  svctm  %util
sda     50   30   2000   1000   5.00   2.00   80.0

# 关键:
await: 平均 I/O 响应时间（ms）→ 高 = 磁盘慢
%util: 磁盘繁忙度 → 接近 100% = 磁盘瓶颈
```

---

## 5. `sar`——历史数据收集

```bash
# 查看 CPU 历史
sar -u        # 当天的 CPU 使用率（每 10 分钟一个点）
sar -u 1 5    # 实时每秒

# 查看内存
sar -r

# 查看磁盘 IO
sar -b

# 查看网络
sar -n DEV     # 各网卡流量

# 查看负载
sar -q         # run queue + load average
```

---

## 6. `free` 与 `/proc/meminfo`——内存分析

```bash
free -h
              total   used   free   shared   buff/cache   available
Mem:           15G    5.0G   2.0G    500M        8.0G        9.5G
Swap:          2G     100M   1.9G

# available = 实际可给新进程用的内存（包括可回收的 cache）
# free = 完全未使用的内存（Linux 会把空闲内存做 cache，小是正常的）
# buff/cache = 缓冲区 + Page Cache（可回收）
```

```bash
# 详细信息
cat /proc/meminfo

# 重点关注:
MemTotal / MemAvailable    # 总内存 vs 可用
Cached                      # Page Cache（文件缓存）
SwapTotal / SwapFree        # Swap 使用
```

---

## 7. 监控组合拳

```
CPU 高？
├─ top 找到高 CPU 的进程
├─ pidstat 看是 usr 还是 sys
├─ perf top 找到具体函数
└─ 火焰图分析调用栈

内存高？
├─ free -h 看整体
├─ top -o %MEM 看哪个进程
├─ cat /proc/<pid>/status 看 VmRSS/VmSize
└─ heaptrack 或 valgrind 查分配

IO 慢？
├─ iostat -x 1 看 await 和 %util
├─ pidstat -d 看哪个进程在读写
└─ strace -p <pid> 看具体的 read/write

负载高但 CPU 不高？
├─ vmstat 看 b 列（阻塞进程）
├─ iostat 看磁盘是否瓶颈
└─ ps aux | grep D 看是否有 D 状态进程（不可中断睡眠）
```

---

## 小结

| 工具 | 监控内容 | 关键用法 |
|------|----------|----------|
| `top/htop` | 进程 CPU/内存 | `top -Hp <pid>` 看线程 |
| `pidstat` | 进程 CPU/IO/上下文切换 | `pidstat -dw 1` |
| `vmstat` | 系统整体（CPU/内存/IO） | `vmstat 1` 看 r/b/wa |
| `iostat` | 磁盘 I/O | `iostat -x 1` 看 await |
| `sar` | 历史数据 | `sar -u -r -n DEV` |
| `free` | 内存概览 | `free -h` |

下一篇 [03 GDB 与 Core Dump](./03-gdb-core-dump.md)。
