# 07 — using 别名、可变参数模板与折叠表达式

> 关联篇目：[第一章 06 模板/SFINAE/CRTP](../01-cpp-basics/06-templates-sfinae-crtp.md) | [06 constexpr](./06-constexpr-consteval.md)

---

## 1. `using` 类型别名——替代 `typedef`

### 1.1 基本对比

```cpp
// C++98 typedef
typedef int (*FuncPtr)(double, char);
typedef std::map<std::string, std::vector<int>> StringToIntVecMap;

// C++11 using——更直观（左边是名字，右边是类型）
using FuncPtr = int (*)(double, char);
using StringToIntVecMap = std::map<std::string, std::vector<int>>;
```

两个等价，但 `using` 的可读性更好——它和变量赋值的方向一致。

### 1.2 `using` 的真正优势：别名模板（Alias Template）

`typedef` 做不到的：**给模板起别名**。

```cpp
// ✅ C++11 using——别名模板
template<typename T>
using Vec = std::vector<T>;

Vec<int> vi;          // std::vector<int>
Vec<std::string> vs;  // std::vector<std::string>

// ❌ C++98 typedef 做不到
// template<typename T>
// typedef std::vector<T> Vec;  // 编译错误！typedef 不能模板化
// 只能用丑陋的间接方式：
template<typename T>
struct VecWrapper { typedef std::vector<T> type; };
VecWrapper<int>::type vi;  // 恶心...
```

### 1.3 实战中的别名模板

```cpp
// 简化复杂模板
template<typename T>
using Ptr = std::unique_ptr<T>;

template<typename K, typename V>
using HashMap = std::unordered_map<K, V>;

// 部分应用模板参数
template<typename T>
using StringMap = std::map<std::string, T>;

StringMap<int> ages;       // std::map<std::string, int>
StringMap<double> scores;  // std::map<std::string, double>

// 型别萃取中的别名
template<typename T>
using RemoveRef = typename std::remove_reference<T>::type;
// C++14 起标准库已提供：std::remove_reference_t<T>
```

---

## 2. 可变参数模板——参数个数不确定时的解决方案

### 2.1 基本语法

```cpp
// Args 是模板参数包（template parameter pack）
// args 是函数参数包（function parameter pack）
template<typename... Args>
void print(Args... args) {
    // args... 是包展开（pack expansion）
    (std::cout << ... << args);  // C++17 折叠表达式（见下一节）
}

print(1, 3.14, "hello", 'x');
```

### 2.2 递归展开（C++11 经典方式）

```cpp
// 递归基——终止条件
void log() { std::cout << "\n"; }

// 每次取出第一个参数，剩下的递归
template<typename First, typename... Rest>
void log(First&& first, Rest&&... rest) {
    std::cout << first << " ";
    log(std::forward<Rest>(rest)...);
}

log("answer:", 42, "pi:", 3.14);
// 输出: answer: 42 pi: 3.14
```

### 2.3 `sizeof...`——参数个数

```cpp
template<typename... Args>
void count(Args... args) {
    std::cout << sizeof...(Args) << " types, "
              << sizeof...(args) << " values\n";
}

count(1, 2, 3);  // 3 types, 3 values
```

---

## 3. 折叠表达式（C++17）——递归展开的终结者

C++11/14 的可变参数模板必须用递归展开，代码冗长。C++17 的折叠表达式让参数包的操作变得简洁。

### 3.1 四种折叠形式

```cpp
// 一元右折叠：(args op ...)
template<typename... Args>
auto sum(Args... args) {
    return (args + ...);  // → arg0 + (arg1 + (arg2 + arg3))
}

// 一元左折叠：(... op args)
template<typename... Args>
auto sumLeft(Args... args) {
    return (... + args);  // → ((arg0 + arg1) + arg2) + arg3
}

// 二元右折叠：(args op ... op init)
template<typename... Args>
auto sumInit(Args... args) {
    return (args + ... + 0);  // → arg0 + (arg1 + (... + (argN + 0)))
                              // 空包时返回 init！
}

// 二元左折叠：(init op ... op args)
template<typename... Args>
auto sumInitLeft(Args... args) {
    return (0 + ... + args);  // → ((0 + arg0) + arg1) + ...
}
```

### 3.2 支持的运算符

几乎所有二元运算符都可用于折叠：

```cpp
// 打印（用逗号运算符）
template<typename... Args>
void printAll(Args&&... args) {
    ((std::cout << args << " "), ...);  // 逗号折叠
    std::cout << "\n";
}

// 逻辑 and
template<typename... Args>
bool allTrue(Args... args) {
    return (... && args);  // 短路求值！
}

// 使用自定义运算符
template<typename... Args>
auto combined(Args&&... args) {
    return (std::forward<Args>(args) + ...);
}
```

### 3.3 折叠表达式的实际应用

```cpp
// push_back 多个元素
template<typename Container, typename... Args>
void pushAll(Container& c, Args&&... args) {
    (c.push_back(std::forward<Args>(args)), ...);
}

std::vector<int> v;
pushAll(v, 1, 2, 3, 4, 5);  // v = {1, 2, 3, 4, 5}

// 多条输出
template<typename... Args>
void logAll(Args&&... args) {
    (std::cout << ... << args);  // cout << arg1 << arg2 << ...
}
```

---

## 4. 可变参数模板 + 完美转发的经典组合

```cpp
// make_unique 的本质
template<typename T, typename... Args>
std::unique_ptr<T> make_unique(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}

// emplace_back 的本质
template<typename... Args>
decltype(auto) emplace_back(Args&&... args) {
    // 在容器的尾部内存上直接构造 T，完美转发所有参数
    new (end_ptr) T(std::forward<Args>(args)...);
}
```

---

## 5. 展开模式（Expansion Patterns）——不只是简单展开

```cpp
// 模式可以包含表达式，不仅仅是参数名
template<typename... Args>
void doubleIt(Args... args) {
    auto result = (args * 2 + ...);  // 折叠表达式带模式
    // 展开为：(arg0 * 2) + (arg1 * 2) + ...
}

// 对每个参数调用函数
template<typename... Args>
void callAll(Args&&... args) {
    (process(std::forward<Args>(args)), ...);  // process(arg0), process(arg1), ...
}

// 传入多个参数到一个函数
template<typename F, typename... Args>
void forEach(F&& f, Args&&... args) {
    (f(std::forward<Args>(args)), ...);  // f(arg0), f(arg1), ...
}

forEach([](int x) { std::cout << x << " "; }, 1, 2, 3, 4);
```

---

## 小结

| 概念 | 一句话 |
|------|--------|
| `using` 别名 | 等效 `typedef` 但支持别名模板 |
| 别名模板 | `template<typename T> using Vec = std::vector<T>;` |
| 参数包 `Args...` | 表示任意数量、任意类型的参数 |
| 包展开 `args...` | 把参数包展开成逗号分隔的列表 |
| 折叠表达式 C++17 | `(args + ...)` 一行替代递归展开 |
| `sizeof...(Args)` | 编译期获取参数包大小 |

下一篇 [08 tuple / optional / variant / any](./08-tuple-optional-variant-any.md)。
