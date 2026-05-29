# 04 — InnoDB 锁机制：行锁、间隙锁、临键锁、死锁

> 关联篇目：[03 事务与 MVCC](./03-mysql-transaction-mvcc.md) | [02 索引](./02-mysql-index.md)

---

## 1. 锁的层级

```
表级锁: 锁整张表 → 并发度最低，开销小
行级锁: 锁特定行 → 并发度高，开销大
页级锁: BDB 引擎使用，MySQL 已淘汰
```

InnoDB 支持**行级锁**，通过**在索引上加锁**实现。

**重要**：如果一条 SQL 没有走索引 → 行级锁退化为表级锁！所以 `WHERE` 条件必须命中索引。

---

## 2. 三种行锁

| 锁类型 | 锁定范围 | 作用 |
|------|------|------|
| **Record Lock** | 单个行记录（索引记录） | 防止该行被修改 |
| **Gap Lock** | 索引记录之间的间隙 | 防止在间隙中插入新行（防幻读） |
| **Next-Key Lock** | Record + 前面的 Gap | = Record + Gap，InnoDB 默认行锁 |

### 2.1 示意图

```
索引值:  5    10    15    20    25
         │  │  │  │  │  │  │  │  │
         └──┴──┴──┴──┴──┴──┴──┴──┘
          gap gap gap gap gap gap

Record Lock: 锁住 10 这个索引记录
Gap Lock:   锁住 (5,10) 这个间隙
Next-Key Lock: 锁住 (5, 10]（前开后闭）
```

### 2.2 各隔离级别下的锁行为

```sql
-- 表: users (id PK, age 有索引), 数据: age=18, 25, 30

-- REPEATABLE READ (默认)
SELECT * FROM users WHERE age = 25 FOR UPDATE;
-- 加 Next-Key Lock: (18, 25] + (25, 30] 
-- 相邻间隙都被锁住 → 其他事务无法插入 age=19 或 age=27

-- READ COMMITTED
SELECT * FROM users WHERE age = 25 FOR UPDATE;
-- 只加 Record Lock: age=25
-- 其他事务可以在间隙中插入 → 可能幻读
```

---

## 3. 共享锁 vs 排他锁

| 锁类型 | SQL | 兼容 S | 兼容 X |
|------|------|:---:|:---:|
| **S (共享锁)** | `SELECT ... LOCK IN SHARE MODE` | ✅ | ❌ |
| **X (排他锁)** | `SELECT ... FOR UPDATE` / `UPDATE` / `DELETE` / `INSERT` | ❌ | ❌ |

### 3.1 意向锁（Intention Lock）

InnoDB 在表级别维护**意向锁**（IS/IX），用于快速判断表中是否有行锁——避免逐行检查。

```
对某行加 X 锁之前 → 先在表上加 IX（意向排他锁）
其他事务想加表级 X 锁 → 先查表上是否有 IX → 有 → 等待

意向锁之间不互斥（IS 和 IX 兼容）
意向锁和表级锁互斥（IX 和表 X 锁不兼容）
```

---

## 4. 死锁

### 4.1 产生条件

```
事务 A: UPDATE users SET name='X' WHERE id=1;  → 锁住 id=1
         UPDATE users SET name='X' WHERE id=2;  → 等待 id=2

事务 B: UPDATE users SET name='Y' WHERE id=2;  → 锁住 id=2
         UPDATE users SET name='Y' WHERE id=1;  → 等待 id=1

→ 循环等待 → 死锁！
```

### 4.2 InnoDB 的死锁处理

- 死锁检测：`innodb_deadlock_detect = ON`（默认），用等待图检测环
- 检测到死锁 → 回滚代价较小的事务（undo 量少的那方）
- 查看最后一次死锁信息：`SHOW ENGINE INNODB STATUS`

### 4.3 死锁预防

```sql
-- 1. 统一操作顺序（最重要！）
--    所有事务都按 id 升序操作 → 消除循环等待

-- 2. 尽量缩小事务（持有锁的时间更短）
-- 3. 合理使用索引（避免行锁升级为表锁）
-- 4. 在 RC 隔离级别下使用基于唯一索引的更新（减少 Gap Lock）
```

---

## 5. 锁监控

```sql
-- 查看当前锁等待
SELECT * FROM information_schema.INNODB_TRX;       -- 当前事务
SELECT * FROM information_schema.INNODB_LOCKS;     -- 当前锁（MySQL 8.0 前）
SELECT * FROM information_schema.INNODB_LOCK_WAITS;-- 锁等待关系

-- MySQL 8.0+
SELECT * FROM performance_schema.data_locks;
SELECT * FROM performance_schema.data_lock_waits;

-- 查看死锁日志
SHOW ENGINE INNODB STATUS\G
-- 搜索 "LATEST DETECTED DEADLOCK" 段落
```

---

## 6. 锁与索引的关系

```sql
-- 表: id PK, name 有索引, age 无索引

-- ✅ 通过索引加行锁
SELECT * FROM t WHERE name = 'Bob' FOR UPDATE;
-- 锁住 name 索引中 'Bob' 对应的行

-- ❌ 不走索引 → 全表所有行被锁 + 间隙被锁！
SELECT * FROM t WHERE age = 25 FOR UPDATE;
-- 相当于锁全表！
```

**黄金规则**：加锁的 SQL 必须走索引，否则退化为表锁。

---

## 小结

| 概念 | 一句话 |
|------|--------|
| Record Lock | 锁住单个索引记录 |
| Gap Lock | 锁住索引记录间的间隙（防 INSERT） |
| Next-Key Lock | Record + Gap = (前开后闭]，InnoDB 默认 |
| 意向锁 | 表级别的 IS/IX，快速判断表中行锁情况 |
| 死锁 | 循环等待 → 自动检测 + 回滚代价小的事务 |
| 不加索引 = 表锁 | 行锁依赖索引 |

下一篇 [05 Redis 数据结构与底层编码](./05-redis-data-structures.md)。
