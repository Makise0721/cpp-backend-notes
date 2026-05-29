# 09 — 命名空间、作用域、异常处理与 noexcept

> 关联篇目：[01 OOP 基础](./01-oop-basics.md) | [02 RAII 与三五零法则](./02-raii-rule-of-five.md)

---

## 第一部分：命名空间与作用域

### 1. 命名空间——名字冲突的防火墙

C 语言的最大痛点之一：所有函数名在全局命名空间内，大型项目中名字冲突是常态。C++ 的 `namespace` 解决这个问题：

```cpp
namespace Audio {
    void init() { /* 初始化音频 */ }
    void play(const char* file) { /* 播放 */ }
}

namespace Video {
    void init() { /* 初始化视频 */ }  // 和 Audio::init 不冲突！
    void play(const char* file) { /* 渲染 */ }
}

Audio::init();
Video::init();
```

### 1.1 嵌套与别名

```cpp
namespace Company::Project::Module {  // C++17 嵌套命名空间
    class Widget {};
}

// 过长时取别名
namespace CPM = Company::Project::Module;
CPM::Widget w;

// 匿名命名空间 → 翻译单元内可见，外部不可见（替代 C 的 static）
namespace {
    int internalCounter = 0;  // 只在当前 .cpp 中可见
}
```

### 1.2 `using` 的正确用法

```cpp
// ❌ 永远不要在头文件的全局作用域写这个
using namespace std;

// ✅ .cpp 文件中可以
using namespace std;

// ✅ 限定使用
using std::cout;
using std::string;

// ✅ 在函数作用域内
void func() {
    using namespace std;
    cout << "ok\n";
}
```

### 1.3 作用域四层结构

```
全局作用域
  └─ 命名空间作用域
       └─ 类作用域
            └─ 块作用域（函数体、if/for 体）

名字查找顺序：从内到外
```

```cpp
int x = 1;                    // 全局

namespace N {
    int x = 2;                // 命名空间
    class C {
        int x = 3;            // 类
        void f() {
            int x = 4;        // 块
            std::cout << x;   // 4（最近的作用域）
            std::cout << C::x;       // 3
            std::cout << N::x;       // 2
            std::cout << ::x;        // 1（::  = 全局）
        }
    };
}
```

### 1.4 ADL（Argument-Dependent Lookup，Koenig Lookup）

当调用函数时，编译器会在**参数类型所在的命名空间**中查找函数——这就是为什么 `std::cout << "hello"` 不需要写全限定名：

```cpp
namespace MyLib {
    class Vec { int x, y; };
    Vec operator+(const Vec& a, const Vec& b) { /* ... */ }
    void print(const Vec& v) { /* ... */ }
}

MyLib::Vec a, b;
auto c = a + b;   // ✅ 编译期在 MyLib 中找到 operator+
print(a);         // ✅ 同样通过 ADL 找到 MyLib::print
```

ADL 是 C++ 中少有的"魔法"——好处是让运算符重载自然地工作，坏处是可能意外调用到错误的函数。

---

## 第二部分：异常处理

### 2. 异常——面向错误恢复的设计

C++ 的异常机制让错误处理代码和正常业务逻辑分离。

```cpp
void process(int n) {
    if (n < 0) throw std::invalid_argument("n must be >= 0");
    if (n == 0) throw std::runtime_error("division by zero would occur");
    std::cout << 100 / n << "\n";
}

int main() {
    try {
        process(-1);
        process(0);
        process(5);
    } catch (const std::invalid_argument& e) {
        std::cerr << "Bad argument: " << e.what() << "\n";
    } catch (const std::runtime_error& e) {
        std::cerr << "Runtime error: " << e.what() << "\n";
    } catch (const std::exception& e) {
        std::cerr << "Unknown: " << e.what() << "\n";  // 兜底
    }
    // 无论是否抛异常，这里都会执行（栈展开已完成）
}
```

### 2.1 标准异常类层次

```
std::exception
├── std::logic_error          ← 程序逻辑错误（可预防）
│   ├── std::invalid_argument
│   ├── std::domain_error
│   ├── std::length_error
│   └── std::out_of_range
├── std::runtime_error        ← 运行时错误（不可预防）
│   ├── std::range_error
│   ├── std::overflow_error
│   └── std::underflow_error
└── std::bad_alloc            ← new 分配失败（是 exception 的直接子类）
    std::bad_cast             ← dynamic_cast 引用类型转换失败
    std::bad_typeid
```

### 2.2 栈展开与 RAII

