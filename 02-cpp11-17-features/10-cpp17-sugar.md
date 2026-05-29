# 10 — C++17 语法糖大集合

> 关联篇目：[08 tuple/optional/variant](./08-tuple-optional-variant-any.md) | [06 constexpr](./06-constexpr-consteval.md) | [01 类型推导](./01-type-deduction.md)

---

## 1. 结构化绑定——一行拆解多个值

```cpp
// pair
std::pair<int, double> p{42, 3.14};
auto [a, b] = p;  // a = 42, b = 3.14

// tuple
auto t = std::make_tuple(1, "hello", 3.0);
auto [x, s, d] = t;

// 结构体（按成员声明顺序）
struct Point { int x, y; };
Point pt{3, 4};
auto [px, py] = pt;  // px = 3, py = 4

// 数组
int arr[] = {1, 2, 3};
auto [v1, v2, v3] = arr;

// map 遍历的神器
std::map<std::string, int> m = {{"a", 1}, {"b", 2}};
for (const auto& [key, value] : m) {
    std::cout << key << " = " << value << "\n";
}
```

---

## 2. 内联变量（C++17）——头文件中的全局变量

C++17 以前，在头文件中定义全局变量需要 `extern` 声明 + `.cpp` 定义。C++17 用 `inline` 把定义放在头文件：

```cpp
// config.h
#pragma once
inline constexpr int kMaxConnections = 100;  // 直接定义在头文件
inline const std::string kAppName = "MyApp";  // 多种编译单元只有一个实例

// 静态成员也可以 inline
class Widget {
    inline static int instanceCount = 0;  // 类内定义！
public:
    Widget() { ++instanceCount; }
};
```

---

## 3. if / switch 初始化语句

```cpp
// 在 if 条件中声明变量，变量作用域仅限于 if 块
if (auto it = map.find(key); it != map.end()) {
    std::cout << "Found: " << it->second;
} else {
    std::cout << "Not found";
}
// it 在此不可见

// 配合 mutex
if (std::lock_guard lock(mtx); !data.empty()) {
    process(data);
}  // lock 在 if 块结束析构

// switch 同样
switch (auto state = getState(); state) {
    case State::Ready:    /* ... */ break;
    case State::Running:  /* ... */ break;
}
```

---

## 4. `std::string_view`——零拷贝字符串引用

```cpp
#include <string_view>

// 不拥有字符串，只是一个"视图"（指针 + 长度）
std::string_view sv = "hello";     // 指向字面量
std::string s = "world";
std::string_view sv2 = s;          // 指向 s 的内容

// 操作类似 string，但不分配内存
sv.substr(1, 3);       // "ell" — O(1)！只调整指针，不拷贝
sv.find('l');          // O(n) 查找
sv.starts_with("he");  // C++20

// 传参时避免拷贝
void process(std::string_view name) {  // string、const char*、string_view 都能传
    std::cout << name;
}
process("literal");    // ✅ 不构造临时 string
process(s);            // ✅ 不拷贝
process(sv);           // ✅

// ⚠️ 陷阱：string_view 不拥有数据
std::string_view dangerous() {
    std::string temp = "hello";
    return temp;       // ❌ 悬空！temp 已销毁
}
```

---

## 5. `std::filesystem`——跨平台路径操作

```cpp
#include <filesystem>
namespace fs = std::filesystem;

// 路径
fs::path p = "/home/user/data/config.json";
p.filename();       // "config.json"
p.stem();           // "config"
p.extension();      // ".json"
p.parent_path();    // "/home/user/data"

// 遍历目录
for (const auto& entry : fs::directory_iterator("/home/user")) {
    std::cout << entry.path() << " — "
              << (entry.is_directory() ? "dir" : "file") << "\n";
}

// 递归遍历
for (const auto& entry : fs::recursive_directory_iterator("/home/user")) {
    // 包括所有子目录
}

// 操作
fs::create_directory("new_dir");
fs::copy_file("a.txt", "b.txt");
fs::remove("old.txt");
fs::exists("file.txt");
auto size = fs::file_size("data.bin");
auto time = fs::last_write_time("file.txt");
```

---

## 6. `std::chrono`——类型安全的时间库

```cpp
#include <chrono>
using namespace std::chrono;

// 时间单位
auto t1 = 30ms;         // milliseconds
auto t2 = 5s;           // seconds
auto t3 = 1min;         // C++20
auto t4 = 500us;        // microseconds
auto t5 = 10ns;         // nanoseconds

// 类型安全的运算
auto total = 5s + 300ms;     // 5300ms，类型是 milliseconds
// auto err = 5s + 5;        // ❌ 必须是有单位的

// duration_cast 转换
auto secs = duration_cast<seconds>(5300ms);  // 5（截断）

// 计时器
auto start = steady_clock::now();
doWork();
auto end = steady_clock::now();
auto elapsed = duration_cast<microseconds>(end - start);
std::cout << "Took " << elapsed.count() << " us\n";

// 三种时钟
system_clock;    // 系统时间（可被 NTP 调整，适合显示时间戳）
steady_clock;    // 单调递增（适合测量间隔，永不回退）
high_resolution_clock;  // 最高精度（通常是 steady_clock 别名）
```

