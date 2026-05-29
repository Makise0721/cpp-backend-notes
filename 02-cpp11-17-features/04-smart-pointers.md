# 04 — 智能指针全面解析：unique_ptr、shared_ptr、weak_ptr

> 关联篇目：[第一章 02 RAII 与三五零法则](../01-cpp-basics/02-raii-rule-of-five.md) | [03 值类别与完美转发](./03-value-categories-move-forward.md)

---

## 1. 智能指针总览——告别 `new` / `delete`

| 智能指针 | 所有权 | 拷贝 | 典型用途 |
|----------|--------|------|----------|
| `unique_ptr` | 独占 | ❌（只可移动） | 工厂函数返回值、容器中的对象、PIMPL |
| `shared_ptr` | 共享 | ✅（引用计数） | 共享所有权、异步回调、多线程访问 |
| `weak_ptr` | 弱引用 | ✅ | 打破循环引用、缓存观察者 |
| `auto_ptr` | — | — | C++11 废弃，C++17 移除，**永远不要用** |

---

## 2. `unique_ptr`——独占所有权

### 2.1 基本用法

```cpp
// 创建
auto p1 = std::make_unique<int>(42);          // ✅ 推荐（C++14）
std::unique_ptr<int> p2(new int(42));         // 可以，但不推荐（多一次写类型）

auto p3 = std::make_unique<std::string>(10, 'x');  // "xxxxxxxxxx"

// 所有权转移
auto p4 = std::move(p1);   // p4 拥有资源，p1 == nullptr
// auto p5 = p4;           // ❌ 编译错误！unique_ptr 不可拷贝

// 释放所有权
int* raw = p4.release();   // p4 放弃所有权，返回裸指针——现在你需要手动 delete！
delete raw;

// 重置
p3.reset();                // 释放 p3 持有的对象
p3.reset(new std::string("new")); // 释放旧对象，接管新对象
```

### 2.2 自定义删除器

`unique_ptr` 的删除器是**模板参数的一部分**（和 `shared_ptr` 不同！），可以用函数指针、lambda 或仿函数：

```cpp
// 方式 1：函数指针（增加一个指针的大小 → 16 字节）
void myDeleter(FILE* f) { if (f) fclose(f); }
std::unique_ptr<FILE, decltype(&myDeleter)> f1(fopen("a.txt", "r"), myDeleter);

// 方式 2：无捕获 lambda（零开销！大小等于裸指针 → 8 字节）
auto f2 = std::unique_ptr<FILE, decltype([](FILE* f) { fclose(f); })>(
    fopen("b.txt", "r"), [](FILE* f) { fclose(f); });
// C++20 起可以更简洁

// 方式 3：std::function（开销最大，32+ 字节）
std::unique_ptr<FILE, std::function<void(FILE*)>> f3(
    fopen("c.txt", "r"), [](FILE* f) { fclose(f); });
```

| 删除器类型 | unique_ptr 大小（64位） | 原因 |
|-----------|----------------------|------|
| 默认 `std::default_delete` | 8 字节 | 空基类优化，无额外存储 |
| 无捕获 lambda | 8 字节 | 空基类优化 |
| 函数指针 | 16 字节 | 多存一个函数指针 |
| 有捕获 lambda / `std::function` | 32-40+ 字节 | 存储捕获变量 + 函数指针 |

### 2.3 数组版本

```cpp
auto arr = std::make_unique<int[]>(100);  // int[100]
arr[0] = 42;   // ✅ 支持 operator[]
// *arr;       // ❌ 不支持 operator*
```

---

## 3. `shared_ptr`——共享所有权 + 引用计数

### 3.1 基本用法

```cpp
auto sp1 = std::make_shared<int>(42);     // ✅ 推荐：一次分配
std::shared_ptr<int> sp2(new int(42));    // 可以，但两次分配（对象 + 控制块）

auto sp3 = sp1;       // 拷贝 → 引用计数 +1
auto sp4 = sp1;       // 引用计数 +1
std::cout << sp1.use_count();  // 3

sp3.reset();          // sp3 放弃引用 → 计数 -1
// 当 use_count 降到 0 → 对象被销毁 → 控制块被释放
```

### 3.2 控制块——shared_ptr 的灵魂

```
shared_ptr 对象 A:       shared_ptr 对象 B:
┌──────────────┐        ┌──────────────┐
│ ptr_to_obj   │──┐     │ ptr_to_obj   │──┐
│ ptr_to_ctrl  │──┼──┐  │ ptr_to_ctrl  │──┼──┐
└──────────────┘  │  │  └──────────────┘  │  │
                  │  │                    │  │
                  ▼  │                    │  │
          ┌─────────┐│                    │  │
          │  对象    ││                    │  │
          │ (int=42) ││                    │  │
          └─────────┘│                    │  │
                      │                    │  │
                      ▼                    ▼  ▼
               ┌─────────────────────────┐
               │       控制块              │
               │  use_count    = 2        │
               │  weak_count   = 0        │
               │  deleter                 │
               │  allocator               │
               └─────────────────────────┘
```

**控制块中存储**：引用计数（强引用）、弱引用计数、删除器、分配器。

### 3.3 `make_shared` vs `new`

