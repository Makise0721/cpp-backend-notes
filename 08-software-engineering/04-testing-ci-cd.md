# 04 — 测试框架与 CI/CD

> 关联篇目：[03 Git 工作流](./03-git-workflow.md) | [第四章 15 Sanitizer](../04-operating-systems/15-sanitizers-valgrind.md)

---

## 1. GoogleTest——C++ 单元测试

### 1.1 基本断言

```cpp
#include <gtest/gtest.h>

TEST(TestSuiteName, TestName) {
    EXPECT_EQ(4, 2 + 2);          // 非致命：失败后继续
    EXPECT_NE(5, 3);
    EXPECT_TRUE(flag);
    EXPECT_FALSE(result);
    EXPECT_STREQ("hello", str);   // C 字符串
    EXPECT_NEAR(3.14, pi, 0.01);  // 浮点近似
    EXPECT_THROW(func(), std::exception);

    ASSERT_EQ(4, 2 + 2);          // 致命：失败后当前 TEST 终止
}
```

| `EXPECT_*` | `ASSERT_*` |
|------------|-----------|
| 失败后继续执行 | 失败后立即退出当前测试 |
| 用于可恢复的错误检查 | 用于"后续逻辑依赖此条件" |

### 1.2 Test Fixture——共享初始化

```cpp
class MyTest : public ::testing::Test {
protected:
    void SetUp() override {
        // 每个 TEST_F 之前执行
        data_ = new int[100];
    }
    void TearDown() override {
        // 每个 TEST_F 之后执行
        delete[] data_;
    }
    int* data_;
};

TEST_F(MyTest, Test1) {
    data_[0] = 42;
    EXPECT_EQ(42, data_[0]);
}

TEST_F(MyTest, Test2) {
    EXPECT_EQ(0, data_[0]);  // 每个测试有独立的 MyTest 实例
}
```

### 1.3 参数化测试

```cpp
class ParamTest : public ::testing::TestWithParam<int> {};

TEST_P(ParamTest, Square) {
    int x = GetParam();
    EXPECT_EQ(x * x, square(x));
}

INSTANTIATE_TEST_SUITE_P(
    Numbers,
    ParamTest,
    ::testing::Values(1, 2, 3, 5, 8, 13)
);
```

---

## 2. GoogleMock——Mock 对象

```cpp
#include <gmock/gmock.h>

// 定义接口
class DataService {
public:
    virtual ~DataService() = default;
    virtual int fetch(const std::string& key) = 0;
    virtual void store(const std::string& key, int value) = 0;
};

// Mock 实现
class MockDataService : public DataService {
public:
    MOCK_METHOD(int, fetch, (const std::string& key), (override));
    MOCK_METHOD(void, store, (const std::string& key, int value), (override));
};

// 使用 Mock
TEST(MyTest, FetchTest) {
    MockDataService mock;

    // 设置期望
    EXPECT_CALL(mock, fetch("score"))
        .Times(1)                         // 被调用 1 次
        .WillOnce(::testing::Return(100)); // 返回 100

    EXPECT_CALL(mock, store(::testing::_, ::testing::_))
        .Times(::testing::AtLeast(1));

    // 使用 mock
    MyClass obj(&mock);
    int result = obj.getScore();
    EXPECT_EQ(100, result);
}
```

---

## 3. Google Benchmark——性能基准测试

```cpp
#include <benchmark/benchmark.h>

static void BM_StringCopy(benchmark::State& state) {
    std::string src = "hello world";
    for (auto _ : state) {
        std::string copy = src;     // 被测量的操作
    }
}
BENCHMARK(BM_StringCopy);

static void BM_VectorPush(benchmark::State& state) {
    for (auto _ : state) {
        std::vector<int> v;
        for (int i = 0; i < state.range(0); ++i)
            v.push_back(i);
    }
}
BENCHMARK(BM_VectorPush)->Arg(1000)->Arg(10000);

BENCHMARK_MAIN();
```

```bash
# 编译
g++ -std=c++17 -O2 -o bench bench.cpp -lbenchmark -lpthread

# 运行
./bench
# 输出:
# BM_StringCopy    18 ns     18 ns    38000000
# BM_VectorPush/1000   2500 ns   2500 ns     280000
# BM_VectorPush/10000  35000 ns  35000 ns     20000
#                      ↑平均      ↑中位数    ↑每秒次数
```

---

## 4. CI/CD 基本概念

### 4.1 持续集成（CI）

代码 push → 自动构建 → 自动测试 → 反馈结果。

```
开发者推送代码
    │
    ▼
┌──────────────────────────┐
│  CI Pipeline              │
│  1. Checkout              │
│  2. Build                 │
│  3. Unit Test             │
│  4. Lint + Format Check   │
│  5. Sanitizer Test (ASan) │
│  6. 报告结果              │
└──────────────────────────┘
```

### 4.2 持续交付/部署（CD）

CI 通过后自动部署到测试/预发布环境。更进一步：自动部署到生产环境（持续部署）。

### 4.3 GitHub Actions 示例

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure
        run: cmake -B build -DCMAKE_BUILD_TYPE=Debug

      - name: Build
        run: cmake --build build -j$(nproc)

      - name: Test
        run: cd build && ctest --output-on-failure

      - name: ASan Test
        run: |
          cmake -B build_asan -DCMAKE_BUILD_TYPE=Debug \
                -DCMAKE_CXX_FLAGS="-fsanitize=address"
          cmake --build build_asan
          cd build_asan && ctest
```

### 4.4 CI/CD 工具对比

| 工具 | 特点 | 适用 |
|------|------|------|
| **GitHub Actions** | 深度集成 GitHub，免费额度 | 开源 + GitHub 托管项目 |
| **GitLab CI** | 自托管 + SaaS，功能全面 | 企业内部 |
| **Jenkins** | 老牌、高度可定制、插件丰富 | 大型企业遗留系统 |
| **TeamCity** | JetBrains 出品 | 中小团队 |

---

## 5. 测试金字塔

```
        ┌─────────┐
        │  E2E    │ ← 少量，全链路（成本高、慢）
        ├─────────┤
        │ 集成测试  │ ← 中等数量（多模块协作）
        ├─────────┤
        │ 单元测试  │ ← 大量（核心逻辑全覆盖，快）
        └─────────┘

单元测试: 70%    — 函数/类级别，毫秒级
集成测试: 20%    — 模块交互，秒级
E2E 测试: 10%   — 用户行为模拟，分钟级
```

---

## 小结

| 工具 | 用途 |
|------|------|
| GoogleTest | 断言/Test Fixture/参数化测试 |
| GoogleMock | Mock 接口/EXPECT_CALL |
| Google Benchmark | 性能基准 |
| GitHub Actions | CI/CD（build + test + sanitizer） |
| ctest | CMake 集成的测试运行器 |

---

## 第九章小结

| # | 篇目 | 核心 |
|---|------|------|
| 01 | 编译链接过程 | 预处理→编译→汇编→链接/.a vs .so/PIC/强弱符号 |
| 02 | CMake 基础 | CMakeLists/PRIVATE-PUBLIC-INTERFACE/find_package |
| 03 | Git 工作流 | merge vs rebase/stash/cherry-pick/bisect/Git Flow |
| 04 | 测试与 CI/CD | GoogleTest/Mock/Benchmark + GitHub Actions |
