# 01 — C++11 类型推导：auto、decltype、decltype(auto)

> 关联篇目：[02 Lambda 全面解析](./02-lambda-deep-dive.md) | [03 值类别与完美转发](./03-value-categories-move-forward.md) | [第一章 06 模板/SFINAE](../01-cpp-basics/06-templates-sfinae-crtp.md)

---

## 1. `auto`——让编译器帮你推断类型

C++11 重新定义了 `auto` 关键字（C 中原本是存储类说明符），用于**从初始化表达式中自动推导变量类型**。

```cpp
auto i = 42;              // int
auto d = 3.14;            // double
auto s = "hello";         // const char*
auto v = std::vector{1,2,3};  // std::vector<int>

// 复杂类型——这才是 auto 真正的价值
std::map<std::string, std::vector<int>>::const_iterator it = m.begin();  // C++98
auto it = m.begin();      // C++11 —— 简洁！
```

### 1.1 `auto` 推导规则（核心：三条规则 + 引用/const 剥离）

`auto` 的推导规则和**函数模板的类型推导完全一致**。也就是说 `auto x = expr` 等价于：

```cpp
template<typename T> void f(T x);
f(expr);  // T 推导为什么，auto 就是什么
```

**规则 1：`auto` 会剥离引用和顶层 const**

```cpp
const int ci = 42;
auto a = ci;        // a 是 int（剥离了 const）
auto& b = ci;       // b 是 const int&（auto& 不剥离 const）

int& ri = a;
auto c = ri;        // c 是 int（剥离了引用）
auto& d = ri;       // d 是 int&（auto& 保留引用）
```

**规则 2：`auto&&` 是万能引用**

```cpp
int x = 42;
auto&& r1 = x;      // x 是左值 → auto = int& → 折叠 → int&
auto&& r2 = 42;     // 42 是右值 → auto = int  → int&&
```

**规则 3：`decltype(auto)` 不剥离任何东西（C++14）**

```cpp
const int ci = 42;
decltype(auto) a = ci;   // const int —— 保留了 const
decltype(auto) b = (ci); // const int& —— 括号产生引用！
```

### 1.2 常见的 auto 陷阱

```cpp
// 陷阱 1：列表初始化
auto x1 = {1, 2, 3};     // std::initializer_list<int>（不是 vector！）
auto x2 {1};             // C++17: int（C++11/14: initializer_list<int>）
// std::vector<int> v = {1, 2, 3};  → 用 auto 得显式写 vector<int>

// 陷阱 2：auto 剥离引用
std::vector<bool> vb = {true, false, true};
auto b = vb[0];          // ⚠️ 不是 bool&！vector<bool> 的 operator[] 返回的是代理对象
b = false;               // 只修改了代理的拷贝，vb[0] 还是 true！
// 修正：
auto& b2 = vb[0];        // 保持引用语义（但 vector<bool> 的代理不是左值，可能编译失败）
bool b3 = vb[0];         // 明确类型最好

// 陷阱 3：auto 不会缩小转换
auto c = 'A' + 'B';      // int（char 运算提升为 int）
auto d = true ? 1 : 3.14;// double（?: 的公共类型）

// 陷阱 4：auto 用作返回值
auto getValue(bool flag) {
    if (flag) return 1;
    else      return 2.0;  // ⚠️ 编译错误！两个 return 推导出不同类型
}
// 修正：
auto getValue(bool flag) -> double { ... }  // 尾随返回类型
```

---

## 2. `decltype`——获取表达式的声明类型

`decltype(expr)` 给出表达式的**确切声明类型**，不做任何剥离或修改。

### 2.1 核心规则

```cpp
int x = 42;
const int& cr = x;

decltype(x)   a;   // int           — 变量名
decltype(cr)  b;   // const int&    — 保留引用和 const
decltype(x+0) c;   // int           — 纯右值表达式，得到值类型
decltype((x)) d;   // int&          — ⚠️ 括号使左值变成引用！
```

**`decltype` 的两条规则：**

