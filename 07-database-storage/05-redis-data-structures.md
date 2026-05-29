# 05 — Redis 核心数据结构与底层编码

> 关联篇目：[06 持久化与事件循环](./06-redis-persistence-eventloop.md) | [07 缓存策略与分布式](./07-redis-cache-advanced.md)

---

## 1. Redis 对象模型

```
Redis 每个键值对都是一个 redisObject:
┌─────────────────────┐
│ type: 数据类型       │ ← STRING/LIST/HASH/SET/ZSET
│ encoding: 底层编码   │ ← 同一种 type 可能有多种编码（自动切换）
│ ptr: 指向实际数据    │
│ lru: LRU 时钟       │
│ refcount: 引用计数   │
└─────────────────────┘
```

---

## 2. String——最简单的类型

### 2.1 底层编码：SDS（Simple Dynamic String）

```c
struct sdshdr {
    int len;    // 已用长度
    int free;   // 剩余空间
    char buf[]; // 柔性数组（实际数据）
};
```

**优于 C 字符串**：
- O(1) 获取长度（不用 `strlen` 遍历）
- 预分配空间（减少 `realloc` 次数，最多分配 1MB 额外空间）
- 惰性空间释放（缩短字符串时不立即释放内存）
- 二进制安全（可以存 `\0`，通过 `len` 而不是 `\0` 确定长度）

### 2.2 embstr vs raw

```
短字符串 (< 44 字节): embstr 编码 — redisObject + SDS 一次分配，连续内存
长字符串 (≥ 44 字节): raw 编码   — redisObject + SDS 分开分配
```

---

## 3. List——双向链表

### 3.1 quicklist（Redis 3.2+）

```
quicklist = 双向链表 + 每个节点是一个 ziplist

┌──────────┐    ┌──────────┐    ┌──────────┐
│ quicklist │⇄   │ quicklist │⇄   │ quicklist │
│  node     │    │  node     │    │  node     │
│ ┌──────┐  │    │ ┌──────┐  │    │ ┌──────┐  │
│ │ziplist│  │    │ │ziplist│  │    │ │ziplist│  │
│ │[a][b]│  │    │ │[c][d]│  │    │ │[e][f]│  │
│ └──────┘  │    │ └──────┘  │    │ └──────┘  │
└──────────┘    └──────────┘    └──────────┘
```

- ziplist 是一段连续内存，存多个元素（紧凑、缓存友好）
- 当 ziplist 过长 → 分裂成新的 quicklist node
- 兼顾了链表的灵活性 + 数组的空间效率

---

## 4. Hash——字典

### 4.1 编码切换

```
数据量小: ziplist（连续内存，key-value 交替存放）
数据量大: dict（哈希表）

dict 结构:
┌──────────┐
│  ht[0]   │ ← 主哈希表
├──────────┤
│  ht[1]   │ ← rehash 时使用（渐进式 rehash）
└──────────┘
```

### 4.2 渐进式 Rehash

Redis 是单线程的，如果一次性 rehash 大哈希表会阻塞服务。**渐进式 rehash**分多次完成：

```
1. 创建 ht[1]（容量 = ht[0] 的 2 倍）
2. 设置 rehashidx = 0
3. 每次对 dict 的增删改查操作，顺带将 ht[0][rehashidx] 迁移到 ht[1]
4. rehashidx++，直到全部迁移完成
5. 释放 ht[0]，ht[1] 变成新的 ht[0]
```

---

## 5. Set——无序集合

| 编码 | 条件 | 结构 |
|------|------|------|
| **intset** | 元素全是整数 + 数量 < 512 | 有序整数数组 |
| **dict** | 其他情况 | 字典（key=元素值，value=NULL） |

---

## 6. ZSet——有序集合

### 6.1 双结构设计

```
ZSet = skiplist（跳表，按 score 排序） + dict（哈希表，按 member 查找）

┌────────────┐
│  skiplist   │ ← 按 score 排序，支持 ZRANGE / ZRANK
│  (跳表)     │
├────────────┤
│    dict     │ ← member → score，O(1) 查找某 member 的分数
│  (哈希表)   │
└────────────┘

两个数据结构指向同一份 member 字符串（通过指针共享，不拷贝）
```

### 6.2 为什么用跳表而不是红黑树？

| 原因 | 说明 |
|------|------|
| 实现简单 | 跳表 ~200 行，红黑树 ~500 行 |
| 范围查询 | 跳表天然支持（找到起点 → 沿 level 0 顺序遍历） |
| 并发友好 | 跳表操作更局部（红黑树旋转影响大片区域） |
| ZRANK | 跳表每个节点维护 span → O(log n) 计算排名 |

---

## 7. 编码选择速查

| 类型 | 小数据编码 | 大数据编码 | 切换阈值（可配置） |
|------|-----------|-----------|-------------------|
| String | embstr | raw | 44 字节 |
| List | quicklist | quicklist | — |
| Hash | ziplist | dict | `hash-max-ziplist-entries 512` |
| Set | intset | dict | `set-max-intset-entries 512` |
| ZSet | ziplist | skiplist+dict | `zset-max-ziplist-entries 128` |

---

## 小结

| 类型 | 底层 | 设计亮点 |
|------|------|----------|
| String | SDS | O(1) len、预分配、二进制安全 |
| List | quicklist | 链表+ziplist 混合 |
| Hash | dict | 渐进式 rehash（不阻塞） |
| Set | intset/dict | 整数时用紧凑数组 |
| ZSet | skiplist+dict | 跳表排分 + 字典查分，共享 member |

下一篇 [06 Redis 持久化与事件循环](./06-redis-persistence-eventloop.md)。
