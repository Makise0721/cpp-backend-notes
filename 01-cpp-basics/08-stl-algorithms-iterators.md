# 08 — STL 算法、迭代器、仿函数与适配器

> 关联篇目：[06 模板/SFINAE/CRTP](./06-templates-sfinae-crtp.md) | [07 STL 容器底层原理](./07-stl-containers-deep-dive.md)

---

## 1. 迭代器——算法与容器的桥梁

迭代器的设计哲学：**让算法不知道自己在操作什么容器**。算法只看到迭代器，容器只提供迭代器——二者通过迭代器接口解耦。

```cpp
// 同一个 find 算法，操作 vector 和 list 完全一样的代码
std::vector<int> v = {1, 2, 3, 4, 5};
auto it1 = std::find(v.begin(), v.end(), 3);  // → 指向 3 的迭代器

std::list<int> l = {1, 2, 3, 4, 5};
auto it2 = std::find(l.begin(), l.end(), 3);  // 完全相同的调用方式
```

### 1.1 五种迭代器类别

不是所有迭代器能力都一样。STL 定义了五级迭代器，形成能力层次：

```
InputIterator  →  ForwardIterator  →  BidirectionalIterator  →  RandomAccessIterator
                                                    ↘
                                              OutputIterator
```

| 类别 | 支持操作 | 典型来源 |
|------|----------|----------|
| **Input** | `*it`（读），`++it`，`it++`，`==`，`!=` | `std::istream_iterator`，一次遍历 |
| **Output** | `*it = val`（写），`++it`，`it++` | `std::ostream_iterator`，`back_inserter` |
| **Forward** | Input + Output + 可多次遍历 | `std::forward_list` |
| **Bidirectional** | Forward + `--it`，`it--` | `std::list`，`std::map`，`std::set` |
| **RandomAccess** | Bidirectional + `it + n`，`it - n`，`it[n]`，`<`，`>` | `std::vector`，`std::deque`，原生指针 |

### 1.2 能力决定算法

算法对其迭代器有最低能力要求：

```cpp
std::sort(v.begin(), v.end());   // 需要 RandomAccess（内部用快排，要随机跳转）
std::reverse(l.begin(), l.end());// 需要 Bidirectional（头尾对调，要能前后移动）
std::find(l.begin(), l.end(), x);// 只需 Input（线性扫描）

// std::sort(l.begin(), l.end()); // ❌ 编译错误：list 的迭代器不支持随机访问
// 所以 list 有自己的 sort 成员函数
l.sort();
```

### 1.3 迭代器失效——每个容器不同

| 操作 | vector | deque | list | map/set | unordered_map |
|------|--------|-------|------|---------|---------------|
| 插入元素 | 扩容→全失效；不扩容→插入点后失效 | 中间插入→全失效；两端→不失效 | 不失效 | 不失效 | rehash→全失效 |
| 删除元素 | 删除点后失效 | 同 vector（中间删除全失效） | 仅被删除的失效 | 仅被删除的失效 | 仅被删除的失效 |

```cpp
// 安全遍历并条件删除的通用模式
for (auto it = container.begin(); it != container.end(); ) {
    if (shouldRemove(*it))
        it = container.erase(it);  // erase 返回下一个有效迭代器
    else
        ++it;
}
```

### 1.4 迭代器辅助函数

```cpp
#include <iterator>

std::vector<int> v = {1, 2, 3, 4, 5};

std::advance(it, 3);       // 前进 3 步。对 RandomAccess 是 O(1)，否则 O(n)
auto d = std::distance(v.begin(), v.end());  // 计算距离
auto prev = std::prev(it); // it - 1
auto next = std::next(it); // it + 1
auto next2 = std::next(it, 2); // it + 2
```

---

## 2. 常用算法——无需手写循环

STL 提供了 >100 个算法。用它们而不是手写循环——更短、更清晰、更难出错。

### 2.1 非修改序列算法

```cpp
std::vector<int> v = {1, 2, 3, 4, 5, 3, 6};

// 查找
auto it = std::find(v.begin(), v.end(), 3);          // 第一个 3
auto it2 = std::find_if(v.begin(), v.end(),          // 第一个 >3 的元素
    [](int x) { return x > 3; });

int count = std::count(v.begin(), v.end(), 3);       // 2
int count_if = std::count_if(v.begin(), v.end(),     // 偶数的个数
    [](int x) { return x % 2 == 0; });

bool all_even = std::all_of(v.begin(), v.end(),      // 全部偶数？
    [](int x) { return x % 2 == 0; });
bool any_neg = std::any_of(v.begin(), v.end(),        // 存在负数？
    [](int x) { return x < 0; });
```

