# 第二章：C++11/14/17/20 核心特性

> 11 篇 · 从类型推导到 C++20 Concepts/Coroutines，覆盖现代 C++ 全部关键特性。

---

## 篇目

| # | 文件 | 核心内容 |
|---|------|----------|
| 01 | [01-type-deduction.md](./01-type-deduction.md) | auto/decltype/decltype(auto) 推导规则与差异 |
| 02 | [02-lambda-deep-dive.md](./02-lambda-deep-dive.md) | Lambda 捕获方式、mutable、底层匿名仿函数类、无状态→函数指针、泛型 lambda |
| 03 | [03-value-categories-move-forward.md](./03-value-categories-move-forward.md) | 值类别（lvalue/prvalue/xvalue）、万能引用、引用折叠、完美转发、RVO/NRVO |
| 04 | [04-smart-pointers.md](./04-smart-pointers.md) | unique_ptr/shared_ptr/weak_ptr、控制块、make_shared、删除器、线程安全 |
| 05 | [05-modern-class-features.md](./05-modern-class-features.md) | nullptr、range-for、类内初始值、=default/=delete、委托/继承构造、enum class |
| 06 | [06-constexpr-consteval.md](./06-constexpr-consteval.md) | constexpr vs const、consteval（C++20）、constinit（C++20）、编译期求值边界 |
| 07 | [07-using-variadic-fold.md](./07-using-variadic-fold.md) | using 别名模板、可变参数模板、折叠表达式（C++17） |
| 08 | [08-tuple-optional-variant-any.md](./08-tuple-optional-variant-any.md) | pair/tuple、optional、variant、any——多值处理四种工具 |
| 09 | [09-concurrency-basics.md](./09-concurrency-basics.md) | thread、mutex 系列、atomic 基础、call_once、shared_mutex（读写锁） |
| 10 | [10-cpp17-sugar.md](./10-cpp17-sugar.md) | 结构化绑定、string_view、filesystem、chrono、if constexpr、属性、copy elision、invoke/apply |
| 11 | [11-cpp20-preview.md](./11-cpp20-preview.md) | Concepts、span、Ranges、`<=>`、Coroutines、jthread |

## 路线

1-3 篇是类型系统和值语义基础；4-5 篇是现代类设计；6-8 篇是编译期编程；后面是并发和语法糖。C++20 前瞻放在最后。
