# 03 — 事务 ACID、隔离级别与 MVCC

> 关联篇目：[01 InnoDB 架构](./01-mysql-innodb.md) | [04 锁机制](./04-mysql-locks.md)

---

## 1. ACID——事务的四项保证

| 属性 | 含义 | InnoDB 如何实现 |
|------|------|----------------|
| **A**tomicity 原子性 | 事务要么全做，要么全不做 | **Undo Log**（回滚） |
| **C**onsistency 一致性 | 事务前后数据满足所有约束 | Undo Log + Redo Log + 锁 |
| **I**solation 隔离性 | 并发事务互不干扰 | **MVCC + 锁** |
| **D**urability 持久性 | 已提交事务永久保存 | **Redo Log**（WAL） |

---

## 2. 隔离级别与并发问题

### 2.1 三种并发问题

| 问题 | 描述 |
|------|------|
| **脏读** | 读到其他事务未提交的修改 |
| **不可重复读** | 同一事务内两次读同一行，结果不同（被其他事务 UPDATE 并提交了） |
| **幻读** | 同一事务内两次查同一范围，结果行数不同（被其他事务 INSERT 了） |

### 2.2 四种隔离级别

```
隔离级别（从低到高）:

READ UNCOMMITTED (读未提交)
  ├─ 可以读到未提交的数据 → 脏读 ✅
  │
READ COMMITTED (读已提交 — Oracle 默认)
  ├─ 只能读已提交数据 → 脏读 ❌
  │ 但同一事务两次读可能不同 → 不可重复读 ✅
  │
REPEATABLE READ (可重复读 — MySQL InnoDB 默认)
  ├─ 同一事务内多次读结果一致 → 不可重复读 ❌
  │ MySQL RR 下通过 Next-Key Lock 也解决了大部分幻读
  │
SERIALIZABLE (可串行化)
  └─ 所有事务串行执行 → 幻读 ❌
```

```sql
-- 查看和设置隔离级别
SELECT @@transaction_isolation;
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

---

## 3. MVCC——多版本并发控制

### 3.1 核心思想

**读不阻塞写，写不阻塞读。** 每个事务看到的是数据的某个"快照版本"，而不是加锁阻塞。

### 3.2 隐藏列

InnoDB 每行有两个隐藏列：

```
┌──────────┬──────────┬──────┬─────────────┬──────────────┐
│   id     │   name   │ ...  │ DB_TRX_ID   │ DB_ROLL_PTR  │
├──────────┼──────────┼──────┼─────────────┼──────────────┤
│    1     │  'Bob'   │      │    100      │ → undo ptr   │
└──────────┴──────────┴──────┴─────────────┴──────────────┘

DB_TRX_ID: 最近修改此行的事务 ID
DB_ROLL_PTR: 指向 Undo Log 中旧版本的指针（版本链）
```

### 3.3 版本链

```
行当前版本 (trx_id=300, name='Bob')
  │ roll_pointer
  ▼
Undo Log (trx_id=200, name='Alice')
  │ roll_pointer
  ▼
Undo Log (trx_id=100, name='NULL')
  │
  ▼ NULL (初始版本)
```

### 3.4 ReadView——决定你能看到哪个版本

ReadView 是事务开始时创建的一个"快照"，包含：

```
ReadView {
    m_ids: [100, 200, 300]  // 当前活跃（未提交）的事务 ID 列表
    min_trx_id: 100         // 最小活跃事务 ID
    max_trx_id: 301         // 下一个将分配的事务 ID
    creator_trx_id: 250     // 创建此 ReadView 的事务 ID
}

可见性判断:
- trx_id == creator_trx_id → 可见（自己改的）
- trx_id < min_trx_id       → 可见（创建 ReadView 前已提交）
- trx_id >= max_trx_id      → 不可见（创建 ReadView 后才开始的事务）
- trx_id in m_ids           → 不可见（创建 ReadView 时还在活跃中）
- 否则                      → 可见
```

### 3.5 快照读 vs 当前读

```sql
-- 快照读（Snapshot Read）：基于 MVCC，不加锁
SELECT * FROM users WHERE id = 1;

-- 当前读（Current Read）：读最新版本，加锁
SELECT * FROM users WHERE id = 1 FOR UPDATE;
SELECT * FROM users WHERE id = 1 LOCK IN SHARE MODE;
UPDATE users SET name='Bob' WHERE id = 1;
DELETE FROM users WHERE id = 1;
INSERT INTO users ... ;
-- 当前读总是读取最新已提交版本，并且会加锁
```

### 3.6 不同隔离级别下的 ReadView 生成时机

| 隔离级别 | ReadView 生成 | 效果 |
|----------|:---:|------|
| READ COMMITTED | **每次查询**都生成新 ReadView | 能看到其他事务已提交的修改 → 不可重复读 |
| REPEATABLE READ | **事务开始时**生成一次 | 整个事务看到同一快照 → 可重复读 |

---

## 4. MVCC 解决了什么？

| 问题 | 解决方式 |
|------|------|
| 脏读 | 通过 ReadView 过滤未提交事务 |
| 不可重复读 | RR 下事务开始生成 ReadView，后续不变 |
| 幻读 | MVCC 的快照读不会幻读；当前读靠 Next-Key Lock（见 [04 锁](./04-mysql-locks.md)） |

---

## 小结

| 概念 | 一句话 |
|------|--------|
| ACID | Undo Log(原子) + Redo Log(持久) + MVCC+锁(隔离) |
| 脏读/不可重复读/幻读 | 分别对应读到未提交/读两次不同/多出行 |
| MVCC | 隐藏列 trx_id + roll_ptr 组成版本链 |
| ReadView | 活跃事务列表 → 决定版本可见性 |
| RC vs RR | ReadView 生成时机：每次查询 vs 事务开始 |

下一篇 [04 锁机制：行锁/间隙锁/临键锁/死锁](./04-mysql-locks.md)。
