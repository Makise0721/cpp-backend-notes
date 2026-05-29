# 07 — 查找算法、二分变体、Top-K 问题、海量数据处理

> 关联篇目：[06 排序算法](./06-sorting-algorithms.md) | [04 哈希表](./04-graph-hash-bloom.md) | [02 二叉树/BST](./02-binary-trees.md)

---

## 1. 查找算法一览

| 算法 | 数据结构 | 平均 | 最坏 | 前提 |
|------|----------|:---:|:---:|------|
| 顺序查找 | 任意 | O(n) | O(n) | 无 |
| 二分查找 | 有序数组 | O(log n) | O(log n) | 有序 + 随机访问 |
| BST 查找 | BST | O(log n) | O(n) | BST 性质 |
| 哈希查找 | 哈希表 | O(1) | O(n) | 好哈希函数 |
| 跳表查找 | 跳表 | O(log n) | O(n) | 概率平衡 |

---

## 2. 二分查找——边界条件最容易写错

### 2.1 标准二分（闭区间 `[l, r]`）

```cpp
int binarySearch(vector<int>& nums, int target) {
    int l = 0, r = nums.size() - 1;
    while (l <= r) {
        int m = l + (r - l) / 2;  // 防溢出
        if (nums[m] == target) return m;
        else if (nums[m] < target) l = m + 1;
        else                       r = m - 1;
    }
    return -1;
}
```

### 2.2 二分查找左边界（第一个等于 target）

```cpp
int leftBound(vector<int>& nums, int target) {
    int l = 0, r = nums.size() - 1;
    while (l <= r) {
        int m = l + (r - l) / 2;
        if (nums[m] >= target) r = m - 1;  // 找到后继续往左找
        else                   l = m + 1;
    }
    // l 指向第一个 >= target 的位置
    return (l < nums.size() && nums[l] == target) ? l : -1;
}
```

**记忆关键**：`>=` 时收缩右边界 → `l` 最终停留在第一个满足条件的位置。

### 2.3 二分查找右边界（最后一个等于 target）

```cpp
int rightBound(vector<int>& nums, int target) {
    int l = 0, r = nums.size() - 1;
    while (l <= r) {
        int m = l + (r - l) / 2;
        if (nums[m] <= target) l = m + 1;  // 找到后继续往右找
        else                   r = m - 1;
    }
    return (r >= 0 && nums[r] == target) ? r : -1;
}
```

### 2.4 二分查找变体速查

| 变体 | 条件 | 返回值 |
|------|------|--------|
| 查找 target | `==` 返回 | mid |
| 左边界 | `>=` 时 `r=m-1` | `l`（第一个 ≥） |
| 右边界 | `<=` 时 `l=m+1` | `r`（最后一个 ≤） |
| 第一个 ≥ target | `>=` 时 `r=m-1` | `l` |
| 最后一个 < target | `>=` 时 `r=m-1` | `r`（`l-1`） |
| 搜索插入位置 | 同左边界 | `l` |

**三个注意点**：
1. `m = l + (r - l) / 2` 防止 `(l + r)` 溢出
2. 循环条件是 `l <= r`（闭区间）还是 `l < r`（左闭右开）——保持一致
3. 边界检查：`l < n` 且 `r >= 0`

---

## 3. Top-K 问题

### 3.1 小顶堆解法（O(n log k)）

**维护一个大小为 k 的小顶堆**，遍历所有元素。堆顶是当前第 K 大。遍历完后堆中即为 Top-K 大。

```cpp
vector<int> topK(vector<int>& nums, int k) {
    priority_queue<int, vector<int>, greater<int>> minHeap;
    for (int x : nums) {
        minHeap.push(x);
        if (minHeap.size() > k) minHeap.pop();
    }
    vector<int> res;
    while (!minHeap.empty()) { res.push_back(minHeap.top()); minHeap.pop(); }
    return res;
}
```

