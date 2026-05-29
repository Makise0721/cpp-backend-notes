# 01 — InnoDB 存储引擎架构：Buffer Pool、Redo Log、Undo Log、Double Write

> 关联篇目：[02 B+ 树索引](./02-mysql-index.md) | [03 事务与 MVCC](./03-mysql-transaction-mvcc.md)

---

## 1. InnoDB 整体架构

```
┌──────────────────────────────────────────────┐
│                  InnoDB                       │
│  ┌──────────────────────────────────────┐    │
│  │          Buffer Pool (内存)           │    │
│  │  数据页 │ 索引页 │ undo页 │ 自适应哈希 │    │
│  │  LRU链表 │ Flush链表 │ Free链表      │    │
│  └──────────┬───────────────────────────┘    │
│             │                                  │
│  ┌──────────┴───────────────────────────┐    │
│  │           磁盘结构                     │    │
│  │  表空间(.ibd): 数据 + 索引             │    │
│  │  Redo Log: ib_logfile0/1             │    │
│  │  Undo Tablespace                     │    │
│  │  Double Write Buffer                 │    │
│  └──────────────────────────────────────┘    │
└──────────────────────────────────────────────┘
```

---

## 2. Buffer Pool——缓存在内存中的数据页

**最大的一块内存**（默认 128MB，生产环境通常配置为物理内存的 50-80%）。

### 2.1 结构

```
Buffer Pool:
┌────────────────────────────────────────────┐
│  数据页 (16KB) × N                         │
│  用 LRU 链表管理淘汰（分为 young/old 两段） │
├────────────────────────────────────────────┤
│  自适应哈希索引 (AHI)                       │
├────────────────────────────────────────────┤
│  Change Buffer（写缓冲，缓存二级索引的变更） │
└────────────────────────────────────────────┘
```

### 2.2 LRU 淘汰

```
标准 LRU 问题：全表扫描会把热数据全挤出去 → 性能雪崩

InnoDB 的改进 LRU:
┌─────────────┬──────────────────┐
│  Young (5/8) │    Old (3/8)     │
│  热数据      │   新读入的页放这里  │
└─────────────┴──────────────────┘
                ↑
          midpoint (innodb_old_blocks_pct 默认 37)
新页先进入 old 区头部，如果在 old 区待够一定时间未被淘汰才进入 young 区
→ 全表扫描的页快速进入 old 区又快速被淘汰，不影响热数据
```

---

## 3. Redo Log——保证持久性（D）

### 3.1 为什么需要 Redo Log？

```
没有 Redo Log:
UPDATE ... SET name='Bob' WHERE id=1
→ 修改 Buffer Pool 中的页（脏页）
→ 此时宕机 → 内存中的修改全丢失 → 数据不一致

有 Redo Log:
UPDATE ... SET name='Bob' WHERE id=1
→ 修改 Buffer Pool 中的页
→ 写入 Redo Log（顺序写，极快！）
→ 返回客户端 "OK"
→ 后台将脏页刷回磁盘

宕机恢复: 重放 Redo Log → 恢复所有已提交但未刷盘的数据
```

### 3.2 WAL（Write-Ahead Log）原则

**先写日志，再写数据。** Redo Log 是顺序写（~100μs），而随机写数据页（~10ms）→ 差 100 倍。

### 3.3 环形结构

```
Redo Log 是固定大小的循环文件:
ib_logfile0 (48MB) + ib_logfile1 (48MB)

┌─────────────────────────────────────┐
│ → → → → → → → → → → → → → → → →  │
│ write_pos                         │
│   已写入的 LSN 位置                  │
│               checkpoint          │
│               已刷盘的位置           │
└─────────────────────────────────────┘

write_pos 追上 checkpoint → 必须等待 checkpoint 推进（刷脏页）→ 写入阻塞
```

---

## 4. Undo Log——保证原子性（A）和隔离性（I）

### 4.1 作用

Undo Log 记录的是**修改前的旧值**，用于：

1. **事务回滚**：将修改恢复为旧值（原子性）
2. **MVCC**：提供快照读（隔离性）——其他事务可以通过 Undo Log 看到旧版本

```
UPDATE ... SET name='Bob' WHERE id=1（原值='Alice'）:
Undo Log: (id=1, name='Alice')  ← 旧版本
Redo Log: (id=1, name='Bob')    ← 新版本
```

### 4.2 版本链

```
每行数据有两个隐藏列:
- DB_TRX_ID: 最近修改此行的 事务ID
- DB_ROLL_PTR: 指向 Undo Log 中上一个版本的指针

行数据 → Undo Log(旧版本1) → Undo Log(旧版本2) → ...
         ↑ 通过 roll_pointer 串联成版本链
```

---

## 5. Double Write——防止页损坏

### 5.1 问题：部分写（Partial Write）

InnoDB 页大小 16KB，OS 页大小 4KB。刷一个 16KB 的页需要写 4 次。如果写到第 2 次时宕机 → 磁盘上的页被写坏 → 无法恢复（Redo Log 的恢复依赖完整的页）。

### 5.2 Double Write 的解决

```
1. 先将脏页写入 Double Write Buffer（磁盘上的 2MB 连续区域，顺序写）
2. fsync ✅ 保证 Double Write Buffer 完整
3. 再将脏页写入实际的表空间位置（随机写）
4. 如果步骤 3 中途宕机 → 从 Double Write Buffer 恢复完整的页
```

---

## 6. 一次 UPDATE 的完整旅程

```sql
UPDATE users SET name='Bob' WHERE id=1;
```

```
1. 从磁盘/Buffer Pool 读取 id=1 的数据页
2. 加锁（行锁）
3. 记录 Undo Log: 旧值 (id=1, name='Alice')
4. 修改 Buffer Pool 中的数据页: name='Bob'
5. 记录 Redo Log: 新值 + Undo 的修改
6. Redo Log 写入磁盘（事务提交时 fsync）
7. 返回 "OK" 给客户端
8. 后台线程将脏页异步刷回磁盘（Checkpoint）
```

---

## 小结

| 组件 | 作用 | 存储位置 |
|------|------|:---:|
| Buffer Pool | 缓存数据页和索引页 | 内存 |
| Redo Log | 保证持久性（顺序写 WAL） | 磁盘（循环文件） |
| Undo Log | 回滚 + MVCC | 磁盘（undo 表空间） |
| Double Write | 防止页损坏 | 磁盘（2MB 连续区域） |

下一篇 [02 B+ 树索引与 EXPLAIN](./02-mysql-index.md)。
