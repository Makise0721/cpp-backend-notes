# 05 — 现代类特性：nullptr、range-for、=default/=delete、委托/继承构造、enum class

> 关联篇目：[第一章 01 OOP 基础](../01-cpp-basics/01-oop-basics.md) | [第一章 02 RAII 与三五零法则](../01-cpp-basics/02-raii-rule-of-five.md)

---

## 1. `nullptr`——空指针的正确表达

C++11 以前，`NULL` 是宏（通常是 `0` 或 `(void*)0`），导致重载决议中的歧义：

```cpp
void f(int x)    { std::cout << "int\n"; }
void f(void* p)  { std::cout << "pointer\n"; }

f(0);       // "int"
f(NULL);    // "int" or "pointer" — 取决于 NULL 的定义！不可移植
f(nullptr); // "pointer" — ✅ 明确、可移植
```

### 1.1 `nullptr` 的本质

`nullptr` 的类型是 `std::nullptr_t`，它可以隐式转换为**任意指针类型**和**任意成员指针类型**，但不能转换为整型：

```cpp
int*  p1 = nullptr;      // ✅
char* p2 = nullptr;      // ✅
void (MyClass::*p3)() = nullptr;  // ✅ 成员函数指针
int   n  = nullptr;      // ❌ 编译错误！不能转换为 int

std::nullptr_t np = nullptr;  // nullptr_t 是单独的类型
```

### 1.2 最佳实践

```cpp
// ✅ 好
int* p = nullptr;
if (p == nullptr) { }
if (!p) { }              // 也可以，但不如显式 nullptr 清晰

// ❌ 避免
int* p = NULL;           // C++11 起不推荐
int* p = 0;              // 语义不清：是数字还是空指针？
if (p == 0) { }          // 同上
```

---

## 2. Range-for 循环——告别索引

```cpp
std::vector<int> v = {1, 2, 3, 4, 5};

// C++98 风格
for (size_t i = 0; i < v.size(); ++i) { std::cout << v[i]; }
for (auto it = v.begin(); it != v.end(); ++it) { std::cout << *it; }

// C++11 range-for
for (int x : v)           { std::cout << x; }  // 按值——拷贝每个元素
for (int& x : v)          { x *= 2; }           // 按引用——修改元素
for (const int& x : v)    { std::cout << x; }   // 按 const 引用——不拷贝，不修改

// auto 版本（推荐）
for (const auto& x : v)   { /* 只读遍历的首选 */ }
for (auto& x : v)         { /* 需要修改时 */ }
for (auto&& x : v)        { /* 万能引用，不关心是否可修改 */ }
```

### 2.1 Range-for 的工作原理

```cpp
// for (auto& x : container) 展开为：
auto&& __range = container;
for (auto __begin = begin(__range), __end = end(__range);
     __begin != __end; ++__begin) {
    auto& x = *__begin;
    // 循环体
}
```

这要求容器有 `begin()` / `end()` 成员函数，或 ADL 能找到 `begin()` / `end()` 自由函数。

### 2.2 常见陷阱

```cpp
// 陷阱 1：修改临时容器
for (auto x : getVector()) { }  // ✅ getVector() 返回的临时对象生命周期会延长

// 陷阱 2：在循环中修改容器
for (auto& x : v) {
    v.push_back(x);  // ⚠️ 迭代器可能失效！vector 扩容 → 未定义行为
}

// 陷阱 3：遍历时删除元素
// range-for 不适合在遍历时删除——用迭代器循环 + erase
```

---

## 3. 类内初始值——声明处直接赋值

C++11 允许在成员变量声明处给默认值，不再必须依赖构造函数的初始化列表：

```cpp
class Widget {
    int count_ = 0;                          // 类内初始值
    std::string name_ = "unnamed";
    std::vector<int> data_ = {1, 2, 3};
    const int id_ = nextId();               // 也可以调用函数

public:
    Widget() {}                              // count_ = 0, name_ = "unnamed"
    Widget(int c) : count_(c) {}             // count_ = c（覆盖默认值）
    Widget(std::string n) : name_(n) {}      // name_ = n
};
```

**构造函数初始化列表优先于类内初始值。** 如果构造函数的初始化列表中没有该成员，才用类内初始值。

---

## 4. `= default` / `= delete`——显式控制特殊成员函数

### 4.1 `= default`

告诉编译器"用默认生成的版本"，即使你定义了其他构造函数：

