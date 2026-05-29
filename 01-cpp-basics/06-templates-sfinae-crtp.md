# 06 — 模板、SFINAE 与 CRTP：编译期多态的三种武器

> 关联篇目：[03 运算符重载](./03-static-operator-overload.md) | [04 虚函数与 vtable](./04-virtual-vtable.md) | [08 STL 算法与迭代器](./08-stl-algorithms-iterators.md)

---

## 1. 函数模板——用类型参数化算法

模板是 C++ 的"代码生成器"——你写一份蓝图，编译器为每个具体类型生成一份代码。

```cpp
// 一个模板，替代无数个重载
template<typename T>
T max(T a, T b) {
    return a > b ? a : b;
}

int    x = max(1, 2);           // 编译器生成 max<int>
double y = max(3.14, 2.71);    // 编译器生成 max<double>
std::string s = max(std::string("a"), std::string("b")); // max<string>
```

### 1.1 模板实参推导 vs 显式指定

```cpp
max(1, 2);          // 推导：T = int
max(1, 2.0);        // ❌ 编译错误：T 推导为 int 还是 double？歧义！
max<double>(1, 2.0);// ✅ 显式指定 T = double，1 隐式转换为 1.0
max<int>(1, 2.0);   // ✅ 显式指定 T = int，2.0 截断为 2
```

### 1.2 模板的两阶段编译

- **阶段 1（定义时）**：检查不依赖模板参数的语法（缺少分号、未声明变量等）
- **阶段 2（实例化时）**：检查依赖模板参数的部分（`a > b` 是否合法、类型是否有 `>` 运算符）

这就是为什么模板编译错误又长又难读——错误是在实例化时才暴露的，编译器看到的已经是展开后的代码。

### 1.3 非类型模板参数

模板参数不一定是类型，也可以是值：

```cpp
template<typename T, size_t N>
class Array {
    T data_[N];
public:
    constexpr size_t size() const { return N; }
    T& operator[](size_t i) { return data_[i]; }
};

Array<int, 10> arr;    // 栈上分配，大小编译期确定
// 对比 std::array<int, 10>
```

---

## 2. 类模板——容器的基石

```cpp
template<typename T>
class Stack {
    std::vector<T> elems_;
public:
    void push(const T& val) { elems_.push_back(val); }
    void push(T&& val)      { elems_.push_back(std::move(val)); }
    T pop() {
        T top = std::move(elems_.back());
        elems_.pop_back();
        return top;
    }
    bool empty() const { return elems_.empty(); }
};

Stack<int> intStack;
Stack<std::string> strStack;
// Stack 和 Stack<std::string> 是**完全不同的类型**
```

### 2.1 成员函数的分离定义

```cpp
template<typename T>
class Stack {
    std::vector<T> elems_;
public:
    void push(const T& val);
};

template<typename T>                         // ← 必须重复 template 声明
void Stack<T>::push(const T& val) {          // ← Stack<T> 不是 Stack
    elems_.push_back(val);
}
```

由于模板在头文件中实例化，**分离编译（.h + .cpp）通常不可行**——要么把实现写在头文件，要么在 .cpp 中显式实例化所有需要的类型。

---

## 3. 模板特化——为特定类型定制行为

### 3.1 全特化（Full Specialization）

```cpp
// 通用版本
template<typename T>
struct TypeName {
    static const char* get() { return "unknown"; }
};

// 全特化：完全指定了所有模板参数
template<>
struct TypeName<int> {
    static const char* get() { return "int"; }
};

template<>
struct TypeName<double> {
    static const char* get() { return "double"; }
};

template<>
struct TypeName<std::string> {
    static const char* get() { return "string"; }
};

std::cout << TypeName<int>::get();     // "int"
std::cout << TypeName<float>::get();   // "unknown"（走通用版本）
```

### 3.2 偏特化（Partial Specialization）

只特化一部分模板参数，或者对参数施加约束：

```cpp
// 通用版本
template<typename T, typename U>
struct IsSame {
    static constexpr bool value = false;
};

// 偏特化：两个类型相同时
template<typename T>
struct IsSame<T, T> {          // ← 偏特化：第二个参数 = 第一个
    static constexpr bool value = true;
};

IsSame<int, double>::value;    // false（通用版本）
IsSame<int, int>::value;       // true（偏特化版本）

// 偏特化指针类型
template<typename T>
struct RemovePointer {
    using type = T;
};
template<typename T>
struct RemovePointer<T*> {     // ← 匹配指针类型
    using type = T;
};
template<typename T>
struct RemovePointer<T* const> {
    using type = T;
};

RemovePointer<int**>::type;    // int*（一层层剥）
```

### 3.3 偏特化 vs 函数重载

函数模板**没有偏特化**——用重载实现类似效果：

