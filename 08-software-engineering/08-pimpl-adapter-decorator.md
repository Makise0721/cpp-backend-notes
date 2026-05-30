# 04 — Pimpl、适配器、装饰器模式

> 关联篇目：[第一章 02 RAII](../01-cpp-basics/02-raii-rule-of-five.md) | [01 编译链接](./01-compilation-link-process.md)

---

## 1. Pimpl 模式——编译防火墙

**P**ointer to **Impl**ementation（指向实现的指针）。将类的实现细节藏到 `.cpp` 文件中。

### 1.1 动机

```cpp
// ❌ bad: header 暴露了所有实现细节
// widget.h
#include "expensive_dep.h"
#include "another_dep.h"
class Widget {
    ExpensiveDep dep_;       // 每次修改这个依赖，所有 include widget.h 的都要重编译
    AnotherDep another_;     // 头文件臃肿，编译慢
public:
    void doSomething();
};
```

### 1.2 Pimpl 实现

```cpp
// widget.h —— 清爽
#include <memory>
class Widget {
    class Impl;                              // 前置声明
    std::unique_ptr<Impl> pimpl_;
public:
    Widget();
    ~Widget();                               // 必须在 .cpp 中实现（Impl 前置声明不完整）
    Widget(Widget&&) noexcept;               // 移动可以 = default 吗？看下方
    Widget& operator=(Widget&&) noexcept;
    void doSomething();
};
```

```cpp
// widget.cpp —— 所有实现细节在这里
#include "widget.h"
#include "expensive_dep.h"

class Widget::Impl {
    ExpensiveDep dep_;
    // 所有私有成员放在这里
public:
    void doSomething() { dep_.process(); }
};

Widget::Widget() : pimpl_(std::make_unique<Impl>()) {}
Widget::~Widget() = default;  // 必须在 Impl 完整定义之后
Widget::Widget(Widget&&) noexcept = default;
Widget::Widget& operator=(Widget&&) noexcept = default;
void Widget::doSomething() { pimpl_->doSomething(); }
```

### 1.3 为什么移动构造不能在头文件 `= default`？

`std::unique_ptr<Impl>` 的析构函数需要在 Impl **完整定义**时才能编译。如果在头文件 `= default` 移动构造，编译器生成的代码中包含了 `unique_ptr` 析构（异常时），而这时 Impl 还不完整 → 编译错误。

### 1.4 Pimpl 的优缺点

| 优点 | 缺点 |
|------|------|
| 编译隔离（修改 .cpp 不需要重编译用户） | 多一次堆分配 |
| 头文件干净 | 多一次间接访问 |
| 二进制兼容（改 Impl 不影响 ABI） | 无法内联（除非在 .cpp 中内联） |

---

## 2. 适配器模式（Adapter）——接口转换

### 2.1 类适配器（继承）

```cpp
// 已有接口（不兼容）
class OldPrinter {
public:
    void printOld(const char* text) { /* ... */ }
};

// 目标接口
class IPrinter {
public:
    virtual void print(const std::string& text) = 0;
    virtual ~IPrinter() = default;
};

// 适配器：继承已有 + 实现接口
class PrinterAdapter : public OldPrinter, public IPrinter {
public:
    void print(const std::string& text) override {
        printOld(text.c_str());  // 转换调用
    }
};
```

### 2.2 对象适配器（组合）

```cpp
class PrinterAdapter : public IPrinter {
    OldPrinter printer_;  // 持有，不继承
public:
    void print(const std::string& text) override {
        printer_.printOld(text.c_str());
    }
};
```

### 2.3 STL 中的适配器

```cpp
// stack 适配 deque
std::stack<int> stk;  // 默认基于 deque

// 改成基于 vector
std::stack<int, std::vector<int>> stk2;

// stack 内部只是把 push_back → push, pop_back → pop, back → top
// 适配器 = 换个接口，底层逻辑不变
```

---

## 3. 装饰器模式（Decorator）——动态添加功能

**不改变原有类，给对象动态添加额外职责。**

### 3.1 经典实现

```cpp
// 被装饰的接口
class Coffee {
public:
    virtual double cost() const = 0;
    virtual std::string desc() const = 0;
    virtual ~Coffee() = default;
};

// 具体组件
class SimpleCoffee : public Coffee {
public:
    double cost() const override { return 2.0; }
    std::string desc() const override { return "Coffee"; }
};

// 装饰器基类
class CoffeeDecorator : public Coffee {
    std::unique_ptr<Coffee> coffee_;
protected:
    Coffee* getCoffee() const { return coffee_.get(); }
public:
    explicit CoffeeDecorator(std::unique_ptr<Coffee> c) : coffee_(std::move(c)) {}
};

// 具体装饰器
class MilkDecorator : public CoffeeDecorator {
public:
    using CoffeeDecorator::CoffeeDecorator;
    double cost() const override { return getCoffee()->cost() + 0.5; }
    std::string desc() const override { return getCoffee()->desc() + " + Milk"; }
};

class SugarDecorator : public CoffeeDecorator {
public:
    using CoffeeDecorator::CoffeeDecorator;
    double cost() const override { return getCoffee()->cost() + 0.2; }
    std::string desc() const override { return getCoffee()->desc() + " + Sugar"; }
};

// 可以嵌套装饰
auto coffee = std::make_unique<SimpleCoffee>();
coffee = std::make_unique<MilkDecorator>(std::move(coffee));
coffee = std::make_unique<SugarDecorator>(std::move(coffee));
// Coffee + Milk + Sugar, cost = 2.0 + 0.5 + 0.2 = 2.7
```

### 3.2 现代 C++ 替代——组合 Lambda

```cpp
// 对于简单场景，用 std::function 链式包装
std::function<double()> cost = [] { return 2.0; };
cost = [cost] { return cost() + 0.5; };   // 加牛奶
cost = [cost] { return cost() + 0.2; };   // 加糖
std::cout << cost();  // 2.7
```

### 3.3 适配器 vs 装饰器 vs 代理

| | 适配器 | 装饰器 | 代理 |
|------|------|------|------|
| 改变接口 | ✅ | ❌ | ❌ |
| 增强功能 | ❌ | ✅ | 控制访问 |
| 关注点 | 接口兼容 | 动态叠加职责 | 访问控制 |

---

## 小结

| 模式 | 核心 | C++ 实现 |
|------|------|----------|
| Pimpl | 隐藏实现细节 | `unique_ptr<Impl>` + 前置声明 |
| 适配器 | 接口转换 | 继承/组合 + 方法转发 |
| 装饰器 | 动态添加功能 | 嵌套包装 + `unique_ptr` |
| STL 适配器 | 限制接口 | `stack`/`queue` 包装 `deque`/`vector` |

下一篇 [05 模板方法 / 责任链模式](./05-template-method-chain.md)。
