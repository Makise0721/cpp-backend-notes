# 07 — STL 容器底层原理深度解析

> 关联篇目：[06 模板/SFINAE/CRTP](./06-templates-sfinae-crtp.md) | [08 STL 算法与迭代器](./08-stl-algorithms-iterators.md)

---

## 1. 容器全景图——先有一个 mental model

STL 容器分四大类：

| 类别 | 容器 | 底层结构 | 核心特点 |
|------|------|----------|----------|
| 顺序容器 | `vector` | 动态数组 | 连续内存，尾端操作 O(1)，随机访问 O(1) |
| 顺序容器 | `deque` | 分段数组 | 双端操作 O(1)，随机访问 O(1)（稍慢于 vector） |
| 顺序容器 | `list` | 双向链表 | 任意位置插入/删除 O(1)，无法随机访问 |
| 关联容器 | `map` / `set` | 红黑树 | 有序，查找/插入/删除 O(log n) |
| 关联容器 | `unordered_map` / `unordered_set` | 哈希表 | 无序，平均 O(1)，最坏 O(n) |
| 适配器 | `stack` / `queue` / `priority_queue` | 包装底层容器 | 限制接口，改变行为语义 |

---

## 2. `vector`——你的默认容器

### 2.1 底层结构

```
vector<T> 对象（通常 3 个指针 = 24 字节）：
┌──────────┐     ┌───┬───┬───┬───┬───┬───┬───┬───┐
│  begin   │───→ │ 0 │ 1 │ 2 │ 3 │   │   │   │   │
├──────────┤     └───┴───┴───┴───┴───┴───┴───┴───┘
│  end     │─────────────────────┘       │
├──────────┤                             │
│  cap_end │───────────────────────────────┘
└──────────┘
    size() = end - begin = 4
    capacity() = cap_end - begin = 8
```

- 连续内存，支持 `data()` 获取裸指针
- 缓存友好——和 C 数组一样快
- 只能在尾端高效插入/删除

### 2.2 扩容机制：1.5x vs 2x

当 `push_back` 时 `size() == capacity()`，vector 需要重新分配更大的内存并搬迁所有元素。

**增长因子**：MSVC 用 1.5x，GCC/libc++ 用 2x。这不是任意的——各有道理。

```
以 2x 扩容（GCC）：
capacity: 1 → 2 → 4 → 8 → 16 → 32 → 64 → 128 → ...
每次扩容后，之前所有分配的内存加起来 < 当前分配的内存
→ 之前的内存块不可能被复用（都太小了）
→ 内存碎片更严重

以 1.5x 扩容（MSVC）：
capacity: 1 → 2 → 3 → 4 → 6 → 9 → 13 → 19 → 28 → 42 → 63 → ...
1 + 2 + 3 + 4 + 6 + 9 + 13 + 19 + 28 + 42 = 127 > 63
→ 在某个时刻，之前释放的内存块总和足够大，可以复用的！
→ 内存利用率更好
```

**均摊复杂度**：`push_back` 的均摊复杂度是 O(1)。虽然偶尔触发 O(n) 的搬迁，但总搬迁次数 ≈ 2n（等比数列求和），均摊到每次操作就是 O(1)。

### 2.3 `reserve` 和 `resize`

```cpp
std::vector<int> v;
v.reserve(1000);  // 预留 capacity，不改变 size
for (int i = 0; i < 1000; ++i)
    v.push_back(i);  // 这 1000 次 push_back 都不会触发扩容！

v.resize(500);     // 改变 size 为 500，多余元素删除
v.resize(800, 42); // size 增加到 800，新元素用 42 填充
```

**黄金规则**：如果你知道最终会有多少元素，先用 `reserve`。这避免了多次扩容和搬迁的开销。

### 2.4 `emplace_back` vs `push_back`

