# 08 — 算法范式：递归、分治、贪心、动态规划、回溯、双指针/滑动窗口

> 关联篇目：[06 排序算法](./06-sorting-algorithms.md) | [07 查找与 Top-K](./07-searching-topk.md) | [11 复杂度分析](./11-complexity-analysis.md)

---

## 1. 递归——自己调用自己

### 1.1 递归三要素

1. **终止条件**（base case）——什么时候不继续递归
2. **递归关系**——大问题如何拆成小问题
3. **返回值合并**——小问题的解如何合成大问题的解

```cpp
// 经典：二叉树深度
int maxDepth(TreeNode* root) {
    if (!root) return 0;                                    // 终止条件
    int left  = maxDepth(root->left);                       // 递归子问题
    int right = maxDepth(root->right);
    return max(left, right) + 1;                            // 合并
}
```

### 1.2 递归 vs 迭代

| | 递归 | 迭代 |
|------|------|------|
| 代码 | 简洁直观 | 可能冗长 |
| 栈空间 | O(深度) | O(1) |
| 栈溢出风险 | ✅（深度大时） | ❌ |
| 可优化 | 尾递归 / 记忆化 | — |

**尾递归**：递归调用是函数的最后一步，编译器可优化为循环（不压栈）。但 C++ 不保证尾递归优化。

---

## 2. 分治（Divide and Conquer）

**三步**：分解 → 解决 → 合并。

```
归并排序 = 分治的经典:
[38,27,43,3,9,82,10]
     ↓ 分解
[38,27,43,3]  [9,82,10]
  ↓     ↓      ↓     ↓
[38,27] [43,3] [9,82] [10]
 ↓  ↓   ↓  ↓   ↓  ↓    ↓
[38][27][43][3][9][82] [10]
  ↓ 合并  ↓ 合并  ↓
[27,38][3,43][9,82] [10]
     ↓ 合并      ↓ 合并
[3,27,38,43] [9,10,82]
         ↓ 合并
[3,9,10,27,38,43,82]
```

**分治算法的 Master Theorem 分析**见 [11 复杂度分析](./11-complexity-analysis.md)。

---

## 3. 贪心——每步选局部最优

**每次都做当下看起来最好的选择，不回溯。** 贪心能奏效的前提是问题有贪心选择性质。

```cpp
// 经典：找零钱（假设硬币面额满足贪心条件，如 1,5,10,25）
int coinChange(vector<int>& coins, int amount) {
    sort(coins.rbegin(), coins.rend());  // 从大到小
    int count = 0;
    for (int coin : coins) {
        count += amount / coin;
        amount %= coin;
    }
    return amount == 0 ? count : -1;
}
```

| 能贪心 | 不能贪心（需要 DP） |
|--------|---------------------|
| 活动选择 | 0-1 背包 |
| 哈夫曼编码 | 最长公共子序列 |
| 最小生成树（Kruskal/Prim） | 最短路径（负权边） |
| Dijkstra（非负权） | 硬币找零（不满足贪心条件的面额） |

---

## 4. 动态规划（DP）——记录子问题的解

### 4.1 DP 的核心：状态定义 + 转移方程

```
DP = 递归 + 记忆化
   = 避免重复计算子问题

五步法：
1. 定义状态 dp[i] 表示什么
2. 写出状态转移方程
3. 确定初始值
4. 确定遍历顺序
5. 返回最终答案
```

### 4.2 经典例题：0-1 背包

```cpp
// dp[i][w] = 前 i 个物品，容量 w，最大价值
// dp[i][w] = max(dp[i-1][w], dp[i-1][w - weight[i]] + value[i])

int knapsack(vector<int>& wt, vector<int>& val, int W) {
    int n = wt.size();
    vector<int> dp(W + 1, 0);
    for (int i = 0; i < n; ++i)
        for (int w = W; w >= wt[i]; --w)  // 倒序：保证每件只用一次
            dp[w] = max(dp[w], dp[w - wt[i]] + val[i]);
    return dp[W];
}
```

### 4.3 DP 常见题型

| 类型 | 例题 | 关键 |
|------|------|------|
| 线性 DP | 爬楼梯、打家劫舍 | `dp[i]` 依赖 `dp[i-1]`, `dp[i-2]` |
| 背包 DP | 0-1 背包、完全背包 | 容量倒序/正序 |
| 区间 DP | 最长回文子串、矩阵连乘 | `dp[l][r]` |
| 树形 DP | 二叉树最大路径和 | 后序遍历 + 返回值 |
| 状态压缩 DP | TSP 旅行商 | 位 mask 表示状态 |

### 4.4 DP 优化

