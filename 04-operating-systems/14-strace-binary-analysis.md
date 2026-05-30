# 04 — strace、ltrace 与二进制分析

> 关联篇目：[13 GDB](./13-gdb-core-dump.md) | [第一章 06 网络排查](../06-network-programming/10-troubleshooting.md)

---

## 1. `strace`——系统调用追踪

### 1.1 基本用法

```bash
# 附加到运行中的进程
strace -p <PID>

# 启动并追踪
strace ./my_app

# 只看特定系统调用
strace -e trace=read,write,open,close ./my_app

# 只看网络相关
strace -e trace=network ./my_app

# 统计各系统调用的次数和耗时
strace -c ./my_app
```

### 1.2 实战场景

```bash
# 场景 1: 程序卡住了？→ 看它在等什么
strace -p <PID>
# 输出: futex(0x..., FUTEX_WAIT, ... → 在等锁
#       read(3, ...)            → 在等 IO 读
#       epoll_wait(4, ...)      → 在等网络事件

# 场景 2: 文件打不开？
strace -e trace=open,openat ./my_app 2>&1 | grep ENOENT
# → 找到是哪个文件不存在

# 场景 3: 网络连不上？
strace -e trace=network ./my_app
# connect(3, {sa_family=AF_INET, sin_port=htons(3306), sin_addr=inet_addr("10.0.0.1")}, 16) = -1 ETIMEDOUT
# → 连接 10.0.0.1:3306 超时

# 场景 4: 显示每个系统调用的时间戳和耗时
strace -T -tt -p <PID>
# -T: 显示每个调用的耗时
# -tt: 显示微秒精度的时间戳
```

### 1.3 常用过滤

```bash
strace -f          # 追踪子进程（fork）
strace -p <PID> -o out.log  # 输出到文件
strace -s 1024     # 增加输出的字符串长度（默认 32 字节）
strace -k          # 显示调用栈（需要 strace 4.17+）
```

---

## 2. `ltrace`——库函数调用追踪

```bash
# 查看程序调用了哪些库函数
ltrace ./my_app

# 输出: malloc(1024) = 0x7f...
#       memcpy(0x..., "hello", 5) = 0x...
#       printf("result=%d\n", 42) = 10

# 只追踪特定函数
ltrace -e malloc+free+strlen ./my_app
```

---

## 3. 二进制分析四件套

### 3.1 `ldd`——查看动态库依赖

```bash
ldd ./my_app
# linux-vdso.so.1
# libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6
# libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6

# 排查"找不到库"问题——看哪个 so 是 "not found"
```

### 3.2 `nm`——查看符号表

```bash
# 列出所有符号（函数名、全局变量等）
nm ./my_app | grep my_function

# 符号类型:
# T = .text 段中的函数
# D = 已初始化数据
# B = 未初始化数据 (BSS)
# U = 未定义（需要从动态库导入）

# 只显示未定义符号（看依赖哪些外部函数）
nm -u ./my_app
```

### 3.3 `objdump`——反汇编

```bash
# 反汇编所有代码段
objdump -d ./my_app | less

# 反汇编某个函数
objdump -d ./my_app | grep -A 20 '<my_function>'

# 查看各段的大小
objdump -h ./my_app

# 配合源码（编译时加 -g）
objdump -S ./my_app
```

### 3.4 `readelf`——ELF 文件详情

```bash
# 查看 ELF 头
readelf -h ./my_app

# 查看段信息
readelf -S ./my_app

# 查看动态段
readelf -d ./my_app | grep NEEDED  # 依赖的 so

# 查看符号表
readelf -s ./my_app
```

### 3.5 `strings`——快速提取字符串

```bash
# 从二进制中提取所有可打印字符串
strings ./my_app | less

# 用于快速查看:
# - 硬编码的配置路径
# - 版本信息
# - 错误消息
strings ./my_app | grep -i version
```

---

## 4. 排查问题决策树

```
程序启动失败？
├─ ldd → 库缺失？
├─ strace -e trace=open,openat → 配置文件找不到？
└─ readelf -d | grep NEEDED → 依赖的 so 版本不对？

程序运行异常（非崩溃）？
├─ strace -p <PID> → 在等什么系统调用？
├─ ltrace → 库函数调用顺序？
└─ /proc/<PID>/fd → 打开了哪些文件/socket？

性能问题？
├─ perf top → 热点函数
├─ strace -c → 哪种系统调用最多/最耗时
└─ strace -T -p <PID> → 哪个系统调用最慢
```

---

## 小结

| 工具 | 用途 | 典型命令 |
|------|------|----------|
| `strace` | 系统调用追踪 | `strace -p PID -e trace=network -T` |
| `ltrace` | 库函数追踪 | `ltrace -e malloc+free ./app` |
| `ldd` | 动态库依赖 | `ldd ./app` |
| `nm` | 符号表 | `nm -u ./app` |
| `objdump -d` | 反汇编 | `objdump -d -S ./app` |
| `strings` | 提取字符串 | `strings ./app \| grep keyword` |

下一篇 [05 Sanitizer 与 Valgrind：内存检测三剑客](./05-sanitizers-valgrind.md)。