### 2.2 修改序列算法

```cpp
// transform — 映射
std::vector<int> squares(v.size());
std::transform(v.begin(), v.end(), squares.begin(),
    [](int x) { return x * x; });

// copy_if — 条件复制
std::vector<int> evens;
std::copy_if(v.begin(), v.end(), std::back_inserter(evens),
    [](int x) { return x % 2 == 0; });

// remove + erase 惯用法（真正删除元素）
v.erase(std::remove(v.begin(), v.end(), 3), v.end());
//     ↑ remove 把要保留的元素前移，返回"新末尾"；erase 真正删除

// replace
std::replace(v.begin(), v.end(), 3, 33);   // 所有 3 → 33

// reverse
std::reverse(v.begin(), v.end());

// unique — 去重（要求先排序！）
std::sort(v.begin(), v.end());
v.erase(std::unique(v.begin(), v.end()), v.end());
```

### 2.3 排序与二分查找

```cpp
std::sort(v.begin(), v.end());                           // 升序，O(n log n)
std::sort(v.begin(), v.end(), std::greater<int>());      // 降序
std::stable_sort(v.begin(), v.end());                    // 稳定排序（相等元素保持原序）

// 部分排序
std::partial_sort(v.begin(), v.begin() + 3, v.end());    // 只排前 3 小
std::nth_element(v.begin(), v.begin() + 3, v.end());     // 第 4 小的在位置 3，左边都 ≤ 它

// 二分查找（要求有序！）
bool found = std::binary_search(v.begin(), v.end(), 42);
auto it = std::lower_bound(v.begin(), v.end(), 42);      // 第一个 ≥ 42 的位置
auto it2 = std::upper_bound(v.begin(), v.end(), 42);     // 第一个 > 42 的位置
auto [lo, hi] = std::equal_range(v.begin(), v.end(), 42);// [lo, hi) 区间的值都 == 42
```

### 2.4 堆算法

```cpp
std::vector<int> v = {3, 1, 4, 1, 5, 9};

std::make_heap(v.begin(), v.end());        // 最大堆
v.push_back(10);
std::push_heap(v.begin(), v.end());        // 维护堆性质
std::pop_heap(v.begin(), v.end());         // 最大元素移到末尾
int max = v.back(); v.pop_back();
std::sort_heap(v.begin(), v.end());        // 堆排序（升序）

// 小顶堆
std::make_heap(v.begin(), v.end(), std::greater<int>());
```

### 2.5 集合操作（要求有序）

```cpp
std::vector<int> a = {1, 2, 3, 5, 7};
std::vector<int> b = {2, 4, 5, 6};
std::vector<int> result;

std::set_union(a.begin(), a.end(), b.begin(), b.end(), std::back_inserter(result));
std::set_intersection(...);
std::set_difference(...);
```

### 2.6 数值算法

```cpp
#include <numeric>

int sum = std::accumulate(v.begin(), v.end(), 0);              // 求和
int sum2 = std::accumulate(v.begin(), v.end(), 1,
    [](int acc, int x) { return acc * x; });                  // 求积

std::vector<int> prefix(v.size());
std::partial_sum(v.begin(), v.end(), prefix.begin());         // 前缀和 [1, 1+2, 1+2+3, ...]

int inner = std::inner_product(a.begin(), a.end(), b.begin(), 0); // 点积
```

---

## 3. 仿函数（Functor）——带状态的可调用对象

重载了 `operator()` 的类/结构体，可以像函数一样使用，但可以携带状态：

```cpp
class GreaterThan {
    int threshold_;
public:
    GreaterThan(int t) : threshold_(t) {}
    bool operator()(int x) const { return x > threshold_; }
};

std::vector<int> v = {1, 5, 3, 8, 2};
int count = std::count_if(v.begin(), v.end(), GreaterThan(3));  // 2 (5 和 8)
```

### 3.1 内建仿函数对象

