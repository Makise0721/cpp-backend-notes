# 08 — LSM-Tree、WAL、SSTable、LevelDB/RocksDB

> 关联篇目：[01 InnoDB（B+ Tree）](./01-mysql-innodb.md) | [第三章 04 布隆过滤器](../03-data-structures-algorithms/04-graph-hash-bloom.md)

---

## 1. LSM-Tree——为写入而生的存储结构

**Log-Structured Merge Tree** 的核心思想：**把随机写变成顺序写，用批量归并摊薄写放大。**

### 1.1 基本架构

```
写入流程:

  Write(1) → [WAL (顺序写)] ──── 持久化保证
         │
         └──→ [MemTable (内存)] ──── 写入后立即返回
                  │
                  │ MemTable 满了 →
                  ▼  flush 到磁盘
            ┌──────────┐
            │ SSTable   │  Level 0  ← 可能重叠
            │ (不可变)   │
            └──────────┘
                  │
                  │ Compaction（后台归并）
                  ▼
            ┌──────────┐
            │ SSTable   │  Level 1  ← 有序、不重叠
            └──────────┘
                  │
                  ▼
            ┌──────────┐
            │ SSTable   │  Level N  ← 每层容量是上层的 10 倍
            └──────────┘
```

### 1.2 读写分析

| 操作 | 过程 | 复杂度 |
|------|------|:---:|
| **写入** | WAL(顺序写) + MemTable(内存) | O(1) 极快 |
| **读取** | MemTable → Level 0 → Level 1 → ... → Level N | 可能多次磁盘读（最坏） |
| **删除** | 写入 tombstone 标记（和写入一样快） | O(1) |
| **空间放大** | Compaction 前旧数据仍占空间 | 10%-50% |

**LSM-Tree 的设计哲学：牺牲读性能换取写性能**。因为顺序写比随机写快 100 倍以上。

---

## 2. WAL——崩溃恢复的保证

```
WAL (Write-Ahead Log) = 顺序追加的日志文件

写入 Key-Value:
1. 将操作记录追加到 WAL → fsync
2. 将数据写入 MemTable → 返回成功

崩溃恢复:
重放 WAL → 重建 MemTable 中未 flush 的数据
```

WAL 和 MySQL 的 Redo Log 是同一原理：预写日志是持久性的基石。

---

## 3. SSTable——有序的不可变文件

```
SSTable 文件结构:
┌──────────────────────────────┐
│         Data Blocks          │ ← 实际 Key-Value 数据（有序）
├──────────────────────────────┤
│        Index Blocks          │ ← key → data block offset
├──────────────────────────────┤
│        Bloom Filter          │ ← 快速判断 key 是否可能在此文件中
├──────────────────────────────┤
│          Footer              │ ← 元信息（索引位置、魔数）
└──────────────────────────────┘

查询过程:
1. 查 Bloom Filter → 不存在 → 跳过此文件（省一次磁盘读）
2. 查 Index Block → 找到可能包含 key 的 Data Block
3. 读 Data Block → 二分查找 key
```

---

## 4. Compaction——归并压缩

随着 MemTable 不断 flush，Level 0 的 SSTable 越来越多，需要后台归并。

### 4.1 Leveled Compaction（RocksDB 默认）

```
Level 0: [SST1] [SST2] [SST3]  ← key 范围可能重叠
            ↓ Compaction
Level 1: [SST4] [SST5] [SST6]  ← key 范围不重叠、有序
            ↓ Compaction
Level 2: [SST7] ... [SST16]    ← 容量是上层的 10 倍
```

每层的容量是上层的 10 倍（放大因子）。当某层大小超过阈值 → 选择该层的文件 → 与下一层中 key 范围重叠的文件合并 → 生成新文件放在下一层。

### 4.2 Universal Compaction

当写量远大于读量时，使用 Universal Compaction 减少写放大（适合写密集型场景）。

---

## 5. LevelDB vs RocksDB

| | LevelDB | RocksDB |
|------|------|------|
| 作者 | Google (Jeff Dean, Sanjay Ghemawat) | Facebook（基于 LevelDB 的 fork） |
| 线程 | 单线程 Compaction | 多线程 Compaction |
| 压缩 | Snappy | Zstd / LZ4 / Snappy 等 |
| MemTable | SkipList | SkipList / HashLinkList / Vector |
| 特性 | 基础 LSM | BlobDB、事务、列族、备份 |
| 实际使用 | — | **MySQL MyRocks、TiKV、Flink** |

---

## 6. B+ Tree vs LSM-Tree

| | B+ Tree (InnoDB) | LSM-Tree (RocksDB) |
|------|:---:|:---:|
| **写性能** | 随机写，B+ 树结构调整 | **顺序写，极快** |
| **读性能** | **O(log n)，极快** | 需查多层，最坏多次磁盘读 |
| **空间放大** | ~10-20%（页分裂碎片） | ~10-50%（旧版本未归并） |
| **写放大** | 1（原地更新） | 10-30（重复写同一数据到新 SST） |
| **适用场景** | OLTP（读多写少/平衡） | 时序数据、日志、写密集型 KV 存储 |

```
选择决策:
读多写少 + 需要复杂查询 → B+ Tree (MySQL)
写多读少 + 简单 KV 访问 → LSM-Tree (RocksDB/LevelDB)
写密集 + 时序数据 → LSM-Tree (TimescaleDB/Cassandra)
```

---

## 小结

| 概念 | 一句话 |
|------|--------|
| LSM-Tree | 顺序写 + 批量归并，用读性能换写性能 |
| WAL | 预写日志，崩溃恢复保障 |
| MemTable | 内存中的写缓冲（跳表） |
| SSTable | 不可变有序文件（Data+Index+Bloom+Footer） |
| Compaction | 后台归并多级 SSTable |
| B+ vs LSM | 读快 vs 写快 |

下一篇 [09 分库分表 / 分布式事务 / Join / 主从复制](./09-database-advanced.md)。
