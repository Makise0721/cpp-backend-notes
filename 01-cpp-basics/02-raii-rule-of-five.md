# 02 — RAII 思想与三五零法则：C++ 资源管理的基石

> 关联篇目：[01 OOP 基础](./01-oop-basics.md) | [07 STL 容器底层原理](./07-stl-containers-deep-dive.md) | [09 异常安全](./09-namespace-exception-noexcept.md)

---

## 1. RAII——C++ 最重要的设计哲学

**RAII** = **R**esource **A**cquisition **I**s **I**nitialization（资源获取即初始化）。

这个名字其实不够直观。更好的理解是：**把资源的生命周期绑定到对象的生命周期上**。构造时获取资源，析构时释放资源——无论函数如何退出（正常返回、异常、提前 return），析构函数都会自动调用。

### 1.1 没有 RAII 的世界有多痛苦

```cpp
// C 风格的资源管理噩梦
void processFile(const char* path) {
    FILE* f = fopen(path, "r");       // 获取资源
    if (!f) return;                    // 提前退出 1

    char* buf = (char*)malloc(1024);
    if (!buf) {
        fclose(f);                     // 手动清理
        return;                        // 提前退出 2
    }

    if (someError) {
        free(buf);                     // 手动清理
        fclose(f);                     // 手动清理
        return;                        // 提前退出 3
    }

    //       如果这里抛异常呢？fclose 和 free 永远不会被调用！

    free(buf);
    fclose(f);                         // 正常退出也要手动清理
}
```

每个退出点都要手动释放资源，而且**异常路径上的清理完全不可靠**。

### 1.2 RAII 的优雅解法

```cpp
#include <fstream>
#include <memory>

void processFile(const char* path) {
    std::ifstream f(path);              // 构造 = 打开文件
    if (!f) return;                     // 提前退出 → 析构自动关闭

    std::unique_ptr<char[]> buf = std::make_unique<char[]>(1024);
    //         构造 = 分配内存

    if (someError) return;              // 提前退出 → 析构自动释放

    // 异常 → 栈展开 → 析构自动清理
    // 正常结束 → 析构自动清理
}
// f 和 buf 离开作用域，析构函数自动执行——无论怎样退出
```

**RAII 的本质就是"让编译器替你记着清理"**，你只需要在析构函数里写一次清理代码。

---

## 2. RAII 的经典应用场景

### 2.1 互斥锁

```cpp
std::mutex mtx;

void bad() {
    mtx.lock();
    // 如果这里抛异常，锁永远不会释放 → 死锁
    mtx.unlock();
}

void good() {
    std::lock_guard<std::mutex> guard(mtx);  // 构造 = 加锁
    // 临界区
}   // 析构 = 解锁——异常安全
```

### 2.2 数据库连接 / 事务

```cpp
class Transaction {
    DBConnection& conn_;
public:
    Transaction(DBConnection& c) : conn_(c) { conn_.execute("BEGIN"); }
    ~Transaction() {
        if (!committed_) conn_.execute("ROLLBACK");  // 析构自动回滚
    }
    void commit() { conn_.execute("COMMIT"); committed_ = true; }
private:
    bool committed_ = false;
};
```

### 2.3 GPU 内存管理（AI Infra 关注）

```cpp
class CudaBuffer {
    float* d_data_;  // device pointer
    size_t size_;
public:
    CudaBuffer(size_t n) : size_(n) {
        cudaMalloc(&d_data_, n * sizeof(float));   // 构造 = 分配 GPU 内存
    }
    ~CudaBuffer() {
        if (d_data_) cudaFree(d_data_);            // 析构 = 释放 GPU 内存
    }
    // 禁止拷贝（GPU 内存拷贝很贵），允许移动
    CudaBuffer(const CudaBuffer&) = delete;
    CudaBuffer& operator=(const CudaBuffer&) = delete;
    CudaBuffer(CudaBuffer&& other) noexcept
        : d_data_(other.d_data_), size_(other.size_) {
        other.d_data_ = nullptr;
        other.size_ = 0;
    }
};
```

