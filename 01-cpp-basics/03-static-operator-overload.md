# 03 — 静态成员与运算符重载

> 关联篇目：[01 OOP 基础](./01-oop-basics.md) | [02 RAII 与三五零法则](./02-raii-rule-of-five.md)

---

## 第一部分：静态成员

### 1. 静态成员变量——属于类，不属于对象

普通成员变量每个对象各有一份；**静态成员变量整个类只有一份**，被所有对象共享。

```cpp
class Soldier {
    static int count_;         // 声明（在类内），不是定义！
    std::string name_;
public:
    Soldier(std::string name) : name_(name) { ++count_; }
    ~Soldier() { --count_; }
    static int getCount() { return count_; }  // 静态成员函数
};

// 定义（在类外，.cpp 文件中）——必须有这一步，否则链接报错
int Soldier::count_ = 0;

Soldier a("Alpha"), b("Bravo");
std::cout << Soldier::getCount();  // 2 → 通过类名调用
```

### 1.1 静态变量的初始化

C++ 中静态变量的初始化顺序是一个非常经典的陷阱。

**同一翻译单元内**（同一个 .cpp 文件）：按定义顺序初始化，保证正确。

**不同翻译单元之间**：初始化顺序**未定义**！这就是著名的 **Static Initialization Order Fiasco**。

```cpp
// file_a.cpp
std::string globalConfig = "default";  // 可能在 file_b.cpp 的 Logger 之后初始化

// file_b.cpp
extern std::string globalConfig;
class Logger {
public:
    Logger() { std::cout << globalConfig; }  // 可能读到空字符串！
};
static Logger globalLogger;  // 在 main 之前构造，但 globalConfig 可能还没初始化
```

**解决方案**：用函数内的局部静态变量（Meyers' Singleton）：

```cpp
std::string& getConfig() {
    static std::string config = "default";  // 首次调用时才初始化，C++11 起线程安全
    return config;
}
```

### 1.2 线程安全

C++11 起，**函数内的局部静态变量初始化是线程安全的**——编译器自动插入锁，保证多线程同时首次调用时只初始化一次。

但**静态成员变量的使用不是线程安全的**。如果多个线程同时修改 `count_`，需要加锁或使用 `std::atomic`。

```cpp
class SafeCounter {
    static std::atomic<int> count_;
public:
    SafeCounter() { ++count_; }
    ~SafeCounter() { --count_; }
    static int getCount() { return count_.load(); }
};
std::atomic<int> SafeCounter::count_{0};
```

---

## 2. 静态成员函数——没有 `this` 的成员函数

```cpp
class Math {
public:
    static double sin(double x);   // 没有 this 指针
    static constexpr double PI = 3.1415926535;  // 静态常量可以在类内初始化
};
```

静态成员函数的特点：
- **没有 `this` 指针** → 不能访问非静态成员
- 可以通过 `ClassName::func()` 调用，也可以通过对象调用
- 不能是 `const`（`const` 修饰的是 `this`，没有 `this` 就没有 `const`）
- 不能是虚函数（虚函数依赖 `this` 的 vptr 进行动态分发）

---

## 第二部分：运算符重载

### 3. 函数重载——同一个名字，不同的参数

函数重载允许同一作用域内多个函数同名，靠**参数类型/数量**区分（返回值不参与重载决议）：

```cpp
void log(int x)         { std::cout << "int: " << x << "\n"; }
void log(double x)      { std::cout << "double: " << x << "\n"; }
void log(const char* x) { std::cout << "string: " << x << "\n"; }
void log(int x, int y)  { std::cout << "two ints: " << x << ", " << y << "\n"; }

log(42);       // int
log(3.14);     // double
log("hello");  // const char*
log(1, 2);     // two ints
```

### 3.1 重载决议的坑

```cpp
void f(int x)    {}
void f(double x) {}

f(0);     // int → 精确匹配
f(0.0);   // double → 精确匹配
f('a');   // int → char 先提升为 int，不是 double
f(0L);    // 编译错误！long 可以转 int 也可以转 double，两个都是"转换"—歧义
f(null);  // 不同编译器行为不同，可能有歧义
```

### 3.2 重载的局限性

重载不能跨作用域。派生类的同名函数会**隐藏**基类的所有同名函数（即使参数不同）：

```cpp
struct Base {
    void print(int x)    { std::cout << "Base int\n"; }
    void print(double x) { std::cout << "Base double\n"; }
};

struct Derived : Base {
    void print(const std::string& s) { std::cout << "Derived string\n"; }
};

Derived d;
d.print(42);    // ❌ 编译错误！Derived::print(string) 隐藏了 Base 的所有 print
d.Base::print(42); // ✅ 显式调用基类版本
```

解决方案：`using Base::print;` 把基类重载集引入派生类作用域。

---

### 4. 运算符重载——让自定义类型用起来像内置类型

