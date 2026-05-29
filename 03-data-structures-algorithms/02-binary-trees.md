# 02 — 二叉树、BST、AVL、红黑树

> 关联篇目：[03 B/B+/Huffman/Trie/跳表](./03-advanced-trees.md) | [第一章 07 STL 容器（红黑树详细）](../01-cpp-basics/07-stl-containers-deep-dive.md)

---

## 1. 二叉树基础

### 1.1 基本概念

```
        A           ← 根节点
       / \
      B   C         ← A 的左/右孩子
     / \   \
    D   E   F       ← 叶子节点：D, E, F
```

| 术语 | 含义 |
|------|------|
| 深度 | 根到该节点的边数（根深度 = 0） |
| 高度 | 该节点到最深叶子的边数 |
| 度 | 节点的孩子数（二叉树最大度 = 2） |

### 1.2 满二叉树 vs 完全二叉树

```
满二叉树（Full）:           完全二叉树（Complete）:
        A                        A
       / \                      / \
      B   C                    B   C
     / \ / \                  / \ /
    D  E F  G                D  E F
每层都满                   最底层左边连续，右边可有空缺
节点数 = 2^(h+1) - 1       适合用数组存储
```

**完全二叉树用数组存储**：

```
         A(0)
       /      \
     B(1)     C(2)
    /   \    /   \
  D(3) E(4) F(5) G(6)

索引关系（从 0 开始）：
parent(i) = (i - 1) / 2
left(i)   = 2 * i + 1
right(i)  = 2 * i + 2
```

### 1.3 二叉树遍历（四种）

```cpp
struct TreeNode { int val; TreeNode *left, *right; };

// 前序：根 → 左 → 右
void preorder(TreeNode* root) {
    if (!root) return;
    visit(root);
    preorder(root->left);
    preorder(root->right);
}

// 中序：左 → 根 → 右  ← BST 中序遍历 = 有序序列！
void inorder(TreeNode* root) {
    if (!root) return;
    inorder(root->left);
    visit(root);
    inorder(root->right);
}

// 后序：左 → 右 → 根  ← 用于先处理孩子再处理父节点的场景
void postorder(TreeNode* root) {
    if (!root) return;
    postorder(root->left);
    postorder(root->right);
    visit(root);
}

// 层序：逐层从左到右（BFS，用队列）
void levelorder(TreeNode* root) {
    if (!root) return;
    std::queue<TreeNode*> q;
    q.push(root);
    while (!q.empty()) {
        auto* node = q.front(); q.pop();
        visit(node);
        if (node->left)  q.push(node->left);
        if (node->right) q.push(node->right);
    }
}
```

---

## 2. 二叉搜索树（BST）——左小右大

```
        8
       / \
      3   10
     / \    \
    1   6    14
       / \   /
      4   7 13
```

**性质**：任意节点的**左子树全小于它，右子树全大于它**。

| 操作 | 平均 | 最坏 |
|------|:---:|:---:|
| 查找 | O(log n) | O(n) |
| 插入 | O(log n) | O(n) |
| 删除 | O(log n) | O(n) |

最坏情况：插入顺序为升序/降序 → 退化成链表。这就是**平衡树**要解决的问题。

### 2.1 删除的三种情况

```
删除节点 X：
Case 1: X 是叶子 → 直接删除
Case 2: X 只有一个孩子 → 用孩子替代 X
Case 3: X 有两个孩子 → 用后继（右子树最小）或前驱（左子树最大）替换 X，再删除后继
```

---

## 3. AVL 树——严格的平衡

**平衡因子** = 左子树高度 - 右子树高度。AVL 要求每节点 `|bf| ≤ 1`。

### 3.1 四种旋转

```
LL（右旋）:           RR（左旋）:           LR（左-右双旋）:      RL（右-左双旋）:
     A                    A                    A                    A
    /                    / \                  / \                  / \
   B          →         B   C    先左旋 B     B   C    先右旋 B     B   C
  /                      \                 /                    \
 C                        C               C                      C
```

### 3.2 AVL vs 红黑树

| 特性 | AVL | 红黑树 |
|------|-----|--------|
| 平衡条件 | 严格：\|bf\| ≤ 1 | 宽松：最长路径 ≤ 2×最短 |
| 树高 | 更低（查找更快） | 更高（查找稍慢） |
| 插入旋转 | 最多 O(log n) | 最多 2-3 次 |
| 删除旋转 | 最多 O(log n) | 最多 3 次 |
| 适用场景 | 读多写少 | 读写均衡 / 写频繁 |

**C++ STL `std::map` / `std::set` 用红黑树**，原因就是插入删除更高效。红黑树的详细插入/删除操作见 [第一章 07](../01-cpp-basics/07-stl-containers-deep-dive.md)。

---

## 4. 二叉树的非递归遍历（面试高频）

所有递归都能用栈模拟：

```cpp
// 中序非递归
vector<int> inorderIterative(TreeNode* root) {
    vector<int> result;
    stack<TreeNode*> stk;
    TreeNode* curr = root;
    while (curr || !stk.empty()) {
        while (curr) {          // 一路向左
            stk.push(curr);
            curr = curr->left;
        }
        curr = stk.top(); stk.pop();
        result.push_back(curr->val);
        curr = curr->right;     // 转向右
    }
    return result;
}

// 前序非递归（根左右）
vector<int> preorderIterative(TreeNode* root) {
    if (!root) return {};
    vector<int> result;
    stack<TreeNode*> stk;
    stk.push(root);
    while (!stk.empty()) {
        auto* node = stk.top(); stk.pop();
        result.push_back(node->val);
        if (node->right) stk.push(node->right);  // 右先入栈
        if (node->left)  stk.push(node->left);   // 左后入栈
    }
    return result;
}

// 后序非递归（左右根）= 前序（根右左）的逆序
vector<int> postorderIterative(TreeNode* root) {
    if (!root) return {};
    vector<int> result;
    stack<TreeNode*> stk;
    stk.push(root);
    while (!stk.empty()) {
        auto* node = stk.top(); stk.pop();
        result.push_back(node->val);
        if (node->left)  stk.push(node->left);
        if (node->right) stk.push(node->right);
    }
    reverse(result.begin(), result.end());  // 反转即后序
    return result;
}
```

---

## 小结

| 结构 | 平衡方式 | 查找 | 插入 | 删除 | 特点 |
|------|----------|:---:|:---:|:---:|------|
| BST | 无 | O(log n)→O(n) | 同左 | 同左 | 最坏退化链表 |
| AVL | 严格（bf≤1） | O(log n) | O(log n) | O(log n) | 旋转代价大，读快 |
| 红黑树 | 宽松（颜色） | O(log n) | O(log n) | O(log n) | 旋转少，写快 |

下一篇 [03 B/B+ 树、Huffman、Trie、跳表](./03-advanced-trees.md)。
