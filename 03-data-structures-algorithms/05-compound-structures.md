# 05 — 工程复合结构：LRU/LFU、并查集、单调栈/队列、概率数据结构

> 关联篇目：[01 线性结构](./01-linear-structures.md) | [04 哈希表](./04-graph-hash-bloom.md)

---

## 1. LRU Cache——最近最少使用

**LRU** = Least Recently Used。当缓存满时，淘汰**最久未被访问**的数据。

### 1.1 数据结构：哈希表 + 双向链表

```
哈希表:                       双向链表（从新到旧）:
┌─────┬─────┐                ┌──────────┐  ┌──────────┐  ┌──────────┐
│ key │ 节点指针 │               │ newest   │⇄│  middle   │⇄│  oldest   │
├─────┼─────┤                │ key3,val3│  │ key2,val2│  │ key1,val1│
│ k1  │  →  │──────────────→ └──────────┘  └──────────┘  └──────────┘
│ k2  │  →  │                      ↑ head                     ↑ tail
│ k3  │  →  │
└─────┴─────┘
```

- **哈希表** → O(1) 查找节点
- **双向链表** → O(1) 移动节点到头部 / 删除尾部

### 1.2 C++ 实现

```cpp
class LRUCache {
    int cap_;
    list<pair<int, int>> cache_;          // key-value 链表
    unordered_map<int, decltype(cache_)::iterator> map_; // key → 链表位置
public:
    LRUCache(int capacity) : cap_(capacity) {}

    int get(int key) {
        auto it = map_.find(key);
        if (it == map_.end()) return -1;
        // 移到链表头部
        cache_.splice(cache_.begin(), cache_, it->second);
        return it->second->second;
    }

    void put(int key, int value) {
        if (auto it = map_.find(key); it != map_.end()) {
            it->second->second = value;
            cache_.splice(cache_.begin(), cache_, it->second);
            return;
        }
        if (cache_.size() == cap_) {
            map_.erase(cache_.back().first);
            cache_.pop_back();
        }
        cache_.push_front({key, value});
        map_[key] = cache_.begin();
    }
};
```

**关键操作**：`splice` 是 O(1) 的链表节点搬移——不分配/释放内存，只改指针。

### 1.3 时间复杂度

| 操作 | 复杂度 |
|------|:---:|
| `get` | O(1) |
| `put` | O(1) |

---

## 2. LFU Cache——最不经常使用

**LFU** = Least Frequently Used。当缓存满时，淘汰**访问频率最低**的数据。

### 2.1 数据结构

```
频率计数映射 + 每个频率一个双向链表

freq=1: [K1] ⇄ [K2] ⇄ [K3]
freq=2: [K4] ⇄ [K5]
freq=3: [K6]

minFreq = 1（最低频率，O(1) 淘汰）
```

### 2.2 核心思路

```cpp
class LFUCache {
    int cap_, minFreq_ = 0;
    unordered_map<int, list<int>::iterator> keyPos_;   // key → 在频率链表中的位置
    unordered_map<int, int> keyFreq_;                   // key → 频率
    unordered_map<int, list<int>> freqKeys_;            // 频率 → key 链表
    // ... 细节省略，核心是维护 minFreq_ 和双向链表
};
```

| 操作 | 复杂度 |
|------|:---:|
| `get` | O(1) |
| `put` | O(1) |

---

## 3. 并查集（Union-Find / Disjoint Set）

### 3.1 核心操作

```cpp
class UnionFind {
    vector<int> parent, rank;  // rank = 树的高度上界
public:
    UnionFind(int n) : parent(n), rank(n, 0) {
        for (int i = 0; i < n; ++i) parent[i] = i;
    }

    // 查找 + 路径压缩
    int find(int x) {
        if (parent[x] != x)
            parent[x] = find(parent[x]);  // 路径压缩
        return parent[x];
    }

    // 合并 + 按秩合并
    void unite(int x, int y) {
        int rx = find(x), ry = find(y);
        if (rx == ry) return;
        if (rank[rx] < rank[ry]) parent[rx] = ry;
        else if (rank[rx] > rank[ry]) parent[ry] = rx;
        else { parent[ry] = rx; rank[rx]++; }
    }
};
```

**两种优化的效果**：

| 优化 | 作用 |
|------|------|
| 路径压缩 | find 时将路径上所有节点直接挂到根，加速后续查找 |
| 按秩合并 | 总是把矮树挂到高树上，避免树变高 |

两者结合 → **均摊 O(α(n))**，其中 α 是阿克曼函数的反函数，实际中 ≤ 4。

### 3.2 应用场景

- **连通分量**：图中两个节点是否连通
- **最小生成树 Kruskal**（见 [09 图算法](./09-graph-algorithms.md)）
- **朋友圈 / 岛屿数量变体**
- **等式方程的可满足性**（LeetCode 990）