```cpp
// 通用版本
template<typename T>
void process(T val) { std::cout << "generic\n"; }

// 用重载"模拟"偏特化
template<typename T>
void process(T* val) { std::cout << "pointer\n"; }  // 更特化的重载

process(42);       // "generic"
process(&x);       // "pointer"（更匹配指针版本）
```

---

## 4. 可变参数模板（Variadic Templates，C++11）

解决"参数个数不确定"的问题——替代 C 的 `printf` 风格。

### 4.1 递归展开

```cpp
// 递归终止条件
void printAll() { std::cout << "\n"; }

// 递归：处理一个，剩下的交给下一个自己
template<typename First, typename... Rest>
void printAll(First&& first, Rest&&... rest) {
    std::cout << first << " ";
    printAll(std::forward<Rest>(rest)...);  // 递归展开
}

printAll(1, 3.14, "hello", 'x');
// 输出：1 3.14 hello x
// 递归过程：
//   printAll(1, 3.14, "hello", 'x')
//   → printAll(3.14, "hello", 'x')
//   → printAll("hello", 'x')
//   → printAll('x')
//   → printAll()
```

### 4.2 折叠表达式（C++17，更简洁）

```cpp
template<typename... Args>
auto sum(Args... args) {
    return (args + ...);  // 折叠表达式 → ((arg1 + arg2) + arg3) + ...
}

sum(1, 2, 3, 4);  // 10
```

### 4.3 实战：`make_unique` 的完美转发

```cpp
template<typename T, typename... Args>
std::unique_ptr<T> make_unique(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}

auto p = make_unique<std::string>(5, 'x');  // new std::string(5, 'x') → "xxxxx"
```

---

## 5. SFINAE——模板元编程的核心法则

**SFINAE** = **S**ubstitution **F**ailure **I**s **N**ot **A**n **E**rror（替换失败并非错误）。

含义：编译器尝试匹配模板时，如果某个候选的模板参数替换失败了，**不报错**，而是悄悄把这个候选从候选集中移除，继续找下一个。

### 5.1 为什么需要 SFINAE？

```cpp
// 需求：对整数类型用一种实现，对浮点类型用另一种
template<typename T>
typename T::value_type process(const T&);   // 版本 A：希望匹配容器

template<typename T>
double process(const T&);                    // 版本 B：匹配基础类型

std::vector<int> v;
process(v);  // 尝试版本 A → vector<int> 有 value_type → 成功 ✅
             // 版本 B 也合法，但 A 更特化 → 选择 A

process(3.14); // 尝试版本 A → double 没有 value_type → 替换失败，不是错误！
               // 尝试版本 B → 成功 ✅
```

如果没有 SFINAE，`double::value_type` 就是硬错误，整个编译失败。

### 5.2 `std::enable_if`——条件性地启用/禁用模板

```cpp
// enable_if<条件, 类型>：条件为 true 时定义类型，false 时无定义（触发 SFINAE）

// 只在 T 是整型时启用
template<typename T>
typename std::enable_if<std::is_integral<T>::value, void>::type
process(T val) {
    std::cout << "整数版本: " << val << "\n";
}

// 只在 T 是浮点型时启用
template<typename T>
typename std::enable_if<std::is_floating_point<T>::value, void>::type
process(T val) {
    std::cout << "浮点版本: " << val << "\n";
}

process(42);     // "整数版本"
process(3.14);   // "浮点版本"
// process("hi"); // ❌ 编译错误：两个版本都不匹配
```

### 5.3 C++14/17 的简化写法

```cpp
// C++14: enable_if_t 别名
template<typename T>
std::enable_if_t<std::is_integral_v<T>, void>  // _v 是 C++17 的变量模板
process(T val) { ... }

// C++20: requires 子句——SFINAE 的继承者
template<typename T>
    requires std::is_integral_v<T>    // 简洁！可读！
void process(T val) { ... }
```

### 5.4 `std::void_t`——检测类型是否有某个成员

```cpp
// 检测 T 是否有 value_type 成员类型
template<typename, typename = void>
struct has_value_type : std::false_type {};

template<typename T>
struct has_value_type<T, std::void_t<typename T::value_type>> : std::true_type {};
//                       ↑ void_t 会吞掉任何类型，返回 void
//                       如果 T::value_type 不合法 → 替换失败 → 回退到通用版本（false_type）

has_value_type<std::vector<int>>::value;  // true
has_value_type<int>::value;               // false
```

`void_t` 是 SFINAE 的瑞士军刀——用它可以在编译期检测类型是否有某个成员函数、成员类型、是否支持某个操作等。标准库中的类型萃取（type traits）大量依赖 `void_t`。

---

## 6. CRTP——用模板实现静态多态

**CRTP** = **C**uriously **R**ecurring **T**emplate **P**attern（奇异递归模板模式）。

