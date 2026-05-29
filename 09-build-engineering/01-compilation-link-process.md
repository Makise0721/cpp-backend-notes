# 01 — C/C++ 编译链接全过程

> 关联篇目：[02 CMake](./02-cmake-basics.md)

---

## 1. 编译四阶段

```
源文件 (.cpp/.c) 的完整旅程:

main.cpp ──→ 预处理 ──→ 编译 ──→ 汇编 ──→ 链接 ──→ 可执行文件
                │        │        │        │
                ▼        ▼        ▼        ▼
            main.i    main.s    main.o    a.out
           (展开后)  (汇编)   (目标文件) (可执行)
```

### 1.1 预处理（Preprocessing）

```
处理指令: #include, #define, #ifdef, #pragma...

g++ -E main.cpp -o main.i    # 只预处理
g++ -E -dM main.cpp          # 查看所有预定义宏
```

做的事情：
- `#include` → 头文件内容原样插入
- `#define` → 宏展开
- `#ifdef / #ifndef` → 条件编译
- 删除注释

### 1.2 编译（Compilation）

```
C/C++ 代码 → 汇编代码

g++ -S main.cpp -o main.s    # 生成汇编
g++ -S -O2 main.cpp          # 看优化后的汇编
```

- 词法分析 → 语法分析 → 语义分析 → 中间代码生成 → 优化 → 汇编生成
- **此时每个 .cpp 独立编译，不看其他文件**

### 1.3 汇编（Assembly）

```
汇编代码 → 机器码（目标文件 .o）

g++ -c main.cpp -o main.o    # 生成目标文件
as main.s -o main.o          # 用汇编器
```

目标文件包含：
- 机器码（`.text`）
- 已初始化数据（`.data`）
- 未初始化数据（`.bss`）
- 符号表（导出和导入的符号）
- 重定位信息（哪些地址需要在链接时修正）

### 1.4 链接（Linking）

```
多个 .o + 静态库 .a → 可执行文件
多个 .o + 动态库 .so → 可执行文件（运行时再加载 .so）

g++ main.o utils.o -o my_app       # 链接多个 .o
g++ main.o -L. -lutils -o my_app    # 链接静态库 libutils.a
```

---

## 2. 静态库 vs 动态库

### 2.1 静态库（`.a` / `.lib`）

```
创建:
ar rcs libutils.a utils.o math.o    # 打包多个 .o

链接:
g++ main.o -L. -lutils -o my_app    # 库代码嵌入可执行文件
```

| 优点 | 缺点 |
|------|------|
| 运行时不需要外部依赖 | 可执行文件大 |
| 版本固定，不冲突 | 库更新需要重编译所有使用者 |

### 2.2 动态库（`.so` / `.dll`）

```
创建:
g++ -shared -fPIC -o libutils.so utils.cpp

链接:
g++ main.o -L. -lutils -o my_app    # 可执行文件只记录依赖

运行时:
ld.so 在 LD_LIBRARY_PATH 和标准路径中查找 libutils.so
```

| 优点 | 缺点 |
|------|------|
| 可执行文件小 | 运行时必须找到 .so |
| 多程序共享同一 .so（节省内存） | 版本不兼容→找不到库 / 符号错误 |
| 升级库不需要重编译程序 | 部署复杂 |

### 2.3 动态库查找路径

```bash
# 运行时查找 .so 的顺序:
# 1. LD_LIBRARY_PATH 环境变量
# 2. /etc/ld.so.cache (由 ldconfig 生成)
# 3. /lib, /usr/lib

# 设置
export LD_LIBRARY_PATH=/opt/mylib:$LD_LIBRARY_PATH

# 永久添加路径
echo "/opt/mylib" | sudo tee /etc/ld.so.conf.d/mylib.conf
sudo ldconfig

# 查看程序的库依赖
ldd ./my_app
```

---

## 3. `-fPIC`——位置无关代码

```
动态库必须用 -fPIC 编译:

g++ -fPIC -shared -o libutils.so utils.cpp

-fPIC: Position Independent Code
→ 生成的代码可以在任意内存地址运行
→ 通过 GOT (Global Offset Table) 和 PLT (Procedure Linkage Table) 实现
```

**为什么动态库需要 PIC？** 多个进程共享同一 .so 时，.so 可能被加载到不同进程的不同虚拟地址。PIC 确保代码不依赖绝对地址。

### 3.1 GOT 与 PLT

```
PLT (Procedure Linkage Table): 函数调用跳板
GOT (Global Offset Table):     存储全局变量/函数的实际地址

调用 printf:
  call printf@PLT
    → 首次: 跳转到 ld.so → 解析 printf 地址 → 写入 GOT → 调用 printf
    → 后续: 跳转到 GOT[printf] → 直接调用 printf（延迟绑定）
```

---

## 4. 强符号与弱符号

```c
// 默认是强符号
int global_var = 42;        // 强符号

// 弱符号
__attribute__((weak)) int global_var = 0;  // 弱符号
__attribute__((weak)) void my_func() { }   // 弱函数
```

| | 强符号 | 弱符号 |
|------|:---:|:---:|
| 定义 | 默认 | `__attribute__((weak))` |
| 链接冲突 | 多个强符号同名 → 链接错误 | 强 + 弱 → 强优先 |
| 用途 | 正常 | 可被覆盖的默认实现 |

**弱符号的应用**：
- 提供默认实现，用户可覆盖
- 可选的函数（如 `pthread_create` 的弱符号桩）
- 中断向量表（ARM 嵌入式）

---

## 5. 静态库链接顺序——被依赖的放后面

```bash
# ❌ 顺序错误 → "undefined reference"
g++ main.o -lutils -lmath -o my_app

# 如果 libmath 依赖 libutils 中的符号 → 链接失败！
# 链接器从左到右处理，处理 libutils 时，libmath 需要的符号还没出现

# ✅ 正确顺序
g++ main.o -lmath -lutils -o my_app
# 或者用 --start-group --end-group（让链接器多遍扫描）
g++ main.o -Wl,--start-group -lmath -lutils -Wl,--end-group -o my_app
```

**规则**：被依赖的库放后面。如果出现循环依赖，用 `--start-group --end-group`。

---

## 小结

| 阶段 | 命令 | 产出 |
|------|------|------|
| 预处理 | `g++ -E` | `.i` 展开后源码 |
| 编译 | `g++ -S` | `.s` 汇编 |
| 汇编 | `g++ -c` | `.o` 目标文件 |
| 链接 | `g++ *.o -L. -lfoo` | 可执行文件 |
| 静态库 | `ar rcs libfoo.a *.o` | `.a` |
| 动态库 | `g++ -shared -fPIC` | `.so` |
| 符号 | `nm` / `readelf -s` | 符号表 |

下一篇 [02 CMake 基础](./02-cmake-basics.md)。
