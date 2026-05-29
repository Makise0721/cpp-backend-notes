# 第一章：C++ 基础

> 10 篇 · 从面向对象到对象内存模型，搭建 C++ 语言核心骨架。

---

## 篇目

| # | 文件 | 核心内容 |
|---|------|----------|
| 01 | [01-oop-basics.md](./01-oop-basics.md) | 类/对象、封装、继承、多态、构造/析构/拷贝/移动、深浅拷贝、this、友元、内部类 |
| 02 | [02-raii-rule-of-five.md](./02-raii-rule-of-five.md) | RAII 资源管理哲学、Rule of Three/Five/Zero、`std::move` 本质、拷贝省略 |
| 03 | [03-static-operator-overload.md](./03-static-operator-overload.md) | 静态成员初始化顺序与线程安全、函数重载、运算符重载（成员/非成员）、不可重载的运算符 |
| 04 | [04-virtual-vtable.md](./04-virtual-vtable.md) | 虚函数、纯虚函数/抽象类、虚析构、vtable/vptr 单继承布局、多继承 thunk、override/final |
| 05 | [05-diamond-virtual-inheritance.md](./05-diamond-virtual-inheritance.md) | 菱形继承的二义性、虚继承、虚基类构造规则、vbase offset |
| 06 | [06-templates-sfinae-crtp.md](./06-templates-sfinae-crtp.md) | 函数/类/可变参数模板、全特化/偏特化、SFINAE、enable_if、void_t、CRTP、if constexpr |
| 07 | [07-stl-containers-deep-dive.md](./07-stl-containers-deep-dive.md) | vector 扩容(1.5x/2x)、deque 中控器、红黑树、哈希表(开链/rehash)、容器适配器 |
| 08 | [08-stl-algorithms-iterators.md](./08-stl-algorithms-iterators.md) | 5 类迭代器、算法分类、仿函数/lambda、适配器、迭代器失效场景 |
| 09 | [09-namespace-exception-noexcept.md](./09-namespace-exception-noexcept.md) | 命名空间/ADL/作用域、异常类层次、三级异常安全保证、noexcept |
| 10 | [10-object-memory-model.md](./10-object-memory-model.md) | 空类大小、对齐/padding、单/多/虚继承布局、指针偏移、alignas/alignof |

## 路线

建议按顺序阅读：OOP 基础 → RAII（C++ 精髓）→ 虚函数机制 → 模板 → STL → 内存模型收尾。
