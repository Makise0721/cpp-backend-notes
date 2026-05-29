# 08 — tuple、pair、optional、variant、any：多值处理的瑞士军刀

> 关联篇目：[07 可变参数模板](./07-using-variadic-fold.md) | [04 智能指针](./04-smart-pointers.md) | [10 C++17 语法糖](./10-cpp17-sugar.md)

---

## 1. `std::pair`——两个值的简单打包

```cpp
#include <utility>

auto p = std::make_pair(42, "hello");  // pair<int, const char*>
std::cout << p.first << " " << p.second;

// C++17 类模板参数推导（CTAD）
std::pair p2(3.14, 'x');  // pair<double, char>

// 结构化绑定
auto [value, name] = p2;
// value = 3.14, name = 'x'
```

### 1.1 piecewise_construct——原地构造

```cpp
// 普通构造：需要先创建临时对象
std::pair<std::string, std::vector<int>> p(
    "key", std::vector<int>{1, 2, 3});

// piecewise 构造：直接在 pair 内构造，零拷贝
std::pair<std::string, std::vector<int>> p2(
    std::piecewise_construct,
    std::forward_as_tuple(10, 'x'),         // string(10, 'x')
    std::forward_as_tuple({1, 2, 3}));      // vector{1, 2, 3}
```

---

## 2. `std::tuple`——任意数量、任意类型的组合

### 2.1 基本用法

```cpp
#include <tuple>

// 创建
auto t = std::make_tuple(42, 3.14, std::string("hello"), 'x');
std::tuple t2{1, "two", 3.0};  // C++17 CTAD

// 访问（索引必须编译期常量）
int i = std::get<0>(t);          // 42
double d = std::get<1>(t);       // 3.14
auto& s = std::get<2>(t);        // 引用

// 按类型访问（如果类型唯一）
auto str = std::get<std::string>(t);  // "hello"

// 大小
std::cout << std::tuple_size<decltype(t)>::value;  // 4
```

### 2.2 实用操作

```cpp
// tie——解包到已有变量
int a; double b; std::string c; char d;
std::tie(a, b, c, d) = t;  // a=42, b=3.14, c="hello", d='x'

// tie + ignore——忽略某些元素
std::tie(a, std::ignore, c, std::ignore) = t;  // 只拿第 0 和第 2 个

// tuple_cat——拼接
auto t1 = std::make_tuple(1, 2);
auto t2 = std::make_tuple(3.0, 4.0);
auto t3 = std::tuple_cat(t1, t2);  // (1, 2, 3.0, 4.0)

// 结构化绑定
auto [x, y, str, ch] = t;  // C++17
```

---

## 3. `std::optional`——可能不存在的值

替代哨兵值（`-1`、空字符串）或指针（`nullptr`）来表达"值可能不存在"。

### 3.1 基本用法

```cpp
#include <optional>

std::optional<int> findUser(const std::string& name) {
    if (auto it = db.find(name); it != db.end())
        return it->age;           // 隐式构造 optional<int>
    return std::nullopt;          // 显式表示"无值"
}

auto result = findUser("Alice");

// 检查
if (result.has_value()) { /* 有值 */ }
if (result) { /* 有值——隐式转 bool */ }

// 取值
int age = result.value();         // 无值时会抛 std::bad_optional_access
int age2 = *result;               // 无值时未定义行为（类似解引用空指针）！
int age3 = result.value_or(-1);   // 安全：无值时返回 -1

// 修改
result = 30;                      // 设值
result.reset();                   // 清除
result.emplace(25);               // 原地构造（已有时先销毁旧的）
```

### 3.2 与指针的对比

```cpp
// ❌ 用指针表示"可能没值"——谁负责 delete？能是 nullptr 吗？
int* findUserOld(const std::string& name);

// ✅ optional——所有权清楚，语义明确
std::optional<int> findUserNew(const std::string& name);

// 链式操作
std::optional<Person> p = findPerson("Alice");
std::optional<std::string> city = p.and_then(
    [](const Person& p) { return p.city(); });
// 如果 p 无值或 city() 返回 nullopt，city 就是 nullopt
```

