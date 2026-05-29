# 06 — 页面置换算法、伙伴系统、Slab 分配器、malloc 内部实现

> 关联篇目：[05 虚拟内存与分页](./05-virtual-memory-paging.md) | [09 文件系统 Page Cache](./09-filesystem.md)

---

## 1. 页面置换算法——物理内存不够时踢谁？

### 1.1 OPT（最佳置换，理论最优）

踢**未来最长时间不会使用的页**。需要预知未来访问序列→ 无法实现，仅作参考基线。

### 1.2 FIFO

踢在内存中待得最久的页。

**Belady 异常**：增加物理页框数反而导致更多缺页！（FIFO 特有缺陷，LRU 等算法没有）

### 1.3 LRU（最近最久未使用）

踢**最近最久没有被访问的页**。理论上接近 OPT，但需要记录每次访问的时间戳 → 硬件开销大。

### 1.4 Clock（NRU，实际最常用）

**LRU 的近似实现**。用一个环形链表 + 访问位（Access bit）：

```
           指针
            ↓
       [A=1] → [A=0] → [A=1] → [A=0]
        ↑_____________________________|

置换时：
1. 检查指针指向的页的 A 位
2. A=0 → 踢出
3. A=1 → 清 0，指针前移，继续检查
```

**为什么不用精确 LRU？** 因为维护精确 LRU 需要每次访问都更新时间戳（或调整链表），开销太大。Clock 只需检查一个 bit（硬件自动设置的 Accessed bit）。

### 1.5 改进 Clock

同时考虑 A 位和 D 位（Dirty bit）：

```
A=0, D=0 → 最佳淘汰候选（既没被访问也没被写入）
A=0, D=1 → 次选（未访问但被写过，需写回磁盘）
A=1, D=0 → 可能很快会再访问
A=1, D=1 → 最不该淘汰
```

### 1.6 置换算法对比

| 算法 | 实现 | Belady 异常 | 实际使用 |
|------|------|:---:|------|
| OPT | 不可实现 | 无 | 理论基准 |
| FIFO | 简单 | **有** | 几乎不用 |
| LRU | 需要硬件支持 | 无 | 代价高 |
| Clock | O(1) | 无 | **Linux 实际使用** |
| 改进 Clock | O(1) + D 位 | 无 | 性能更好 |

---

## 2. 内存碎片——内部 vs 外部

```
内部碎片: 分配器给你 4KB，你只用 100 字节 → 浪费 3996 字节
外部碎片: 总空闲 1MB，但分散成 100 个 10KB 小块 → 申请 100KB 连续内存失败
```

| 类型 | 原因 | 谁负责 |
|------|------|--------|
| 内部碎片 | 分配粒度（固定大小） | 分配器多给了 |
| 外部碎片 | 多次分配释放后的离散空闲块 | 需压缩/重新整理 |

---

## 3. 伙伴系统（Buddy System）——内核分配连续物理页

### 3.1 原理

将物理内存视为 2^order 个连续页的块。分配时找到不小于请求大小的最小块，必要时**分裂**大块。释放时尝试与相邻同大小的空闲块**合并**（coalesce）。

```
请求 1 页（4KB），当前只有一个 8 页的空闲块:

[8 页空闲块]
   ↓ 分裂
[4 页] [4 页]
   ↓ 再分裂
[2 页] [2 页] [4 页]
   ↓ 再分裂
[1 页] [1 页] [2 页] [4 页]
   ↑ 分配这个

释放时：两个相邻且同大小的空闲伙伴 → 合并回更大的块
```

```bash
$ cat /proc/buddyinfo
Node 0, zone DMA32
    0    1    2    3    4    5    6    7    8    9   10  ← order
  12    8    4    2    1    0    1    0    0    0    1  ← 该 order 的空闲块数
```

### 3.2 伙伴系统的优缺点

| 优点 | 缺点 |
|------|------|
| 分配/释放 O(log n) | 内部碎片（分配大小必须是 2 的幂） |
| 自动合并减少外部碎片 | 分配小对象时效率低 |
| 实现简单可靠 | — |

---

## 4. Slab 分配器——内核对象的高速缓存

内核中频繁分配/释放小对象（`task_struct`、`inode`、`dentry` 等），每次都用伙伴系统太浪费。**Slab 在伙伴系统之上，为固定大小的对象提供缓存。**

### 4.1 结构

```
Slab 分配器:

Cache（特定类型对象，如 inode_cache）
├── Slab 1（已满）
├── Slab 2（部分空闲）
└── Slab 3（全空闲）

每个 Slab = 1 个或多个连续物理页，内部是固定大小的槽位
```

### 4.2 分配流程

```
请求 inode 对象:
1. 查 inode_cache 的"部分空闲" Slab → 直接分配
2. 没有"部分空闲" Slab → 从伙伴系统申请新页 → 创建新 Slab → 分配
3. 释放时 → 标记槽位为空闲 → 整个 Slab 空闲时归还伙伴系统
```

```bash
$ cat /proc/slabinfo | head
# 查看内核各种对象缓存的统计
```

---

## 5. 用户态 malloc 内部实现

### 5.1 三层结构

```
用户程序
  │ malloc / free
  ▼
分配器（ptmalloc / jemalloc / tcmalloc）
  │ brk() / mmap()
  ▼
内核（伙伴系统 / Slab）
```

### 5.2 brk vs mmap

```
brk: 调整数据段末尾（program break），扩展堆
     → 小块内存（通常 < 128KB）
     → 释放时只能从末尾收缩

mmap: 映射一块匿名内存到进程地址空间
      → 大块内存（≥ 128KB）
      → 可以独立 munmap
```

### 5.3 ptmalloc（glibc 默认）

- 基于 Doug Lea 的 dlmalloc，多线程优化
- 用 **arena**（分配区）减少锁竞争：多线程时创建多个 arena
- 用 bin（fastbin / smallbin / largebin / unsorted bin）管理不同大小的空闲块

### 5.4 jemalloc vs tcmalloc

| | jemalloc | tcmalloc |
|------|------|------|
| 作者 | Jason Evans | Google |
| 核心思想 | 线程缓存（tcache）+ 大小类 | 线程本地缓存 + 中心堆 |
| 碎片控制 | ✅ 很好 | ✅ 好 |
| 使用方 | FreeBSD、Redis、Rust | Chrome、gperftools |

---

## 6. 内存泄漏检测

```bash
# Valgrind
valgrind --leak-check=full ./my_program

# AddressSanitizer (ASan) —— 编译期插桩，比 valgrind 快
g++ -fsanitize=address -g -o my_program my_program.cpp

# 野指针 / 悬垂指针 → ASan 也能检测（use-after-free）
```

---

## 小结

| 概念 | 一句话 |
|------|--------|
| Clock 算法 | 环形扫描 + A 位，Linux 使用的 LRU 近似 |
| 伙伴系统 | 2^order 页块分配/合并，内核分配物理页 |
| Slab | 固定大小对象缓存，避免伙伴系统的碎片和开销 |
| malloc | glibc 用 ptmalloc（arena + bin），替代方案有 jemalloc/tcmalloc |
| 内部 vs 外部碎片 | 内部=分配器多给，外部=空闲离散 |

下一篇 [07 大页 / CPU Cache / MESI / 伪共享 / NUMA](./07-hugepages-cache-mesi-numa.md)。
