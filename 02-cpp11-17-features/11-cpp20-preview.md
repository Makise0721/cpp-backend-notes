# 11 — C++20 前瞻：Concepts、span、Ranges、Spaceship、Coroutines、jthread

> 关联篇目：[06 constexpr/consteval](./06-constexpr-consteval.md) | [第一章 06 模板/SFINAE/CRTP](../01-cpp-basics/06-templates-sfinae-crtp.md) | [08 STL 算法](../01-cpp-basics/08-stl-algorithms-iterators.md)

> C++20 是 C++11 以来最大的更新。以下选取面试和实际开发中最常涉及的六个特性。

---

## 1. Concepts（概念）——给模板加"类型约束"

### 1.1 问题：模板错误信息的天书

```cpp
template<typename T>
T max(T a, T b) { return a > b ? a : b; }

struct NoComparison {};
max(NoComparison{}, NoComparison{});
// 编译器错误：200+ 行，全在标准库深处
// 真正的问题藏在第 73 行：NoComparison 没有 operator>
```

### 1.2 Concepts 的解法

```cpp
#include <concepts>

// 定义一个 concept：要求 T 支持 < 比较
template<typename T>
concept Comparable = requires(T a, T b) {
    { a < b } -> std::convertible_to<bool>;
};

// 使用 concept 约束模板
template<Comparable T>
T max(T a, T b) { return a > b ? a : b; }

max(NoComparison{}, NoComparison{});
// 错误信息：1-2 行，清楚指出 "NoComparison doesn't satisfy Comparable"
```

### 1.3 四种写法

```cpp
// 方式 1：requires 子句（最灵活）
template<typename T>
    requires std::integral<T>
T gcd(T a, T b) { return b == 0 ? a : gcd(b, a % b); }

// 方式 2：concept 替代 typename
template<std::integral T>
T add(T a, T b) { return a + b; }

// 方式 3：简写（auto + concept）
auto add2(std::integral auto a, std::integral auto b) { return a + b; }

// 方式 4：trailing requires
template<typename T>
T add3(T a, T b) requires std::integral<T> { return a + b; }
```

### 1.4 标准库自带 Concepts

```cpp
std::integral<T>       // 整型
std::floating_point<T> // 浮点型
std::same_as<T, U>     // 同一类型
std::derived_from<T, Base>  // 派生自 Base
std::convertible_to<T, U>   // 可转换为 U
std::movable<T>        // 可移动
std::copyable<T>       // 可拷贝
std::invocable<F, Args...>  // 可调用
```

Concepts 本质上是 **SFINAE 的继任者**——更可读、错误信息更好、编译更快。

---

## 2. `std::span`——轻量级数组视图

```cpp
#include <span>

// span 不拥有数据，只是一个指针 + 长度的 view
void process(std::span<int> data) {
    for (int& x : data) x *= 2;
    std::cout << data.size() << " elements\n";
    std::cout << data[0];  // 支持下标
}

int arr[] = {1, 2, 3, 4, 5};
std::vector<int> v = {1, 2, 3};

process(arr);  // ✅ C 数组
process(v);    // ✅ vector
process(std::span(arr).subspan(1, 3));  // ✅ 子范围（零拷贝）

// 固定大小的 span
std::span<int, 5> fixed = arr;  // 编译期知道大小

// span vs string_view
std::span<const int>  nums;    // 替代 int* + size
std::string_view      str;     // 替代 const char* + size
```

---

## 3. Ranges——管道式算法

```cpp
#include <ranges>
namespace views = std::views;

std::vector<int> data = {1, 5, 3, 8, 2, 9, 4, 6};

// 传统方式：多步操作，中间结果多次拷贝
std::vector<int> temp1, temp2, result;
std::copy_if(data.begin(), data.end(), std::back_inserter(temp1),
    [](int x) { return x % 2 == 0; });
std::transform(temp1.begin(), temp1.end(), std::back_inserter(temp2),
    [](int x) { return x * x; });
std::copy(temp2.begin(), temp2.end(), std::back_inserter(result));

// 返回前三个
// ... 更繁琐的代码

// C++20 Ranges：管道式，延迟求值，零中间分配
auto result = data
    | views::filter([](int x) { return x % 2 == 0; })   // 过滤偶数
    | views::transform([](int x) { return x * x; })      // 平方
    | views::take(3)                                      // 取前 3 个
    | std::ranges::to<std::vector>();                     // C++23 收集到 vector
```

**Ranges 的核心优势**：
- **延迟求值**：在最终遍历前不做任何计算
- **组合性**：`|` 管道链接读起来像英文
- **安全**：range 知道自己的边界，比迭代器对更安全

---

## 4. 三路比较 `<=>`（Spaceship Operator）

一个运算符生成全部六种比较：

