# 01 — 线性结构：数组、链表、栈、队列、循环队列

> 关联篇目：[05 LRU/LFU/并查集](./05-compound-structures.md) | [06 排序算法](./06-sorting-algorithms.md) | [07 查找](./07-searching-topk.md)

---

## 1. 数组——连续内存的王者

### 1.1 结构与特点

```
数组内存布局（int arr[5]）:
┌───┬───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │ 4 │  ← 连续内存，每个元素等距
└───┴───┴───┴───┴───┘
地址: base + index × sizeof(int)
```

| 操作 | 复杂度 | 说明 |
|------|:---:|------|
| 随机访问 `arr[i]` | O(1) | 地址 = base + i × stride |
| 尾部插入 | O(1) | 均摊（动态数组扩容时 O(n)） |
| 中间插入/删除 | O(n) | 需要移动后续所有元素 |
| 查找 | O(n) | 无序时线性扫描 |

### 1.2 动态数组（vector）扩容

C++ `std::vector` 扩容系数通常是 1.5x（MSVC）或 2x（GCC），详细分析见 [第一章 07 STL 容器](../01-cpp-basics/07-stl-containers-deep-dive.md)。

---

## 2. 链表——离散节点，灵活插入

### 2.1 单链表

```
head → [data|next] → [data|next] → [data|next] → nullptr
```

```cpp
struct ListNode {
    int val;
    ListNode* next;
    ListNode(int x) : val(x), next(nullptr) {}
};
```

| 操作 | 复杂度 |
|------|:---:|
| 访问第 k 个 | O(n) |
| 头部插入/删除 | O(1) |
| 已知位置后插入/删除 | O(1) |
| 查找 | O(n) |

### 2.2 双链表

```
head ⇄ [prev|data|next] ⇄ [prev|data|next] ⇄ [prev|data|next] → nullptr
```

比单链表多了前驱指针，支持双向遍历 + O(1) 删除已知节点（不需要找前驱）。代价是每节点多一个指针（8 字节）。

### 2.3 循环链表

尾节点的 `next` 指向头节点（单向循环）或头尾互指（双向循环）。适合环形缓冲、轮转调度等场景。

### 2.4 链表操作的经典技巧

```cpp
// 1. 哑节点（dummy head）——简化边界处理
ListNode dummy(0);
dummy.next = head;
// 现在"删除头节点"和"删除中间节点"是一样的代码

// 2. 快慢指针——找中点、判环
ListNode *slow = head, *fast = head;
while (fast && fast->next) {
    slow = slow->next;        // 每次走 1 步
    fast = fast->next->next;  // 每次走 2 步
}
// slow 现在指向中点

// 3. 反转链表（迭代）
ListNode* reverse(ListNode* head) {
    ListNode *prev = nullptr, *curr = head;
    while (curr) {
        ListNode* next = curr->next;  // 保存后继
        curr->next = prev;            // 反转指针
        prev = curr;
        curr = next;
    }
    return prev;
}
```

---

## 3. 栈——后进先出（LIFO）

```
push(1)  push(2)  push(3)  pop()
  │        │        │        │
  ▼        ▼        ▼        ▼
┌───┐    ┌───┐    ┌───┐    ┌───┐
│   │    │   │    │ 3 │    │   │
│   │    │ 2 │    │ 2 │    │ 2 │
│ 1 │    │ 1 │    │ 1 │    │ 1 │
└───┘    └───┘    └───┘    └───┘
```

| 操作 | 复杂度 |
|------|:---:|
| `push` | O(1) |
| `pop` | O(1) |
| `top` | O(1) |

### 3.1 经典应用

- **括号匹配**：左括号入栈，右括号出栈匹配
- **函数调用栈**：递归的底层机制
- **表达式求值**：中缀转后缀 + 后缀求值
- **撤销操作**：Ctrl+Z 的历史记录
- **单调栈**：见 [05 复合结构](./05-compound-structures.md)

---

## 4. 队列——先进先出（FIFO）

```
enqueue(1)  enqueue(2)  enqueue(3)  dequeue()
    │           │           │           │
    ▼           ▼           ▼           ▼
入队 → ┌───┬───┬───┐ → 出队
      │ 3 │ 2 │ 1 │
      └───┴───┴───┘
```

| 操作 | 复杂度 |
|------|:---:|
| `enqueue`（入队） | O(1) |
| `dequeue`（出队） | O(1) |
| `front` | O(1) |

### 4.1 循环队列——固定大小，避免搬移

普通队列用数组实现时，出队只会移动 `front` 指针，导致数组前面空出来浪费。循环队列让 `rear` 指针绕回数组开头：

```
循环队列（容量 6，当前有 3 个元素）:
索引:  0   1   2   3   4   5
     ┌───┬───┬───┬───┬───┬───┐
     │   │   │ a │ b │ c │   │
     └───┴───┴───┴───┴───┴───┘
            rear↑       front↑
```

```cpp
class CircularQueue {
    std::vector<int> data_;
    int front_ = 0, rear_ = 0, size_ = 0, cap_;
public:
    CircularQueue(int k) : data_(k), cap_(k) {}

    bool enqueue(int val) {
        if (isFull()) return false;
        data_[rear_] = val;
        rear_ = (rear_ + 1) % cap_;  // 绕回
        size_++;
        return true;
    }

    bool dequeue() {
        if (isEmpty()) return false;
        front_ = (front_ + 1) % cap_;
        size_--;
        return true;
    }

    int Front() { return isEmpty() ? -1 : data_[front_]; }
    int Rear()  { return isEmpty() ? -1 : data_[(rear_ - 1 + cap_) % cap_]; }
    bool isEmpty() { return size_ == 0; }
    bool isFull()  { return size_ == cap_; }
};
```

**区分空和满**：维护 `size` 变量是最清晰的方式。也可以用"牺牲一个位置"（`(rear + 1) % cap == front` 表示满）或额外标志位。

---

## 5. 双端队列（Deque）——两头都能操作

```
push_front → ┌───┬───┬───┬───┬───┐ ← push_back
             │ 0 │ 1 │ 2 │ 3 │ 4 │
             └───┴───┴───┴───┴───┘
pop_front ←                      → pop_back
```

双端均可 O(1) 插入删除。C++ `std::deque` 底层实现为中控器 + 分段数组（见 [第一章 07](../01-cpp-basics/07-stl-containers-deep-dive.md)）。

---

## 6. 线性结构选型速查

| 场景 | 推荐结构 |
|------|----------|
| 频繁随机访问 | 数组（vector） |
| 频繁头部插入/删除 | 链表 / deque |
| 后进先出 | 栈 |
| 先进先出 | 队列 |
| 固定大小的 FIFO | 循环队列 |
| 双端操作 | deque |
| 滑动窗口极值 | 单调队列（见 [05](./05-compound-structures.md)） |

---

## 小结

| 结构 | 核心操作 | 关键技巧 |
|------|----------|----------|
| 数组 | O(1) 随机访问 | 扩容均摊分析 |
| 单链表 | O(1) 头插 | 哑节点 + 快慢指针 + 反转 |
| 双链表 | O(1) 删除已知节点 | 额外指针开销 8 字节/节点 |
| 栈 | push/pop O(1) | 括号匹配 / 单调栈 / 表达式求值 |
| 队列 | enqueue/dequeue O(1) | BFS / 消息队列 |
| 循环队列 | 固定大小 O(1) | (rear+1)%cap 绕回 |

下一篇 [02 二叉树 / BST / AVL / 红黑树](./02-binary-trees.md)。