---

## 4. `std::variant`——类型安全的联合体

### 4.1 基本用法

```cpp
#include <variant>

std::variant<int, double, std::string> v;

v = 42;                       // 持有 int
v = 3.14;                     // 持有 double
v = std::string("hello");     // 持有 string

// 访问（必须匹配实际类型）
std::cout << std::get<int>(v);     // ❌ 抛 std::bad_variant_access（当前持有 string）
std::cout << std::get<std::string>(v);  // ✅ "hello"

// 检查当前持有的类型
if (std::holds_alternative<int>(v)) { }
std::cout << v.index();  // 当前活跃类型的索引：string 在 variant<int,double,string> 中的 index=2
```

### 4.2 `std::visit`——多类型统一处理

```cpp
struct Printer {
    void operator()(int x)          const { std::cout << "int: " << x; }
    void operator()(double x)       const { std::cout << "double: " << x; }
    void operator()(const std::string& x) const { std::cout << "string: " << x; }
};

std::visit(Printer{}, v);  // 根据 v 的实际类型调用对应的 operator()

// 用泛型 lambda（C++17）
std::visit([](auto&& arg) {
    using T = std::decay_t<decltype(arg)>;
    if constexpr (std::is_same_v<T, int>)
        std::cout << "int: " << arg;
    else if constexpr (std::is_same_v<T, double>)
        std::cout << "double: " << arg;
    else
        std::cout << "string: " << arg;
}, v);
```

### 4.3 Variant vs Union vs 虚函数

| 方案 | 类型安全 | 堆分配 | 大小 |
|------|:---:|:---:|------|
| `std::variant` | ✅ | 无 | 最大类型 + 索引 |
| `union` | ❌（需手动判断） | 无 | 最大类型 |
| 虚函数 + 派生类 | ✅ | 通常有 | vptr + 各类型自担 |

---

## 5. `std::any`——任意类型的容器

可以存任何可拷贝的类型，类似 `void*` 但类型安全。

```cpp
#include <any>

std::any a = 42;
a = 3.14;
a = std::string("hello");
a = std::vector<int>{1, 2, 3};

// 取回值——必须知道确切类型
if (a.type() == typeid(std::vector<int>)) {
    auto& v = std::any_cast<std::vector<int>&>(a);
    std::cout << v.size();
}

// 错误的类型会抛异常
try {
    std::any_cast<int>(a);  // a 当前是 vector<int>
} catch (const std::bad_any_cast& e) {
    std::cerr << "bad cast\n";
}
```

### 5.1 any 的实现原理（简化）

```cpp
// 类型擦除：
// any 内部持有一个基类指针，指向模板化的派生类对象
class AnyBase { virtual ~AnyBase() {} };
template<typename T>
class AnyHolder : public AnyBase { T value_; };
// any_cast 通过 typeid 检查后，static_cast 回具体类型
```

### 5.2 什么时候用哪个？

| 场景 | 工具 |
|------|------|
| 两个值 | `std::pair` |
| 多个值、异质 | `std::tuple` |
| 值可能不存在 | `std::optional` |
| 值是若干已知类型之一 | `std::variant` |
| 值可以是任何类型 | `std::any`（最后的选择） |

---

## 小结

| 类型 | 用途 | 关键操作 |
|------|------|----------|
| `pair` | 两个值 | `.first` / `.second` |
| `tuple` | N 个值 | `get<N>()` / `tie` / 结构化绑定 |
| `optional` | 可能无值 | `.value()` / `.value_or()` / `*` |
| `variant` | 多选一 | `get<T>()` / `visit` / `holds_alternative` |
| `any` | 任意类型 | `any_cast<T>()` / `.type()` |

下一篇 [09 并发基础：atomic / thread / mutex](./09-concurrency-basics.md)。
