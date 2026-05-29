# 06 — constexpr、consteval、constinit：编译期计算三部曲

> 关联篇目：[第一章 06 模板/SFINAE/CRTP](../01-cpp-basics/06-templates-sfinae-crtp.md) | [05 现代类特性](./05-modern-class-features.md)

---

## 1. `const` vs `constexpr`——先厘清区别

这是 C++ 最容易被混淆的两个关键字：

```cpp
const int a = 42;            // "a 的值不会改变"（运行时或编译期）
constexpr int b = 42;        // "b 的值在编译期就能确定"（强制编译期）

int x = 10;
const int c = x;             // ✅ 合法：x 是运行时值，c 是运行时不可变
// constexpr int d = x;      // ❌ x 不是编译期常量

const int size1 = 10;
constexpr int size2 = 10;
int arr1[size1];             // ✅ const 的编译期常量可以当数组大小（变长数组除外）
int arr2[size2];             // ✅ 同样
```

| | `const` | `constexpr` |
|---|---|---|
| 含义 | 不可修改 | 编译期可求值 |
| 初始化值 | 运行时或编译期都可以 | 必须是编译期常量 |
| 函数 | 成员函数承诺不修改 `*this` | 函数可以在编译期执行 |
| 变量能否当数组大小 | 如果编译期已知就行 | 必定可以 |

---

## 2. `constexpr` 变量——编译期常量

```cpp
constexpr int answer = 42;
constexpr double pi = 3.14159;
constexpr const char* msg = "hello";  // constexpr 已经是 const，但习惯上还是会叠加

// 编译期计算
constexpr int sum = 1 + 2 + 3;
constexpr int factorial5 = 5 * 4 * 3 * 2 * 1;
static_assert(factorial5 == 120);  // 编译期断言
```

---

## 3. `constexpr` 函数——编译期可执行函数

### 3.1 C++11 的严格限制（只能有 return 语句）

```cpp
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);  // 只有一个 return！
}
```

### 3.2 C++14 的放宽（可以有分支和循环）

```cpp
constexpr int factorial(int n) {
    int result = 1;
    for (int i = 2; i <= n; ++i)
        result *= i;
    return result;
}

static_assert(factorial(5) == 120);
```

### 3.3 C++17/20——几乎什么都能做

```cpp
// C++17: constexpr lambda
auto factorial = [](int n) constexpr {
    int r = 1;
    for (int i = 2; i <= n; ++i) r *= i;
    return r;
};

// C++20: constexpr 动态分配（在编译期用 new/delete！）
constexpr auto createVector() {
    std::vector<int> v = {1, 2, 3};  // C++20 起 vector 可以在编译期使用
    v.push_back(4);
    return v;
}
constexpr auto v = createVector();  // 编译期构造的 vector！
```

### 3.4 编译期求值的能力边界

```cpp
constexpr int runtime_call(int n) {
    // 如果在编译期求值，所有子表达式也必须是编译期
    static int counter = 0;         // ❌ 不能有 static 局部变量
    // try { } catch { }            // ❌ 不能有异常处理（C++20 前）
    // goto label;                  // ❌ 不能有 goto
    return n;
}

// C++20 大幅放宽：
constexpr int cxx20_ok() {
    // ✅ try-catch（如果不在编译期抛）
    // ✅ union 的活跃成员切换
    // ✅ dynamic_cast / typeid（如果多态对象是编译期的）
    return 0;
}
```

---

## 4. `consteval`（C++20）——必须在编译期执行

`constexpr` 函数既可以在编译期执行，也可以在运行期执行。`consteval` 强制**只在编译期执行**：

```cpp
consteval int compileOnly(int n) {
    return n * n;
}

int x = compileOnly(5);   // ✅ 编译期执行，x = 25
int y = 5;
// int z = compileOnly(y); // ❌ 编译错误！y 不是编译期常量
```

**`consteval` 的即时函数（immediate function）**：每次调用都产生编译期常量。适合需要保证编译期求值的场景（比如生成查找表、编译期字符串处理）。

---

## 5. `constinit`（C++20）——编译期初始化，运行时可修改

`constinit` 保证变量**在编译期初始化**，但运行期可以修改。它解决的是**静态初始化顺序问题**（Static Initialization Order Fiasco）：

```cpp
// 传统方式——初始化顺序不确定
constinit int globalCounter = 0;  // 编译期初始化为 0，保证在 main 之前完成

void increment() {
    globalCounter++;  // 运行期可以修改
}

// 与传统 const 的区别
constinit int x = 42;   // 编译期初始化，运行时可变
const int y = 42;       // 不可变
constexpr int z = 42;   // 编译期，不可变
```

`constinit` 不能用于局部变量——只能用于静态存储期（全局、静态成员、static 局部）的变量。

---

## 6. 三者的交叉对比

```cpp
// 一个 constexpr 函数
constexpr int square(int n) { return n * n; }

constexpr int a = square(5);    // 编译期常量，不可变
constinit int b = square(5);    // 编译期初始化，运行时可修改（b = 10; 合法）
const int c = square(5);        // 可能是编译期，但主要用于"不可变"的语义

consteval int cube(int n) { return n * n * n; }

constexpr int d = cube(3);      // ✅ cube 必须在编译期求值
constinit int e = cube(3);      // ✅
// int f = cube(readInput());   // ❌ readInput() 不是编译期常量，consteval 拒绝
```

| 关键字 | 编译期求值 | 运行时可修改 | 适用范围 |
|--------|:---:|:---:|------|
| `constexpr` | 可能 | ❌ | 变量 + 函数 |
| `consteval` | 必须 | ❌ | 仅函数 (C++20) |
| `constinit` | 必须 | ✅ | 静态存储期变量 (C++20) |
| `const` | 可能 | ❌ | 变量 + 成员函数 |

---

## 7. 实战场景

```cpp
// 编译期哈希——用在 switch-case / 模板参数
constexpr size_t hash(const char* s) {
    size_t h = 0;
    for (; *s; ++s) h = h * 31 + *s;
    return h;
}

switch (hash(cmd)) {
    case hash("start"): /* ... */ break;
    case hash("stop"):  /* ... */ break;
}

// 编译期查找表（AI Infra 场景）
consteval std::array<float, 256> generateLogTable() {
    std::array<float, 256> table{};
    for (int i = 0; i < 256; ++i)
        table[i] = std::log2(1.0f + i / 256.0f);
    return table;
}
constexpr auto logTable = generateLogTable();  // 编译期生成，运行期零开销查表
```

---

## 小结

| 概念 | 一句话 |
|------|--------|
| `constexpr` 变量 | 编译期常量 |
| `constexpr` 函数 | 可编译期也可运行期（C++14/17/20 边界不断扩大） |
| `consteval` | 即时函数，只在编译期执行（C++20） |
| `constinit` | 编译期初始化，运行时可修改（C++20） |
| `const` | 纯粹的"不可修改"，不强制编译期 |

下一篇 [07 using 别名 / 可变参数模板 / 折叠表达式](./07-using-variadic-fold.md)。