```cpp
// new：两次分配
std::shared_ptr<int> sp1(new int(42));
// 1. new int(42)             → 分配对象内存
// 2. make_shared 控制块      → 分配控制块内存
// 两次堆分配 + 内存不连续 → 缓存不友好

// make_shared：一次分配
auto sp2 = std::make_shared<int>(42);
// 1. 分配一块连续内存 [控制块 | int(42)]
// 一次堆分配 + 内存连续 → 缓存友好 + 更快

// ⚠️ make_shared 的一个坑：weak_ptr 延迟释放
auto sp = std::make_shared<BigObject>();
std::weak_ptr<BigObject> wp = sp;
sp.reset();  // 对象析构，但内存不释放！weak_count > 0
// 只有当所有 weak_ptr 也销毁后，整块内存才释放
// 如果用 new：对象析构后只需析构控制块，对象内存立即释放
```

### 3.4 线程安全性

```cpp
auto sp = std::make_shared<int>(42);

// ✅ 引用计数的原子操作是线程安全的
std::thread t1([sp] { auto copy = sp; });  // use_count++ （原子）
std::thread t2([sp] { auto copy = sp; });  // use_count++ （原子）

// ❌ 对象本身的访问不是线程安全的
std::thread t3([&] { *sp = 100; });  // 数据竞争！
std::thread t4([&] { *sp = 200; });  // 数据竞争！

// 需要加锁保护对象
std::mutex mtx;
std::thread t5([&] {
    std::lock_guard lock(mtx);
    *sp = 100;
});
```

**关键**：`shared_ptr` 的控制块是线程安全的，但**它指向的对象不是**。

---

## 4. `weak_ptr`——打破循环引用

### 4.1 循环引用问题

```cpp
struct Node {
    std::shared_ptr<Node> next;
    std::shared_ptr<Node> prev;  // ← 循环引用！
    ~Node() { std::cout << "destroyed\n"; }
};

auto a = std::make_shared<Node>();
auto b = std::make_shared<Node>();
a->next = b;
b->prev = a;
// a 和 b 离开作用域 → use_count 各为 1（互相持有）→ 永远不会析构！内存泄漏！
```

### 4.2 weak_ptr 的解法

```cpp
struct Node {
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> prev;    // ← weak_ptr 不增加引用计数
    ~Node() { std::cout << "destroyed\n"; }
};

auto a = std::make_shared<Node>();
auto b = std::make_shared<Node>();
a->next = b;
b->prev = a;

// a 离开作用域 → a.use_count 从 2 降到 1（b->next 持有）
// b 离开作用域 → b.use_count 从 1 降到 0 → 析构 b
// b 析构 → b->prev（指向 a 的 weak_ptr）销毁 → a.use_count 从 1 降到 0 → 析构 a
// ✅ 全部正确释放
```

### 4.3 weak_ptr 的使用

```cpp
auto sp = std::make_shared<int>(42);
std::weak_ptr<int> wp = sp;

// 检查是否过期
if (wp.expired()) {
    // 所指向的对象已被销毁
}

// 获取 shared_ptr（临时提升引用计数，安全访问）
if (auto locked = wp.lock()) {  // locked 是 shared_ptr
    std::cout << *locked;       // 安全——locked 持有引用计数
} else {
    // 对象已销毁
}
```

**`wp.lock()` 是原子操作**——它要么返回一个有效的 `shared_ptr`（引用计数 +1），要么返回空。不会出现"检查通过但使用时已销毁"的 TOCTOU 问题。

---

## 5. 智能指针选择指南

```
需要所有权吗？
├─ 不需要 → 裸指针 / 引用 / std::optional<std::reference_wrapper<T>>
│
├─ 独占所有权
│   ├─ 单一所有者 → std::unique_ptr
│   └─ 需要自定义删除器 → std::unique_ptr<T, Deleter>
│
├─ 共享所有权
│   ├─ 多个所有者 → std::shared_ptr
│   ├─ 打破循环引用 → std::weak_ptr
│   └─ 缓存/观察者 → std::weak_ptr
│
└─ 传给 C API
    └─ p.get() 获取裸指针（不转移所有权！）
```

### 5.1 常见反模式

```cpp
// ❌ 永远不要这样
int* raw = new int(42);
std::shared_ptr<int> sp1(raw);
std::shared_ptr<int> sp2(raw);  // 两个独立的控制块！double free！

// ✅ 正确
auto sp1 = std::make_shared<int>(42);
auto sp2 = sp1;                 // 共享同一个控制块

// ❌ 不要从 this 裸指针创建 shared_ptr
class Bad {
public:
    std::shared_ptr<Bad> getShared() {
        return std::shared_ptr<Bad>(this);  // 和已有的 shared_ptr 冲突！
    }
};
// ✅ 用 enable_shared_from_this
class Good : public std::enable_shared_from_this<Good> {
public:
    std::shared_ptr<Good> getShared() {
        return shared_from_this();  // 安全
    }
};
```

---

## 小结

| 指针 | 所有权 | 关键操作 | 与删除器关系 |
|------|--------|----------|-------------|
| `unique_ptr` | 独占 | `std::move` 转移，`.release()` 放弃 | 删除器是类型一部分，lambda 版零开销 |
| `shared_ptr` | 共享 | 拷贝 → 计数 +1，`.use_count()` 查询 | 删除器在控制块中，不占对象大小 |
| `weak_ptr` | 弱引用 | `.lock()` 提升为 shared_ptr | 不参与所有权 |

下一篇 [05 现代类特性：nullptr / range-for / =default / =delete / enum class](./05-modern-class-features.md)。