```cpp
// 空间优化：二维 → 一维（滚动数组）
// 时间优化：斜率优化、四边形不等式（进阶）

// 记忆化搜索 = 自顶向下 DP
unordered_map<int, int> memo;
int fib(int n) {
    if (n <= 1) return n;
    if (memo.count(n)) return memo[n];
    return memo[n] = fib(n-1) + fib(n-2);
}
```

---

## 5. 回溯——穷举 + 剪枝

**系统地搜索所有解空间，在遇到死胡同时回退。**

```cpp
// 全排列
void backtrack(vector<int>& nums, vector<bool>& used,
               vector<int>& path, vector<vector<int>>& res) {
    if (path.size() == nums.size()) {
        res.push_back(path);
        return;
    }
    for (int i = 0; i < nums.size(); ++i) {
        if (used[i]) continue;                 // 剪枝
        used[i] = true;
        path.push_back(nums[i]);
        backtrack(nums, used, path, res);      // 递归
        path.pop_back();                        // 回溯
        used[i] = false;
    }
}
```

### 5.1 回溯 vs DP vs 贪心

| | 回溯 | DP | 贪心 |
|------|:---:|:---:|:---:|
| 解空间 | 穷举所有 | 记录最优子结构 | 单步最优 |
| 时间复杂度 | 指数级 | 多项式 | 线性/O(n log n) |
| 典型问题 | N 皇后、全排列 | 背包、LCS | 活动选择、Huffman |

### 5.2 回溯优化技巧

- **剪枝**：提前排除不可能的解路径
- **排序 + 去重**：`if (i > start && nums[i] == nums[i-1]) continue;`
- **分支限界**：用当前最优解做界限，排除不可能更好的分支

---

## 6. 双指针 / 滑动窗口——面试最高频技巧

### 6.1 双指针

```cpp
// 有序数组两数之和
vector<int> twoSum(vector<int>& nums, int target) {
    int l = 0, r = nums.size() - 1;
    while (l < r) {
        int sum = nums[l] + nums[r];
        if (sum == target) return {l, r};
        else if (sum < target) ++l;
        else                   --r;
    }
    return {};
}

// 主要变体：
// - 快慢指针（判环、找中点）
// - 左右指针（有序数组两数和、三数和）
// - 分离指针（归并两个有序数组）
```

### 6.2 滑动窗口

**维护一个窗口 `[l, r)`，右扩满足条件，左缩优化结果。**

```cpp
// 最长无重复字符子串
int lengthOfLongestSubstring(string s) {
    unordered_set<char> window;
    int l = 0, ans = 0;
    for (int r = 0; r < s.size(); ++r) {
        while (window.count(s[r]))  // 右扩发现重复
            window.erase(s[l++]);   // 左缩直到不重复
        window.insert(s[r]);
        ans = max(ans, r - l + 1);
    }
    return ans;
}
```

### 6.3 滑动窗口模板

```cpp
int slidingWindow(string s) {
    unordered_map<char, int> window;
    int l = 0, r = 0, ans = 0;
    while (r < s.size()) {
        char c = s[r++];           // 右边界扩张
        window[c]++;               // 更新窗口状态

        while (/* 窗口需要收缩的条件 */) {
            char d = s[l++];       // 左边界收缩
            window[d]--;           // 更新窗口状态
        }

        ans = max(ans, r - l);     // 更新答案
    }
    return ans;
}
```

---

## 7. 范式选择决策树

```
优化问题？
├─ 最优子结构 + 重叠子问题 → 动态规划
├─ 贪心选择性质 → 贪心算法
└─ 无 → 回溯 / 暴力枚举

非优化问题？
├─ 搜索所有解 → 回溯
├─ 大问题可分解为相同子问题 → 分治
├─ 子串/子数组 + 连续约束 → 滑动窗口
└─ 有序数组两数和/三元组 → 双指针
```

---

## 小结

| 范式 | 复杂度 | 何时适用 | 典型题目 |
|------|--------|----------|----------|
| 递归 | 指数 | 可拆分子问题 | 树遍历 |
| 分治 | O(n log n) | 子问题独立 | 归并/快排 |
| 贪心 | O(n log n) | 贪心选择性质 | 活动选择 |
| DP | O(n²) ~ O(n) | 最优子结构 + 重叠 | 背包/LCS |
| 回溯 | O(k^n) | 穷举所有解 | N 皇后/全排列 |
| 双指针 | O(n) | 有序/两端 | 两数之和 |
| 滑动窗口 | O(n) | 连续子数组/子串 | 无重复最长子串 |

下一篇 [09 图算法](./09-graph-algorithms.md)。