> 在 AI 推理/训练系统中，RAII 是管理 GPU 显存、CUDA stream、NCCL communicator 的标准范式。

---

## 3. Rule of Three / Five / Zero

这是 C++ 社区总结出的三个黄金法则，帮你正确管理资源。

### 3.1 Rule of Three（C++98/03 时代）

**如果你需要自定义析构函数、拷贝构造函数或拷贝赋值运算符中的任何一个，你几乎肯定需要自定义全部三个。**

为什么？因为需要自定义析构函数通常意味着你在管理一个资源（堆内存、文件句柄、锁等）。此时默认的拷贝构造和拷贝赋值只做浅拷贝，必然出错。

```cpp
class RuleOfThree {
    char* data_;
public:
    // 1. 析构函数
    ~RuleOfThree() { delete[] data_; }

    // 2. 拷贝构造函数
    RuleOfThree(const RuleOfThree& other) {
        data_ = new char[strlen(other.data_) + 1];
        strcpy(data_, other.data_);
    }

    // 3. 拷贝赋值运算符
    RuleOfThree& operator=(const RuleOfThree& other) {
        if (this != &other) {
            delete[] data_;                      // 先释放旧资源
            data_ = new char[strlen(other.data_) + 1];  // 再分配新资源
            strcpy(data_, other.data_);
        }
        return *this;
    }
};
```

### 3.2 Rule of Five（C++11 起）

**在 Rule of Three 的基础上，加上移动构造函数和移动赋值运算符。**

C++11 引入了移动语义。如果你管理资源，移动操作可以显著提升性能（比如函数返回大对象时避免不必要的拷贝）。编译器不会自动生成移动操作如果你定义了拷贝操作。

```cpp
class RuleOfFive {
    char* data_;
public:
    ~RuleOfFive() { delete[] data_; }

    // 拷贝
    RuleOfFive(const RuleOfFive& other) {
        data_ = new char[strlen(other.data_) + 1];
        strcpy(data_, other.data_);
    }
    RuleOfFive& operator=(const RuleOfFive& other) {
        if (this != &other) {
            delete[] data_;
            data_ = new char[strlen(other.data_) + 1];
            strcpy(data_, other.data_);
        }
        return *this;
    }

    // 移动（C++11 新增）
    RuleOfFive(RuleOfFive&& other) noexcept
        : data_(other.data_) {           // 直接"偷"走指针
        other.data_ = nullptr;           // 源对象置空，防止 double-free
    }
    RuleOfFive& operator=(RuleOfFive&& other) noexcept {
        if (this != &other) {
            delete[] data_;              // 释放自己的旧资源
            data_ = other.data_;         // 偷走别人的资源
            other.data_ = nullptr;
        }
        return *this;
    }
};
```

**为什么要 `noexcept`？** 移动操作标记 `noexcept` 有两个好处：

1. `std::vector` 扩容时，如果元素的移动构造是 `noexcept` 的，`vector` 才会用移动而不是拷贝——性能差距巨大
2. 移动操作本就不该分配资源（只是转移所有权），抛异常的概率几乎为零

### 3.3 Rule of Zero（现代 C++ 推荐）

**不要自己写任何析构/拷贝/移动函数——让编译器默认生成。**

前提是你的成员变量都是 RAII 类型：`std::string`、`std::vector`、`std::unique_ptr` 等。这些标准库类型已经正确实现了三五法则，编译器为你合成的默认版本就是正确的。

```cpp
// Rule of Zero：一个特殊成员函数都没写——完全依赖标准库
class RuleOfZero {
    std::string name_;
    std::vector<int> data_;
    std::unique_ptr<Config> config_;  // unique_ptr 不可拷贝，所以 RuleOfZero 也不可拷贝
public:
    RuleOfZero(std::string name) : name_(std::move(name)) {}
    // 编译器生成的：
    //   - 析构：调用每个成员的析构 ✓
    //   - 拷贝构造：调用每个成员的拷贝构造 ✓
    //   - 拷贝赋值：调用每个成员的拷贝赋值 ✓
    //   - 移动构造：调用每个成员的移动构造 ✓（但因为 unique_ptr 不可拷贝，拷贝操作被删除）
    //   - 移动赋值：调用每个成员的移动赋值 ✓
};
```

