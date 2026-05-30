# 01 — 单例模式（Singleton）

> 关联篇目：[第二章 09 并发基础](../02-cpp11-17-features/09-concurrency-basics.md)

---

## 1. 单例模式概述

确保一个类**只有一个实例**，并提供全局访问点。

**适用场景**：配置管理器、日志系统、数据库连接池、线程池。

---

## 2. 五种 C++ 实现

### 2.1 懒汉式——第一次使用时创建（非线程安全）

```cpp
class Singleton {
    static Singleton* instance_;
    Singleton() = default;
public:
    static Singleton* getInstance() {
        if (!instance_)
            instance_ = new Singleton();
        return instance_;
    }
};
// ❌ 多线程下可能创建多个实例
```

### 2.2 懒汉式——加锁（线程安全，但重开销）

```cpp
class Singleton {
    static Singleton* instance_;
    static std::mutex mtx_;
public:
    static Singleton* getInstance() {
        std::lock_guard lock(mtx_);
        if (!instance_)
            instance_ = new Singleton();
        return instance_;
    }
};
// ✅ 线程安全，但每次调用都加锁（性能差）
```

### 2.3 双重检查锁定（DCLP）——C++11 前的不归路

```cpp
// ❌ C++11 之前 DCLP 有内存序问题——不要用！
// C++11 之后可以用 atomic + acquire/release 实现，但没必要
```

### 2.4 Meyers' Singleton（C++11 最佳方案）

```cpp
class Singleton {
    Singleton() = default;
public:
    static Singleton& getInstance() {
        static Singleton instance;  // C++11 保证线程安全！
        return instance;
    }
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;
};
```

**为什么是最佳？**
- C++11 起**保证 static 局部变量初始化是线程安全的**（编译器自动插入锁）
- 代码极简
- 自动析构（程序退出时销毁）
- 无需手动管理内存

### 2.5 饿汉式——程序启动时创建

```cpp
class Singleton {
    static Singleton instance_;
public:
    static Singleton& getInstance() { return instance_; }
};
Singleton Singleton::instance_;  // main 之前初始化
// ✅ 天然线程安全
// ⚠️ 但无法控制初始化顺序（多个饿汉单例之间）
```

---

## 3. 单例的销毁

### 3.1 自动析构（Meyers' Singleton 默认）

```cpp
// static 局部变量在程序退出时自动调用析构函数
// 如果析构顺序有依赖（A 析构时调用了 B，但 B 已析构）→ 可能问题
```

### 3.2 `atexit` 注册

```cpp
class Singleton {
    static Singleton* instance_;
public:
    static Singleton* getInstance() {
        if (!instance_) {
            instance_ = new Singleton();
            std::atexit([] { delete instance_; instance_ = nullptr; });
        }
        return instance_;
    }
};
```

### 3.3 `std::call_once` + `unique_ptr`

```cpp
class Singleton {
    static std::unique_ptr<Singleton> instance_;
    static std::once_flag flag_;
public:
    static Singleton& getInstance() {
        std::call_once(flag_, [] {
            instance_.reset(new Singleton());
        });
        return *instance_;
    }
};
```

---

## 4. 单例 vs 全局变量 vs 依赖注入

| | 单例 | 全局变量 | 依赖注入 |
|------|:---:|:---:|:---:|
| 唯一性保证 | ✅ | ❌ | 由容器保证 |
| 可测试性 | 差（难 mock） | 差 | **好** |
| 耦合 | 紧（调用方直接依赖单例类） | 紧 | 松 |
| 使用 | 简单 | 简单 | 需要 DI 框架 |

**现代 C++ 建议**：优先考虑**依赖注入**而非单例。单例适合真正需要全局唯一的场景（如日志系统）。

---

## 5. 模板化单例基类

```cpp
template<typename T>
class Singleton {
public:
    static T& getInstance() {
        static T instance;
        return instance;
    }
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;
protected:
    Singleton() = default;
};

// 使用
class Logger : public Singleton<Logger> {
    friend class Singleton<Logger>;  // 允许基类调用私有构造
    Logger() = default;
public:
    void log(const std::string& msg) { /* ... */ }
};

Logger::getInstance().log("hello");
```

---

## 小结

| 实现 | 线程安全 | 推荐 |
|------|:---:|:---:|
| 懒汉（无锁） | ❌ | ❌ |
| 懒汉 + 互斥锁 | ✅ | 可，但每次加锁 |
| Meyers' (static 局部变量) | ✅ (C++11) | ❤️ **首选** |
| 饿汉 | ✅ | 可，注意初始化顺序 |
| `call_once` | ✅ | 灵活版本 |

下一篇 [06 工厂模式](./06-factory.md)。
