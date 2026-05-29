# 10 — 字符串算法：KMP、Rabin-Karp、Manacher、Trie + AC 自动机

> 关联篇目：[03 字典树（Trie）](./03-advanced-trees.md) | [08 算法范式](./08-algorithm-paradigms.md)

---

## 1. 字符串匹配问题

**输入**：文本串 `T[n]`，模式串 `P[m]`
**输出**：P 在 T 中出现的位置

| 算法 | 时间 | 空间 | 核心思想 |
|------|:---:|:---:|------|
| 暴力 | O(n×m) | O(1) | 逐个比较 |
| KMP | O(n+m) | O(m) | 前缀函数 / next 数组，避免回溯 |
| Rabin-Karp | O(n+m) 平均 | O(1) | 滚动哈希 |
| Boyer-Moore | O(n/m) 最好 | O(k) | 从右往左匹配 + 坏字符规则 |

---

## 2. KMP——永不回退的指针

### 2.1 核心：`next` 数组（前缀函数）

`next[i]` = `P[0...i]` 的最长相等前后缀长度。

```
P = "ababc"
i=0: "a"       → next=0（单字符，无前后缀）
i=1: "ab"      → next=0（a≠b）
i=2: "aba"     → next=1（前缀"a"=后缀"a"）
i=3: "abab"    → next=2（前缀"ab"=后缀"ab"）
i=4: "ababc"   → next=0

next = [0, 0, 1, 2, 0]
```

### 2.2 匹配过程

```cpp
vector<int> kmp(const string& text, const string& pattern) {
    int n = text.size(), m = pattern.size();
    if (m == 0) return {};

    // 构建 next 数组
    vector<int> next(m, 0);
    for (int i = 1, j = 0; i < m; ++i) {
        while (j > 0 && pattern[i] != pattern[j])
            j = next[j - 1];           // 回退到上一个可匹配的前缀
        if (pattern[i] == pattern[j])
            j++;
        next[i] = j;
    }

    // 匹配
    vector<int> matches;
    for (int i = 0, j = 0; i < n; ++i) {
        while (j > 0 && text[i] != pattern[j])
            j = next[j - 1];           // 利用 next 跳转
        if (text[i] == pattern[j])
            j++;
        if (j == m) {
            matches.push_back(i - m + 1);
            j = next[j - 1];           // 继续找下一处匹配
        }
    }
    return matches;
}
```

**为什么 O(n+m)**：`j` 值在整个过程中最多增加 n 次（每次 `j++`），每次回退 `j = next[j-1]` 至少减少 1 → 总回退次数 ≤ 总增加次数 ≤ n。匹配文本指针 `i` 从来不回退。

---

## 3. Rabin-Karp——用哈希做匹配

### 3.1 滚动哈希

```cpp
vector<int> rabinKarp(const string& text, const string& pattern) {
    int n = text.size(), m = pattern.size();
    const int BASE = 256, MOD = 1e9 + 7;

    // 计算 BASE^(m-1) % MOD
    long long power = 1;
    for (int i = 0; i < m - 1; ++i) power = (power * BASE) % MOD;

    // 计算 pattern 的哈希
    long long pHash = 0;
    for (char c : pattern) pHash = (pHash * BASE + c) % MOD;

    // 滑动窗口计算 text 的哈希
    long long tHash = 0;
    vector<int> matches;
    for (int i = 0; i < n; ++i) {
        if (i >= m)  // 移除窗口最左字符的贡献
            tHash = (tHash - text[i - m] * power % MOD + MOD) % MOD;
        tHash = (tHash * BASE + text[i]) % MOD;
        if (i >= m - 1 && tHash == pHash) {
            // 哈希匹配 → 逐字符验证（防止哈希碰撞）
            if (text.substr(i - m + 1, m) == pattern)
                matches.push_back(i - m + 1);
        }
    }
    return matches;
}
```

**优势**：容易扩展到多模式匹配（多个 pattern 预计算哈希），且常数小。
**劣势**：需要验证哈希碰撞。

---

## 4. Manacher——最长回文子串（O(n)）

### 4.1 预处理：插入分隔符

```
原串:  "abba"
预处理: "#a#b#b#a#"    → 所有回文串变成奇数长度
```

### 4.2 核心算法

