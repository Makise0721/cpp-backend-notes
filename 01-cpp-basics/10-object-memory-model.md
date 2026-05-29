# 10 — C++ 对象内存模型

> 关联篇目：[04 虚函数与 vtable](./04-virtual-vtable.md) | [05 菱形继承与虚继承](./05-diamond-virtual-inheritance.md) | [01 OOP 基础](./01-oop-basics.md)

---

## 1. 空类的大小——为什么是 1 字节？

```cpp
class Empty {};

std::cout << sizeof(Empty);  // 1
```

直觉上应该是 0——没有数据。但 C++ 标准要求**每个对象必须有唯一的地址**，否则两个不同的 `Empty` 对象会有同一个地址：

```cpp
Empty a, b;
&a == &b;  // 如果 size=0，a 和 b 可能在同一地址 → 无法区分
```

所以编译器给空类分配 1 字节占位。

### 1.1 有了虚函数后

```cpp
class EmptyWithVirtual {
    virtual ~EmptyWithVirtual() {}
};

std::cout << sizeof(EmptyWithVirtual);  // 8（64 位系统，vptr 大小）
```

空类 + 虚函数 → 编译器插入 vptr（8 字节）。1 字节占位不再需要（vptr 已经提供了唯一地址）。

---

## 2. 成员对齐与 Padding

CPU 读取内存有对齐要求——读一个 `int`（4 字节）要求地址是 4 的倍数，读一个 `double`（8 字节）要求地址是 8 的倍数。未对齐的访问要么慢，要么在某些架构上直接崩溃。

编译器在成员间插入 padding 以满足对齐要求：

```cpp
struct Example {
    char  a;   // 1 字节，偏移 0
    //       padding: 3 字节（让 int b 的偏移 = 4）
    int   b;   // 4 字节，偏移 4
    char  c;   // 1 字节，偏移 8
    //       padding: 7 字节（让整个结构体大小是 8 的倍数，满足 double 对齐）
    double d;  // 8 字节，偏移 16
};

std::cout << sizeof(Example);  // 24（不是 1+4+1+8=14！）
```

内存布局：

```
偏移  0   1   2   3   4   5   6   7   8   9  10  11  12  13  14  15
    ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
    │ a │   padding   │        b        │ c │          padding         │
    └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘

偏移 16  17  18  19  20  21  22  23
    ┌───┬───┬───┬───┬───┬───┬───┬───┐
    │             d                 │
    └───┴───┴───┴───┴───┴───┴───┴───┘

总大小 = 24 字节（必须是最大成员 double 的对齐值 8 的倍数）
```

### 2.1 重排成员减少空间浪费

```cpp
// 糟糕的排列：24 字节
struct Bad {
    char  a;
    int   b;
    char  c;
    double d;
};

// 优化排列：16 字节
struct Good {
    double d;   // 8 字节，偏移 0
    int    b;   // 4 字节，偏移 8
    char   a;   // 1 字节，偏移 12
    char   c;   // 1 字节，偏移 13
    // padding: 2 字节
};              // 总计 16 字节
```

**经验**：从大到小排列成员，减少 padding 浪费。这在 AI Infra 存储大量 tensor 元数据时尤其重要。

### 2.2 `alignof` 和 `alignas`

```cpp
std::cout << alignof(int);      // 4
std::cout << alignof(double);   // 8
std::cout << alignof(Example);  // 8（最大成员的对齐值）

// 手动指定对齐（常用于 SIMD / GPU 内存对齐）
struct alignas(64) CacheLineAligned {
    int data[16];
};
// sizeof(CacheLineAligned) = 64（必须是 alignas 的倍数）
```

---

## 3. 单继承内存布局

```cpp
class Base {
    int a_;
    double b_;
public:
    virtual void f() {}
};

class Derived : public Base {
    int c_;
public:
    void f() override {}
    virtual void g() {}
};
```

内存布局：