```cpp
struct Vec2 {
    double x, y;

    Vec2(double x, double y) : x(x), y(y) {}

    // 成员函数形式（二元运算符，左操作数是 *this）
    Vec2 operator+(const Vec2& rhs) const {
        return Vec2(x + rhs.x, y + rhs.y);
    }
    Vec2 operator-(const Vec2& rhs) const {
        return Vec2(x - rhs.x, y - rhs.y);
    }
    Vec2 operator*(double s) const {
        return Vec2(x * s, y * s);
    }

    // 一元运算符
    Vec2 operator-() const { return Vec2(-x, -y); }  // 取负
    Vec2& operator+=(const Vec2& rhs) {
        x += rhs.x; y += rhs.y;
        return *this;
    }

    // 比较运算符
    bool operator==(const Vec2& rhs) const {
        return x == rhs.x && y == rhs.y;
    }
    bool operator!=(const Vec2& rhs) const { return !(*this == rhs); }
};

Vec2 a(1, 2), b(3, 4);
Vec2 c = a + b;         // 等价于 a.operator+(b)
Vec2 d = a - b;         // a.operator-(b)
Vec2 e = a * 3.0;       // a.operator*(3.0)
// Vec2 f = 3.0 * a;    // ❌ 等价于 3.0.operator*(a)，double 没有这个成员
```

`3.0 * a` 不行的原因：成员形式的运算符重载要求**左操作数是该类对象**。所以 `operator*` 在需要对称性时常写成**友元函数**：

```cpp
struct Vec2 {
    double x, y;
    Vec2 operator*(double s) const { return Vec2(x * s, y * s); }
    friend Vec2 operator*(double s, const Vec2& v) { return Vec2(v.x * s, v.y * s); }
};

Vec2 f = 3.0 * a;  // ✅ 调用友元函数版本
```

### 4.1 成员 vs 非成员——选哪个？

| 运算符 | 推荐形式 | 原因 |
|--------|----------|------|
| `=` `[]` `()` `->` | 必须成员 | 语言要求 |
| `+=` `-=` `*=` `/=` 等复合赋值 | 成员 | 修改自身状态 |
| `+` `-` `*` `/` 等二元算术 | 非成员（友元） | 支持两边隐式转换 |
| `<<` `>>`（流插入/提取） | 非成员（友元） | 左操作数是 `std::ostream&` |
| `==` `!=` `<` `>` 等比较 | 非成员（友元） | 对称性 |
| `++` `--`（前后缀） | 成员 | 修改自身 |

### 4.2 `operator<<` 的标准写法

```cpp
class Point {
    double x_, y_;
public:
    Point(double x, double y) : x_(x), y_(y) {}
    friend std::ostream& operator<<(std::ostream& os, const Point& p) {
        return os << "(" << p.x_ << ", " << p.y_ << ")";
    }
};

Point p(3, 4);
std::cout << p;  // 输出 (3, 4)
```

### 4.3 前置/后置 `++` 的区分

```cpp
class Counter {
    int value_;
public:
    // 前置 ++：返回修改后的引用
    Counter& operator++() { ++value_; return *this; }
    // 后置 ++：int 参数是哑元，仅用于区分签名；返回旧值的拷贝
    Counter operator++(int) { Counter old = *this; ++value_; return old; }
};

Counter c{0};
++c;   // 调用 operator++()，返回 Counter&
c++;   // 调用 operator++(int)，返回 Counter（旧值拷贝）
```

### 4.4 `operator[]` 与 `operator()`

```cpp
class Array {
    int data_[100];
public:
    int& operator[](size_t i) { return data_[i]; }        // 可读写
    const int& operator[](size_t i) const { return data_[i]; } // 只读
};
// 用法：arr[5] = 10;

class Adder {
public:
    int operator()(int a, int b) const { return a + b; }  // 函数对象
};
// 用法：Adder add; int x = add(3, 4);  // 像函数一样调用
```

---

### 5. 不可重载的运算符

以下运算符**不能**重载——它们是语言核心，重载会破坏语法一致性：

| 运算符 | 原因 |
|--------|------|
| `.`（成员访问） | 语义太基础，重载会破坏 `obj.member` 的含义 |
| `::`（作用域解析） | 编译期名字查找，不是运行时操作 |
| `?:`（三元条件） | 语法特殊（短路求值），参数不是普通表达式 |
| `sizeof` | 编译期求值 |
| `typeid` | RTTI 机制，语义固定 |
| `.*` `->*`（成员指针） | 极其少用，语义固定 |
| `static_cast` `dynamic_cast` `const_cast` `reinterpret_cast` | 类型转换语义由语言保证 |

---

### 6. C++20 三路比较运算符 `<=>`（太空船运算符）

一个运算符生成全部六种比较：

```cpp
struct Point {
    int x, y;
    auto operator<=>(const Point&) const = default;  // 编译器自动生成 ==, !=, <, <=, >, >=
};
// 一行代码，六种比较全有了。前提：成员类型都支持 <=>
```

这是 C++20 的变革——以前需要手写 `operator<` + `operator==`，然后用它们推导其他四种，现在一个 `= default` 搞定。

---

## 小结

| 概念 | 一句话 |
|------|--------|
| 静态成员变量 | 类级别共享，类外定义，注意初始化顺序陷阱 |
| 静态成员函数 | 无 `this`，不能 `const`/虚，通过 `类名::` 调用 |
| 函数重载 | 同域同名不同参，返回值不参与决议；跨作用域会被隐藏 |
| 运算符重载（成员） | 左操作数是 `*this`，必须用于 `=` `[]` `()` `->` |
| 运算符重载（非成员） | 对称性好，支持两边隐式转换，`<<` `>>` 的标配 |
| 不可重载 | `.` `::` `?:` `sizeof` `typeid` — 语言核心 |
| `<=>` | C++20 一个运算符顶六个 |

下一篇 [04 虚函数与 vtable](./04-virtual-vtable.md)。
