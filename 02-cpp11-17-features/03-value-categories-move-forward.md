# 03 — 值类别、移动语义、万能引用与完美转发

> 关联篇目：[01 类型推导](./01-type-deduction.md) | [第一章 02 RAII 与三五零法则](../01-cpp-basics/02-raii-rule-of-five.md) | [第一章 06 模板](../01-cpp-basics/06-templates-sfinae-crtp.md)

> 注：移动构造/移动赋值的基本用法已在 [第一章 02](../01-cpp-basics/02-raii-rule-of-five.md) 中详述，本篇聚焦值类别体系和完美转发机制。

---

## 1. C++ 值类别全景——lvalue / prvalue / xvalue

C++11 之后，表达式的值分为三种基本类别：

```
          expression
         /          \
    glvalue        rvalue
    /      \      /      \
lvalue     xvalue       prvalue
```

| 类别 | 全称 | 一句话 | 有标识（地址）？ | 可被移动？ |
|------|------|--------|:---:|:---:|
| **lvalue** | left value | 有名字、有地址、能取址 | ✅ | ❌ |
| **xvalue** | eXpiring value | 即将消亡、资源可被"偷" | ✅ | ✅ |
| **prvalue** | pure rvalue | 临时对象、字面量、无地址 | ❌ | —（本身就是临时） |
| **glvalue** | generalized lvalue | lvalue ∪ xvalue | ✅ | — |
| **rvalue** | right value | prvalue ∪ xvalue | — | — |

```cpp
int x = 42;           // x 是 lvalue
int& ref = x;         // ref 是 lvalue

42;                   // prvalue（字面量）
x + 1;                // prvalue（运算结果）
std::string("hi");    // prvalue（临时对象）

std::move(x);         // xvalue（将亡值——标记 x 可被移动）
static_cast<int&&>(x);// xvalue

std::vector<int> v;
v[0];                 // lvalue（返回引用）
std::move(v)[0];      // xvalue（移动后的 vector 的下标）
```

### 1.1 左值 → 右值的身份转换

```cpp
int x = 42;
int&& r1 = 42;        // ✅ 右值引用绑定到 prvalue
int&& r2 = x;         // ❌ 右值引用不能绑定到 lvalue
int&& r3 = std::move(x); // ✅ x 被转为 xvalue，可以绑定

// 关键：r1 虽然类型是 int&&，但 r1 本身是 lvalue！
int&& r4 = r1;        // ❌ r1 是 lvalue（有名字！），不能绑定到 int&&
int&& r5 = std::move(r1); // ✅ 需要 std::move
```

**核心洞见**：**有名字的变量永远是左值，即使它的类型是右值引用。**

---

## 2. 万能引用（Forwarding Reference / Universal Reference）

当 `T&&` 出现在**类型推导**的上下文中（`auto&&` 或函数模板的 `T&&`），它就不再是普通的右值引用，而是**万能引用**——可以绑定任何东西。

```cpp
template<typename T>
void f(T&& param);     // param 是万能引用

int x = 42;
f(x);                  // x 是左值 → T 推导为 int&  → param 类型 int&
f(42);                 // 42 是右值 → T 推导为 int   → param 类型 int&&
f(std::move(x));       // move(x) 是右值 → T 推导为 int → param 类型 int&&

// 对比：不是万能引用的右值引用
template<typename T>
void g(std::vector<T>&& param); // 就是右值引用，不是万能引用
void h(int&& param);            // 就是右值引用，不是万能引用
```

### 2.1 万能引用的条件

**必须是 `T&&` + `T` 是被推导的**。以下都不是万能引用：

```cpp
template<typename T>
void f(const T&& param);          // ❌ 加了 const → 普通右值引用

template<typename T>
void g(std::vector<T>&& param);   // ❌ 不是直接的 T&&

template<typename T>
class Widget {
    void h(T&& param);            // ❌ T 在类实例化时就确定了，不再推导
};
```

---

## 3. 引用折叠——万能引用如何工作

C++ 不允许引用的引用（`int& &`），但当模板推导产生时，会触发**引用折叠**：

| 折叠表达式 | 结果 |
|-----------|------|
| `T& &` | `T&` |
| `T& &&` | `T&` |
| `T&& &` | `T&` |
| `T&& &&` | `T&&` |

**规则：只要有一个是左值引用，结果就是左值引用。只有两个都是右值引用时，结果才是右值引用。**

在万能引用中：

```cpp
template<typename T>
void f(T&& param) { }  // param 的最终类型由引用折叠决定

int x = 42;
f(x);   // T 推导为 int&，  param 类型 = int& &&  → 折叠为 int&
f(42);  // T 推导为 int，   param 类型 = int&&     → 保持 int&&
```

---

## 4. 完美转发——`std::forward`

**问题**：万能引用函数接收参数后，参数本身（有名字）变成了左值。如果把它传给下一个函数，会丢失右值信息。

