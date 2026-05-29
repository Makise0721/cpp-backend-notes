# 第七章：数据库与存储

> 9 篇 · 从 MySQL InnoDB 到 Redis/LSM-Tree，覆盖数据库内核与存储引擎原理。

---

## 篇目

| # | 文件 | 核心内容 |
|---|------|----------|
| 01 | [01-mysql-innodb.md](./01-mysql-innodb.md) | Buffer Pool、Redo Log(WAL)、Undo Log、Double Write、一次 UPDATE 的完整旅程 |
| 02 | [02-mysql-index.md](./02-mysql-index.md) | B+ 树索引、聚簇/非聚簇索引、回表、覆盖索引、最左前缀、EXPLAIN 解读 |
| 03 | [03-mysql-transaction-mvcc.md](./03-mysql-transaction-mvcc.md) | ACID 实现原理、四种隔离级别、MVCC(隐藏列/ReadView/版本链)、快照读 vs 当前读 |
| 04 | [04-mysql-locks.md](./04-mysql-locks.md) | Record Lock/Gap Lock/Next-Key Lock、意向锁、死锁检测与预防 |
| 05 | [05-redis-data-structures.md](./05-redis-data-structures.md) | String(SDS)/List(quicklist)/Hash(dict+渐进式rehash)/ZSet(skiplist+dict) |
| 06 | [06-redis-persistence-eventloop.md](./06-redis-persistence-eventloop.md) | RDB/AOF/混合持久化、事件循环、Redis 6.0 多线程 IO |
| 07 | [07-redis-cache-advanced.md](./07-redis-cache-advanced.md) | 缓存穿透/击穿/雪崩、Cache Aside、主从/哨兵/集群、淘汰策略(LRU/LFU) |
| 08 | [08-lsm-tree-storage.md](./08-lsm-tree-storage.md) | LSM-Tree 架构、WAL/MemTable/SSTable、Compaction、LevelDB vs RocksDB、B+ vs LSM |
| 09 | [09-database-advanced.md](./09-database-advanced.md) | 分库分表、读写分离、分布式事务(2PC/TCC/本地消息表)、Join 算法、主从复制 |

## 路线

MySQL(1-4)→Redis(5-7)→LSM存储引擎(8)→分布式进阶(9)。
