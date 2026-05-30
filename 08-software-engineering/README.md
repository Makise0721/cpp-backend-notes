# 第八章：软件工程

> 9 篇 · 从编译链接到设计模式，覆盖 C++ 工程化的核心环节和常用设计模式。

### 编译构建与工程 (01-04)

| # | 核心内容 |
|---|------|
| 01 | 编译四阶段、静态库 vs 动态库、-fPIC/GOT/PLT、强弱符号、链接顺序 |
| 02 | CMakeLists.txt 结构、PUBLIC/PRIVATE/INTERFACE、find_package |
| 03 | merge vs rebase、cherry-pick/stash、bisect、Git Flow/GitHub Flow |
| 04 | GoogleTest、GoogleMock、Google Benchmark、GitHub Actions CI |

### 设计模式（C++ 实现）(05-09)

| # | 核心内容 |
|---|------|
| 05 | Meyers' Singleton(C++11 static 线程安全)、call_once |
| 06 | 简单工厂、工厂方法、抽象工厂、注册式工厂 |
| 07 | 策略(std::function 替代虚函数)、观察者(信号槽) |
| 08 | Pimpl 编译防火墙、适配器、装饰器 |
| 09 | 模板方法(NVI/CRTP)、责任链(函数管道) |

路线：构建工具链(01-04)→设计模式(05-09)
