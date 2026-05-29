# 05 — 菱形继承、虚继承与虚基类

> 关联篇目：[04 虚函数与 vtable](./04-virtual-vtable.md) | [10 对象内存模型](./10-object-memory-model.md)

---

## 1. 菱形继承——多继承的经典陷阱

考虑一个经典的 OOP 层次：

```
        Person
       /      \
   Student   Teacher
       \      /
      TeachingAssistant
```

用代码表达：

```cpp
class Person {
public:
    std::string name_;
    Person(std::string n) : name_(n) {}
};

class Student : public Person {
public:
    Student(std::string n) : Person(n) {}
    void study() {}
};

class Teacher : public Person {
public:
    Teacher(std::string n) : Person(n) {}
    void teach() {}
};

class TA : public Student, public Teacher {
public:
    TA(std::string n) : Student(n), Teacher(n) {}
};

TA ta("Alice");
```

### 1.1 问题：数据冗余与二义性

`TA` 对象中有**两份 `Person` 子对象**——一份通过 `Student`，一份通过 `Teacher`。

```
TA 对象的内存布局（非虚继承）:

┌─────────────────────┐
│  Student 子对象       │
│  ┌───────────────┐  │
│  │ Person::name_ │  │ ← 第一份 name_
│  └───────────────┘  │
│  Student 的成员...   │
├─────────────────────┤
│  Teacher 子对象       │
│  ┌───────────────┐  │
│  │ Person::name_ │  │ ← 第二份 name_！！！重复了
│  └───────────────┘  │
│  Teacher 的成员...   │
├─────────────────────┤
│  TA 自己的成员       │
└─────────────────────┘
```

直接后果：

```cpp
TA ta("Alice");
ta.name_;         // ❌ 编译错误！歧义——用哪个 name_？
ta.Student::name_;// ✅ 显式指定
ta.Teacher::name_;// ✅ 显式指定（但它俩不是一个变量！）

Person& p = ta;   // ❌ 歧义！转换成哪个 Person？
```

---

## 2. 虚继承——解决问题的钥匙

**虚继承**保证：无论派生路径有多少条，被虚继承的基类在最终对象中只存在一份。

```cpp
class Student : public virtual Person {    // ← virtual！
public:
    Student(std::string n) : Person(n) {}
};

class Teacher : public virtual Person {    // ← virtual！
public:
    Teacher(std::string n) : Person(n) {}
};

class TA : public Student, public Teacher {
public:
    TA(std::string n) : Person(n), Student(n), Teacher(n) {}
    // ⚠️ 注意：TA 必须直接调用 Person 的构造函数！
    // 虚基类的构造由最派生类负责，不是中间类
};
```

### 2.1 虚继承下的对象布局

```
TA 对象的内存布局（虚继承）:

┌──────────────────────┐
│ Student 子对象         │
│ ├ vptr → vtable      │ ← 包含虚基类偏移信息
│ └ Student 成员        │
├──────────────────────┤
│ Teacher 子对象         │
│ ├ vptr → vtable      │ ← 包含虚基类偏移信息
│ └ Teacher 成员        │
├──────────────────────┤
│ TA 成员               │
├──────────────────────┤
│ Person 子对象（唯一）    │ ← 只有一份！放在对象末尾
│ └ name_               │
└──────────────────────┘
```

**虚基类指针（vbase pointer / vbase offset）**：编译器在 vtable 中记录了从子对象到虚基类的偏移量。当通过 `Student*` 访问 `name_` 时，编译器的做法：

```cpp
// 伪代码：通过虚基类访问成员
void accessName(Student* s) {
    // 1. 从 vtable 查虚基类 Person 的偏移
    int offset = s->vptr->vbase_offset_Person;
    // 2. 计算 Person 子对象的实际地址
    Person* p = reinterpret_cast<Person*>(reinterpret_cast<char*>(s) + offset);
    // 3. 访问成员
    return p->name_;
}
```

### 2.2 虚基类构造函数——由最派生类负责

```cpp
class TA : public Student, public Teacher {
public:
    TA(std::string n)
        : Person(n)      // TA 直接初始化虚基类 Person
        , Student(n)     // Student(n) 中初始化 Person 的操作被忽略
        , Teacher(n)     // Teacher(n) 中初始化 Person 的操作也被忽略
    {}
};
```

