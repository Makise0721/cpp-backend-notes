# 03 — GDB 调试与 Core Dump 分析

> 关联篇目：[04 strace/ltrace/二进制分析](./04-strace-binary-analysis.md)

---

## 1. GDB 启动与断点

```bash
# 编译时加调试信息
g++ -g -O0 -o my_app my_app.cpp

# 启动调试
gdb ./my_app

# 附加到运行中的进程
gdb -p <PID>

# 分析 core dump
gdb ./my_app core
```

### 1.1 断点（Breakpoint）

```
(gdb) break main            # 在 main 函数入口
(gdb) b myfile.cpp:42       # 某文件某行
(gdb) b myfile.cpp:42 if x > 10   # 条件断点
(gdb) info breakpoints      # 查看所有断点
(gdb) delete 1              # 删除 1 号断点
(gdb) disable 1             # 禁用但不删除
```

### 1.2 执行控制

```
(gdb) run                   # 开始运行
(gdb) continue (c)          # 继续执行
(gdb) step (s)              # 单步进入（进入函数调用）
(gdb) next (n)              # 单步跳过（不进入函数）
(gdb) finish                # 执行完当前函数并返回
(gdb) until 100             # 运行到第 100 行
```

### 1.3 查看变量

```
(gdb) print x               # 打印变量
(gdb) p *ptr                # 解引用指针
(gdb) p array[0]@10         # 打印数组前 10 个元素
(gdb) display x             # 每次停下自动显示 x
(gdb) info locals           # 当前栈帧所有局部变量
(gdb) info args             # 当前函数参数
```

---

## 2. 调用栈分析

```
(gdb) backtrace (bt)        # 当前线程调用栈
(gdb) bt full               # 调用栈 + 每帧的局部变量
(gdb) frame 3               # 切换到栈帧 3
(gdb) up / down             # 上移/下移栈帧

# 多线程
(gdb) info threads          # 所有线程
(gdb) thread 5              # 切换到线程 5
(gdb) thread apply all bt   # 所有线程的调用栈 ← 诊断死锁必备
```

---

## 3. Watchpoint——监控变量变化

```
(gdb) watch x               # 当 x 被写入时暂停
(gdb) watch x if x == 0     # 条件 watchpoint
(gdb) rwatch x              # 读 watchpoint（被读取时暂停）
(gdb) awatch x              # 读写 watchpoint

# watchpoint 依赖硬件断点寄存器，数量有限（通常 4 个）
```

---

## 4. Core Dump 分析

### 4.1 启用 Core Dump

```bash
# 允许生成 core 文件
ulimit -c unlimited

# 设置 core 文件路径和命名
echo "/tmp/core.%e.%p" | sudo tee /proc/sys/kernel/core_pattern
# %e = 可执行文件名, %p = PID, %t = 时间戳
```

### 4.2 分析 Core Dump

```bash
gdb ./my_app /tmp/core.my_app.12345

# 进入后:
(gdb) bt full           # 崩溃时的调用栈 + 变量值
(gdb) info threads      # 所有线程
(gdb) thread apply all bt  # 所有线程栈（多线程崩溃必备）
(gdb) info registers    # 寄存器状态
(gdb) x/10x $rsp        # 查看栈内容（16 进制）
(gdb) frame 0           # 崩溃点
(gdb) list              # 崩溃点附近的源代码
```

### 4.3 常见崩溃排查

```
SIGSEGV (段错误):
  bt 看是哪一行 → 大概率是空指针 / 野指针 / 数组越界
  print ptr 确认指针值
  x/10x ptr 看指针指向的内存是否合法

SIGABRT (异常中止):
  通常是 assert 失败或调用 abort()
  bt 找到 abort 调用点

SIGFPE (浮点异常):
  除以零 / 整数溢出
```

---

## 5. TUI 模式与实用技巧

```bash
# TUI 模式（带源码窗口）
gdb -tui ./my_app
# 或启动后: Ctrl+X+A

# layout 切换
(gdb) layout src       # 源码
(gdb) layout asm       # 汇编
(gdb) layout split     # 源码 + 汇编
(gdb) layout regs      # 寄存器

# 在 GDB 内执行 shell 命令
(gdb) shell ls -la

# 保存断点
(gdb) save breakpoints bp.txt
(gdb) source bp.txt     # 恢复断点
```

---

## 6. 快速排查演示

```bash
# 进程卡住了？→ attach + 看调用栈
gdb -p <PID>
(gdb) info threads
(gdb) thread apply all bt
# → 看哪个线程在哪一行卡住了（大概率在等锁 / IO / 死循环）

# 进程崩溃了？→ core dump
gdb ./app core
(gdb) bt full
# → 看崩溃时的调用栈和变量
```

---

## 小结

| 命令 | 用途 |
|------|------|
| `break` | 设置断点 |
| `bt full` | 调用栈 + 局部变量 |
| `info threads` | 线程列表 |
| `thread apply all bt` | 所有线程栈（死锁/卡死诊断） |
| `watch` | 监控变量写 |
| `core dump` | `ulimit -c unlimited` + gdb 分析 |

下一篇 [04 strace / ltrace / 二进制分析](./04-strace-binary-analysis.md)。
