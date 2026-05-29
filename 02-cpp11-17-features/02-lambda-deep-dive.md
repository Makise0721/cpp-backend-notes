# 02 — Lambda 表达式全面解析

> 关联篇目：[01 类型推导](./01-type-deduction.md) | [03 值类别与完美转发](./03-value-categories-move-forward.md) | [第一章 08 STL 算法与迭代器](../01-cpp-basics/08-stl-algorithms-iterators.md)

---

## 1. Lambda 语法——从零到一

```cpp
// 完整语法
[capture](parameters) mutable noexcept -> return_type { body }

// 最简形式
[]{};  // 合法！无捕获、无参数、无返回的空 lambda
```

每个部分：

| 部分 | 必需？ | 说明 |
|------|--------|------|
| `[capture]` | ✅ | 捕获列表——决定 lambda 能访问哪些外部变量 |
| `(params)` | 无参可省 | 参数列表 |
| `mutable` | 可选 | 允许修改按值捕获的变量（默认是 const） |
| `noexcept` | 可选 | 标记不抛异常 |
| `-> type` | 可推导时可省 | 尾随返回类型 |
| `{ body }` | ✅ | 函数体 |

---

## 2. 捕获方式——lambda 的核心能力

### 2.1 六种捕获语法

```cpp
int a = 1, b = 2, c = 3;

// 1. [] — 不捕获任何外部变量
auto f1 = [] { return 42; };  // 只能用函数体内定义的变量

// 2. [=] — 按值捕获所有（隐式）
auto f2 = [=] { return a + b + c; };  // a, b, c 都被拷贝进 lambda

// 3. [&] — 按引用捕获所有（隐式）
auto f3 = [&] { a = 10; b = 20; };    // a, b, c 通过引用访问

// 4. [a, &b] — a 按值，b 按引用（显式混合）
auto f4 = [a, &b] { return a + b; };  // a 被拷贝，b 是引用

// 5. [=, &c] — 默认按值，但 c 按引用
auto f5 = [=, &c] { return a + b + c; };

// 6. [&, a] — 默认按引用，但 a 按值
auto f6 = [&, a] { b = c = 0; return a; };
```

### 2.2 捕获 `this`（C++11 / C++17 / C++20）

```cpp
class Widget {
    int value_ = 42;
public:
    auto getLambda() {
        // C++11: 捕获 this 指针（[=] 或 [this]），通过 this 访问成员
        return [this] { return value_; };   // 实际是 this->value_

        // ⚠️ 危险：如果 Widget 被析构，lambda 中的 this 变成悬空指针
    }

    auto getSafeLambda() {
        // C++17: 按值捕获 *this——复制整个对象
        return [*this] { return value_; };  // lambda 持有 Widget 的拷贝
    }
};
```

### 2.3 初始化捕获（C++14）——把移动语义带进 lambda

C++11 无法将 move-only 对象"移动"进 lambda。C++14 解决了：

```cpp
auto ptr = std::make_unique<int>(42);

// ❌ C++11：unique_ptr 不可拷贝，[=] 无法编译
// auto f = [ptr] { return *ptr; };  // 编译错误！

// ✅ C++14 初始化捕获：
auto f = [p = std::move(ptr)] { return *p; };  // p 从 ptr 移动构造

// 等价的泛型写法（C++14 泛型 lambda）
auto f2 = [p = std::make_unique<int>(42)] { return *p; };

// 初始化捕获可以改名
std::string name = "Alice";
auto greet = [n = std::move(name)] { return "Hello, " + n; };
// name 现在是空字符串，n 持有 "Alice"
```

---

## 3. `mutable`——修改按值捕获的变量

默认情况下，lambda 的 `operator()` 是 **const** 的——不能修改按值捕获的变量：

```cpp
int counter = 0;

// ❌ 默认 const
auto f1 = [counter] { return counter++; };  // 编译错误！

// ✅ mutable 去掉了 const
auto f2 = [counter]() mutable { return counter++; };
f2();  // 返回 0（counter 变为 1）
f2();  // 返回 1（counter 变为 2）
std::cout << counter;  // 还是 0！外部 counter 没变

// 对比：引用捕获可以直接修改
auto f3 = [&counter] { return counter++; };
f3();  // 外部 counter 也变成了 1
```

**`mutable` 只影响按值捕获的副本，不影响外部变量。**

---

## 4. Lambda 底层实现——编译器生成的匿名仿函数类