```
Derived 对象（64 位）:

┌──────────────────────┐ 偏移 0
│  vptr (8 bytes)      │ ← 指向 Derived 的 vtable
├──────────────────────┤ 偏移 8
│  Base::a_ (4 bytes)  │
│  padding  (4 bytes)  │ ← 对齐 double
├──────────────────────┤ 偏移 16
│  Base::b_ (8 bytes)  │
├──────────────────────┤ 偏移 24
│  Derived::c_ (4)     │
│  padding  (4)        │ ← 保证大小是 8 的倍数
├──────────────────────┤ 偏移 32
└──────────────────────┘

sizeof(Derived) = 32
```

**关键观察**：单继承下，基类子对象在派生类对象的开头。因此 `Derived*` 和 `Base*` 指向同一个地址——不需要指针调整。

```cpp
Derived d;
Base* pb = &d;
Derived* pd = &d;
// pb == pd（值相等），不需要偏移调整
```

---

## 4. 多继承内存布局

```cpp
class Base1 { int a_; virtual void f() {} };
class Base2 { double b_; virtual void g() {} };
class Derived : public Base1, public Base2 {
    int c_;
    void f() override {}
    void g() override {}
};
```

内存布局：

```
Derived 对象:

┌──────────────────────┐ 偏移 0
│  vptr1 (8)           │ → Derived 的 vtable（for Base1）
├──────────────────────┤ 偏移 8
│  Base1::a_ (4)       │
│  padding  (4)        │
├──────────────────────┤ 偏移 16
│  vptr2 (8)           │ → Derived 的 vtable（for Base2，含 thunk）
├──────────────────────┤ 偏移 24
│  Base2::b_ (8)       │
├──────────────────────┤ 偏移 32
│  Derived::c_ (4)     │
│  padding  (4)        │
├──────────────────────┤ 偏移 40
└──────────────────────┘

sizeof(Derived) = 40
```

**指针转换时发生偏移**：

```cpp
Derived d;
Base1* pb1 = &d;   // pb1 指向偏移 0（和 &d 相同）
Base2* pb2 = &d;   // pb2 指向偏移 16（≠ &d ！）

std::cout << &d;    // 比如 0x1000
std::cout << pb1;   // 0x1000
std::cout << pb2;   // 0x1010（偏移了 16 字节！）
```

`pb2` 的值不等于 `&d` 的值——编译器在 `Derived* → Base2*` 转换时自动加了偏移量（这是为什么 `reinterpret_cast` 在类层次间是危险的）。

---

## 5. 虚继承内存布局

```cpp
class Base {
    int a_;
public:
    virtual void f() {}
};

class Derived1 : public virtual Base {
    int b_;
    void f() override {}
};

class Derived2 : public virtual Base {
    int c_;
};

class MostDerived : public Derived1, public Derived2 {
    int d_;
};
```

内存布局（示意，具体由编译器实现）：

```
MostDerived 对象:

┌─────────────────────────┐ 偏移 0
│ Derived1 子对象           │
│ ├ vptr_d1 + vbase off  │ ← 包含到虚基类 Base 的偏移
│ └ Derived1::b_          │
├─────────────────────────┤
│ Derived2 子对象           │
│ ├ vptr_d2 + vbase off  │ ← 包含到虚基类 Base 的偏移
│ └ Derived2::c_          │
├─────────────────────────┤
│ MostDerived::d_         │
├─────────────────────────┤
│ Base 子对象（唯一一份）    │ ← 放在最后！
│ ├ vptr_base             │
│ └ Base::a_              │
└─────────────────────────┘
```

### 5.1 访问虚基类成员的过程

```cpp
void accessViaDerived1(Derived1* d1) {
    d1->a_ = 42;  // 编译器的做法：
    // 1. 通过 d1 的 vptr → vtable → 获取 vbase_offset_Base
    // 2. this_adjusted = reinterpret_cast<char*>(d1) + vbase_offset_Base
    // 3. reinterpret_cast<Base*>(this_adjusted)->a_ = 42
}
```