---

## 4. 单调栈 / 单调队列

### 4.1 单调栈——找下一个更大/更小元素

```
问题：找到数组中每个元素的下一个更大元素

arr = [2, 1, 2, 4, 3]
       ↓  ↓  ↓  ↓  ↓
res = [4, 2, 4,-1,-1]

单调递减栈（从底到顶递减）:
遍历 2: push 2       → 栈 [2]
遍历 1: push 1       → 栈 [2, 1]       (1 < 2，递减)
遍历 2: 1 < 2 → pop → 栈 [2]           ans[1]=2
        push 2       → 栈 [2, 2]
遍历 4: 2 < 4 → pop → 栈 [2]           ans[2]=4
        2 < 4 → pop → 栈 []             ans[0]=4
        push 4       → 栈 [4]
遍历 3: push 3       → 栈 [4, 3]
```

```cpp
vector<int> nextGreaterElement(vector<int>& nums) {
    int n = nums.size();
    vector<int> ans(n, -1);
    stack<int> st;  // 存索引
    for (int i = 0; i < n; ++i) {
        while (!st.empty() && nums[st.top()] < nums[i]) {
            ans[st.top()] = nums[i];
            st.pop();
        }
        st.push(i);
    }
    return ans;
}
```

### 4.2 单调队列——滑动窗口最大值

```
问题：数组 [1,3,-1,-3,5,3,6,7]，窗口大小 k=3，求每个窗口的最大值

单调递减队列（存索引，队首永远是当前窗口最大值）:
窗口 [1,3,-1]: deque [1(3), 2(-1)]      → max=3
窗口 [3,-1,-3]: deque [1(3), 2(-1), 3(-3)] → max=3
窗口 [-1,-3,5]: 去掉索引<1的 → deque [4(5)] → max=5
...
```

```cpp
vector<int> maxSlidingWindow(vector<int>& nums, int k) {
    vector<int> ans;
    deque<int> dq;  // 存索引，单调递减
    for (int i = 0; i < nums.size(); ++i) {
        while (!dq.empty() && dq.front() <= i - k) dq.pop_front();
        while (!dq.empty() && nums[dq.back()] <= nums[i]) dq.pop_back();
        dq.push_back(i);
        if (i >= k - 1) ans.push_back(nums[dq.front()]);
    }
    return ans;
}
```

---

## 5. 位图（Bitmap）与概率数据结构

### 5.1 位图

用 1 bit 表示一个元素是否存在。适合**稠密整数集合**。

```
bitmap 存储 {1, 3, 5, 7}:
位: 0 1 2 3 4 5 6 7
   [0,1,0,1,0,1,0,1]
字节: [01010101] = 0x55

空间：n 个元素只需 n/8 字节
```

### 5.2 HyperLogLog——基数估计

**问题**：统计 UV（独立访客），传统方式需要 O(n) 空间存所有 ID。HyperLogLog 只需 **12 KB**，误差约 0.81%。

**原理**：利用哈希值的"前导零数量"来估计基数。前导零越长 → 样本越多 → 基数越大。

```
哈希值: h(x)
ρ(x) = 哈希值二进制表示中第一个 1 的位置
基数估计 ≈ 常数 × 2^(max(ρ(x)))

Redis PFADD / PFCOUNT 用的就是 HyperLogLog
```

### 5.3 Count-Min Sketch——频率估计

**问题**：统计高频元素（如 Top-K 热搜词），但数据量太大不能全存。

**原理**：二维计数器数组 + 多个哈希函数。更新时对应的计数都 +1；查询时取最小值。

```
    h1 h2 h3
k1: +1 +1 +1
k2:    +1    +1

查询 k1 = min(Count[h1(k1)], Count[h2(k1)], Count[h3(k1)])
→ 只会高估不会低估
```

---

## 小结

| 结构 | 用途 | 核心技巧 |
|------|------|----------|
| LRU | 缓存淘汰 | 哈希表 + 双向链表，`splice` O(1) 搬移 |
| LFU | 按频率淘汰 | 频率计数 + freq→list 映射 |
| 并查集 | 连通性判断 | 路径压缩 + 按秩合并，均摊 O(α(n)) |
| 单调栈 | 下一个更大元素 | 维护递减/递增栈 |
| 单调队列 | 滑动窗口极值 | 双端队列维护窗口内单调性 |
| 位图 | 整数集合 | 1 bit/元素 |
| HyperLogLog | 基数估计 | 前导零估计，12KB ≈ 0.81% 误差 |
| Count-Min Sketch | 频率估计 | 多哈希 + 取 min，只高估 |

下一篇 [06 十大排序算法](./06-sorting-algorithms.md)。