| 表达式 | decltype 结果 |
|--------|--------------|
| 变量名（无括号） | 变量的声明类型 |
| 左值表达式（包括 `(x)`） | T& |
| 纯右值表达式 | T（值类型） |
| 将亡值表达式 | T&& |

### 2.2 decltype 实战场景

```cpp
// 场景 1：尾随返回类型——返回类型依赖参数类型
template<typename T, typename U>
auto add(T&& t, U&& u) -> decltype(t + u) {  // 根据 t+u 的结果类型推导返回
    return t + u;
}

// 场景 2：推导容器元素类型
std::vector<int> v;
decltype(v)::value_type x = 0;  // int —— 不实例化 v 也知道 value_type

// 场景 3：完美转发返回值
template<typename F, typename... Args>
auto invoke(F&& f, Args&&... args) -> decltype(f(args...)) {
    return f(std::forward<Args>(args)...);
}
```

---

## 3. `decltype(auto)`——C++14 的精确推导

`decltype(auto)` 结合了 `auto` 的便利和 `decltype` 的精确：**用 `decltype` 的规则推导类型，但像 `auto` 一样从初始化表达式推导**。

```cpp
int x = 42;
const int& cr = x;

auto          a1 = cr;   // int           — 剥离了 const 和引用
decltype(cr)  a2;        // const int&    — 但必须写两次 cr
decltype(auto) a3 = cr;  // const int&    — 方便 + 精确！

// 真正有价值的地方：函数返回
auto getRef1(int& x)      { return x; }        // int — 返回了拷贝！
auto& getRef2(int& x)     { return x; }        // int& — 显式写 auto&
decltype(auto) getRef3(int& x) { return x; }   // int& — 推导出引用！

auto getRef4(int& x)      { return (x); }      // int — 括号不影响 auto
decltype(auto) getRef5(int& x) { return (x); } // int& — ⚠️ 括号产生引用！
```

### 3.1 关键差异总结

```cpp
int x = 10;
const int cx = 20;
int& rx = x;

// auto: 永远是值类型（剥离引用和顶层const）
auto a1 = x;    // int
auto a2 = cx;   // int
auto a3 = rx;   // int
auto&& a4 = x;  // int& （万能引用规则）

// decltype: 保留引用和const
decltype(x)  d1; // int
decltype(cx) d2; // const int
decltype(rx) d3; // int&

// decltype(auto): 用decltype规则
decltype(auto) da1 = x;   // int
decltype(auto) da2 = cx;  // const int
decltype(auto) da3 = rx;  // int&
decltype(auto) da4 = (x); // int&  ← 注意括号！
```

---

## 4. 什么时候用哪个？

| 场景 | 推荐 | 原因 |
|------|------|------|
| 普通变量初始化 | `auto` | 想要的几乎都是值类型 |
| 需要引用 / const | `auto&` / `const auto&` | 显式意图 |
| 转发函数返回值 | `decltype(auto)` | 保持值类别 |
| 泛型代码中推导返回类型 | `decltype(expr)` 或 `decltype(auto)` | 精确控制 |
| Range-for 不修改元素 | `const auto&` | 避免拷贝 |
| Range-for 修改元素 | `auto&` | 通过引用修改 |

```cpp
// 推荐写法
for (const auto& item : container) { /* 只读 */ }    // 不拷贝
for (auto& item : container)       { /* 修改 */ }    // 通过引用修改
for (auto&& item : container)      { /* 万能 */ }    // 不在意是否可修改

// 避免
for (auto item : container)        { /* 每次拷贝！*/ }  // 大对象时很浪费
```

---

## 小结

| 概念 | 行为 |
|------|------|
| `auto` | 模板推导规则：剥离引用和顶层 const |
| `auto&` | 保留引用，保留底层 const |
| `auto&&` | 万能引用 |
| `decltype(expr)` | 精确：变量名→声明类型，`(x)`→引用，纯右值→值 |
| `decltype(auto)` | 用 decltype 规则 + auto 便利 |

下一篇 [02 Lambda 全面解析](./02-lambda-deep-dive.md)。
