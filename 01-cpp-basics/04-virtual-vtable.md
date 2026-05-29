# 04 — 虚函数、虚函数表与多态实现原理

> 关联篇目：[01 OOP 基础](./01-oop-basics.md) | [05 菱形继承与虚继承](./05-diamond-virtual-inheritance.md) | [10 对象内存模型](./10-object-memory-model.md)

---

## 1. 虚函数——运行期多态的钥匙

普通成员函数是**静态绑定**——调用哪个版本在编译期根据指针/引用的**声明类型**决定。虚函数是**动态绑定**——在运行期根据对象的**实际类型**决定。

```cpp
class Animal {
public:
    virtual void speak() const { std::cout << "???\n"; }  // 虚函数
};

class Dog : public Animal {
public:
    void speak() const override { std::cout << "Woof!\n"; }  // override：显式覆写
};

class Cat : public Animal {
public:
    void speak() const override { std::cout << "Meow!\n"; }
};

// 关键演示
Dog dog;
Animal& ref = dog;        // ref 的静态类型是 Animal&，实际指向 Dog
ref.speak();              // 输出 "Woof!" —— 动态绑定！

Animal* ptr = &dog;
ptr->speak();             // 同样输出 "Woof!"

Animal copy = dog;        // ⚠️ 对象切片！copy 就是一个 Animal
copy.speak();             // 输出 "???" —— 切片后丢失了 Dog 的身份
```

**为什么切片会丢失多态？** 因为虚函数表指针 `vptr` 在拷贝时被设为 `Animal` 的，而不是 `Dog` 的。下面详述。

---

## 2. 纯虚函数与抽象类

**纯虚函数** = 只声明不实现，强制派生类必须覆写：

```cpp
class Shape {                          // 有纯虚函数的类 = 抽象类
public:
    virtual double area() const = 0;   // = 0 → 纯虚函数
    virtual void draw() const = 0;
    virtual ~Shape() {}                // 虚析构（后面讲）
};

// Shape s;       // ❌ 抽象类不能实例化
// Shape* p = new Shape(); // ❌ 同样不行

class Circle : public Shape {
    double radius_;
public:
    double area() const override { return 3.14159 * radius_ * radius_; }
    void draw() const override { /* ... */ }
};

Circle c;          // ✅ 必须实现所有纯虚函数才能实例化
Shape* s = &c;     // ✅ 可以用基类指针/引用操作派生类对象
```

**纯虚函数可以有实现**（罕见但合法），派生类需要显式调用 `Base::func()`。

---

## 3. 虚析构——多态 delete 的救命稻草

```cpp
class Base {
public:
    ~Base() { std::cout << "~Base\n"; }  // ⚠️ 非虚析构！
};

class Derived : public Base {
    std::string data_;
public:
    ~Derived() { std::cout << "~Derived\n"; }
};

Base* p = new Derived();
delete p;  // 输出：~Base
           // ⚠️ Derived 的析构函数没有被调用！data_ 泄漏！
```

**原因**：`delete p` 时，编译器只根据 `p` 的静态类型 `Base*` 调用 `Base::~Base()`，根本不知道实际对象是 `Derived`。非虚析构 = 不经过虚函数表。

**解决方案**：将基类析构声明为虚函数：

```cpp
class Base {
public:
    virtual ~Base() { std::cout << "~Base\n"; }  // ✅ 虚析构
};

delete p;  // 输出：~Derived → ~Base ✅ 链式调用，Derived 的资源被正确释放
```

**铁律**：如果一个类设计为"作为基类被继承"，必须有虚析构。反之，如果一个类不是为继承设计的，不要加虚析构（会引入 vptr，增加对象大小）。

---

## 4. 虚函数表（vtable）与虚函数表指针（vptr）

这是多态的实现机制——理解它对调试、性能优化、面试都很关键。

### 4.1 基本思想

每个**有虚函数的类**有一个虚函数表（vtable），是一张函数指针数组。每个**对象**有一个虚函数表指针（vptr），指向它所属类的 vtable。

