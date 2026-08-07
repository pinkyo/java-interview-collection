# SQL 优化与分库分表

---

## 一、SQL 优化步骤

```
1. 开启慢查询日志 → 定位慢 SQL
2. EXPLAIN 分析 → 查看执行计划
3. 分析索引使用情况 → 优化索引
4. 优化 SQL 写法 → 改写查询
5. 表结构优化 → 分库分表
```

### 1.1 慢查询日志

```sql
-- 开启慢查询
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;  -- 超过 1s 记录
SET GLOBAL log_queries_not_using_indexes = ON;  -- 记录未用索引的
```

### 1.2 慢查询分析工具

- **mysqldumpslow**：MySQL 自带，简单
- **pt-query-digest**（Percona Toolkit）：功能强大，推荐
- **EXPLAIN + 优化**

---

## 二、SQL 优化技巧

### 2.1 查询优化

```sql
-- ❌ SELECT * （不需要的列也查，IO 浪费，无法覆盖索引）
SELECT * FROM user;

-- ✅ 只查需要的列
SELECT id, name FROM user;

-- ❌ WHERE 中函数/计算
SELECT * FROM user WHERE YEAR(birthday) = 2000;

-- ✅ 直接比较
SELECT * FROM user WHERE birthday >= '2000-01-01' AND birthday < '2001-01-01';

-- ❌ 不等于（不走索引）
SELECT * FROM user WHERE status != 0;

-- ✅ 用 IN 替代（根据数据分布决定）
SELECT * FROM user WHERE status IN (1, 2, 3);

-- ❌ OR 条件不够优化
SELECT * FROM user WHERE name = 'a' OR age = 18;

-- ✅ 用 UNION 替代
SELECT * FROM user WHERE name = 'a'
UNION
SELECT * FROM user WHERE age = 18;
```

### 2.2 分页优化

```sql
-- ❌ 深分页（offset 很大时扫描大量数据）
SELECT * FROM user ORDER BY id LIMIT 100000, 20;

-- ✅ 延迟关联
SELECT * FROM user a
INNER JOIN (SELECT id FROM user ORDER BY id LIMIT 100000, 20) b
ON a.id = b.id;

-- ✅ 记录上次位置（最佳，但需要前端配合）
SELECT * FROM user WHERE id > 100000 ORDER BY id LIMIT 20;
```

### 2.3 JOIN 优化

```sql
-- ✅ 小表驱动大表
-- ✅ JOIN 的字段建索引
-- ✅ 避免 SELECT * 在 JOIN 查询中
-- ✅ 用 EXPLAIN 检查有没有 Using join buffer
```

### 2.4 COUNT 优化

```sql
-- COUNT(*) 和 COUNT(1) 没区别（都走优化）
-- COUNT(column) 不统计 NULL
-- MyISAM 的 COUNT(*) 有优化（记录总数），InnoDB 没有

-- 大表统计用近似值
SELECT TABLE_ROWS FROM information_schema.TABLES WHERE TABLE_NAME = 'user';
-- 或用 Redis 计数器
```

### 2.5 ORDER BY 优化

```sql
-- Using filesort 需要优化
-- 1. 给 ORDER BY 的字段建索引
-- 2. WHERE 和 ORDER BY 使用相同的索引
-- 3. ORDER BY 的顺序要和联合索引一致
```

---

## 三、分库分表 🔥

### 3.1 何时需要分库分表？

| 维度 | 阈值（参考） |
|------|------------|
| 单表数据量 | > 500W ~ 1000W |
| 单库 QPS | > 1000 |
| 单库磁盘 | 快满了 |

### 3.2 分库分表方式

#### 垂直拆分

```
          ┌──────────────────────┐
          │      原始大表          │
          │ id, name, avatar,    │
          │ detail, created_at   │
          └──────────────────────┘
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
  ┌───────────┐          ┌───────────┐
  │ 用户主表    │          │ 用户扩展表  │
  │ id, name  │          │ id, avatar│
  └───────────┘          │ , detail  │
                         └───────────┘
```

**垂直分库**：按业务拆分（用户库、订单库、商品库）

#### 水平拆分

```
              用户表 → 按 user_id 分片

          ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
          │ DB0  │  │ DB1  │  │ DB2  │  │ DB3  │
          ├──────┤  ├──────┤  ├──────┤  ├──────┤
          │ user │  │ user │  │ user │  │ user │
          │ _0   │  │ _1   │  │ _2   │  │ _3   │
          └──────┘  └──────┘  └──────┘  └──────┘
            0-7        8-15      16-23      24-31
              (user_id % 32 / 8 路由到对应库)

          ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
          │ user │  │ user │  │ user │  │ user │
          │ _0   │  │ _1   │  │ _2   │  │ _3   │
          └──────┘  └──────┘  └──────┘  └──────┘
          user_id % 4 = 分表
```

### 3.3 分片策略

| 策略 | 实现 | 优点 | 缺点 |
|------|------|------|------|
| 取模（Hash） | `db = id % N` | 分布均匀 | 扩容困难（迁移数据） |
| 范围（Range） | 1~1000W 一库 | 扩容方便 | 写热点（最新数据集中） |
| 一致性 Hash | 哈希环 | 扩容数据迁移少 | 实现复杂 |
| 时间 | 按月分表 | 归档方便 | 跨月查询困难 |

### 3.4 中间件

| 中间件 | 类型 | 特点 |
|--------|------|------|
| **ShardingSphere-JDBC** | 客户端 SDK | 阿里开源，最常用，支持分库分表+读写分离 |
| Mycat | 代理 | 需要独立部署，功能全面 |
| Vitess | 代理 | Google 开源的 MySQL 集群方案 |

### 3.5 分库分表后的问题

| 问题 | 解决方案 |
|------|---------|
| **分布式 ID** | 雪花算法、美团 Leaf |
| **跨库 JOIN** | 应用层组装 / 冗余字段 / 全局表 |
| **跨库事务** | 柔性事务（Seata、TCC、可靠消息） |
| **分页/排序** | 每个分片查 N 条排序后截取 / 搜索引擎辅助 |
| **扩容** | 一致性 Hash / 双写迁移 |

---

## 四、读写分离

```
         ┌─────────┐
         │ 应用服务  │
         └────┬────┘
              │
    ┌─────────┼─────────┐
    ▼         │         ▼
┌────────┐   │   ┌────────┐
│ Master │   │   │ Master │
│ (写)    │───┼──►│ (写)    │
└────────┘   │   └────────┘
    │Binlog   │
    ▼         │       ▼
┌────────┐   │   ┌────────┐
│ Slave1 │   │   │ Slave2 │
│ (读)    │   │   │ (读)    │
└────────┘   │   └────────┘
```

**追问：主从延迟问题如何处理？**

1. 写后立即读 → 强制读主库
2. 关键业务读主库
3. 容忍短暂延迟（非关键场景）
4. Semi-Sync 半同步复制（至少一个 Slave 确认收到 Binlog）

---

## 面试追问集

**Q：分库分表后如何保证全局唯一 ID？**

1. 雪花算法（Snowflake）：趋势递增，不依赖 DB
2. DB 号段模式：每次取一批 ID 缓存使用（美团 Leaf）
3. Redis 自增
4. UUID：无序，不适合做主键

**Q：你们公司用过什么分库分表方案？**

结合自己的经验回答。典型答案：**ShardingSphere-JDBC** + 雪花算法 ID + 按 biz_id 取模分片。
