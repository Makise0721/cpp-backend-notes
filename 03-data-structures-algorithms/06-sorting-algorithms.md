# 06 — 十大排序算法：原理、稳定性与复杂度对比

> 关联篇目：[07 查找与 Top-K](./07-searching-topk.md) | [11 复杂度分析](./11-complexity-analysis.md)

---

## 1. 排序算法全景图

### 1.1 复杂度一表打尽

| 算法 | 最好 | 平均 | 最坏 | 空间 | 稳定 | 原地 |
|------|:---:|:---:|:---:|:---:|:---:|:---:|
| 冒泡 | O(n) | O(n²) | O(n²) | O(1) | ✅ | ✅ |
| 选择 | O(n²) | O(n²) | O(n²) | O(1) | ❌ | ✅ |
| 插入 | O(n) | O(n²) | O(n²) | O(1) | ✅ | ✅ |
| 希尔 | O(n log n) | O(n^1.3) | O(n²) | O(1) | ❌ | ✅ |
| 归并 | O(n log n) | O(n log n) | O(n log n) | O(n) | ✅ | ❌ |
| 快速 | O(n log n) | O(n log n) | O(n²) | O(log n) | ❌ | ✅ |
| 堆 | O(n log n) | O(n log n) | O(n log n) | O(1) | ❌ | ✅ |
| 计数 | O(n+k) | O(n+k) | O(n+k) | O(k) | ✅ | ❌ |
| 基数 | O(d(n+k)) | O(d(n+k)) | O(d(n+k)) | O(n+k) | ✅ | ❌ |
| 桶 | O(n+k) | O(n+k) | O(n²) | O(n+k) | ✅ | ❌ |

---

## 2. 比较排序（O(n²) 三兄弟）

### 2.1 冒泡排序

```cpp
void bubbleSort(vector<int>& arr) {
    int n = arr.size();
    for (int i = 0; i < n - 1; ++i) {
        bool swapped = false;
        for (int j = 0; j < n - 1 - i; ++j) {
            if (arr[j] > arr[j+1]) {
                swap(arr[j], arr[j+1]);
                swapped = true;
            }
        }
        if (!swapped) break;  // 已经有序，提前退出
    }
}
```

**稳定**：相邻元素比较，相等时不交换。

### 2.2 选择排序

```cpp
void selectionSort(vector<int>& arr) {
    int n = arr.size();
    for (int i = 0; i < n - 1; ++i) {
        int minIdx = i;
        for (int j = i + 1; j < n; ++j)
            if (arr[j] < arr[minIdx]) minIdx = j;
        swap(arr[i], arr[minIdx]);
    }
}
```

**不稳定**：`swap` 可能打破相等元素的相对顺序。例：`[5₁, 5₂, 2]` → `[2, 5₂, 5₁]`（两个 5 的相对顺序变了）。

### 2.3 插入排序

```cpp
void insertionSort(vector<int>& arr) {
    int n = arr.size();
    for (int i = 1; i < n; ++i) {
        int key = arr[i];
        int j = i - 1;
        while (j >= 0 && arr[j] > key) {
            arr[j + 1] = arr[j];
            --j;
        }
        arr[j + 1] = key;
    }
}
```

**稳定 + 对小数组极快**。快速排序在小分区上切换到插入排序就是利用了这一点。

---

## 3. 归并排序——稳定 O(n log n)

```cpp
void merge(vector<int>& arr, int l, int m, int r) {
    vector<int> L(arr.begin() + l, arr.begin() + m + 1);
    vector<int> R(arr.begin() + m + 1, arr.begin() + r + 1);
    int i = 0, j = 0, k = l;
    while (i < L.size() && j < R.size())
        arr[k++] = (L[i] <= R[j]) ? L[i++] : R[j++];
    while (i < L.size()) arr[k++] = L[i++];
    while (j < R.size()) arr[k++] = R[j++];
}

void mergeSort(vector<int>& arr, int l, int r) {
    if (l >= r) return;
    int m = l + (r - l) / 2;
    mergeSort(arr, l, m);
    mergeSort(arr, m + 1, r);
    merge(arr, l, m, r);
}
```

**稳定**：`L[i] <= R[j]` 的 `<=` 保证了左边优先——相等元素保持原序。

---

## 4. 快速排序——实际最快的通用排序

