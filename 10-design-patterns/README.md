# 第十章：设计模式（C++ 实现）

> 5 篇 · 经典 GoF 模式的现代 C++ 实现——用 `std::function`、模板、`unique_ptr` 替代传统虚函数。

---

## 篇目

| # | 文件 | 核心内容 |
|---|------|----------|
| 01 | [01-singleton.md](./01-singleton.md) | Meyers' Singleton(C++11 static 线程安全)、call_once、atexit 销毁 |
| 02 | [02-factory.md](./02-factory.md) | 简单工厂、工厂方法、抽象工厂、注册式工厂(模板+map) |
| 03 | [03-strategy-observer.md](./03-strategy-observer.md) | 策略模式(std::function 替代虚函数/模板策略)、观察者模式(信号槽) |
| 04 | [04-pimpl-adapter-decorator.md](./04-pimpl-adapter-decorator.md) | Pimpl 编译防火墙、适配器(STL stack/queue)、装饰器(unique_ptr 嵌套) |
| 05 | [05-template-method-chain.md](./05-template-method-chain.md) | 模板方法(NVI 惯用法/CRTP)、责任链(函数管道/Pipeline) |