```cpp
#include <compare>

struct Point {
    int x, y;

    // C++20：一行搞定全部比较
    auto operator<=>(const Point&) const = default;
};

Point a{1, 2}, b{1, 3};
a == b;   // false（y 不同）
a != b;   // true
a < b;    // true（x 相同，y 更小）
a <= b;   // true
a > b;    // false
a >= b;   // false

// 编译器自动按成员声明顺序逐个比较（x 再 y）
// 就像手写了所有 six operators 一样
```

### 4.1 三种序

```cpp
std::strong_ordering    // 全序：每个值都可比，相等意味着可替代（int、string）
std::weak_ordering      // 弱序：相等但可能不可替代（大小写不敏感的字符串）
std::partial_ordering   // 偏序：部分值可能不可比（浮点 NaN）

// 浮点数的比较自动返回 partial_ordering
struct WithDouble {
    double val;
    auto operator<=>(const WithDouble&) const = default;  // 返回 std::partial_ordering
};
```

---

## 5. Coroutines（协程）——异步编程新范式

```cpp
#include <coroutine>
#include <iostream>

// 最简单的生成器
template<typename T>
struct Generator {
    struct promise_type {
        T current;
        Generator get_return_object() { return Generator{this}; }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        std::suspend_always yield_value(T val) { current = val; return {}; }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };

    struct iterator {
        std::coroutine_handle<promise_type> h;
        iterator& operator++() { h.resume(); return *this; }
        T operator*() const { return h.promise().current; }
        bool operator!=(const iterator&) const { return !h.done(); }
    };

    iterator begin() { h.resume(); return {h}; }
    iterator end() { return {nullptr}; }

    std::coroutine_handle<promise_type> h;
    ~Generator() { if (h) h.destroy(); }
};

// 使用协程的生成器
Generator<int> fibonacci() {
    int a = 0, b = 1;
    while (true) {
        co_yield a;          // 暂停，返回 a
        int tmp = a + b;
        a = b;
        b = tmp;
    }
}

// 遍历
for (int f : fibonacci()) {
    if (f > 100) break;
    std::cout << f << " ";  // 0 1 1 2 3 5 8 13 21 34 55 89
}
```

### 5.1 三个关键字

| 关键字 | 含义 |
|--------|------|
| `co_await` | 暂停并等待一个异步操作完成 |
| `co_yield` | 暂停并返回一个值（生成器） |
| `co_return` | 完成协程并返回值 |

协程是 C++20 最复杂的特性——标准库只提供了底层基础设施，高层封装（如 `std::generator`）在 C++23 才加入。实际项目中常用第三方库（如 `cppcoro`）。

---

## 6. `std::jthread`——自动 join 的线程

C++11 `std::thread` 的最大问题：析构时如果没 join/detach → `std::terminate`。`jthread` 自动 join：

```cpp
#include <thread>

// jthread：析构时自动 join
{
    std::jthread t([] { std::this_thread::sleep_for(1s); });
    // 不需要显式 join！
}  // t 析构，自动等待线程完成

// 支持 stop_token（协作式取消）
std::jthread worker([](std::stop_token stoken) {
    while (!stoken.stop_requested()) {
        doWork();
    }
    std::cout << "Stopped cleanly\n";
});

worker.request_stop();  // 请求停止
// worker 析构时也会自动 request_stop + join
```

---

## 附：C++20 其他值得知道的

| 特性 | 一句话 |
|------|--------|
| `std::latch` | 一次性计数器倒计时，所有线程等待 |
| `std::barrier` | 可复用的阶段同步屏障 |
| `std::semaphore` | 计数信号量 |
| `char8_t` | UTF-8 字符类型 |
| `using enum` | 把枚举值引入作用域 |
| Designated initializers | C 风格的结构体初始化：`Point{.x=1, .y=2}` |
| `contains()` | map/set 有了 `.contains()` 方法 |

---

## 第二章小结

| # | 篇目 | 核心 |
|---|------|------|
| 01 | 类型推导 | auto / decltype / decltype(auto) |
| 02 | Lambda | 捕获方式 / mutable / 底层实现 / 泛型 lambda |
| 03 | 值类别与完美转发 | lvalue/prvalue/xvalue / 万能引用 / forward / RVO |
| 04 | 智能指针 | unique_ptr / shared_ptr / weak_ptr / 删除器 / 线程安全 |
| 05 | 现代类特性 | nullptr / range-for / =default/=delete / 委托构造 / enum class |
| 06 | constexpr 三部曲 | constexpr / consteval / constinit / const vs constexpr |
| 07 | using + 可变参数 + 折叠 | 别名模板 / 参数包 / 折叠表达式 |
| 08 | tuple / optional / variant / any | 多值处理的四种工具 |
| 09 | 并发基础 | thread / mutex / atomic / call_once |
| 10 | C++17 语法糖 | 结构化绑定 / string_view / filesystem / chrono / if constexpr / 属性 |
| 11 | C++20 前瞻 | Concepts / span / Ranges / <=> / Coroutines / jthread |
