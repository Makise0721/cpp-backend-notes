# 02 — B+ 树索引、聚簇索引、覆盖索引、EXPLAIN

> 关联篇目：[01 InnoDB 架构](./01-mysql-innodb.md) | [第三章 03 B/B+ 树](../03-data-structures-algorithms/03-advanced-trees.md)

---

## 1. 为什么用 B+ 树而不是 B 树？

### 1.1 结构差异

```
B 树:                           B+ 树:
      [key+data]                    [key]       ← 内部节点只有 key
     /    |    \                   /    \
[key+data] [key+data]        [key]    [key]     ← 所有 data 在叶子
                            /    \    /    \
                         [key+data]...[key+data] ← 叶子是完整数据
                                                 叶子之间有双向链表
```

### 1.2 B+ 树的四大优势

| 优势 | 说明 |
|------|------|
| **树更矮** | 内部节点只存 key 不存 data → 一页能存更多 key → 树高更低 → 更少磁盘 I/O |
| **查询稳定** | 每次查找都必须走到叶子节点 → I/O 次数固定 |
| **范围查询快** | 叶子节点有双向链表 → `WHERE id BETWEEN 100 AND 200` 只需找到起点然后顺序扫描 |
| **全表扫描高效** | 遍历叶子链表即可，无需中序遍历整棵树 |

---

## 2. 聚簇索引 vs 非聚簇索引

### 2.1 聚簇索引（Clustered Index）

```
聚簇索引 = 主键索引。叶子节点存储的是**完整行数据**。

表数据按主键顺序存储在 B+ 树的叶子节点中:
          [30]
         /    \
    [10,20]  [40,50]
    /  |  \   /  |  \
   10  20  30 40 50 60
   |   |   |  |  |  |
  row row row row row row  ← 叶子存整行数据
```

- **每张表只有一个聚簇索引**（数据只能按一种顺序存放）
- 如果没有显式定义主键 → InnoDB 用第一个 NOT NULL UNIQUE 列 → 都没有则创建隐藏的 `row_id`

### 2.2 非聚簇索引（二级索引 / Secondary Index）

```
二级索引的叶子节点存储的是**主键值**（不是行数据）:

索引 (name):                       聚簇索引:
    ['Bob']                            [30]
    /    \                            /    \
['Alice']['Charlie']              [10,20]  [40,50]
   |       |                       /  |  \   /  |  \
  id=3   id=1                     10  20  30 40 50 60
                                   |   |   |  |  |  |
SELECT * WHERE name='Bob':       row row row row row row
1. 在 name 索引中查到 id=3
2. **回表**: 用 id=3 去聚簇索引中查完整行
```

### 2.3 回表的代价

每次在二级索引上查到主键后，还需要去聚簇索引中查一次 → **多一次磁盘 I/O**。如果查询涉及很多行，回表代价很大。

---

## 3. 覆盖索引——不用回表

**如果查询需要的所有列都在二级索引中（包括主键），就不需要回表。**

```sql
-- 在 (name, age) 上建了联合索引

-- ✅ 覆盖索引（不需要回表）
SELECT name, age FROM users WHERE name = 'Bob';
-- 索引中已经包含 name 和 age，直接返回
-- EXPLAIN 中 Extra = Using index

-- ❌ 需要回表
SELECT name, age, email FROM users WHERE name = 'Bob';
-- email 不在索引中，需要回表
```

---

## 4. 联合索引与最左前缀原则

```sql
CREATE INDEX idx_a_b_c ON t(a, b, c);

-- ✅ 可以使用索引
WHERE a = 1                 -- 用到 a
WHERE a = 1 AND b = 2       -- 用到 a, b
WHERE a = 1 AND b = 2 AND c = 3  -- 用到 a, b, c
WHERE a = 1 AND c = 3       -- 用到 a（c 不行，中间断了）

-- ❌ 不能使用索引
WHERE b = 2                 -- 不满足最左前缀
WHERE c = 3                 -- 不满足最左前缀

-- 范围查询会中断后续列的使用:
WHERE a = 1 AND b > 5 AND c = 3
-- 用到 a, b(range)，c 不能使用索引
```

**最左前缀原则**：联合索引从最左边的列开始匹配，不能跳过中间列。遇到范围查询（`>` `<` `BETWEEN` `LIKE 'x%'`），后续列无法使用索引。

---

## 5. EXPLAIN 执行计划解读

```sql
EXPLAIN SELECT * FROM users WHERE name = 'Bob';
```

### 5.1 关键字段

| 字段 | 含义 | 好坏排序 |
|------|------|----------|
| **type** | 访问类型 | **system > const > eq_ref > ref > range > index > ALL** |
| **key** | 实际使用的索引 | NULL = 没用到索引 |
| **rows** | 估计需要扫描的行数 | 越小越好 |
| **Extra** | 额外信息 | Using index（覆盖索引）= 好；Using filesort/Using temporary = 需要优化 |

### 5.2 type 类型速查

| type | 含义 | 示例 |
|------|------|------|
| `system` | 表只有一行 | 系统表 |
| `const` | 主键或唯一索引等值查询 | `WHERE id = 1` |
| `eq_ref` | Join 时使用主键/唯一索引 | `ON a.id = b.id` |
| `ref` | 非唯一索引等值查询 | `WHERE name = 'Bob'` |
| `range` | 索引范围扫描 | `WHERE id BETWEEN 1 AND 100` |
| `index` | 全索引扫描 | 比 ALL 快（索引通常比表小） |
| **ALL** | 全表扫描 | ❌ 必须优化 |

### 5.3 Extra 关键信息

```sql
-- Using index: 覆盖索引，最好
EXPLAIN SELECT id, name FROM users WHERE name = 'Bob';
-- Extra: Using index

-- Using filesort: 需要额外排序，可能需要优化
EXPLAIN SELECT * FROM users ORDER BY name;
-- Extra: Using filesort

-- Using temporary: 需要临时表，必须优化
EXPLAIN SELECT DISTINCT name FROM users;
-- Extra: Using temporary
```

---

## 6. 索引优化策略

```sql
-- 1. 选择性高的列放联合索引前面
--    (status, name) ❌ status 只有几个值
--    (name, status) ✅ name 选择性高

-- 2. 范围查询的列放联合索引最后
--    (a, b, c) 如果 b 常做范围查询 → (a, c, b)

-- 3. 不要为每列单独建索引（浪费空间 + 写操作慢）
--    ❌ INDEX(a), INDEX(b), INDEX(c) — 只能用一个
--    ✅ INDEX(a, b, c) — 一个联合索引覆盖多个查询

-- 4. 避免在索引列上使用函数
--    ❌ WHERE YEAR(created_at) = 2024
--    ✅ WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31'
```

---

## 小结

| 概念 | 一句话 |
|------|--------|
| B+ 树 | 数据只在叶子，叶子有链表，范围查询 O(log n + k) |
| 聚簇索引 | 主键索引，叶子存整行 |
| 二级索引 | 叶子存主键值，查完整行需**回表** |
| 覆盖索引 | 查询列都在索引中 → 不回表 |
| 最左前缀 | 联合索引必须从最左列开始匹配 |
| EXPLAIN type | system > const > eq_ref > ref > range > index > ALL |

下一篇 [03 事务 ACID / 隔离级别 / MVCC](./03-mysql-transaction-mvcc.md)。
