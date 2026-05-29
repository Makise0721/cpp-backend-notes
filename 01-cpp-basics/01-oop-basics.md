# 01 — C++ 面向对象核心：类、对象、封装、继承、多态

> 关联篇目：[02 RAII 与三五零法则](./02-raii-rule-of-five.md) | [04 虚函数与 vtable](./04-virtual-vtable.md) | [10 对象内存模型](./10-object-memory-model.md)

---

## 1. 类与对象 — 从"数据+行为"的组合开始

**类（class）**是 C++ 中用户自定义的类型蓝图，**对象（object）**是类的实例。类和 C 的 `struct` 本质区别在于：**类不仅打包数据，还打包操作数据的行为（成员函数）**，并通过访问控制决定谁能碰哪些东西。

```cpp
class Person {
private:                    // 只有自己能看到
    std::string name_;
    int age_;

public:                     // 所有人都能看到
    Person(std::string name, int age) : name_(name), age_(age) {}

    void sayHello() const { // const 承诺不修改成员
        std::cout << "Hi, I'm " << name_ << ", " << age_ << " years old.\n";
    }

    void birthday() {       // 非 const，会修改 age_
        ++age_;
    }
};

int main() {
    Person alice("Alice", 30);   // 栈上创建对象
    alice.sayHello();
    alice.birthday();
}
```

### 关键概念：`this` 指针

每个非静态成员函数内部都有一个隐式的 `this` 指针，指向调用该函数的对象。上面 `sayHello()` 里的 `name_` 等价于 `this->name_`。为什么需要它？

```cpp
Person& Person::birthday() {
    ++age_;
    return *this;   // 返回自身的引用，支持链式调用
}
// 用法：alice.birthday().sayHello();
```

---

## 2. 封装（Encapsulation）—— 把数据藏起来，只暴露接口

封装是 OOP 的第一支柱。核心思想：**内部状态私有化，通过公开的成员函数控制访问**。

### 为什么封装？

```cpp
// 坏设计：直接暴露数据
struct BadDate {
    int year, month, day;
};
BadDate d; d.month = 13;  // 非法值！没有校验

// 好设计：数据私有，通过接口访问
class Date {
private:
    int year_, month_, day_;
public:
    void setMonth(int m) {
        if (m < 1 || m > 12) throw std::invalid_argument("month must be 1-12");
        month_ = m;
    }
    int getMonth() const { return month_; }
};
```

封装的好处：修改内部实现不影响外部代码（比如把 `int month_` 改成 `enum`），可以加校验逻辑，可以跟踪谁访问了什么。

### 访问限定符的三个级别

| 限定符 | 类自身 | 派生类 | 外部代码 |
|--------|--------|--------|----------|
| `private` | ✅ | ❌ | ❌ |
| `protected` | ✅ | ✅ | ❌ |
| `public` | ✅ | ✅ | ✅ |

经典经验：**成员变量一律 `private`，只把需要暴露的接口设为 `public`**。`protected` 用在"派生类需要访问、但外部不该碰"的场景。

> 访问限定符与继承方式（`class B : public A`）的交叉影响在 [04 虚函数与 vtable](./04-virtual-vtable.md) 中详述。

---

## 3. 继承（Inheritance）—— "is-a" 关系的表达

继承让一个类获得另一个类的成员，表达"是一种"的层次关系。

```cpp
class Animal {
protected:
    std::string name_;
public:
    Animal(std::string name) : name_(name) {}
    virtual ~Animal() {}              // 虚析构，后面详述
    virtual void speak() const = 0;   // 纯虚函数
    std::string getName() const { return name_; }
};

class Dog : public Animal {           // public 继承：is-a
public:
    Dog(std::string name) : Animal(name) {}
    void speak() const override {
        std::cout << name_ << " says: Woof!\n";
    }
};

class Cat : public Animal {
public:
    Cat(std::string name) : Animal(name) {}
    void speak() const override {
        std::cout << name_ << " says: Meow!\n";
    }
};
```

### 三种继承方式

```cpp
class B : public A    {};  // A 的 public → B 的 public；protected → protected
class B : protected A {};  // A 的 public/protected → B 的 protected
class B : private A   {};  // A 的 public/protected → B 的 private（默认 for class）
```