```cpp
struct Point { int x, y; Point(int x, int y) : x(x), y(y) {} };

v.push_back(Point(1, 2));      // 步骤：构造临时 Point → 移动（或拷贝）到 vector
v.emplace_back(1, 2);          // 步骤：直接在 vector 内部构造 Point。没有临时对象！
```

`emplace_back` 通过完美转发直接在容器内存上构造——消除一次移动/拷贝。对于 `std::string` 等有堆分配的类型，这可能是显著的优化。

### 2.5 迭代器失效

```cpp
std::vector<int> v = {1, 2, 3, 4, 5};
auto it = v.begin() + 2;  // 指向 3

v.push_back(6);  // 可能触发扩容 → 内存重新分配 → it 指向释放的内存！
// *it;           // ❌ 未定义行为

// 同样：
v.insert(v.begin(), 0);  // 插入在 it 之前 → 所有元素后移 → it 指向的元素变了
v.erase(v.begin());      // 删除在 it 之前的元素 → it 指向的元素变了
```

**vector 迭代器失效规则**：
- 重新分配（扩容）→ 所有迭代器失效
- 在 `it` 之前插入/删除 → `it` 及之后的迭代器失效
- 在 `it` 之后插入/删除 → `it` 之前的迭代器仍有效（但位置可能偏移）

---

## 3. `deque`——双端队列，分段存储

### 3.1 底层结构：中控器 + 分段数组

`deque` 的核心思想是**分块存储**——一个"中控器"数组管理多个固定大小的数据块（bucket），所有块组成一个逻辑上连续的双端队列。

```
deque 的内存结构（简化）：

中控器（map of pointers）:
┌──────┬──────┬──────┬──────┬──────┐
│ ptr0 │ ptr1 │ ptr2 │ ptr3 │ ptr4 │  ← 指向各个数据块
└──│───┴──│───┴──│───┴──│───┴──│───┘
   │      │      │      │      │
   ▼      ▼      ▼      ▼      ▼
 ┌────┐┌────┐┌────┐┌────┐┌────┐
 │ B0 ││ B1 ││ B2 ││ B3 ││ B4 │   ← 每个块大小固定（通常 512 字节 / sizeof(T) 个元素）
 └────┘└────┘└────┘└────┘└────┘
          ▲
          └── 数据从这里开始
```

### 3.2 为什么用分段存储？

好处是**两端插入不需要搬迁所有元素**：

- `push_front`：如果当前起始块还有空间，直接在前面放；否则分配新块，更新中控器指针
- `push_back`：同理，在末端块操作
- 不像 `vector`，`push_front` 是 O(n)（所有元素后移）

代价是：
- 随机访问比 `vector` 略慢：需要两次指针解引用（中控器 → 块 → 元素）
- 内存不连续（虽然对用户透明，但缓存局部性不如 vector）

### 3.3 deque vs vector 的选择

```cpp
// 用 deque 的场景
deque<int> dq;
dq.push_front(1);   // ✅ O(1)
dq.push_back(2);    // ✅ O(1)

// 用 vector 的场景
vector<int> v;
// 需要传递给 C API（需要裸指针）
c_api(v.data());     // ✅ vector 连续内存
// c_api(dq.data()); // ❌ deque 没有 data() 方法！
```

---

## 4. `list`——双向链表

```cpp
// list 节点结构（简化）
template<typename T>
struct ListNode {
    T data;
    ListNode* prev;
    ListNode* next;
};
```

- 任意位置插入/删除：O(1)（已有迭代器的前提下）
- 无法随机访问（`list[5]` 不存在，必须遍历）
- 每个元素有 2 个指针的开销（16 字节 on 64-bit）
- 内存不连续，缓存不友好

**list 的 sweet spot**：频繁在中间插入/删除 + 不介意遍历开销。大多数场景下 `vector` 更快——即使 `vector` 在中间插入是 O(n)，连续内存带来的缓存优势通常压倒 list 的 O(1) 理论优势。

### 迭代器失效

