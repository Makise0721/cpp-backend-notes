# 01 — perf 与火焰图：CPU 性能分析利器

> 关联篇目：[02 系统监控](./02-system-monitoring.md) | [05 Sanitizer](./05-sanitizers-valgrind.md)

---

## 1. perf 概览

`perf` 是 Linux 内核自带的性能分析工具（kernel source: `tools/perf/`），利用 CPU 的 PMU（Performance Monitoring Unit）硬件计数器。

### 1.1 三大核心用途

```
perf record: 采样 → 记录哪些函数在消耗 CPU
perf report: 解析采样数据 → 展示热点
perf top:    实时查看（类似 top，但是函数级别的 CPU 消耗）
perf stat:   统计性能计数器（IPC、cache-miss 等）
```

---

## 2. `perf record` + `perf report`——离线分析

```bash
# 采样某个进程（PID），持续 30 秒，频率 99 Hz
perf record -F 99 -p <PID> -g -- sleep 30

# 采样某个命令
perf record -F 99 -g ./my_program

# -F 99: 每秒采样 99 次（避免与定时器频率 100Hz 共振）
# -g:    记录调用栈（call graph）
# 结果写入 perf.data
```

```bash
# 查看报告
perf report

# 输出示例:
Samples: 10K of event 'cycles', Event count: 2.5e9
  Children      Self  Command  Symbol
+   80.00%     0.00%  my_app   [.] main
+   80.00%    40.00%  my_app   [.] computeMatrix
+   30.00%    30.00%  my_app   [.] matMul
+   10.00%    10.00%  my_app   [.] normalize

# Self: 函数自身消耗的 CPU 占比
# Children: 函数 + 其调用的子函数总共消耗的 CPU 占比
```

---

## 3. `perf top`——实时热点

```bash
# 实时查看系统全局热点函数
perf top

# 实时查看特定进程
perf top -p <PID>

# 只看用户态
perf top -p <PID> --exclude-kernel
```

---

## 4. `perf stat`——性能计数器

```bash
# 统计程序的硬件事件
perf stat ./my_program

# 输出示例:
Performance counter stats for './my_program':
   2,500.42 msec  task-clock           # CPU 利用率
      523,145    cache-misses          # L1 数据缓存未命中
    8,500,000    cache-references
      0.52       IPC                   # 每周期指令数（Instructions Per Cycle）
    1,200,000    branch-misses         # 分支预测失败
      0.30       cache-miss-rate

# 重点关注指标:
# IPC < 1 → CPU 经常在等内存（stall）
# IPC > 2 → 超标量流水线利用充分
# cache-miss 高 → 数据结构缓存不友好
# branch-miss 高 → 分支预测失败多（大量 if/switch？）
```

```bash
# 指定事件
perf stat -e cycles,instructions,cache-references,cache-misses,branch-misses ./prog
```

---

## 5. 火焰图（Flame Graph）

### 5.1 生成步骤

```bash
# 1. 采样（记录调用栈）
perf record -F 99 -p <PID> -g -- sleep 30

# 2. 转换成火焰图格式
perf script > out.perf

# 3. 用 Brendan Gregg 的脚本生成 SVG
git clone https://github.com/brendangregg/FlameGraph.git
./FlameGraph/stackcollapse-perf.pl out.perf > out.folded
./FlameGraph/flamegraph.pl out.folded > flamegraph.svg
```

### 5.2 如何解读火焰图

```
火焰图（y 轴 = 调用栈深度，x 轴 = CPU 占比）:

func_main           ← 底部 = 调用者
├─ parseJson        ← 宽 = CPU 占比大
│  └─ strlen        ← 窄 = 占比小
├─ compute          ← 平板顶 = 没有子函数调用
└─ ...

解读:
- 越宽 = 越耗 CPU → 优化的目标
- 平顶（plateau）= 函数自身耗时（没有子调用）
- 尖峰 = 深层调用链
- 颜色：随机，无特殊含义（但同一函数同色）
```

---

## 6. 实战：排查 CPU 瓶颈

```bash
# 1. 找到 CPU 使用率高的进程
top -c

# 2. 采样该进程
perf record -F 99 -p <PID> -g -- sleep 10

# 3. 看报告
perf report --stdio | head -50

# 4. 如果热点在某个函数 → 看调用栈 → 定位到具体代码行
perf annotate <function_name>  # 反汇编 + 源码级注释

# 5. 生成火焰图给老板看
```

---

## 小结

| 命令 | 用途 |
|------|------|
| `perf record -g` | 采样调用栈 |
| `perf report` | 查看热点函数 |
| `perf top` | 实时热点 |
| `perf stat` | 硬件性能计数器（IPC/cache-miss） |
| 火焰图 | perf record → stackcollapse → flamegraph.pl |

下一篇 [02 系统级监控：top/htop/vmstat/iostat/sar](./02-system-monitoring.md)。