每次访问虚基类成员都多一次间接跳转——这就是虚继承的性能代价。在 AI Infra 中，一般应避免在热路径涉及的数据结构上使用虚继承。

---

## 6. 综合示例与验证

```cpp
#include <iostream>

struct A {
    virtual ~A() {}
    int x;
};  // 8 (vptr) + 4 (int) + 4 (padding) = 16

struct B : A {
    int y;
};  // 16 (A 部分) + 4 (int) + 4 (padding) = 16？不对——
    // vptr 属于 B，来自 A：8 + 4(x) + 4(y) + 0(padding?)=16
    // 实际上：8(vptr) + 4(x) + 4(y) = 16，正好 8 的倍数

struct C : virtual A {
    int z;
};  // 8 (vptr_C + vbase offset) + 4 (z) + 4 (padding) +
    // 8 (vptr_A) + 4 (x) + 4 (padding) = 32

int main() {
    std::cout << "A: " << sizeof(A) << "\n";  // 16
    std::cout << "B: " << sizeof(B) << "\n";  // 16
    std::cout << "C: " << sizeof(C) << "\n";  // 32 (具体因编译器而异)
}
```

> **注意**：内存布局是实现定义的（implementation-defined），不同编译器可能不同。以上基于 Itanium C++ ABI（GCC/Clang 在 Linux/macOS 上使用）。MSVC 有自己的布局规则。

---

## 7. 与 AI Infra 的关联

对象内存模型直接影响 AI 框架性能：

- **class 成员排列** → Tensor 元数据结构的 padding 会浪费 GPU 侧的内存带宽。PyTorch 的 TensorImpl 精心排列成员以避免 padding
- **虚函数开销** → AI 推理引擎（如 ONNX Runtime）大量使用虚函数做算子分发，每次 `kernel->compute()` 都是一次间接跳转 → 可以改用 CRTP 或函数指针表消除
- **缓存行** → CUDA 中 128 字节缓存行意味着结构体大小需要对齐到 128 字节来避免 false sharing → 用 `alignas(128)`
- **对象切片** → 在序列化/反序列化 model 参数时，如果把派生类对象按基类拷贝，成员会被截断 → 导致模型加载后行为异常

---

## 小结

| 概念 | 一句话 |
|------|--------|
| 空类大小 = 1 | 保证每个对象有唯一地址 |
| 成员对齐 padding | 按最大对齐要求插入填充字节，按大小降序排列减少浪费 |
| 单继承 | 基类子对象在偏移 0，`Derived*` 和 `Base*` 值相同 |
| 多继承 | 多个 vptr，`Base2*` 不等于 `Derived*`——编译器自动加偏移 |
| 虚继承 | 虚基类放在对象末尾，vtable 中有偏移信息，访问多一次间接 |
| `alignas` / `alignof` | 手动控制对齐，常用于 SIMD / GPU 内存 |

---

## 全系列小结

| # | 篇目 | 核心主题 |
|---|------|----------|
| 01 | OOP 基础 | 类/对象/封装/继承/多态/构造析构/深浅拷贝/friend |
| 02 | RAII 与三五零法则 | 资源管理哲学 + 特殊成员函数生成规则 |
| 03 | 静态成员与运算符重载 | 类级共享状态 + 自定义运算符语法 |
| 04 | 虚函数与 vtable | 动态多态的实现机制（vptr/vtable/thunk） |
| 05 | 菱形继承与虚继承 | 多继承的陷阱与解决方案 |
| 06 | 模板/SFINAE/CRTP | 编译期多态的三种武器 |
| 07 | STL 容器底层原理 | 六大容器的数据结构和复杂度分析 |
| 08 | STL 算法与迭代器 | 算法分类 + 迭代器层次 + 适配器 |
| 09 | 命名空间与异常 | 作用域管理 + 异常安全三级保证 + noexcept |
| 10 | 对象内存模型 | 字节级布局 + 对齐/padding + 多继承指针偏移 |

知识点之间相互关联，建议按顺序阅读，遇到不熟悉的概念回头看前序篇目中的解释。