- 时间：O(n log k)
- 空间：O(k)
- 适合：**k 很小**（如 Top-10）

### 3.2 快速选择（Quick Select，O(n) 平均）

```cpp
int quickSelect(vector<int>& nums, int l, int r, int k) {
    if (l == r) return nums[l];
    int pivot = nums[r], i = l;
    for (int j = l; j < r; ++j)
        if (nums[j] <= pivot) swap(nums[i++], nums[j]);
    swap(nums[i], nums[r]);

    int cnt = i - l + 1;  // 包括 pivot 在内的左半部分元素数
    if (cnt == k) return nums[i];
    else if (cnt > k) return quickSelect(nums, l, i - 1, k);
    else              return quickSelect(nums, i + 1, r, k - cnt);
}
```

- 时间：O(n) 平均，O(n²) 最坏
- 空间：O(log n) 递归栈
- 适合：**k 不固定，追求平均最优**

| 方法 | 时间 | 空间 | 适用 |
|------|:---:|:---:|------|
| 全局排序 | O(n log n) | O(1) | k ≈ n |
| 小顶堆 | O(n log k) | O(k) | k 很小 |
| 快速选择 | O(n) 平均 | O(log n) | k 不小，追求平均性能 |

---

## 4. 海量数据处理——当内存放不下时

### 4.1 场景与策略

| 场景 | 策略 | 内存需求 |
|------|------|:---:|
| 10 亿个整数去重 | 位图（2³² bit = 512 MB） | 512 MB |
| 10 亿个 URL Top-100 | Hash 分片 → 每片 Top-K → 合并 | 可控 |
| 100 亿个数排序 | **外部排序**（多路归并） | 磁盘 I/O |
| 10 亿个 IP 统计 UV | HyperLogLog | 12 KB |
| 搜索热词 Top-K | Count-Min Sketch + 堆 | 可控 |

### 4.2 外部排序

```
数据量超过内存 → 分块排序 + 多路归并：

Phase 1（分块内排序）:
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│ Chunk 1 │  │ Chunk 2 │  │ Chunk 3 │  │ Chunk N │
│ 排序 →   │  │ 排序 →   │  │ 排序 →   │  │ 排序 →   │
│ file_1  │  │ file_2  │  │ file_3  │  │ file_N  │
└─────────┘  └─────────┘  └─────────┘  └─────────┘

Phase 2（K 路归并）:
file_1, file_2, ..., file_N
    │        │            │
    └────────┼────────────┘
             ▼
     ┌──────────────┐
     │  K 路归并     │ ← 小顶堆选最小值
     │ （每次从 K   │
     │  个文件中选  │
     │  最小元素）   │
     └──────┬───────┘
            ▼
     ┌──────────────┐
     │  最终有序文件  │
     └──────────────┘
```

### 4.3 Hash 分片法

**核心思想**：用哈希函数把数据分到若干小文件，保证相同数据一定在同一文件，然后每个小文件可以单机处理。

```
原始数据（1 TB）
  │ hash(key) % 1000
  ├→ file_000（约 1 GB）→ 单机处理
  ├→ file_001（约 1 GB）→ 单机处理
  ├→ ...
  └→ file_999（约 1 GB）→ 单机处理
```

---

## 小结

| 问题 | 方法 | 关键 |
|------|------|------|
| 有序数组查找 | 二分查找 O(log n) | 注意左右边界写法 |
| Top-K 小 k | 小顶堆 O(n log k) | 堆大小始终保持 k |
| Top-K 大 k | 快速选择 O(n) | partition 只递归一边 |
| 海量数据排序 | 外部排序 | 分块 + 多路归并 |
| 海量数据聚合 | Hash 分片 | 相同 key 进同一文件 |
| UV 统计 | HyperLogLog | 12 KB，0.81% 误差 |

下一篇 [08 算法范式：递归/分治/贪心/DP/回溯/滑动窗口](./08-algorithm-paradigms.md)。