```cpp
class NonCopyable {
public:
    NonCopyable() = default;                       // 显式要求编译器生成默认构造
    NonCopyable(const NonCopyable&) = delete;      // 禁止拷贝
    NonCopyable& operator=(const NonCopyable&) = delete;
    NonCopyable(NonCopyable&&) = default;          // 显式要求编译器生成移动构造
    NonCopyable& operator=(NonCopyable&&) = default;
    ~NonCopyable() = default;
};
```

### 4.2 `= delete`

**禁止**某个函数被使用——不仅仅是特殊成员函数，任何函数都可以 `= delete`：

```cpp
// 禁止特定类型的参数
void process(int x) { }
void process(double) = delete;    // 禁止 double，防止隐式转换

process(42);    // ✅ int
process(3.14);  // ❌ 编译错误！double 被删除

// 禁止特定模板实例化
template<typename T>
void handle(T* ptr) { }

template<>
void handle<void>(void*) = delete;  // 禁止 void*
```

---

## 5. 委托构造函数——一个构造调用另一个

C++11 以前，多个构造函数中重复的初始化代码只能抽成 `init()` 函数。C++11 允许一个构造函数调用另一个：

```cpp
class Widget {
    int a_, b_, c_;
public:
    Widget() : Widget(0, 0, 0) {}                 // 委托给三参数版本
    Widget(int a) : Widget(a, 0, 0) {}             // 委托给三参数版本
    Widget(int a, int b) : Widget(a, b, 0) {}      // 委托给三参数版本
    Widget(int a, int b, int c) : a_(a), b_(b), c_(c) {
        // 真正的初始化逻辑只写一次
    }
};
```

**注意**：委托构造时，初始化列表中只能有被委托的构造函数调用，不能再有其他成员初始化。

---

## 6. 继承构造函数——`using Base::Base`

C++11 以前，派生类要"透传"基类的构造函数，必须一个一个手写。C++11 用 `using` 声明直接继承：

```cpp
class Base {
public:
    Base(int x) {}
    Base(int x, double y) {}
    Base(const std::string& s) {}
};

class Derived : public Base {
    using Base::Base;  // 继承所有 Base 的构造函数
    // 编译器生成：
    // Derived(int x) : Base(x) {}
    // Derived(int x, double y) : Base(x, y) {}
    // Derived(const std::string& s) : Base(s) {}
};

Derived d1(42);
Derived d2(1, 3.14);
Derived d3("hello");
```

---

## 7. `enum class`——强类型枚举

C++98 的 `enum` 有两个问题：值泄漏到外层作用域，且能隐式转换为 `int`。

```cpp
// C++98 的旧枚举
enum Color { Red, Green, Blue };
enum Mood  { Happy, Sad, Blue };  // ❌ Blue 重定义！

Color c = Red;
int n = Red;  // ✅ Red 隐式转换为 0——但通常不是你想要的行为

// C++11 枚举类
enum class Color { Red, Green, Blue };
enum class Mood  { Happy, Sad, Blue };  // ✅ 各自的 Blue 互不冲突

Color c = Color::Red;     // 必须加作用域限定
// int n = Color::Red;    // ❌ 不能隐式转换为 int
int n = static_cast<int>(Color::Red);  // ✅ 显式转换

// 指定底层类型
enum class Permissions : uint8_t {
    Read = 1, Write = 2, Execute = 4
};  // 只占 1 字节

// 位运算需要重载
Permissions operator|(Permissions a, Permissions b) {
    return static_cast<Permissions>(
        static_cast<uint8_t>(a) | static_cast<uint8_t>(b));
}

auto rw = Permissions::Read | Permissions::Write;  // Read | Write
```

| 特性 | `enum` (C++98) | `enum class` (C++11) |
|------|---------------|---------------------|
| 作用域 | 泄漏到外层 | 枚举名限定 |
| 隐式转 int | ✅ | ❌（需 `static_cast`） |
| 可以指定底层类型 | C++11 起支持 | ✅ |
| 前置声明 | ❌（C++11 前） | ✅ |

---

## 小结

| 特性 | 一句话 |
|------|--------|
| `nullptr` | 类型安全的空指针，不隐式转 int |
| Range-for | `for (const auto& x : container)` 只读遍历首选 |
| 类内初始值 | 成员声明处给默认值，构造函数可覆盖 |
| `= default` | 要求编译器生成默认版本 |
| `= delete` | 禁止任何函数，不仅限于特殊成员 |
| 委托构造 | 一个构造调用另一个，避免重复代码 |
| 继承构造 | `using Base::Base` 透传基类构造 |
| `enum class` | 强类型、作用域限定、不隐式转 int |

下一篇 [06 constexpr / consteval / constinit](./06-constexpr-consteval.md)。
