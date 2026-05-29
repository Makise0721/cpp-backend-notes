# 05 — 模板方法模式与责任链模式

> 关联篇目：[04 Pimpl/适配器/装饰器](./04-pimpl-adapter-decorator.md) | [第一章 虚函数](../01-cpp-basics/04-virtual-vtable.md)

---

## 1. 模板方法模式——基类定义骨架，子类填充细节

**定义一个操作的算法骨架，将某些步骤延迟到子类实现。**

### 1.1 经典实现（虚函数）

```cpp
class DataProcessor {
public:
    void process() {                         // 模板方法（固定流程）
        load();
        transform();
        validate();
        save();
    }
    virtual ~DataProcessor() = default;
protected:
    virtual void load()      { /* 默认实现，子类可选覆写 */ }
    virtual void transform() = 0;             // 纯虚：子类必须实现
    virtual void validate()  { /* 默认空 */ }
    virtual void save()      = 0;             // 纯虚
};

class CSVProcessor : public DataProcessor {
protected:
    void load() override      { /* 从 CSV 文件加载 */ }
    void transform() override { /* 按 CSV 格式解析 */ }
    void save() override      { /* 保存为 CSV */ }
};

class JSONProcessor : public DataProcessor {
protected:
    void transform() override { /* 按 JSON 格式解析 */ }
    void save() override      { /* 保存为 JSON */ }
};

CSVProcessor p;
p.process();  // 执行 load → transform → validate → save（流程固定）
```

### 1.2 非虚函数接口（NVI）——C++ 惯用法

```cpp
class DataProcessor {
public:
    void process() {          // 公开的非虚接口（NVI）
        doLoad();
        doTransform();
        doSave();
    }
private:
    virtual void doLoad()      { }
    virtual void doTransform() = 0;
    virtual void doSave()      = 0;
};
// 公共接口 = 非虚，虚函数 = 私有
// 基类可以在 process() 前后加上锁、日志、检查等
```

### 1.3 CRTP 实现——编译期模板方法

```cpp
template<typename Derived>
class DataProcessor {
public:
    void process() {
        static_cast<Derived*>(this)->load();
        static_cast<Derived*>(this)->transform();
        static_cast<Derived*>(this)->save();
    }
    // 不提供默认实现 → 派生类如果不实现 → 编译错误
};

class CSVProcessor : public DataProcessor<CSVProcessor> {
    friend class DataProcessor<CSVProcessor>;  // 允许基类调用私有方法
    void load()      { }
    void transform() { }
    void save()      { }
};
// 优势：无虚函数开销，可内联
// 劣势：每个派生类是不同的类型，不能放在同一个容器中
```

---

## 2. 责任链模式——请求沿链路传递

### 2.1 经典实现

```cpp
class Handler {
protected:
    std::unique_ptr<Handler> next_;
public:
    Handler& setNext(std::unique_ptr<Handler> next) {
        next_ = std::move(next);
        return *next_;  // 支持链式
    }

    virtual void handle(int request) {
        if (next_)
            next_->handle(request);  // 传递给下一个
    }
    virtual ~Handler() = default;
};

class AuthHandler : public Handler {
public:
    void handle(int request) override {
        if (request & AUTH_REQUIRED) {
            // 做认证...
            if (!authenticated) return;  // 终止链路
        }
        Handler::handle(request);  // 传给下一个
    }
};

class LogHandler : public Handler {
public:
    void handle(int request) override {
        std::cout << "Processing: " << request << "\n";
        Handler::handle(request);
    }
};

class CoreHandler : public Handler {
public:
    void handle(int request) override {
        // 真正处理请求
    }
};

// 组装链路
auto core = std::make_unique<CoreHandler>();
auto log  = std::make_unique<LogHandler>();
auto auth = std::make_unique<AuthHandler>();

auth->setNext(std::move(log))->setNext(std::move(core));
// auth → log → core

auth->handle(someRequest);
```

### 2.2 现代 C++——函数链

```cpp
using Middleware = std::function<bool(int& request)>;

class Pipeline {
    std::vector<Middleware> middlewares_;
    std::function<void(int)> handler_;
public:
    void addMiddleware(Middleware m) { middlewares_.push_back(std::move(m)); }
    void setHandler(std::function<void(int)> h) { handler_ = std::move(h); }

    void process(int request) {
        for (auto& m : middlewares_)
            if (!m(request)) return;  // 中间件返回 false → 中断
        handler_(request);
    }
};

Pipeline pipeline;
pipeline.addMiddleware([](int& req) {
    // 认证
    return true;  // 继续
});
pipeline.addMiddleware([](int& req) {
    std::cout << "Log: " << req << "\n";
    return true;
});
pipeline.setHandler([](int req) { /* 核心处理 */ });
pipeline.process(42);
```

### 2.3 实际应用

- **Web 中间件**：认证 → 权限 → 日志 → 限流 → 核心处理
- **GUI 事件传递**：子控件未处理 → 传给父控件
- **日志级别过滤**：Debug → Info → Warning → Error（每层决定是否记录）

---

## 3. 模板方法 vs 策略——对比

| | 模板方法 | 策略 |
|------|------|------|
| 关系 | 子类 IS-A 基类 | 上下文 HAS-A 策略 |
| 变化点 | 算法中的某些步骤 | **整个算法**可替换 |
| 复用 | 基类复用骨架 | 算法独立复用 |
| 耦合 | 紧（继承） | 松（组合） |
| 粒度 | 细（步骤级别） | 粗（算法级别） |

---

## 小结

| 模式 | 核心 | C++ 实现 |
|------|------|----------|
| 模板方法 | 基类定义骨架，子类实现步骤 | 虚函数 / NVI 惯用法 / CRTP |
| 责任链 | 请求沿链路传递直到被处理 | `unique_ptr` 链表 / `std::function` 管道 |

---

## 第十章小结

| # | 篇目 | 核心 |
|---|------|------|
| 01 | 单例模式 | Meyers' Singleton (C++11 static 局部变量) |
| 02 | 工厂模式 | 简单工厂/工厂方法/抽象工厂 + 注册式工厂 |
| 03 | 策略与观察者 | `std::function` 替代虚函数 / 信号槽 |
| 04 | Pimpl/适配器/装饰器 | 编译防火墙/STL 适配器/嵌套包装 |
| 05 | 模板方法/责任链 | NVI 惯用法/CRTP/函数管道 |