核心思想：**派生类将自己作为模板参数传给基类**，基类可以在编译期就知道派生类的类型，从而在基类中调用派生类的方法——无需虚函数。

```cpp
// CRTP 基类
template<typename Derived>
class Animal {
public:
    void speak() const {
        // static_cast<const Derived*>(this) → 在编译期确定类型，无需 vtable
        static_cast<const Derived*>(this)->speakImpl();
    }
};

class Dog : public Animal<Dog> {   // ← Dog 把自己传给 Animal
public:
    void speakImpl() const { std::cout << "Woof!\n"; }
};

class Cat : public Animal<Cat> {
public:
    void speakImpl() const { std::cout << "Meow!\n"; }
};

template<typename T>
void makeSound(const Animal<T>& animal) {
    animal.speak();
}

Dog dog;
Cat cat;
makeSound(dog);  // "Woof!" — 编译期绑定，无虚函数开销
makeSound(cat);  // "Meow!"
```

### 6.1 CRTP vs 虚函数

| 方面 | 虚函数 | CRTP |
|------|--------|------|
| 绑定时机 | 运行期（动态多态） | 编译期（静态多态） |
| 开销 | vptr + 间接调用 | 零开销（内联） |
| 灵活性 | 可以存异构容器 | 每个派生类是不同类型 |
| 代码膨胀 | 一份代码 | 每类型生成一份 |
| `override` 检查 | ✅ | ❌（函数名写错就悄悄变新函数） |

### 6.2 CRTP 实战：`std::enable_shared_from_this`

标准库中的经典 CRTP 应用：

```cpp
class MyClass : public std::enable_shared_from_this<MyClass> {
public:
    std::shared_ptr<MyClass> getShared() {
        return shared_from_this();  // 安全地返回指向自己的 shared_ptr
    }
};

auto p = std::make_shared<MyClass>();
auto q = p->getShared();  // p 和 q 共享同一个引用计数
```

### 6.3 CRTP 在 AI Infra 中的应用

在 AI 框架中，CRTP 常用于**避免虚函数开销但保持接口统一**：

```cpp
// 不同后端（CUDA / CPU / MPS）的算子实现
template<typename Derived>
class OpKernel {
public:
    void compute(const Tensor& in, Tensor& out) {
        // 公共逻辑：维度检查、内存对齐等
        static_cast<Derived*>(this)->computeImpl(in, out);
    }
};

class CudaConv2D : public OpKernel<CudaConv2D> {
public:
    void computeImpl(const Tensor& in, Tensor& out) {
        // CUDA 特定实现
        cudaConv2DForward(in.data(), out.data(), ...);
    }
};

class CpuConv2D : public OpKernel<CpuConv2D> {
public:
    void computeImpl(const Tensor& in, Tensor& out) {
        // CPU 特定实现
        im2col(in.data(), out.data(), ...);
    }
};
```

这里 CRTP 避免了将 `compute()` 做成虚函数——在 AI 推理的热路径上，一次虚函数调用的开销可能就决定了一个 kernel 延迟是否达标。

---

## 7. 模板的编译期计算——比运行时快，但别滥用

### 7.1 编译期阶乘

```cpp
template<int N>
struct Factorial {
    static constexpr int value = N * Factorial<N - 1>::value;
};

template<>
struct Factorial<0> {    // 全特化终止递归
    static constexpr int value = 1;
};

int x = Factorial<5>::value;  // 编译期计算出 120，运行时就是常量 120
```

### 7.2 C++17 `if constexpr`——编译期分支

```cpp
template<typename T>
auto getValue(T t) {
    if constexpr (std::is_pointer_v<T>) {
        return *t;         // 编译期分支，T 不是指针时这行代码根本不被编译
    } else {
        return t;
    }
}

getValue(42);     // 编译时只保留 else 分支
getValue(&x);     // 编译时只保留 if 分支
```

不需要 SFINAE 或重载，一个函数即可处理不同类型。

---

## 小结

| 概念 | 一句话 |
|------|--------|
| 函数模板 | 类型参数化的函数，编译期生成具体版本 |
| 类模板 | 类型参数化的类，如 `std::vector<T>` |
| 全特化 | 为特定类型提供完全不同的实现 |
| 偏特化 | 对符合模式的类型提供不同实现（如 `T*`） |
| 可变参数模板 | `typename... Args` + 递归/折叠展开 |
| SFINAE | 替换失败不报错，悄悄排除候选 |
| `enable_if` | 条件性启用/禁用模板重载 |
| `void_t` | 检测类型是否拥有某个成员 |
| CRTP | 派生类把自己传给基类模板，实现静态多态 |
| `if constexpr` | C++17 编译期分支，替代部分 SFINAE |

下一篇 [07 STL 容器底层原理](./07-stl-containers-deep-dive.md)。
