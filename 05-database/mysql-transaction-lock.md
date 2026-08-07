# MySQL 事务与锁机制

---

## 一、ACID 特性

| 特性 | 含义 | 实现机制 |
|------|------|---------|
| 原子性（Atomicity） | 事务要么全部成功，要么全部失败 | Undo Log |
| 一致性（Consistency） | 事务前后数据保持一致性 | 其他三者共同保证 |
| 隔离性（Isolation） | 多个事务互不干扰 | 锁 + MVCC |
| 持久性（Durability） | 事务提交后数据永久保存 | Redo Log |

---

## 二、事务隔离级别 🔥

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
|---------|------|-----------|------|
| READ UNCOMMITTED | ✅ | ✅ | ✅ |
| READ COMMITTED（RC） | ❌ | ✅ | ✅ |
| REPEATABLE READ（RR，默认） | ❌ | ❌ | ✅（部分解决） |
| SERIALIZABLE | ❌ | ❌ | ❌ |

**概念解释：**
- **脏读**：读到其他事务未提交的数据
- **不可重复读**：同一事务两次查询之间，其他事务修改并提交了数据
- **幻读**：同一事务两次查询之间，其他事务插入/删除了数据

```sql
-- 设置隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

**追问：RR 下如何解决幻读？**

通过 **Next-Key Lock**（行锁 + 间隙锁）解决：
- 行锁锁住已有记录
- 间隙锁锁住记录之间的范围，防止插入
- 例如 `SELECT * FROM t WHERE id BETWEEN 10 AND 20 FOR UPDATE`，不仅锁住 10~20 之间的记录，还锁住这段范围

但在快照读下，MVCC 即可避免幻读（除非 `FOR UPDATE`）。

---

## 三、MVCC 多版本并发控制 🔥

### 3.1 核心思想

MVCC 通过**不阻塞读操作**来提高并发，读不加锁，写加锁。

### 3.2 实现原理

```
每行记录有两个隐藏字段：
- trx_id：创建/修改此行的事务 ID
- roll_pointer：指向 undo log 版本链的指针

ReadView：读操作时生成的一致性视图
- m_ids：活跃事务 ID 列表
- min_trx_id：最小活跃事务 ID
- max_trx_id：下一个分配的事务 ID
- creator_trx_id：创建 ReadView 的事务 ID
```

**版本链可见性判断：**

```
如果 trx_id == creator_trx_id → 可见（自己的修改）
如果 trx_id < min_trx_id → 可见（已提交）
如果 trx_id > max_trx_id → 不可见（还未开始）
如果 min_trx_id <= trx_id <= max_trx_id：
    如果 trx_id 在 m_ids 中 → 不可见（活跃中）
    否则 → 可见（已提交）
```

### 3.3 RC 和 RR 下 MVCC 的差异

| 对比 | READ COMMITTED | REPEATABLE READ |
|------|---------------|-----------------|
| ReadView 生成时机 | 每次查询都生成新 ReadView | 第一次查询生成，事务内复用 |
| 效果 | 能看到其他事务已提交的修改 | 读取的是事务开始时的快照，不受其他事务影响 |

### 3.4 快照读 vs 当前读

```sql
-- 快照读：读 MVCC 版本（不加锁）
SELECT * FROM user WHERE id = 1;

-- 当前读：读最新版本（加锁）
SELECT * FROM user WHERE id = 1 FOR UPDATE;
SELECT * FROM user WHERE id = 1 LOCK IN SHARE MODE;
UPDATE user SET name = 'x' WHERE id = 1;
DELETE FROM user WHERE id = 1;
-- 当前读读取的是最新已提交的数据，并且会对记录加锁
```

---

## 四、InnoDB 锁机制 🔥

### 4.1 锁类型

| 锁 | 粒度 | 说明 |
|----|------|------|
| 表锁 | 表 | 意向锁（IS/IX），表锁和行锁共存时防止冲突 |
| 行锁 | 行 | Record Lock / Gap Lock / Next-Key Lock |
| 共享锁（S） | 行/表 | SELECT ... LOCK IN SHARE MODE |
| 排他锁（X） | 行/表 | UPDATE / DELETE / SELECT ... FOR UPDATE |

### 4.2 三种行锁

```
假设表数据：id = 5, 10, 15

Record Lock（记录锁）：
  SELECT * FROM t WHERE id = 10 FOR UPDATE;
  → 只锁 id=10 这行

Gap Lock（间隙锁）：
  SELECT * FROM t WHERE id = 8 FOR UPDATE;
  → 锁住 (5, 10) 这个间隙，防止插入 id=6,7,8,9

Next-Key Lock（临键锁）= Record Lock + Gap Lock：
  SELECT * FROM t WHERE id >= 10 AND id <= 15 FOR UPDATE;
  → 锁住 (5,10] + (10,15] + (15,+∞) 区间
```

**追问：间隙锁在什么隔离级别下生效？**

只在 **REPEATABLE READ** 级别下生效。RC 级别没有间隙锁。

### 4.3 锁兼容性

|  | 共享锁（S） | 排他锁（X） | 意向共享锁（IS） | 意向排他锁（IX） |
|--|-----------|-----------|----------------|----------------|
| S | ✅ | ❌ | ✅ | ❌ |
| X | ❌ | ❌ | ❌ | ❌ |
| IS | ✅ | ❌ | ✅ | ✅ |
| IX | ❌ | ❌ | ✅ | ✅ |

---

## 五、死锁

### 5.1 常见死锁场景

```sql
-- 事务 A
UPDATE t SET name = 'a' WHERE id = 10;  -- 锁住 id=10
UPDATE t SET name = 'a' WHERE id = 20;  -- 等待 id=20

-- 事务 B
UPDATE t SET name = 'b' WHERE id = 20;  -- 锁住 id=20
UPDATE t SET name = 'b' WHERE id = 10;  -- 等待 id=10

-- → 死锁！
```

### 5.2 死锁检测

```sql
-- 查看最近一次死锁
SHOW ENGINE INNODB STATUS;

-- 查看当前锁等待
SELECT * FROM information_schema.INNODB_LOCKS;
SELECT * FROM information_schema.INNODB_LOCK_WAITS;
```

### 5.3 死锁处理

1. InnoDB 自动检测，回滚代价较小的事务
2. 业务上保证一致的加锁顺序
3. 尽量缩短事务
4. 降低隔离级别（RR → RC）

---

## 面试追问集

**Q：为什么 RR 是默认隔离级别？**

1. 解决不可重复读和大部分幻读问题
2. Binlog 格式为 STATEMENT 时需要 RR，否则主从可能不一致
3. 性能优于 SERIALIZABLE

**Q：事务的原子性和持久性怎么实现？**

- 原子性：Undo Log（出错了回滚到之前状态）
- 持久性：Redo Log（即使宕机也能恢复已提交的数据）

**Q：一条 UPDATE 语句加锁的过程？**

```
1. 通过索引找到目标记录（聚簇索引或二级索引）
2. 对二级索引记录加 Next-Key Lock
3. 对对应的聚簇索引记录加 Record Lock
4. 如果更新了索引列，会有额外的锁操作
```