```cpp
int partition(vector<int>& arr, int l, int r) {
    int pivot = arr[r];  // 取最右为 pivot（也可以三数取中优化）
    int i = l;           // i 指向第一个 > pivot 的位置
    for (int j = l; j < r; ++j) {
        if (arr[j] <= pivot)
            swap(arr[i++], arr[j]);
    }
    swap(arr[i], arr[r]);
    return i;
}

void quickSort(vector<int>& arr, int l, int r) {
    if (l >= r) return;
    int p = partition(arr, l, r);
    quickSort(arr, l, p - 1);
    quickSort(arr, p + 1, r);
}
```

**不稳定**：partition 的交换会打破相等元素的相对顺序。

**最坏 O(n²) 的触发**：pivot 总是最值（已排序数组 + 选最右为 pivot）。优化手段：
- 随机选 pivot
- 三数取中（头、中、尾的中位数）
- 小分区切换到插入排序

---

## 5. 堆排序——原地 O(n log n)

```cpp
void heapify(vector<int>& arr, int n, int i) {
    int largest = i;
    int left = 2 * i + 1, right = 2 * i + 2;
    if (left < n && arr[left] > arr[largest])  largest = left;
    if (right < n && arr[right] > arr[largest]) largest = right;
    if (largest != i) {
        swap(arr[i], arr[largest]);
        heapify(arr, n, largest);
    }
}

void heapSort(vector<int>& arr) {
    int n = arr.size();
    for (int i = n / 2 - 1; i >= 0; --i) heapify(arr, n, i); // 建堆
    for (int i = n - 1; i > 0; --i) {
        swap(arr[0], arr[i]);         // 把最大值移到末尾
        heapify(arr, i, 0);           // 修复堆
    }
}
```

**不稳定**：堆中的交换不保证相等元素的相对顺序。

---

## 6. 希尔排序——插入排序的升级版

```cpp
void shellSort(vector<int>& arr) {
    int n = arr.size();
    for (int gap = n / 2; gap > 0; gap /= 2) {
        for (int i = gap; i < n; ++i) {
            int temp = arr[i], j;
            for (j = i; j >= gap && arr[j - gap] > temp; j -= gap)
                arr[j] = arr[j - gap];
            arr[j] = temp;
        }
    }
}
```

**不稳定**：不同 gap 组之间交换不保证稳定性。

---

## 7. 非比较排序（线性时间，但有适用范围）

### 7.1 计数排序

条件：**数据范围 k 不大**。用数组统计每个值出现次数，然后重建。

```cpp
void countingSort(vector<int>& arr) {
    if (arr.empty()) return;
    int maxVal = *max_element(arr.begin(), arr.end());
    vector<int> count(maxVal + 1, 0);
    for (int x : arr) count[x]++;
    int idx = 0;
    for (int i = 0; i <= maxVal; ++i)
        for (int j = 0; j < count[i]; ++j)
            arr[idx++] = i;
}
```

### 7.2 基数排序

条件：**按位排序**（数字的个十百位、字符串的字符）。

```
LSD（最低位优先）:
原始: [170, 45, 75, 90, 802, 24, 2, 66]
按个位: [170, 90, 802, 2, 24, 45, 75, 66]
按十位: [802, 2, 24, 45, 66, 170, 75, 90]
按百位: [2, 24, 45, 66, 75, 90, 170, 802]
```

### 7.3 桶排序

将数据分到若干有序桶中，桶内用其他排序（如插入排序），最后合并。

---

## 8. 稳定性分析——为什么重要？

**稳定排序保持相等元素的原有相对顺序**。当按多关键字排序时，稳定性让"先按 A 排，再按 B 排"等价于"一次按 (B, A) 排"。

```
不稳定排序:  quick, heap, selection, shell
稳定排序:    bubble, insertion, merge, counting, radix

多关键字排序的经典做法（先按次要关键字排，再按主要关键字排）:
records.sortBy("score");   // 稳定排序
records.sortBy("name");    // 现在既按 name 又按 score
```

---

## 9. 排序算法选择指南

```
数据量小（n < 50）→ 插入排序（常数因子最小）
数据量中 + 需要稳定 → 归并排序
数据量中 + 通用 → 快速排序（通常最快）
数据量大 + 不能 O(n) 额外空间 → 堆排序
数据是整数 + 范围小 → 计数排序
数据是字符串 → 基数排序
外部排序（磁盘）→ 多路归并
```

---

## 小结

排序的核心就是权衡：**时间 × 空间 × 稳定性 × 实现复杂度**。面试中快速排序（partition 思想）和归并排序最重要，大多数 Top-K / 第 K 大 / 逆序对都用它们的思想。

下一篇 [07 查找算法 / 二分变体 / Top-K / 海量数据](./07-searching-topk.md)。
