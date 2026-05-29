# 07 — CPU Cache、MESI 一致性协议、伪共享、NUMA 架构

> 关联篇目：[05 虚拟内存（大页）](./05-virtual-memory-paging.md) | [02 线程](./02-threads.md)

---

## 1. CPU Cache 层级——内存访问的金字塔

```
        ┌─────────────┐
        │    CPU 核    │
        │  ┌────────┐  │
        │  │寄存器   │  │ ← 0 cycles（不计延迟）
        │  ├────────┤  │
        │  │ L1 Cache│  │ ← ~1 ns, ~32 KB（指令+数据分离）
        │  ├────────┤  │
        │  │ L2 Cache│  │ ← ~4 ns, ~256 KB（每个核私有）
        │  ├────────┤  │
        │  │ L3 Cache│  │ ← ~12 ns, ~8-32 MB（同 socket 内所有核共享）
        └──┴────────┴──┘
               │
        ┌──────┴──────┐
        │   主存 RAM   │ ← ~60-100 ns
        └─────────────┘
```

**各级延迟的数量级差距**：

| 层级 | 延迟（约） | 大小 |
|------|:---:|------|
| L1 | 1 ns（~4 cycles） | 32 KB |
| L2 | 4 ns（~12 cycles） | 256 KB |
| L3 | 12 ns（~40 cycles） | 8-32 MB |
| 主存 | 60-100 ns | GB 级 |
| NUMA 远端内存 | ~130 ns | — |

### 1.1 Cache Line

CPU 缓存不是按字节读写的，而是按 **Cache Line**（通常是 **64 字节**）为最小单位。

```bash
$ getconf LEVEL1_DCACHE_LINESIZE
64    # L1 数据缓存行大小 = 64 字节
```

---

## 2. MESI——多核 Cache 一致性协议

四个状态（首字母组成 MESI）：

```
         Modified ──→ Exclusive
           │  ↖         │
           │    \       │
           ↓      ↘     ↓
         Shared  ←──  Invalid
```

| 状态 | 含义 | 该核有数据？ | 其他核有？ | 与内存一致？ |
|------|------|:---:|:---:|:---:|
| **M**odified | 已修改 | ✅ 且最新 | ❌ | ❌（脏数据） |
| **E**xclusive | 独占未改 | ✅ 且最新 | ❌ | ✅ |
| **S**hared | 多核共享 | ✅ | ✅ | ✅ |
| **I**nvalid | 无效 | ❌ | — | — |

**状态转换示例**：

```
核 A 读取地址 X：
  → X 不在 A 的 cache → 发出 Read 请求
  → 其他核都没有 X → A 获得 X，状态 = Exclusive
  → 再写 X → M 状态（不通知其他核，因为只有 A 有）

核 B 也想读 X：
  → A 有 X 在 M 状态 → A 必须写回内存并降级为 Shared
  → B 获得 X，状态 = Shared

核 B 要写 X：
  → 发出 Read-For-Ownership → A 的 Shared 副本变 Invalid
  → B 获得 X 的 Exclusive 权限 → 写入 → M 状态
```

**关键性能含义**：**两个线程频繁写同一 Cache Line → 不断触发 MESI 状态切换 → 性能灾难。**

---

## 3. 伪共享（False Sharing）——隐蔽的性能杀手

### 3.1 问题

```cpp
// 两个线程各自累加不同的变量
struct Counters {
    int a;  // 线程 1 频繁更新
    int b;  // 线程 2 频繁更新
};
// a 和 b 在同一个 64 字节 Cache Line 内！

Counters c;
thread t1([&] { for (;;) c.a++; });
thread t2([&] { for (;;) c.b++; });
// t1 写 c.a → 其核获得 M 状态
// t2 想写 c.b → 需要将 cache line 转移到 t2 的核
// → 两个核之间 cache line 不断 bounce！
// 性能可能比单线程还差！
```