99% 的情况用 `public` 继承。`private` 继承表达"用基类实现，但不是 is-a"（很少用，组合通常更好）。

---

## 4. 多态（Polymorphism）—— 同一个接口，不同的行为

C++ 多态分两类：

### 4.1 编译期多态（静态多态）

函数重载和模板——编译时就确定调用哪个版本。

```cpp
void print(int x)    { std::cout << "int: " << x << "\n"; }
void print(double x) { std::cout << "double: " << x << "\n"; }
void print(std::string x) { std::cout << "string: " << x << "\n"; }

print(42);       // 调用 int 版本
print(3.14);     // 调用 double 版本
print("hello");  // 调用 string 版本（const char* → string 隐式转换）
```

> 详见 [03 静态成员与运算符重载](./03-static-operator-overload.md) 和 [06 模板/SFINAE/CRTP](./06-templates-sfinae-crtp.md)。

### 4.2 运行期多态（动态多态）

通过**虚函数**实现——运行时根据对象的实际类型决定调用哪个函数。

```cpp
void makeSound(const Animal& a) {
    a.speak();  // 运行期才知道是 Dog 还是 Cat
}

Dog dog("Buddy");
Cat cat("Kitty");
makeSound(dog);  // 输出 "Buddy says: Woof!"
makeSound(cat);  // 输出 "Kitty says: Meow!"
```

核心机制：虚函数表（vtable），详见 [04 虚函数与 vtable](./04-virtual-vtable.md)。

### 多态的必要条件

1. **指针或引用**调用——`Animal a = Dog("x");` 会发生**对象切片**（slicing），丢失派生类信息
2. 函数标记为 `virtual`
3. 派生类 `override` 该虚函数

---

## 5. 构造函数家族——对象一生的起点

### 5.1 默认构造函数

```cpp
class Widget {
    int id_;
    std::string label_;
public:
    Widget() : id_(0), label_("default") {}  // 默认构造
    Widget(int id) : id_(id), label_("unnamed") {}  // 带参数构造
};
Widget w1;        // 调用默认构造
Widget w2(42);    // 调用带参数构造
Widget w3{};      // C++11：值初始化，调用默认构造（区别：对基本类型清零）
```

### 5.2 拷贝构造函数

用已有对象初始化新对象时调用：

```cpp
Widget(const Widget& other) : id_(other.id_), label_(other.label_) {
    std::cout << "Copy ctor called\n";
}

Widget a(1);
Widget b(a);     // 拷贝构造
Widget c = a;    // 也是拷贝构造（不是赋值！是初始化）
```

### 5.3 移动构造函数（C++11）

"偷"走临时对象的资源，避免昂贵拷贝：

```cpp
Widget(Widget&& other) noexcept
    : id_(other.id_), label_(std::move(other.label_)) {
    other.id_ = 0;  // 将源对象置于"有效但未指定"状态
}

Widget d(std::move(a));  // 显式移动
Widget e(getWidget());   // 函数返回的临时对象，自动移动
```

移动后原对象的状态：通常被重置为默认值或"空"状态。**不应再使用被移动的对象，除非先重新赋值**。

> 移动语义是 RAII 的自然延伸，详见 [02 RAII 与三五零法则](./02-raii-rule-of-five.md)。

### 5.4 析构函数

对象生命周期结束时自动调用：

```cpp
~Widget() {
    // 释放资源：关闭文件、释放内存、断开连接等
}
```

析构函数**绝不抛异常**（C++11 起默认 `noexcept`）。如果在析构中抛异常且当前已有异常在传播，程序直接 `std::terminate`。

### 5.5 拷贝赋值运算符 vs 移动赋值运算符

```cpp
// 拷贝赋值：a = b（a 和 b 都已经存在）
Widget& operator=(const Widget& other) {
    if (this != &other) {  // 防止自赋值
        id_ = other.id_;
        label_ = other.label_;
    }
    return *this;
}

// 移动赋值：a = std::move(b)
Widget& operator=(Widget&& other) noexcept {
    if (this != &other) {
        id_ = other.id_;
        label_ = std::move(other.label_);
        other.id_ = 0;
    }
    return *this;
}
```

注意拷贝赋值和移动赋值的返回值是 `Widget&`（非 const），以支持 `a = b = c` 链式赋值。

---

## 6. 深浅拷贝——指针成员是陷阱的起点