```
虚函数调用过程：
ptr->speak()
  → 通过对象的 vptr 找到 vtable
  → 在 vtable 中查找 speak 对应的函数指针
  → 调用该函数
```

### 4.2 单继承下的 vtable 布局

```cpp
class Base {
public:
    virtual void f() { std::cout << "Base::f\n"; }
    virtual void g() { std::cout << "Base::g\n"; }
    virtual void h() { std::cout << "Base::h\n"; }
};

class Derived : public Base {
public:
    void f() override { std::cout << "Derived::f\n"; }   // 覆写 f
    void g() override { std::cout << "Derived::g\n"; }   // 覆写 g
    // h() 没有覆写，继承 Base::h
    virtual void k() { std::cout << "Derived::k\n"; }    // 新增虚函数
};
```

内存布局（概念示意）：

```
Base 对象:                    Base vtable:
┌──────────┐                 ┌──────────────────┐
│  vptr    │──────────────→  │ slot 0: &Base::f │
├──────────┤                 │ slot 1: &Base::g │
│  成员...  │                 │ slot 2: &Base::h │
└──────────┘                 └──────────────────┘

Derived 对象:                 Derived vtable:
┌──────────┐                 ┌──────────────────────┐
│  vptr    │──────────────→  │ slot 0: &Derived::f  │ ← 覆写
├──────────┤                 │ slot 1: &Derived::g  │ ← 覆写
│ Base成员  │                 │ slot 2: &Base::h     │ ← 未覆写，指向基类版本
├──────────┤                 │ slot 3: &Derived::k  │ ← 新增
│ 成员...   │                 └──────────────────────┘
└──────────┘
```

**关键观察**：
1. vtable 按虚函数声明顺序排列（先基类后派生类新增）
2. 覆写的函数替换对应 slot 中的指针
3. 未覆写的函数保持基类指针
4. 每个对象只存一个 vptr（通常 8 字节，64 位系统）

### 4.3 多继承下的 vtable 布局——多个 vptr 与 thunk

多继承引入了一个新问题：派生类对象包含多个基类子对象，每个基类子对象可能需要自己的 vptr。

```cpp
class Base1 {
public:
    virtual void f() { std::cout << "Base1::f\n"; }
    virtual void g() { std::cout << "Base1::g\n"; }
};

class Base2 {
public:
    virtual void h() { std::cout << "Base2::h\n"; }
    virtual void i() { std::cout << "Base2::i\n"; }
};

class Derived : public Base1, public Base2 {
public:
    void f() override { std::cout << "Derived::f\n"; }  // 覆写 Base1::f
    void h() override { std::cout << "Derived::h\n"; }  // 覆写 Base2::h
    virtual void k() { std::cout << "Derived::k\n"; }   // 新增
};
```

多继承下的对象布局：

```
Derived 对象:
┌──────────────────┐
│  vptr → vtable1  │ ← Base1 子对象的 vptr
├──────────────────┤
│  Base1 成员       │
├──────────────────┤
│  vptr → vtable2  │ ← Base2 子对象的 vptr
├──────────────────┤
│  Base2 成员       │
├──────────────────┤
│  Derived 新增成员  │
└──────────────────┘
```

#### 为什么需要 thunk？

当通过 `Base2*` 指针调用 `h()` 时：

```cpp
Derived d;
Base2* pb2 = &d;   // pb2 指向 Derived 对象中的 Base2 子对象部分
                   // pb2 的值 ≠ &d！指针发生了偏移
pb2->h();          // 应该调用 Derived::h()
```

`pb2->h()` 的流程：

1. 通过 `pb2` 找到 `vtable2`（Base2 子对象的 vptr 指向的 vtable）
2. `vtable2` 中 `h()` 对应的 slot……指向谁？

这里是关键。如果 slot 直接指向 `Derived::h()`，那么 `Derived::h()` 收到的 `this` 指针值是 `pb2`（偏移后的地址），但 `Derived::h()` 期望的 `this` 是整个 `Derived` 对象的起始地址。**指针不匹配。**

**thunk 就是解决这个偏移问题的**：它是一个小的调整代码：