`list` 的插入和删除**只让被操作节点的迭代器失效**，其他迭代器完全不受影响——这是 list 相比 vector 的重大优势。

---

## 5. `map` 与 `set`——红黑树

### 5.1 为什么用红黑树而不是 AVL 树？

`std::map` 和 `std::set` 的底层标准要求是**红黑树**。选择红黑树而非 AVL 的原因是：

| 特性 | 红黑树 | AVL 树 |
|------|--------|--------|
| 平衡条件 | 宽松（最长路径 ≤ 2×最短路径） | 严格（高度差 ≤ 1） |
| 查找 | O(log n)，稍慢（树更高） | O(log n)，稍快（树更矮） |
| 插入/删除旋转次数 | 最多 2-3 次 | 可能 O(log n) 次 |
| 重染色 vs 重平衡 | 多数情况只改颜色 | 经常需要旋转 |

STL 的 map/set 更多用于"频繁插入删除 + 偶尔查找"的场景，红黑树在此场景下总开销更低。

### 5.2 红黑树的核心性质

1. 每个节点是红色或黑色
2. 根节点是黑色
3. 每个叶子（NIL）是黑色
4. 红色节点的两个子节点必须都是黑色（不能连续红）
5. 从任意节点到其所有后代叶子的简单路径上，黑色节点数量相同

### 5.3 插入操作的关键 case

```
新插入的节点默认是红色。

Case 1：父节点是黑色 → 直接插入，不影响性质 ✅
Case 2：父节点是红色（违反性质 4）→ 看叔叔节点：
  Case 2a：叔叔是红色 → 父、叔变黑，祖父变红，上溯
  Case 2b：叔叔是黑色 + 当前节点是内侧孙子 → 旋转父节点变成 Case 2c
  Case 2c：叔叔是黑色 + 当前节点是外侧孙子 → 旋转祖父 + 变色
```

动画理解：红黑树的核心操作就是**旋转 + 变色**。旋转改变树结构但不破坏 BST 性质；变色调整颜色以恢复红黑树性质。

### 5.4 `map` 的迭代器与 `erase`

```cpp
std::map<int, std::string> m = {{1, "a"}, {2, "b"}, {3, "c"}};

// ⚠️ 经典陷阱
for (auto it = m.begin(); it != m.end(); ++it) {
    if (it->first == 2)
        m.erase(it);  // C++11 之前：it 失效，++it 是 UB
}

// ✅ 正确写法
for (auto it = m.begin(); it != m.end(); ) {
    if (it->first == 2)
        it = m.erase(it);  // erase 返回下一个有效迭代器（C++11）
    else
        ++it;
}
```

---

## 6. `unordered_map` 与 `unordered_set`——哈希表

### 6.1 底层结构：开链法（Separate Chaining）

```
unordered_map 的内存结构：

bucket 数组（指针数组）:
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│  0  │  1  │  2  │  3  │  4  │  5  │  6  │  7  │
│  │  │     │  │  │     │     │  │  │     │  │  │
└──│──┴─────┴──│──┴─────┴─────┴──│──┴─────┴──│──┘
   │           │                 │           │
   ▼           ▼                 ▼           ▼
 [K1,V1]    [K2,V2]          [K3,V3]     [K4,V4]
    │           │                            │
    ▼           ▼                            ▼
 [K5,V5]    [K6,V6]                      [K7,V7]
    │
    ▼
 [K8,V8]

每个 bucket 是一个单向链表（forward list）
当 bucket 链表过长时触发 rehash
```

### 6.2 负载因子（Load Factor）与 rehash

```
load_factor = size() / bucket_count()
max_load_factor 默认 = 1.0

当 load_factor > max_load_factor 时 → rehash：
1. 分配新的 bucket 数组（大小约 2x）
2. 重新计算每个元素的 hash % new_bucket_count
3. 搬迁到新 bucket
4. 旧 bucket 数组释放
```

