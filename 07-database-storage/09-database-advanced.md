# 09 — 分库分表、分布式事务、Join 算法、主从复制

> 关联篇目：[02 索引](./02-mysql-index.md) | [01 InnoDB 架构](./01-mysql-innodb.md)

---

## 1. 分库分表

### 1.1 垂直拆分 vs 水平拆分

```
垂直拆分（按列/业务模块）:
┌──────────────┐     ┌────────┐  ┌────────┐
│ 大表          │ →   │ 用户表  │  │ 订单表  │
│ id, name,    │     └────────┘  └────────┘
│ order, addr  │
└──────────────┘

水平拆分（按行/数据范围）:
┌──────────────┐     ┌────────────┐
│ orders       │     │ orders_0   │ ← id % 4 = 0?
│ id:1..1M     │ →   │ orders_1   │ ← id % 4 = 1?
└──────────────┘     │ orders_2   │
                     │ orders_3   │
                     └────────────┘
```

### 1.2 分片键选择

| 策略 | 优点 | 缺点 |
|------|------|------|
| **取模 hash** | 分布均匀 | 扩容需要重新分布 |
| **范围分片** | 扩容简单 | 热点数据可能集中 |
| **一致性哈希** | 扩容影响小 | 实现复杂 |

### 1.3 跨分片问题

```
问题: SELECT * FROM orders WHERE user_id = ? AND status = ?
      → user_id 是分片键 → 路由到正确的片 ✅

      SELECT * FROM orders WHERE status = ?
      → 需要查所有片 → 结果合并（scatter-gather） ⚠️ 性能差

      JOIN 跨分片 → 几乎不可能高效 → 需要反范式/应用层 JOIN
```

---

## 2. 读写分离

```
         ┌─────────┐
 写请求 →│ Master  │
         └────┬────┘
              │ 主从复制 (Binlog)
    ┌─────────┼─────────┐
    ▼         ▼         ▼
┌──────┐ ┌──────┐ ┌──────┐
│Slave1│ │Slave2│ │Slave3│ ← 读请求分发到 Slave
└──────┘ └──────┘ └──────┘
```

### 2.1 主从延迟处理

主从复制有延迟（通常 < 1s，但高峰期可能到几秒）。写完立刻读可能读到旧数据：

```
解决方案:
1. 写后立即读强制走 Master
2. 延迟要求不高的场景读 Slave
3. 关键业务（如支付）全部走 Master
4. 用中间件（ShardingSphere）自动路由
```

---

## 3. 分布式事务

### 3.1 方案对比

| 方案 | 一致性 | 性能 | 复杂度 | 适用 |
|------|:---:|:---:|:---:|------|
| **2PC** (XA) | 强 | 低 | 中 | 传统数据库 |
| **TCC** (Try-Confirm-Cancel) | 最终 | 较高 | 高 | 业务层实现 |
| **Saga** | 最终 | 高 | 高 | 长事务、微服务 |
| **本地消息表** | 最终 | 中 | 中 | 异步解耦 |

### 3.2 2PC 两阶段提交

```
阶段 1 (Prepare): 协调者问所有参与者 → "能提交吗？"
阶段 2 (Commit):  所有都 YES → 提交；有 NO → 全部回滚

问题: 协调者单点、同步阻塞、阶段 2 失败时数据不一致
```

### 3.3 TCC

```
Try:    预留资源（冻结库存）
Confirm: 确认使用资源（扣减库存）
Cancel:  释放资源（解冻库存）

需要业务自己实现 Try/Confirm/Cancel 三个接口
比 2PC 更灵活，但开发量大
```

### 3.4 本地消息表

```
服务 A:
1. 执行业务 + 插入本地消息表（同一事务）
2. 异步发送 MQ → 服务 B

服务 B:
3. 消费消息 → 执行业务
4. 回调/回查 A 保证最终一致性
```

---

## 4. Join 算法

| 算法 | 原理 | 适用 |
|------|------|------|
| **Nested Loop** | 外层每行 → 扫描内层 | 小表 Join 大表（小表驱动） |
| **Block Nested Loop** | 外层分批 → 内层扫描 | 无索引的 Join |
| **Hash Join** | 建哈希表 → 匹配 | 等值 Join、大表 Join |
| **Merge Join** | 两表预排序 → 归并 | 已按 Join 列排序的表 |

```sql
-- MySQL 8.0 起支持 Hash Join
EXPLAIN SELECT * FROM orders o JOIN users u ON o.user_id = u.id;
-- Extra: Using where; Using join buffer (Hash Join)
```

---

## 5. MySQL 主从复制原理

```
Master                           Slave
  │                                │
  │ 事务提交                        │
  │ 写入 Binlog                     │
  │                                │
  │ ←─── IO Thread 拉取 Binlog ────│ 写入 Relay Log
  │                                │
  │                                │ SQL Thread 回放 Relay Log
  │                                │ → 数据同步
```

### 5.1 三种复制格式

| 格式 | 内容 | 优点 | 缺点 |
|------|------|------|------|
| **STATEMENT** | SQL 语句 | 日志量小 | 某些函数导致不一致（NOW()） |
| **ROW** | 行的变更 | 精确 | 日志量大 |
| **MIXED** | 混合 | 平衡 | — |

MySQL 5.7+ 默认 ROW。

### 5.2 GTID 复制

传统复制靠 `(binlog_file, position)` 定位，切换麻烦。**GTID**（全局事务 ID）：

```
每个事务有唯一的 GTID → Slave 自动找到需要从哪个事务开始同步
主从切换简化 → 高可用方案的基础
```

---

## 小结

| 概念 | 一句话 |
|------|--------|
| 垂直拆分 | 按业务模块拆分字段 |
| 水平拆分 | 按行拆分到多个库/表（取模/范围/一致性哈希） |
| 读写分离 | 写 Master、读 Slave，注意主从延迟 |
| 2PC | 两阶段提交，强一致但性能差 |
| TCC | Try-Confirm-Cancel，业务层补偿 |
| Hash Join | MySQL 8.0 新特性 |
| Binlog | 主从复制的核心，ROW 格式最精确 |

---

## 第七章小结

| # | 篇目 | 核心 |
|---|------|------|
| 01 | InnoDB 架构 | Buffer Pool / Redo / Undo / Double Write |
| 02 | B+ 树索引 | 聚簇/非聚簇/回表/覆盖索引/最左前缀/EXPLAIN |
| 03 | 事务与 MVCC | ACID / 隔离级别 / ReadView / 快照读 vs 当前读 |
| 04 | 锁机制 | Record/Gap/Next-Key Lock / 死锁 / 意向锁 |
| 05 | Redis 数据结构 | SDS/quicklist/dict/skiplist/编码切换 |
| 06 | Redis 持久化 | RDB/AOF/混合/事件循环/多线程 IO |
| 07 | 缓存与分布式 | 穿透击穿雪崩/Cache Aside/主从/哨兵/集群/淘汰 |
| 08 | LSM-Tree | WAL/MemTable/SSTable/Compaction/RocksDB vs B+ |
| 09 | 数据库进阶 | 分库分表/分布式事务/Join/主从复制 |
