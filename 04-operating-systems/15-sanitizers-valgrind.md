# 05 — Sanitizer 与 Valgrind：内存错误检测

> 关联篇目：[11 perf](./11-perf-flamegraph.md) | [13 GDB](./13-gdb-core-dump.md)

---

## 1. Sanitizer 三剑客——编译器插桩检测

### 1.1 共同特性

- **编译期插桩**：在代码中插入检查逻辑（而非运行时解释，比 Valgrind 快很多）
- **~2x 慢**（Valgrind 可能 20-30x 慢）
- 需要重新编译，加对应的 `-fsanitize=xxx` 标志

---

## 2. AddressSanitizer（ASan）——内存错误检测

### 2.1 检测什么

```
- 堆缓冲区溢出（heap buffer overflow）
- 栈缓冲区溢出（stack buffer overflow）
- 全局缓冲区溢出
- Use-after-free（释放后使用）
- Use-after-return
- Double-free / Invalid free
- 内存泄漏（LeakSanitizer，ASan 集成）
```

### 2.2 使用

```bash
# 编译
g++ -fsanitize=address -g -O1 -o my_app my_app.cpp
# 可选: ASAN_OPTIONS 环境变量

# 运行
./my_app
# 发现错误 → 打印详细的错误信息 + 调用栈

# 输出示例:
==12345==ERROR: AddressSanitizer: heap-buffer-overflow
READ of size 4 at 0x60300000fff0
    #0 0x4a2c3e in main my_app.cpp:42
    #1 0x7f...

0x60300000fff0 is located 0 bytes to the right of 16-byte region
allocated by thread T0 here:
    #0 0x4a1b8d in operator new[](unsigned long)
    #1 0x4a2c1e in main my_app.cpp:40
```

### 2.3 常用 ASAN_OPTIONS

```bash
export ASAN_OPTIONS=detect_leaks=1:halt_on_error=0:log_path=asan.log
# detect_leaks=1: 启用内存泄漏检测（默认）
# halt_on_error=0: 遇到错误继续运行（而不是立刻 abort）
# log_path: 输出日志文件
```

---

## 3. ThreadSanitizer（TSan）——数据竞争检测

### 3.1 检测什么

- 两个线程并发访问同一内存位置
- 至少一个是写操作
- 没有同步（锁/原子操作）保护

```bash
g++ -fsanitize=thread -g -O1 -o my_app my_app.cpp -pthread

./my_app
# 输出:
WARNING: ThreadSanitizer: data race (pid=12345)
  Write of size 4 at 0x7b0400000000 by thread T2:
    #0 increment() my_app.cpp:10
  Previous read of size 4 at 0x7b0400000000 by thread T1:
    #0 read_counter() my_app.cpp:15

# 有些误报是可能的（如用户态自旋锁），但不常见
```

### 3.2 注意

- 不与 ASan 同时使用（`-fsanitize=address,thread` 会冲突）
- 性能开销较大（~5-15x 慢）
- 不能和 ASan/Valgrind 同时运行

---

## 4. UndefinedBehaviorSanitizer（UBSan）——未定义行为检测

```bash
g++ -fsanitize=undefined -g -o my_app my_app.cpp
```

检测：
- 整数溢出（有符号）
- 除以零
- 空指针解引用
- 非法类型转换
- 数组越界（部分）
- `memcpy` 参数重叠

```bash
# 更全面的 UBSan
g++ -fsanitize=undefined,integer,nullability,alignment -o my_app my_app.cpp
```

---

## 5. Valgrind——无需重新编译的内存分析

### 5.1 Memcheck（默认工具）

```bash
# 不需要重新编译！（但最好有调试符号 -g）
valgrind --leak-check=full ./my_app

# 输出:
==12345== Invalid write of size 4
==12345==    at 0x4005F4: main (my_app.cpp:10)
==12345==  Address 0x5203044 is 0 bytes after a block of size 4 alloc'd

==12345== 4 bytes in 1 blocks are definitely lost
==12345==    at 0x4C2DB8F: malloc
==12345==    by 0x4005F0: main (my_app.cpp:8)
```

**Valgrind vs ASan**：

| | ASan | Valgrind |
|------|:---:|:---:|
| 速度 | ~2x 慢 | ~20-30x 慢 |
| 需重编译 | ✅ | ❌ |
| 检测范围 | 栈 + 堆 + 全局 | 主要堆 |
| 精确度 | 高 | 中（有误报可能） |

---

## 6. 堆内存剖析

### 6.1 `heaptrack`

```bash
# 追踪堆分配
heaptrack ./my_app
# 生成 heaptrack.my_app.PID.gz

# 分析
heaptrack_gui heaptrack.my_app.PID.gz
# 或命令行
heaptrack_print heaptrack.my_app.PID.gz | less
```

输出：哪些函数分配了多少内存、峰值内存、分配次数。

### 6.2 Valgrind Massif

```bash
valgrind --tool=massif ./my_app
# 生成 massif.out.PID

ms_print massif.out.PID
# 输出内存使用随时间变化的图表（ASCII art）
```

---

## 7. 内存检测选择

```
开发阶段：
├─ 编译时加 -fsanitize=address,undefined  → 日常开发/测试
├─ 怀疑数据竞争 → -fsanitize=thread
└─ CI 用 ASan + UBSan 自动检测

排查线上问题（core dump）：
└─ 已经有 core → GDB 分析（无法重新编译线上代码时）

无法重新编译的二进制：
└─ valgrind --leak-check=full ./binary

内存膨胀排查：
├─ heaptrack → 看哪个函数分配最多
└─ massif → 看内存增长趋势
```

---

## 小结

| 工具 | 检测内容 | 需要重编译 |
|------|----------|:---:|
| ASan | 内存越界/use-after-free/泄漏 | ✅ |
| TSan | 数据竞争 | ✅ |
| UBSan | 未定义行为 | ✅ |
| Valgrind Memcheck | 内存错误（堆为主） | ❌ |
| Heaptrack | 堆分配剖析 | ❌ |

下一篇 [06 网络调试工具](./06-network-debugging.md)。