```
vtable2 中 h() 的 slot → thunk → Derived::h()
                                  └── 将 this 指针从 Base2 子对象偏移量调整回 Derived 对象起始地址
```

thunk 做的事就是 `this -= offset_of_Base2_in_Derived;` 然后跳转到真正的 `Derived::h()`。

```
Derived 的 vtable2（Base2 子对象用）:
┌──────────────────────────────┐
│ slot 0: thunk → Derived::h  │ ← 有偏移调整
│ slot 1: &Base2::i           │ ← 未覆写，无需 thunk
│ slot 2: &Derived::k         │ ← 新增（通过 second base 的 vtable 暴露）
└──────────────────────────────┘
```

**多继承 vtable 总结**：
- 有 N 个基类就有 N 个 vptr（因此 N 个 vtable）
- 覆写"非第一个基类"的虚函数时，需要用 thunk 调整 `this` 指针
- 新增的虚函数通常附加到第一个基类的 vtable 中（也可能出现在其他 vtable 中）

### 4.4 虚函数调用的性能代价

```
直接调用（编译期确定）：        虚函数调用：
call 0x123456                  mov rax, [rdi]        ; 取 vptr
                               mov rax, [rax + N*8]  ; 取 vtable[N]
                               call rax              ; 间接调用
```

- 多一次指针解引用（vptr → vtable）
- 间接跳转（`call rax` 比 `call imm` 更难预测，可能增加 CPU 分支预测失败）
- 无法内联（编译器不知道运行时会调哪个版本）

在 AI Infra 的热路径上（比如 CUDA kernel 启动、tensor 操作），虚函数的这几次间接跳转有时就需要避免，改用模板实现静态多态（CRTP）——见 [06 模板/SFINAE/CRTP](./06-templates-sfinae-crtp.md)。

---

## 5. `override` 和 `final`

```cpp
class Base {
public:
    virtual void f(int) const;
    virtual void g();
};

class Derived : public Base {
    void f(int) override;        // ✅ 正确覆写
    // void f(double) override;  // ❌ 编译错误！Base 没有 virtual void f(double)，防止拼写错误
    void g() final;              // ✅ 覆写，且禁止子类再覆写 g()
};

class MoreDerived : public Derived {
    // void g() override;  // ❌ 编译错误！g() 是 final
};
```

**永远用 `override`**——当你以为在覆写但实际参数/const 稍有不同时（这是无声的 bug），编译器会直接报错。

---

## 6. 虚函数与默认参数——危险组合

```cpp
class Base {
public:
    virtual void print(int level = 1) { std::cout << "Base: " << level << "\n"; }
};

class Derived : public Base {
public:
    void print(int level = 2) override { std::cout << "Derived: " << level << "\n"; }
};

Derived d;
Base* p = &d;
p->print();  // 输出 "Derived: 1"
             // 函数是动态绑定（调用了 Derived::print）
             // 默认参数是静态绑定（用了 Base 的默认值 1）！
```

**原因**：默认参数在调用点由编译器根据静态类型填入，不经过虚函数表。结论：**虚函数不要用默认参数**。

---

## 小结

| 概念 | 一句话 |
|------|--------|
| 虚函数 | 通过 vptr → vtable 实现运行期动态绑定 |
| 纯虚函数 `= 0` | 强制派生类覆写，包含纯虚函数的类是抽象类 |
| 虚析构 | 基类析构必须 virtual，否则 `delete` 派生类对象时不调派生类析构 |
| vtable | 每个有虚函数的类有一张函数指针数组 |
| vptr | 每个对象有一个指针指向其类的 vtable（8 字节开销） |
| 单继承 | 一个 vptr，派生类覆写的函数替换 vtable 对应 slot |
| 多继承 | N 个基类 → N 个 vptr + N 个 vtable，thunk 调整 `this` |
| `override` | 编译期检查是否真正覆写了基类虚函数 |
| `final` | 禁止进一步覆写 |

下一篇 [05 菱形继承与虚继承](./05-diamond-virtual-inheritance.md)。