`rehash` 使所有迭代器失效（和 vector 扩容类似）。

### 6.3 为什么 `unordered_map` 在实际中常常不如 `map`？

| 方面 | `map`（红黑树） | `unordered_map`（哈希表） |
|------|----------------|--------------------------|
| 理论查找 | O(log n) | O(1) 平均 |
| 实际查找（小 n） | 通常更快 | 哈希计算 + 链表遍历可能更慢 |
| 内存 | 紧凑 | bucket 数组 + 链表节点 → 碎片多 |
| 缓存 | 树节点分散 | bucket 数组连续但链表分散 |
| 有序性 | 有序 ✅ | 无序 |
| 迭代 | 中序遍历 O(n) | 遍历 bucket + 链表（顺序随机） |

实际建议：
- n < 100：`map` 通常更快
- 需要有序遍历：必须 `map`
- n 很大 + 随机访问为主：`unordered_map`

---

## 7. 容器适配器——换个视角看容器

适配器不自己管理存储，而是在底层容器上包装接口：

```cpp
template<typename T, typename Container = std::deque<T>>
class stack {
protected:
    Container c;
public:
    void push(const T& x) { c.push_back(x); }
    void pop()            { c.pop_back(); }
    T& top()              { return c.back(); }
    bool empty() const    { return c.empty(); }
};
```

| 适配器 | 默认底层容器 | 接口限制 |
|--------|-------------|----------|
| `stack` | `deque` | 只暴露 top / push / pop（LIFO） |
| `queue` | `deque` | 只暴露 front / back / push / pop（FIFO） |
| `priority_queue` | `vector` | 只暴露 top / push / pop（最大堆） |

### 7.1 `priority_queue` = 二叉堆

底层用 `vector` 存储，通过 `std::make_heap` / `push_heap` / `pop_heap` 维护堆性质：

```
索引关系（从 0 开始）：
parent(i) = (i - 1) / 2
left(i)   = 2 * i + 1
right(i)  = 2 * i + 2

插入（push）：
vector        堆表示
[3,1,4,1,5]   5
             / \
            3   4
           / \
          1   1

push(9) → 放到 vector 末尾 → 上浮（sift up）→ 和 parent 比较/交换直到堆性质恢复
```

默认是大顶堆（`std::less` 生成最大堆）。小顶堆用 `std::greater`：

```cpp
std::priority_queue<int, std::vector<int>, std::greater<int>> minHeap;
```

---

## 8. 容器选择决策树

```
需要什么操作？
│
├─ 随机访问 + 尾端插入 → vector（默认首选）
│
├─ 双端插入/删除 + 随机访问 → deque
│
├─ 中间插入/删除（已有迭代器） + 不关心中间遍历 → list
│
├─ 有序 + 按键查找 → map / set
│
├─ 无序 + 按键查找 + 数据量大 → unordered_map / unordered_set
│
├─ LIFO → stack
├─ FIFO → queue
└─ 优先级 → priority_queue
```

**默认选择永远是用 `vector`**——只有在确实不合适时才换。

---

## 小结

| 容器 | 底层 | 关键操作复杂度 | 内存 |
|------|------|---------------|------|
| `vector` | 动态数组 | `push_back` 均摊 O(1)，扩容 1.5x/2x | 连续 |
| `deque` | 中控器 + 分段块 | 双端 O(1)，随机 O(1)（两次解引用） | 分段 |
| `list` | 双向链表 | 插入/删除 O(1) | 分散，每节点 2 指针 |
| `map`/`set` | 红黑树 | O(log n)，插入旋转少 | 树节点 + 颜色位 |
| `unordered_map` | 哈希表 + 链表 | 平均 O(1)，rehash 时 O(n) | bucket 数组 + 链表 |
| 适配器 | 包装底层 | 接口受限 | 和底层一致 |

下一篇 [08 STL 算法与迭代器](./08-stl-algorithms-iterators.md)。