```cpp
#include <functional>

std::sort(v.begin(), v.end(), std::greater<int>());  // 降序
std::sort(v.begin(), v.end(), std::less<int>());     // 升序（默认）
std::transform(a.begin(), a.end(), b.begin(), result.begin(), std::plus<int>());
std::transform(a.begin(), a.end(), b.begin(), result.begin(), std::multiplies<int>());
```

### 3.2 Lambda——现代替代品

C++11 lambda 让仿函数几乎不再需要手写：

```cpp
int threshold = 3;
auto count = std::count_if(v.begin(), v.end(),
    [threshold](int x) { return x > threshold; });  // 捕获 threshold
// 编译器自动生成一个匿名仿函数类，threshold 作为成员存储
```

Lambda 捕获列表：

```cpp
int a = 1, b = 2;
[=]        { return a + b; }   // 按值捕获所有
[&]        { a = b; }          // 按引用捕获所有
[a, &b]    { b = a; }          // a 按值，b 按引用
[this]     { return member_; } // 捕获 this 指针
[*this]    { }                 // C++17：按值捕获 *this
```

---

## 4. 适配器——改变接口

### 4.1 迭代器适配器

```cpp
// back_inserter：把赋值变成 push_back
std::vector<int> src = {1, 2, 3}, dst;
std::copy(src.begin(), src.end(), std::back_inserter(dst));  // dst: [1, 2, 3]

// front_inserter：把赋值变成 push_front（需要容器支持 push_front）
std::list<int> lst;
std::copy(src.begin(), src.end(), std::front_inserter(lst)); // lst: [3, 2, 1]

// inserter：在指定位置插入
std::set<int> s;
std::copy(src.begin(), src.end(), std::inserter(s, s.begin()));

// reverse_iterator：反向遍历
for (auto it = v.rbegin(); it != v.rend(); ++it) {}  // 从尾到头
```

### 4.2 函数适配器

```cpp
#include <functional>

// std::bind：绑定参数
auto plus10 = std::bind(std::plus<int>(), std::placeholders::_1, 10);
plus10(5);  // 15

// std::function：类型擦除的通用可调用包装器
std::function<int(int, int)> f;
f = [](int a, int b) { return a + b; };   // 可以装 lambda
f = std::plus<int>();                      // 可以装仿函数
f = old_c_function;                        // 可以装函数指针

// std::ref / std::cref：引用包装（按引用传递到接受值的地方）
int x = 42;
auto ref = std::ref(x);    // reference_wrapper<int>
auto cref = std::cref(x);  // const 版本
```

---

## 5. 实战：用算法链替代手写循环

```cpp
// ❌ 手写循环——5 行，意图埋在代码中
std::vector<int> result;
for (int x : data) {
    if (x % 2 == 0)
        result.push_back(x * x);
}

// ✅ 算法链——意图清晰
std::vector<int> result;
std::copy_if(data.begin(), data.end(), std::back_inserter(result),
    [](int x) { return x % 2 == 0; });
std::transform(result.begin(), result.end(), result.begin(),
    [](int x) { return x * x; });

// ✅✅ C++20 ranges——一个管道搞定
auto result = data
    | std::views::filter([](int x) { return x % 2 == 0; })
    | std::views::transform([](int x) { return x * x; })
    | std::ranges::to<std::vector>();
```

### 5.1 为什么用算法而不是循环？

1. **意图明确**：`std::find` 一看就知道在查找；`for` 循环要读完才知道
2. **减少 off-by-one 错误**：循环边界写错是最高频的 bug
3. **编译器优化更好**：算法内部可能有特化（比如 `std::copy` 对 trivially copyable 类型用 `memmove`）
4. **未来可并行化**：C++17 引入了带执行策略的算法 `std::sort(std::execution::par, v.begin(), v.end())`

---

## 小结

| 概念 | 一句话 |
|------|--------|
| 5 种迭代器 | Input → Forward → Bidirectional → RandomAccess，能力递增 |
| 算法 | >100 个，操作迭代器区间，与容器类型无关 |
| remove-erase | `remove` 返回新末尾，`erase` 真正删除 |
| Lambda | 编译器生成的匿名仿函数，现代 C++ 首选 |
| `back_inserter` | 把赋值变成 `push_back` |
| `std::bind` | 参数绑定，部分应用 |
| C++20 ranges | 管道语法，比传统算法链更直观 |

下一篇 [09 命名空间、异常处理与 noexcept](./09-namespace-exception-noexcept.md)。
