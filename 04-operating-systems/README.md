# 第四章：操作系统与性能调试

> 16 篇 · 从进程线程到 eBPF，从 perf 火焰图到网络抓包——操作系统内核 + 性能分析全工具链。

### 操作系统基础 (01-10)

| # | 核心内容 |
|---|------|
| 01 | 进程概念、PCB(task_struct)、状态模型、上下文切换 |
| 02 | 线程/TCB、用户态/内核态线程、NPTL、进程 vs 线程 |
| 03 | fork(COW)/exec/vfork、孤儿/僵尸、守护进程、IPC 六大方式 |
| 04 | FCFS/SJF/RR/MLFQ、Linux CFS(vruntime+红黑树) |
| 05 | 虚拟内存、4 级页表、TLB、缺页异常、大页 |
| 06 | Clock 算法、伙伴系统、Slab、malloc 内部、内存碎片 |
| 07 | L1/L2/L3 Cache、MESI、伪共享+alignas、NUMA |
| 08 | 中断/异常分类、IDT、上半部下(softirq/tasklet/workqueue) |
| 09 | VFS 四大对象、Page Cache、硬链接 vs 软链接 |
| 10 | 系统调用流程、/proc、cgroups/namespaces、CPU 亲和性、eBPF |

### 性能分析与调试 (11-16)

| # | 核心内容 |
|---|------|
| 11 | perf record/report/top/stat、火焰图生成与解读 |
| 12 | top/htop、pidstat、vmstat、iostat、sar、free |
| 13 | GDB 断点/watchpoint、Core Dump 分析、多线程调试 |
| 14 | strace/ltrace、ldd/nm/objdump/readelf/strings |
| 15 | ASan/TSan/UBSan、Valgrind Memcheck、Heaptrack |
| 16 | tcpdump/Wireshark、ss、dig、curl、iftop/nethogs |

路线：前半部分 (01-10) OS 内核原理线，后半部分 (11-16) 性能分析和故障排查工具线。
