# 第四章：操作系统

> 10 篇 · 从进程线程到 eBPF，覆盖后端工程师必备的 OS 内核知识。

---

## 篇目

| # | 文件 | 核心内容 |
|---|------|----------|
| 01 | [01-process-basics.md](./01-process-basics.md) | 进程概念、PCB(task_struct)、状态模型(三/五/七态)、上下文切换开销 |
| 02 | [02-threads.md](./02-threads.md) | 线程/TCB、用户态/内核态线程、NPTL(LWP+clone)、进程 vs 线程对比、共享/私有资源 |
| 03 | [03-process-create-ipc.md](./03-process-create-ipc.md) | fork(COW)/exec/vfork、孤儿/僵尸进程、守护进程化、IPC 六大方式(管道/共享内存/消息队列/信号量/信号/套接字) |
| 04 | [04-cpu-scheduling.md](./04-cpu-scheduling.md) | FCFS/SJF/RR/MLFQ、Linux CFS(vruntime+红黑树) |
| 05 | [05-virtual-memory-paging.md](./05-virtual-memory-paging.md) | 虚拟内存、4 级页表(PGD→PUD→PMD→PTE)、TLB、缺页异常流程、大页 |
| 06 | [06-page-replacement-allocator.md](./06-page-replacement-allocator.md) | Clock 算法、伙伴系统、Slab 分配器、malloc 内部(ptmalloc/jemalloc/tcmalloc)、内存碎片 |
| 07 | [07-hugepages-cache-mesi-numa.md](./07-hugepages-cache-mesi-numa.md) | L1/L2/L3 Cache、MESI 一致性协议、伪共享(False Sharing)+alignas、NUMA 架构 |
| 08 | [08-interrupts.md](./08-interrupts.md) | 中断/异常分类、IDT、中断上下文限制、上半部下(softirq/tasklet/workqueue) |
| 09 | [09-filesystem.md](./09-filesystem.md) | VFS 四大对象(superblock/inode/dentry/file)、Page Cache、硬链接 vs 软链接 |
| 10 | [10-system-basics.md](./10-system-basics.md) | 系统调用流程(syscall)、/proc 文件系统、cgroups/namespaces(容器基石)、CPU 亲和性、eBPF |

## 路线

进程→线程→调度是 CPU 抽象线；虚拟内存→页面置换→分配器是内存线；Cache/NUMA 是性能线；中断/文件系统/系统调用是 I/O 线。