抛异常时，运行时从 `throw` 点向上遍历调用栈，依次调用每个作用域中局部对象的析构函数——**这是 RAII 的核心保障**。

```cpp
void riskyOperation() {
    auto conn = Database::connect();     // RAII：析构时断开连接
    auto transaction = conn.beginTx();   // RAII：析构时回滚
    auto buffer = std::make_unique<char[]>(1024);  // RAII：自动释放

    doSomething();  // 如果抛异常 → 以上三个对象全被自动析构
                    // buffer 释放，transaction 回滚，conn 断开
}   // 正常结束也会析构
```

### 2.3 异常安全的三级保证

这是写健壮 C++ 代码必须理解的：

| 级别 | 含义 | 示例 |
|------|------|------|
| **基本保证**（Basic） | 不泄漏资源，对象处于"有效但可能已修改"状态 | 大多数容器操作 |
| **强保证**（Strong） | 操作要么完全成功，要么完全回滚（commit-or-rollback） | `std::vector::push_back`（扩容失败时原 vector 不变） |
| **不抛异常保证**（Nothrow） | 操作绝不会抛异常 | `std::swap`，析构函数，移动构造（应标记 noexcept） |

```cpp
// 强异常安全 → copy-and-swap 惯用法
class Widget {
    std::string name_;
    std::vector<int> data_;

    Widget& operator=(const Widget& other) {
        // 先拷贝到一个临时对象——如果这一步抛异常，*this 不受影响
        Widget temp(other);
        // 用不抛异常的 swap 替换内容
        swap(temp);
        return *this;
    }  // temp 析构，释放旧资源
};
```

### 2.4 析构函数中不要抛异常

```cpp
class Bad {
public:
    ~Bad() {
        throw std::runtime_error("fail");  // ❌ 绝对不要！
    }
};
// 如果在栈展开时抛异常（即当前已有一个异常在传播），
// 析构再抛异常 → std::terminate() → 程序直接终止，无法恢复
```

如果析构中的某个操作可能失败，吞掉或记录日志：

```cpp
~Safe() {
    try { riskyCleanup(); }
    catch (...) { /* log and swallow */ }
}
```

---

## 第三部分：noexcept

### 3. `noexcept`——对编译器的承诺

```cpp
void mayThrow();                    // 可能抛异常
void neverThrows() noexcept;        // 承诺不抛异常
void maybeNever() noexcept(false);  // 显式说明可能抛异常

// noexcept 运算符——编译期检查表达式是否可能抛异常
static_assert(noexcept(std::swap(a, b)));  // swap 应标记 noexcept
```

### 3.1 为什么 noexcept 重要？

```cpp
std::vector<Widget> v;
v.reserve(100);
// 当 vector 扩容时：
// 如果 Widget 的移动构造是 noexcept → vector 用移动（快！）
// 如果 Widget 的移动构造不是 noexcept → vector 用拷贝（安全但慢！）
```

原因：如果移动过程中抛异常，已移动的部分无法恢复（被移动的元素已是"空"状态）。拷贝则可以安全回滚（只要删除临时拷贝即可）。所以 `vector` 只对 `noexcept` 移动使用移动语义。

### 3.2 标记 noexcept 的时机

| 应该标记 noexcept | 不应该标记 |
|-------------------|-----------|
| 析构函数（C++11 起默认） | 可能抛异常的函数 |
| 移动构造/赋值 | 调用 C 库的函数 |
| `swap` | 分配内存的函数 |
| `operator delete` | |

### 3.3 noexcept 与异常安全

```cpp
// noexcept 函数如果抛异常 → 直接 std::terminate()
void guaranteedSafe() noexcept {
    // throw std::runtime_error("oops"); // 如果执行 → terminate！
}
```

`noexcept` 是向调用方发的契约：我不会抛异常。不要标记为 `noexcept` 除非你真的能保证——违反契约的代价是程序直接终止，比未捕获的异常更糟糕。

---

## 小结

| 概念 | 一句话 |
|------|--------|
| namespace | 防止名字冲突，匿名 namespace 替代 C 的 static |
| ADL | 在参数类型所在命名空间中查找函数名 |
| 作用域 | 块→类→命名空间→全局，从内到外查找 |
| 异常 | try/catch/throw，栈展开时自动析构局部对象 |
| 强异常保证 | 操作要么全部成功，要么完全回滚（copy-and-swap） |
| noexcept | 承诺不抛异常，移动构造必须加以启用 vector 优化 |
| noexcept 运算符 | 编译期检查表达式是否 noexcept |

下一篇 [10 C++ 对象内存模型](./10-object-memory-model.md)。