**规则**：虚基类的构造函数只由**最派生类（most derived class）**调用一次，中间类的构造函数中对虚基类的初始化被跳过。

这意味着如果 `TA` 忘记直接初始化 `Person`，`Person` 会用默认构造函数。如果 `Person` 没有默认构造——编译错误。

---

## 3. 虚继承与虚函数同时存在

当一个类同时涉及虚继承和虚函数，vtable 的组织更复杂：

```cpp
class Person {
public:
    virtual ~Person() {}
    std::string name_;
};

class Student : public virtual Person {
public:
    virtual void study() = 0;
};
```

`Student` 的 vtable 中不仅包括虚函数指针，还包括：
- 虚基类 `Person` 的偏移量（vbase offset）
- `Person` 中虚函数通过 `Student` 的路径调用时需要的 thunk

---

## 4. 虚继承的代价

| 代价 | 说明 |
|------|------|
| **空间开销** | vptr（或 vbase ptr）+ vtable 中偏移信息 |
| **时间开销** | 访问虚基类成员多一次间接跳转（查偏移、计算地址） |
| **复杂度** | 构造函数规则特殊、类型转换路径复杂 |
| **不可平凡的拷贝** | 编译器不能简单地 memcpy，需要正确处理虚基类 |

### 性能对比（伪汇编）

```
// 非虚继承下的 name_ 访问：
// this + fixed_offset → 直接，编译期已知

// 虚继承下的 name_ 访问：
// this → vptr → vbase_offset → this + vbase_offset → 间接，运行期计算
```

**教训**：虚继承只在设计层面确实需要"共享基类"时才用（如 `std::basic_ios` → `std::basic_istream` / `std::basic_ostream` → `std::basic_iostream`）。不要为了"以防万一"加 virtual。

---

## 5. 实战：iostream 的菱形继承

C++ 标准库的 `iostream` 就是菱形继承 + 虚继承的经典应用：

```cpp
// 简化版标准库层次
template<typename CharT>
class basic_ios { /* 状态位、缓冲区指针等 */ };

template<typename CharT>
class basic_istream : public virtual basic_ios<CharT> { /* 输入 */ };

template<typename CharT>
class basic_ostream : public virtual basic_ios<CharT> { /* 输出 */ };

template<typename CharT>
class basic_iostream : public basic_istream<CharT>,
                       public basic_ostream<CharT> { /* 输入+输出 */ };

// std::cin 是 basic_istream<char>
// std::cout 是 basic_ostream<char>
// std::fstream 是 basic_iostream<char>（通常）
```

这里用虚继承是正确的：`basic_ios` 中的状态位（`rdstate()`、`exceptions()` 等）在 `basic_iostream` 中只需要一份——读写共享同一个状态。

---

## 6. 菱形继承 vs 组合——什么时候该避免继承？

```cpp
// 继承方案（表达"TA 既是 Student 也是 Teacher"）
class TA : public Student, public Teacher { ... };

// 组合方案（表达"TA 拥有 Student 和 Teacher 的能力"）
class TA {
    std::unique_ptr<StudentRole> studentRole_;
    std::unique_ptr<TeacherRole> teacherRole_;
    // 通过委托来复用行为
};
```

**组合优先于继承**——这是 GoF 设计模式的核心原则。菱形继承的出现通常暗示"也许该用组合而不是多继承"。

---

## 小结

| 概念 | 一句话 |
|------|--------|
| 菱形继承 | 多层多继承导致基类子对象重复 |
| 虚继承 `virtual` | 保证虚基类只出现一次 |
| 虚基类构造 | 由最派生类负责，中间类的初始化被忽略 |
| vbase offset | vtable 中的偏移信息，实现虚基类成员的间接访问 |
| 代价 | 空间+1 vptr，时间+1 间接查找，构造函数规则特殊 |
| 组合优先 | 菱形继承前先考虑：能不能用组合？ |

下一篇 [06 模板、SFINAE 与 CRTP](./06-templates-sfinae-crtp.md)。