Lambda 不是魔法——编译器为每个 lambda 生成一个匿名的仿函数类（functor class）。

```cpp
// 你写的：
int threshold = 5;
auto lambda = [threshold](int x) { return x > threshold; };

// 编译器生成（简化）：
class __lambda_12345 {
    int threshold_;          // 捕获的变量成为成员
public:
    explicit __lambda_12345(int t) : threshold_(t) {}

    auto operator()(int x) const { return x > threshold_; }
    //                ↑ const：默认不可修改按值捕获的成员
    // 如果加了 mutable：没有 const
};

auto lambda = __lambda_12345(threshold);
```

### 4.1 不同捕获方式的成员类型

```cpp
int x = 1;
// [x]    → 编译器成员：int x_;
// [&x]   → 编译器成员：int& x_;
// [p = std::move(ptr)] → 编译器成员：std::unique_ptr<int> p_;（C++14）

// [=]    → 所有用到的变量按值成为成员
// [&]    → 所有用到的变量按引用成为成员
```

---

## 5. 无状态 Lambda → 函数指针的隐式转换

**无捕获（stateless）的 lambda 可以隐式转换为函数指针**：

```cpp
// 无捕获 lambda → 函数指针
int (*fp)(int, int) = [](int a, int b) { return a + b; };
fp(3, 4);  // 7

// 实用场景：作为 C API 的回调
void registerCallback(void (*cb)(int));
registerCallback([](int val) { std::cout << val; });  // ✅ 隐式转换

// ⚠️ 只要有任何捕获，就不能转换
int n = 10;
// int (*fp2)(int) = [n](int x) { return x + n; };  // ❌ 编译错误！
// 有捕获的 lambda 必须用 std::function
std::function<int(int)> f2 = [n](int x) { return x + n; };  // ✅
```

**原理**：无状态 lambda 的仿函数类没有成员变量，`operator()` 不依赖 `this`，所以编译器可以额外生成一个静态函数版本，该函数指针就是它的地址。

---

## 6. 泛型 Lambda（C++14）——用 `auto` 做参数

```cpp
// C++14 泛型 lambda：参数类型用 auto
auto generic = [](auto x, auto y) { return x + y; };

generic(1, 2);            // int + int → int
generic(1.5, 2.3);        // double + double → double
generic(std::string("Hello "), std::string("World")); // string + string

// 等价于：
struct __generic {
    template<typename T, typename U>
    auto operator()(T x, U y) const { return x + y; }
};

// 用 decltype 推导返回类型
auto generic2 = [](auto x, auto y) -> decltype(x + y) {
    return x + y;
};
```

---

## 7. 实战：Lambda 在现代 C++ 中的典型用法

### 7.1 自定义排序

```cpp
std::vector<Person> people = {...};

// 按年龄降序
std::sort(people.begin(), people.end(),
    [](const Person& a, const Person& b) { return a.age > b.age; });
```

### 7.2 RAII 风格的资源管理（Scope Guard）

```cpp
template<typename F>
class ScopeGuard {
    F f_;
public:
    explicit ScopeGuard(F f) : f_(std::move(f)) {}
    ~ScopeGuard() { f_(); }
};

// 用法
void processFile(const char* path) {
    FILE* f = fopen(path, "r");
    auto guard = ScopeGuard([f] { fclose(f); });
    // 无论怎样退出，guard 析构时都会 fclose
}
```

### 7.3 延迟计算

```cpp
std::map<std::string, std::function<int()>> callbacks;
callbacks["add"] = [a=1, b=2] { return a + b; };
callbacks["mul"] = [a=3, b=4] { return a * b; };

std::cout << callbacks["add"]();  // 3（此时才计算）
```

---

## 小结

| 概念 | 关键点 |
|------|--------|
| `[=]` vs `[&]` | 值拷贝 vs 引用，注意生命周期 |
| `mutable` | 允许修改按值捕获的副本（去掉 const） |
| 初始化捕获 C++14 | `[p = std::move(ptr)]` — 移动语义进 lambda |
| `[*this]` C++17 | 安全拷贝整个对象进 lambda |
| 底层实现 | 编译器生成匿名仿函数类，捕获变量 → 成员 |
| 无状态 → 函数指针 | 无捕获的 lambda 可以隐式转换为函数指针 |
| 泛型 lambda C++14 | `[](auto x, auto y) { ... }` — 参数类型模板化 |

下一篇 [03 值类别与完美转发](./03-value-categories-move-forward.md)。