当类包含指针成员时，默认的拷贝构造只拷贝**指针的值**（浅拷贝），导致两个对象指向同一块内存，析构时 double-free：

```cpp
class ShallowBuffer {
    char* data_;
public:
    ShallowBuffer(const char* s) {
        data_ = new char[strlen(s) + 1];
        strcpy(data_, s);
    }
    ~ShallowBuffer() { delete[] data_; }
    // 缺少自定义拷贝构造 → 默认的浅拷贝
};

ShallowBuffer a("hello");
ShallowBuffer b = a;  // a.data_ 和 b.data_ 指向同一块内存
// 析构时：a 删一次，b 再删一次 → double free → crash!
```

**深拷贝**：复制指针指向的内容，而不是指针本身。

```cpp
class DeepBuffer {
    char* data_;
public:
    DeepBuffer(const char* s) {
        data_ = new char[strlen(s) + 1];
        strcpy(data_, s);
    }
    // 拷贝构造：深拷贝
    DeepBuffer(const DeepBuffer& other) {
        data_ = new char[strlen(other.data_) + 1];
        strcpy(data_, other.data_);
    }
    // 拷贝赋值
    DeepBuffer& operator=(const DeepBuffer& other) {
        if (this != &other) {
            delete[] data_;
            data_ = new char[strlen(other.data_) + 1];
            strcpy(data_, other.data_);
        }
        return *this;
    }
    ~DeepBuffer() { delete[] data_; }
};
```

**现代 C++ 的最佳实践：用 `std::string` 代替 `char*`，用 `std::vector` 代替动态数组**——标准库类已经正确实现了深拷贝和移动语义，你根本不需要自己管理裸指针。这直接通向 **Rule of Zero**（见 [02 RAII 与三五零法则](./02-raii-rule-of-five.md)）。

---

## 7. 友元（friend）—— 在封装的墙上开一扇门

`friend` 让外部函数或另一个类可以访问本类的 `private` 和 `protected` 成员。

```cpp
class BankAccount {
    friend class Auditor;           // 类级友元
    friend void printBalance(const BankAccount&);  // 函数级友元

private:
    double balance_;
public:
    BankAccount(double bal) : balance_(bal) {}
};

class Auditor {
public:
    void inspect(const BankAccount& acc) {
        std::cout << "Balance: " << acc.balance_ << "\n";  // 可以访问 private
    }
};

void printBalance(const BankAccount& acc) {
    std::cout << acc.balance_ << "\n";  // 友元函数也可以
}
```

**使用原则**：友元破坏封装，应尽量少用。最常见的合理场景是运算符重载（`operator<<` 需要访问私有成员但又不能是成员函数）。

---

## 8. 内部类（Nested Class）—— 辅助类的收纳

在类内部定义的类，用于封装实现细节：

```cpp
class LinkedList {
private:
    struct Node {          // 内部 struct（默认 public），外部不可见
        int value;
        Node* next;
        Node(int v, Node* n = nullptr) : value(v), next(n) {}
    };
    Node* head_;

public:
    class Iterator {       // 公开的内部类
        Node* current_;
    public:
        explicit Iterator(Node* n) : current_(n) {}
        int operator*() const { return current_->value; }
        Iterator& operator++() { current_ = current_->next; return *this; }
        bool operator!=(const Iterator& other) const {
            return current_ != other.current_;
        }
    };

    Iterator begin() { return Iterator(head_); }
    Iterator end()   { return Iterator(nullptr); }
};
```

内部类可以访问外部类的 `private` 成员（如果持有外部类对象的引用/指针），这是实现迭代器等模式的常用手法。

---

## 小结

| 概念 | 一句话 |
|------|--------|
| 封装 | 数据私有，接口公开，控制访问 |
| 继承 | "is-a" 层次，派生类获得基类接口 |
| 多态 | 同一接口不同行为——编译期（重载/模板）或运行期（虚函数） |
| 拷贝 vs 移动 | 拷贝复制资源，移动转移资源（更快） |
| 深拷贝 | 复制指针指向的内容，不是指针本身 |
| 友元 | 封装的例外，慎用 |

下一篇 [02 RAII 与三五零法则](./02-raii-rule-of-five.md) 深入 C++ 最核心的资源管理哲学。
