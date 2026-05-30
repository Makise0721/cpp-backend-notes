# 03 — 策略模式与观察者模式

> 关联篇目：[06 工厂模式](./06-factory.md) | [第二章 02 Lambda](../02-cpp11-17-features/02-lambda-deep-dive.md)

---

## 1. 策略模式（Strategy）——运行时切换算法

### 1.1 经典实现（虚函数）

```cpp
// 策略接口
class SortStrategy {
public:
    virtual void sort(std::vector<int>& data) = 0;
    virtual ~SortStrategy() = default;
};

// 具体策略
class QuickSort : public SortStrategy {
public: void sort(std::vector<int>& data) override { /* 快排 */ }
};
class MergeSort : public SortStrategy {
public: void sort(std::vector<int>& data) override { /* 归并 */ }
};

// 上下文
class DataProcessor {
    std::unique_ptr<SortStrategy> strategy_;
public:
    void setStrategy(std::unique_ptr<SortStrategy> s) { strategy_ = std::move(s); }
    void process(std::vector<int>& data) { strategy_->sort(data); }
};

DataProcessor dp;
dp.setStrategy(std::make_unique<QuickSort>());
dp.process(data);
// 切换到另一种策略
dp.setStrategy(std::make_unique<MergeSort>());
dp.process(data);
```

### 1.2 现代 C++ 实现——`std::function` 替代虚函数

```cpp
class DataProcessor {
    std::function<void(std::vector<int>&)> strategy_;
public:
    void setStrategy(std::function<void(std::vector<int>&)> s) { strategy_ = std::move(s); }
    void process(std::vector<int>& data) { strategy_(data); }
};

DataProcessor dp;
dp.setStrategy([](std::vector<int>& v) { std::sort(v.begin(), v.end()); });
dp.process(data);

// 动态切换
dp.setStrategy([](std::vector<int>& v) { std::stable_sort(v.begin(), v.end()); });
```

### 1.3 模板策略——编译期选择

```cpp
// 策略作为模板参数（编译期确定，零运行时开销）
template<typename SortStrategy>
class DataProcessor {
    SortStrategy strategy_;
public:
    void process(std::vector<int>& data) { strategy_.sort(data); }
};

struct QuickSort { void sort(std::vector<int>& d) { /* ... */ } };
struct MergeSort { void sort(std::vector<int>& d) { /* ... */ } };

DataProcessor<QuickSort> dp1;  // 快排版本
DataProcessor<MergeSort> dp2;  // 归并版本
// 无法运行时切换，但性能最优（可内联）
```

---

## 2. 观察者模式（Observer）——一对多通知

### 2.1 经典实现

```cpp
class Observer {
public:
    virtual void update(const std::string& msg) = 0;
    virtual ~Observer() = default;
};

class Subject {
    std::vector<Observer*> observers_;
public:
    void attach(Observer* o) { observers_.push_back(o); }
    void detach(Observer* o) {
        observers_.erase(std::remove(observers_.begin(), observers_.end(), o), observers_.end());
    }
    void notify(const std::string& msg) {
        for (auto* o : observers_) o->update(msg);
    }
};

// 具体观察者
class Logger : public Observer {
public:
    void update(const std::string& msg) override {
        std::cout << "[LOG] " << msg << "\n";
    }
};

class EmailAlarm : public Observer {
public:
    void update(const std::string& msg) override {
        // 发送邮件...
    }
};

Subject subject;
Logger logger;
EmailAlarm alarm;
subject.attach(&logger);
subject.attach(&alarm);
subject.notify("CPU usage > 90%");
```

### 2.2 现代 C++——信号槽（Signal-Slot）

```cpp
// 轻量信号槽实现（简化版 boost::signals2）
template<typename... Args>
class Signal {
    using Slot = std::function<void(Args...)>;
    std::vector<Slot> slots_;
public:
    void connect(Slot slot) { slots_.push_back(std::move(slot)); }

    void emit(Args... args) {
        for (auto& slot : slots_) slot(args...);
    }
};

// 使用
Signal<const std::string&> onDataChanged;
onDataChanged.connect([](const std::string& s) { std::cout << s; });
onDataChanged.connect([](const std::string& s) { /* 另一个处理 */ });
onDataChanged.emit("data updated");

// C++ 实际会用 Qt 的 signal/slot 或 boost::signals2
```

### 2.3 基于回调的观察者（`std::function` 列表）

```cpp
class EventEmitter {
    std::vector<std::function<void(int)>> listeners_;
public:
    // 返回 subscription ID，支持取消订阅
    int subscribe(std::function<void(int)> f) {
        listeners_.push_back(std::move(f));
        return listeners_.size() - 1;
    }
    void emit(int value) {
        for (auto& f : listeners_) f(value);
    }
};
```

---

## 3. 策略 vs 观察者——对比

| | 策略 | 观察者 |
|------|------|------|
| 关系 | **一对一**（上下文 ↔ 策略） | **一对多**（Subject ↔ N 个 Observer） |
| 数据流向 | 上下文 → 策略（委托） | Subject → Observer（通知） |
| 目的 | 替换算法/行为 | 状态变化通知 |
| 实现 | 虚函数 / `std::function` | 回调列表 / 信号槽 |

---

## 小结

| 模式 | C++ 推荐实现 |
|------|-------------|
| 策略（编译期） | 模板参数 — 零开销 |
| 策略（运行时） | `std::function` 替代虚函数 |
| 观察者 | `std::function` 回调列表 / 信号槽 |

下一篇 [08 Pimpl / 适配器 / 装饰器模式](./08-pimpl-adapter-decorator.md)。