**Rule of Zero 是现代 C++ 的终极目标**：你只关心业务逻辑，资源管理交给标准库。

### 3.4 三法则速查表

| 场景 | 适用法则 | 典型写法 |
|------|----------|----------|
| 类不管理资源（纯数据） | Rule of Zero | 什么都不写 |
| 类管理一个资源（如 `FILE*`） | Rule of Five | 写 5 个 + 禁用拷贝或全部实现 |
| 类管理资源 + 禁止拷贝（如 `mutex`） | Rule of Five | 5 个全写，拷贝操作 `= delete` |
| C++98 遗留代码 | Rule of Three | 写 3 个，编译器不生成移动 |

---

## 4. 移动语义深入——不仅仅是"快一点"

### 4.1 为什么需要移动？

拷贝的代价有时大得不可接受：

```cpp
std::vector<std::string> buildNames() {
    std::vector<std::string> result;
    result.push_back("Alice");
    result.push_back("Bob");
    result.push_back("Charlie");
    return result;  // C++11 起：移动返回，不是拷贝
}

std::vector<std::string> names = buildNames();
// 如果拷贝：所有 string 都要复制一遍 → O(n)
// 如果移动：只复制三个指针（vector 的 begin/end/capacity）→ O(1)
```

### 4.2 `std::move` 的真实面目

`std::move` 不移动任何东西——它只是做一个**强制类型转换**，把左值变成右值引用：

```cpp
template<typename T>
typename std::remove_reference<T>::type&& move(T&& t) noexcept {
    return static_cast<typename std::remove_reference<T>::type&&>(t);
}
```

它的作用是**标记"这个对象我以后不用了，你可以偷它的资源"**。真正的移动发生在接收方（移动构造函数/移动赋值运算符）里。

### 4.3 移动后的对象

标准保证：被移动的对象处于"**有效但未指定**（valid but unspecified）"状态。你可以：
- 销毁它 ✅
- 重新赋值 ✅
- 调用没有前置条件的成员函数（如 `.empty()` `.size()`）✅

不应该：
- 假设它保持原值 ❌
- 解引用被移动的 `unique_ptr` ❌

```cpp
std::string s = "hello";
std::string t = std::move(s);  // s 被"掏空"
// s 现在是什么？标准说"valid but unspecified"
// 实践中通常是空字符串，但不要依赖这一点
std::cout << s;         // 可能输出空，但不要依赖
s = "world";            // ✅ 重新赋值，安全
```

---

## 5. 拷贝省略（Copy Elision）与 RVO

编译器比你聪明——即使你写了拷贝构造，它也可能不调：

```cpp
Widget createWidget() {
    return Widget(42);  // NRVO / RVO：直接在调用方的栈上构造
}

Widget w = createWidget();  // 拷贝/移动构造都不调——编译器省略了
```

C++17 起，某些场景的拷贝省略是**强制的**（mandatory），不再是可选的优化。这意味着返回局部对象时，你不需要担心拷贝开销。

但这**不意味着你可以不写移动构造**——不是所有场景都能省略拷贝。比如：

```cpp
Widget w;
w = createWidget();  // 赋值，不是初始化 → 需要移动赋值
```

---

## 小结

| 概念 | 一句话 |
|------|--------|
| RAII | 资源绑定对象生命周期，构造获取，析构释放 |
| Rule of Three | 管理资源 → 必须写析构+拷贝构造+拷贝赋值 |
| Rule of Five | Rule of Three + 移动构造 + 移动赋值 |
| Rule of Zero | 用标准库 RAII 类型 → 什么都不写 |
| `std::move` | 只是一个类型转换，标记"可被偷" |
| 移动后状态 | 有效但未指定，可以重新赋值或销毁 |

下一篇 [03 静态成员与运算符重载](./03-static-operator-overload.md)。