```cpp
template<typename T>
void wrapper(T&& arg) {
    // arg 是左值（有名字！）
    inner(arg);  // ⚠️ 永远调用 inner 的左值版本！即使传入的是右值
}

void inner(int& x)  { std::cout << "lvalue\n"; }
void inner(int&& x) { std::cout << "rvalue\n"; }

wrapper(42);  // 期望输出 "rvalue"，实际输出 "lvalue"！
```

**`std::forward` 修复这个问题**：

```cpp
template<typename T>
void wrapper(T&& arg) {
    inner(std::forward<T>(arg));  // 恢复 arg 的原始值类别
}
// 如果传入的是左值，T=int&， forward 返回 int&
// 如果传入的是右值，T=int，  forward 返回 int&&
```

### 4.1 `std::move` vs `std::forward`

| | `std::move` | `std::forward<T>` |
|---|---|---|
| 做什么 | 无条件转为右值 | 条件性转发：左值→左值，右值→右值 |
| 使用场景 | 你明确知道不再需要这个对象 | 泛型代码中转发参数 |
| 本质 | `static_cast<T&&>` | 条件性的 `static_cast<T&&>` |

```cpp
// 简化实现
template<typename T>
T&& forward(std::remove_reference_t<T>& arg) noexcept {
    return static_cast<T&&>(arg);  // 引用折叠决定最终类型
}
```

### 4.2 完美转发的经典模式

```cpp
// make_unique 的实现模式
template<typename T, typename... Args>
std::unique_ptr<T> make_unique(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}

// 包装器的标准写法
template<typename F, typename... Args>
decltype(auto) invokeAndLog(F&& f, Args&&... args) {
    log("calling...");
    return std::forward<F>(f)(std::forward<Args>(args)...);
}
```

---

## 5. RVO / NRVO——编译器优化掉你的 move

返回值优化（RVO）让编译器直接在调用方的栈帧上构造返回对象，跳过拷贝/移动。

```cpp
// NRVO（Named Return Value Optimization）
std::vector<int> buildVector() {
    std::vector<int> v;          // 编译器可能在调用方栈上直接构造 v
    v.push_back(1);
    v.push_back(2);
    return v;                    // ✅ 编译器优化：不拷贝也不移动
}

// RVO（未命名临时对象返回）
std::vector<int> buildVector2() {
    return std::vector<int>{1, 2, 3};  // ✅ RVO 保证（C++17 强制）
}
```

### 5.1 不要给返回值加 `std::move`

```cpp
// ❌ 坏习惯——限制了 RVO
std::vector<int> buildBad() {
    std::vector<int> v;
    return std::move(v);  // 阻止了 NRVO！强制移动
}

// ✅ 好习惯——让编译器优化
std::vector<int> buildGood() {
    std::vector<int> v;
    return v;  // NRVO 生效，零开销
}
```

**规则**：`return` 局部变量时，不要 `std::move`——编译器比你聪明。C++17 起对纯右值甚至强制省略拷贝。

### 5.2 RVO 不适用的情况

```cpp
// RVO 不适用：return 的不是局部变量
std::vector<int> buildFromParam(std::vector<int> v) {
    return v;           // 可能移动（编译器不能直接在调用方构造 v，因为它是参数）
}

// 条件返回——RVO 无法应用
std::vector<int> buildConditional(bool flag) {
    std::vector<int> a, b;
    if (flag) return a;  // 编译器不知道该在调用方构造 a 还是 b
    else      return b;  // → 只能用移动
}
```

---

## 6. 值类别速查——你在写什么？

```cpp
std::string s = "hello";

s;                    // lvalue（有名字的变量）
s + " world";         // prvalue（临时结果）
s[0];                 // lvalue（返回 char&）
std::move(s);         // xvalue（标记可移动）
"hello";              // lvalue（字符串字面量是 const char[6]——有地址！）
std::string("hi");    // prvalue（临时对象）

std::string&& r = std::move(s);
r;                    // lvalue！（有名字的右值引用……是左值）
std::move(r);         // xvalue

void f(std::string&& param);
// param 在 f 内部是 lvalue
```

---

## 小结

| 概念 | 一句话 |
|------|--------|
| 值类别 | lvalue（有地址）→ prvalue（临时）→ xvalue（将亡=可移动） |
| 万能引用 `T&&` | 类型推导语境下的 `T&&`，左值→左值引用，右值→右值引用 |
| 引用折叠 | `&` 取胜；只有 `&& + &&` = `&&`，其他全 `&` |
| `std::forward` | 恢复参数的原始值类别，泛型代码的标准转发方式 |
| `std::move` | 无条件转右值，标记"不再使用" |
| RVO/NRVO | 编译器直接在调用方构造，比移动更快——`return` 时别加 `std::move` |

下一篇 [04 智能指针全面解析](./04-smart-pointers.md)。