虽然两个线程写的是**不同变量**，但因为它们在同一个 Cache Line，CPU 以为它们在争用同一数据。

### 3.2 解决方案：填充对齐

```cpp
struct alignas(64) Counter {
    int value;
    char padding[60];  // 填充到 64 字节，独占一个 Cache Line
};

alignas(64) Counter a;  // a 和 b 在不同 Cache Line
alignas(64) Counter b;
// 现在 t1 更新 a → t2 更新 b → 互不干扰！
```

```cpp
// C++17 标准做法
#include <new>
alignas(std::hardware_destructive_interference_size) int a;
alignas(std::hardware_destructive_interference_size) int b;
```

---

## 4. NUMA 架构——非统一内存访问

### 4.1 为什么需要 NUMA？

SMP（对称多处理）所有 CPU 共享一条内存总线 → 核多了就成瓶颈。NUMA 把内存和 CPU 分成多个"节点"，每个节点有自己的本地内存。

```
NUMA 架构（双路服务器）:
┌──────────────────────┐  ┌──────────────────────┐
│    Socket 0（NUMA 0） │  │    Socket 1（NUMA 1） │
│  ┌────┐ ┌────┐       │  │  ┌────┐ ┌────┐       │
│  │CPU0│ │CPU1│       │  │  │CPU2│ │CPU3│       │
│  └────┘ └────┘       │  │  └────┘ └────┘       │
│  ┌──────────────┐    │  │  ┌──────────────┐    │
│  │  本地内存     │    │  │  │  本地内存     │    │
│  │  (快)        │    │  │  │  (快)        │    │
│  └──────────────┘    │  │  └──────────────┘    │
└──────────┬───────────┘  └──────────┬───────────┘
           └───────── QPI/UPI ───────┘
           (跨节点访问远端内存 → 更慢)
```

### 4.2 延迟差异

| 访问类型 | 延迟 |
|----------|------|
| 本地内存 | ~100 ns |
| 远端内存 | ~130 ns（多 ~30%） |

```bash
$ numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3
node 0 size: 32768 MB
node 1 cpus: 4 5 6 7
node 1 size: 32768 MB
node distances:
node   0   1
  0:  10  21    ← 本地 10 vs 远端 21（相对值）
  1:  21  10
```

### 4.3 NUMA 绑定

```bash
# 将进程绑定到 NUMA node 0 上运行
numactl --cpunodebind=0 --membind=0 ./my_app

# 编程接口
#include <numa.h>
numa_run_on_node(0);
```

AI 推理场景：确保 GPU 所在 NUMA 节点的 CPU 来驱动该 GPU，避免跨节点数据传输。

---

## 5. 与 AI Infra 的关联

| 概念 | AI Infra 影响 |
|------|--------------|
| **大页** | 减少 TLB miss → DNN 推理中模型参数的大页映射提升 5-15% |
| **NUMA** | GPU 分配内存时绑定到近端 NUMA 节点，避免跨 socket 数据传输 |
| **伪共享** | 多线程更新 tensor 元数据时填充对齐 |
| **MESI** | 避免多线程频繁更新同一 cache line 的共享数据结构 |
| **Cache Line** | 数据结构对齐到 64B 边界提高缓存利用率 |

---

## 小结

| 概念 | 一句话 |
|------|--------|
| Cache 层级 | L1(~1ns)→L2(~4ns)→L3(~12ns)→RAM(~100ns) |
| Cache Line | CPU 缓存基本单位 64 字节 |
| MESI | 多核缓存一致性协议：Modified/Exclusive/Shared/Invalid |
| 伪共享 | 不同变量在同一 Cache Line → 多核频繁 invalidation |
| `alignas(64)` | 解决伪共享的标准方法 |
| NUMA | 近端内存快于远端；用 `numactl` 绑定 |

下一篇 [08 中断与异常](./08-interrupts.md)。
