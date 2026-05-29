# 02 — 工厂模式：简单工厂、工厂方法、抽象工厂

> 关联篇目：[03 策略与观察者](./03-strategy-observer.md) | [第一章 模板/CRTP](../01-cpp-basics/06-templates-sfinae-crtp.md)

---

## 1. 简单工厂——用一个函数创建不同对象

```cpp
enum class CarType { SUV, Sedan };

class Car { public: virtual void drive() = 0; virtual ~Car() = default; };
class SUV : public Car { public: void drive() override { /* ... */ } };
class Sedan : public Car { public: void drive() override { /* ... */ } };

class CarFactory {
public:
    static std::unique_ptr<Car> create(CarType type) {
        switch (type) {
            case CarType::SUV:   return std::make_unique<SUV>();
            case CarType::Sedan: return std::make_unique<Sedan>();
        }
        return nullptr;
    }
};

auto car = CarFactory::create(CarType::SUV);
```

| 优点 | 缺点 |
|------|------|
| 创建逻辑集中 | 新增类型要修改 switch（违反开闭原则） |
| 调用方不需要知道具体类 | 只有一个工厂 |

---

## 2. 工厂方法——每个子类各有一个工厂

```cpp
// 产品
class Transport { public: virtual void deliver() = 0; virtual ~Transport() = default; };
class Truck : public Transport { public: void deliver() override { /* 陆运 */ } };
class Ship  : public Transport { public: void deliver() override { /* 海运 */ } };

// 工厂（抽象）
class Logistics {
public:
    virtual std::unique_ptr<Transport> createTransport() = 0;  // 工厂方法
    void planDelivery() {
        auto t = createTransport();
        t->deliver();
    }
    virtual ~Logistics() = default;
};

// 具体工厂
class RoadLogistics : public Logistics {
public:
    std::unique_ptr<Transport> createTransport() override {
        return std::make_unique<Truck>();
    }
};

class SeaLogistics : public Logistics {
public:
    std::unique_ptr<Transport> createTransport() override {
        return std::make_unique<Ship>();
    }
};

// 使用
std::unique_ptr<Logistics> logistics = std::make_unique<RoadLogistics>();
logistics->planDelivery();  // 创建 Truck 并 deliver
```

**关键**：`createTransport()` 是"工厂方法"——子类决定创建哪种产品。客户端代码使用 `Logistics` 基类，不关心具体产品。

---

## 3. 抽象工厂——创建一组相关产品

```cpp
// 产品族：按钮 + 文本框
class Button { public: virtual void render() = 0; virtual ~Button() = default; };
class TextBox { public: virtual void render() = 0; virtual ~TextBox() = default; };

class WinButton : public Button { public: void render() override { } };
class WinTextBox : public TextBox { public: void render() override { } };
class MacButton : public Button { public: void render() override { } };
class MacTextBox : public TextBox { public: void render() override { } };

// 抽象工厂
class GUIFactory {
public:
    virtual std::unique_ptr<Button> createButton() = 0;
    virtual std::unique_ptr<TextBox> createTextBox() = 0;
    virtual ~GUIFactory() = default;
};

class WinFactory : public GUIFactory {
public:
    std::unique_ptr<Button> createButton() override { return std::make_unique<WinButton>(); }
    std::unique_ptr<TextBox> createTextBox() override { return std::make_unique<WinTextBox>(); }
};

class MacFactory : public GUIFactory {
public:
    std::unique_ptr<Button> createButton() override { return std::make_unique<MacButton>(); }
    std::unique_ptr<TextBox> createTextBox() override { return std::make_unique<MacTextBox>(); }
};
```

| | 简单工厂 | 工厂方法 | 抽象工厂 |
|------|:---:|:---:|:---:|
| 创建几个产品 | 1 个 | 1 个 | **多个**（产品族） |
| 新增产品 | 改工厂 | 新增子工厂 | 改接口 |
| 适用 | 简单场景 | 单个产品多态创建 | 跨平台/主题（一组相关对象） |

---

## 4. C++ 现代工厂——用模板替代虚函数

```cpp
// 不需要虚函数的工厂（编译期多态）
template<typename T>
std::unique_ptr<T> create() {
    return std::make_unique<T>();
}

// 注册式工厂（运行时注册新类型）
class Factory {
    using Creator = std::function<std::unique_ptr<void>()>;
    std::unordered_map<std::string, Creator> registry_;
public:
    template<typename T>
    void registerType(const std::string& name) {
        registry_[name] = [] { return std::unique_ptr<void>(new T()); };
    }

    template<typename T>
    std::unique_ptr<T> create(const std::string& name) {
        return std::unique_ptr<T>(static_cast<T*>(registry_.at(name)().release()));
    }
};

Factory f;
f.registerType<SUV>("suv");
auto car = f.create<Car>("suv");
```

---

## 小结

| 模式 | 核心 | 适合 |
|------|------|------|
| 简单工厂 | 一个函数用参数决定创建哪个 | 产品类型少 |
| 工厂方法 | 子类决定创建哪个产品 | 一对一，需要扩展 |
| 抽象工厂 | 创建产品族（一组相关对象） | 跨平台 UI、主题切换 |

下一篇 [03 策略与观察者模式](./03-strategy-observer.md)。
