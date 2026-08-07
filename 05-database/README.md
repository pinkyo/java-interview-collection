# 数据库

> 数据库是后端面试的另一个核心模块，MySQL 和 Redis 几乎是每家公司必问的内容。

## 📂 内容索引

| 文件 | 核心知识点 |
|------|-----------|
| [mysql-architecture.md](./mysql-architecture.md) | 整体架构、存储引擎对比（InnoDB/MyISAM）、SQL 执行流程 |
| [mysql-index.md](./mysql-index.md) | B+Tree 原理、聚簇与非聚簇索引、索引优化策略、失效场景 |
| [mysql-transaction-lock.md](./mysql-transaction-lock.md) | ACID、隔离级别、MVCC、行锁/表锁/间隙锁、死锁 |
| [mysql-optimization.md](./mysql-optimization.md) | SQL 优化、Explain 分析、分库分表方案、慢查询优化 |
| [redis.md](./redis.md) | 五种数据结构、持久化（RDB/AOF）、集群、缓存策略 |

## 🔥 高频考点速览

1. **B+Tree**：为什么使用 B+Tree？与 B-Tree/Hash 的对比
2. **聚簇索引**：回表查询、覆盖索引、最左前缀原则
3. **MVCC**：ReadView、undo log、RR 与 RC 下 MVCC 的差异
4. **事务隔离级别**：脏读/不可重复读/幻读定义与解决
5. **SQL 优化**：Explain 各字段含义、索引优化实战
6. **Redis**：数据结构底层实现、缓存穿透/击穿/雪崩、分布式锁