```cpp
string longestPalindrome(string s) {
    // 预处理
    string t = "#";
    for (char c : s) { t += c; t += '#'; }

    int n = t.size();
    vector<int> radius(n, 0);  // radius[i] = 以 i 为中心的回文半径
    int center = 0, right = 0;  // 最右回文边界
    int maxLen = 0, maxCenter = 0;

    for (int i = 0; i < n; ++i) {
        // 利用对称性初始化
        if (i < right)
            radius[i] = min(radius[2 * center - i], right - i);

        // 中心扩展
        while (i - radius[i] - 1 >= 0 && i + radius[i] + 1 < n &&
               t[i - radius[i] - 1] == t[i + radius[i] + 1])
            radius[i]++;

        // 更新最右边界
        if (i + radius[i] > right) {
            center = i;
            right = i + radius[i];
        }

        if (radius[i] > maxLen) {
            maxLen = radius[i];
            maxCenter = i;
        }
    }

    int start = (maxCenter - maxLen) / 2;
    return s.substr(start, maxLen);
}
```

**O(n) 的关键**：利用已计算的回文半径 + 对称性，避免重复扩展。每个字符最多被扩展一次。

---

## 5. AC 自动机——多模式串匹配

**Trie + KMP 的 fail 指针 = AC 自动机**。一次扫描文本，同时匹配多个模式串。

### 5.1 结构

```
模式串: {"he", "she", "his", "hers"}

Trie + fail 指针（虚线）:
               root
              /    \
          h(s)      s(1)
         /   \       \
        e(2)  i(3)    h(4)
       /       \        \
      r(8)      s(9)     e(5)
     /                    \
    s(9)                   r(6)
                            \
                             s(7)

fail 指针：指向最长可匹配后缀对应的节点
```

### 5.2 构建步骤

1. 将模式串建成 Trie
2. BFS 构建 fail 指针（类似 KMP 的 next，但针对多模式）
3. 扫描文本时，沿 fail 指针跳转

```cpp
struct AhoCorasick {
    struct Node {
        Node* children[26] = {};
        Node* fail = nullptr;
        bool isEnd = false;
    };
    Node root_;

    void insert(const string& word) {
        Node* node = &root_;
        for (char c : word) {
            int idx = c - 'a';
            if (!node->children[idx]) node->children[idx] = new Node();
            node = node->children[idx];
        }
        node->isEnd = true;
    }

    void buildFail() {
        queue<Node*> q;
        for (auto* child : root_.children)
            if (child) { child->fail = &root_; q.push(child); }

        while (!q.empty()) {
            Node* curr = q.front(); q.pop();
            for (int i = 0; i < 26; ++i) {
                Node* child = curr->children[i];
                if (!child) continue;

                Node* f = curr->fail;
                while (f != &root_ && !f->children[i])
                    f = f->fail;
                if (f->children[i]) f = f->children[i];
                child->fail = f;

                // 如果 fail 指向的节点是单词结尾，当前节点也是
                child->isEnd |= f->isEnd;
                q.push(child);
            }
        }
    }

    // 查询文本中有哪些模式串出现
    void query(const string& text) {
        Node* node = &root_;
        for (char c : text) {
            int idx = c - 'a';
            while (node != &root_ && !node->children[idx])
                node = node->fail;
            if (node->children[idx])
                node = node->children[idx];
            if (node->isEnd) {
                // 匹配到了某个模式串
            }
        }
    }
};
```

### 5.3 应用

- **敏感词过滤**：多个关键词一次扫描
- **DNA 序列匹配**：多个基因片段匹配
- **搜索引擎分词**：词典匹配

---

## 6. 字符串算法选择

```
单模式匹配
├─ 简单场景 → KMP（O(n+m)，标准解法）
├─ 追求常数 → Rabin-Karp（滚动哈希）
└─ 大字母表 → Boyer-Moore（实际常用）

多模式匹配 → AC 自动机（Trie + fail 指针）

回文问题 → Manacher（O(n) 最长回文子串）

前缀/字典 → Trie（O(L) 插入/查找）
```

---

## 小结

| 算法 | 问题 | 时间 | 核心 |
|------|------|:---:|------|
| KMP | 单模式匹配 | O(n+m) | next 数组，i 不回溯 |
| Rabin-Karp | 单/多模式匹配 | O(n+m) | 滚动哈希 |
| Manacher | 最长回文子串 | O(n) | 对称性 + 中心扩展 |
| AC 自动机 | 多模式匹配 | O(n + 总匹配数) | Trie + fail 指针 |

下一篇 [11 复杂度分析与主定理](./11-complexity-analysis.md)。