---

## 7. `if constexpr`——编译期条件分支

```cpp
template<typename T>
auto getValue(T t) {
    if constexpr (std::is_pointer_v<T>) {
        return *t;          // 如果 T 不是指针，这段代码根本不编译
    } else if constexpr (std::is_integral_v<T>) {
        return static_cast<double>(t);
    } else {
        return t;
    }
}

getValue(42);      // 走第二个分支——其他分支的代码被丢弃
getValue(&x);      // 走第一个分支
getValue(3.14);    // 走 else
```

**与普通 `if` 的关键区别**：`if constexpr` 的分支在编译期就被选定了，未选中的分支代码不被实例化——所以可以在一个分支里写只在特定类型下合法的代码。

---

## 8. C++17 属性

```cpp
// [[nodiscard]] —— 返回值不应被丢弃
[[nodiscard]] int computeExpensive() { return 42; }
computeExpensive();  // ⚠️ 编译器警告：忽略了 nodiscard 返回值

// 用于类/枚举/函数
[[nodiscard]] class ErrorCode { };  // 任何返回 ErrorCode 的函数都会被警告

// [[maybe_unused]] —— 可能不被使用，不要警告
void f([[maybe_unused]] int param) { }

// [[fallthrough]] —— 有意不写 break
switch (x) {
    case 1:
        init();
        [[fallthrough]];  // 明确告诉编译器：这不是遗漏
    case 2:
        process();
        break;
}
```

---

## 9. C++17 保证的复制消除（Guaranteed Copy Elision）

```cpp
// C++17 起，以下场景的拷贝/移动被强制省略
Widget w = Widget(Widget(Widget(42)));
// 等价于 Widget w(42)；中间没有任何拷贝或移动构造调用

// 从函数返回临时对象
Widget create() {
    return Widget(42);  // 直接在调用方构造，不经过移动
}
Widget w = create();  // 0 次拷贝/移动——即使 Widget 的移动构造 = delete 也能编译！
```

**核心**：当用 prvalue 初始化同类型对象时，编译器**必须**在原地构造，跳过所有临时对象。这不是优化——是语言保证。

---

## 10. `std::invoke`、`std::apply`——统一调用

```cpp
#include <functional>

// invoke —— 统一调用函数、成员函数、仿函数
struct Widget {
    int value = 0;
    int getValue() const { return value; }
    void setValue(int v) { value = v; }
};

Widget w;
std::invoke(&Widget::setValue, w, 42);   // 通过对象调用成员函数
int v = std::invoke(&Widget::getValue, w); // 同上
std::invoke(&Widget::getValue, &w);       // 通过指针也可以

// apply —— unpack tuple 作为函数参数
auto t = std::make_tuple(3, 4);
int sum = std::apply([](int a, int b) { return a + b; }, t);  // 7
```

---

## 11. `std::function` 与 `std::bind`

```cpp
#include <functional>

// function —— 类型擦除的可调用包装器
std::function<int(int, int)> op;
op = [](int a, int b) { return a + b; };   // lambda
op = std::plus<int>();                      // 仿函数
op = std::fmax;                             // 函数指针

// bind —— 部分应用参数
using namespace std::placeholders;
auto plus10 = std::bind(std::plus<int>(), _1, 10);
plus10(5);  // 15

auto isGreater = std::bind(std::greater<int>(), _2, _1);  // 翻转参数
isGreater(3, 5);  // greater(5, 3) = true
```

> C++14 起 lambda 通常比 `std::bind` 更清晰——能用 lambda 就用 lambda。

---

## 小结

| 特性 | 一句话 |
|------|--------|
| 结构化绑定 | `auto [a, b] = pair/tuple/struct` |
| 内联变量 | 头文件中定义全局变量/静态成员 |
| if/switch 初始化 | `if (auto x = ...; cond)` — 限制变量作用域 |
| `string_view` | 字符串视图，不拥有数据，传参首选 |
| `filesystem` | 跨平台路径和文件操作 |
| `chrono` | 类型安全的时间单位和计时器 |
| `if constexpr` | 编译期条件分支，丢弃不匹配的代码 |
| `[[nodiscard]]` / `[[maybe_unused]]` / `[[fallthrough]]` | 编译器提示 |
| 保证复制消除 | 临时对象初始化跳过拷贝/移动 |
| `invoke` / `apply` | 统一调用接口 |

下一篇 [11 C++20 前瞻](./11-cpp20-preview.md)。
