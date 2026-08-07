# MySQL 索引原理与优化

---

## 一、索引数据结构演进

| 数据结构 | 问题 |
|---------|------|
| 哈希表 | 无法范围查询、排序 |
| 二叉树 | 极端情况退化为链表 |
| 平衡二叉树(AVL) | 树高太大（百万级数据 ~20 层） |
| B-Tree | 一个节点存多个键，但仍存了数据 |
| **B+Tree** ✅ | 非叶子节点只存键，叶子节点存数据+链表 |

---

## 二、B+Tree 为什么适合索引？🔥

```
              ┌─────┬─────┬─────┐
              │ 10  │ 20  │ 30  │   ← 非叶子节点：只存键值
              └──┬──┴──┬──┴──┬──┘
                 │     │     │
    ┌────────────┼─────┼─────┼────────────┐
    ▼            ▼     ▼     ▼            ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│ 1→5→8  │→│11→12→15│→│22→23→28│→│31→35→38│   ← 叶子节点：存数据 + 双向链表
└────────┘ └────────┘ └────────┘ └────────┘
```

**优点**：
1. **矮胖**：一个节点存多个键，树高度很低（百万级数据 3 层）
2. **磁盘友好**：节点大小 = 磁盘页大小（16KB），一次 I/O 加载一个节点
3. **范围查询**：叶子节点有双向链表，直接遍历
4. **有序**：叶子节点按键值排序，支持排序查询

**追问：为什么不用 B-Tree？**

B-Tree 非叶子节点也存数据，导致每个节点能存的键减少，树变高。B+Tree 数据只存叶子节点，非叶子节点能存更多键，更矮、I/O 更少。

**追问：为什么不用 Hash 索引？**

Hash 只能等值查询，不支持范围查询、排序、最左前缀匹配。

---

## 三、聚簇索引 vs 非聚簇索引 🔥

| 对比 | 聚簇索引（Clustered） | 非聚簇索引（Secondary） |
|------|---------------------|------------------------|
| 数据存储 | 叶子节点就是数据行 | 叶子节点存主键值 |
| 数量 | 只能有 1 个 | 可以有多个 |
| 查找行数据 | 直接获取 | **回表**查询 |
| 默认 | 主键索引 | 普通索引 |

```
聚簇索引：
┌──────────────────────────────┐
│ 主键索引树 (叶子节点 = 完整行)  │
│ [id=5] [id=10] [id=15]      │  直接拿到全部数据
└──────────────────────────────┘

非聚簇索引 (回表)：
┌────────────────┐     ┌──────────────────────┐
│ 普通索引树      │ ──► │  聚簇索引树 (主键查找)  │
│ (叶子=主键id)   │     │  (叶子=完整行)         │
│ [name=zhang,id=5]   │ [id=5,name=zhang,age=18]
└────────────────┘     └──────────────────────┘
```

### 覆盖索引（Covering Index）

```sql
-- 创建联合索引
CREATE INDEX idx_name_age ON user(name, age);

-- 覆盖索引：查询列都在索引中，不需要回表
SELECT name, age FROM user WHERE name = 'zhang';
-- Extra: Using index ← 覆盖索引

-- 不回表的查询
SELECT * FROM user WHERE name = 'zhang';
-- Extra: Using index condition ← 需要回表
```

---

## 四、索引类型

| 类型 | 说明 |
|------|------|
| 主键索引（PRIMARY KEY） | 聚簇索引，唯一不为空 |
| 唯一索引（UNIQUE） | 值唯一，可为 NULL |
| 普通索引（INDEX） | 无约束 |
| 联合索引（复合索引） | 多列组合（最左前缀原则） |
| 全文索引（FULLTEXT） | 文本搜索 |
| 前缀索引 | 对长字符串取前 N 位 |

---

## 五、最左前缀原则 🔥

```sql
-- 联合索引 (a, b, c) 相当于创建了三个索引：
-- 索引(a)
-- 索引(a, b)
-- 索引(a, b, c)

-- ✅ 命中索引
SELECT * FROM t WHERE a = 1 AND b = 2 AND c = 3;
SELECT * FROM t WHERE a = 1 AND b = 2;
SELECT * FROM t WHERE a = 1;
SELECT * FROM t WHERE a = 1 AND c = 3;  -- 只用到 a

-- ❌ 不命中
SELECT * FROM t WHERE b = 2 AND c = 3;   -- 没有 a
SELECT * FROM t WHERE b = 2;
```

**追问：查询条件中 a 和 b 顺序颠倒还能命中索引吗？**

可以。MySQL 优化器会自动调整顺序。但**联合索引中列的顺序有讲究**：区分度高的列放前面。

---

## 六、索引失效场景

```sql
-- 1. 函数/计算
SELECT * FROM user WHERE YEAR(create_time) = 2024;   -- ❌
SELECT * FROM user WHERE create_time >= '2024-01-01'; -- ✅

-- 2. 隐式类型转换
SELECT * FROM user WHERE phone = 13800138000;  -- phone 是 varchar ❌

-- 3. 前导模糊查询
SELECT * FROM user WHERE name LIKE '%zhang';    -- ❌
SELECT * FROM user WHERE name LIKE 'zhang%';    -- ✅

-- 4. OR 两边都有索引才生效
SELECT * FROM user WHERE name = 'a' OR age = 18;  -- age 没索引 ❌

-- 5. NOT / != / <>
SELECT * FROM user WHERE status != 1;   -- 全表扫描概率大

-- 6. IS NULL / IS NOT NULL（视数据分布）
SELECT * FROM user WHERE name IS NULL;  -- NULL 多了可能走，少了全表

-- 7. 联合索引不满足最左前缀（见前文）

-- 8. 范围查询右边的列索引失效
-- 联合索引 (a, b, c)
SELECT * FROM t WHERE a = 1 AND b > 2 AND c = 3;
-- b 范围查询后，c 的索引失效
```

---

## 七、Explain 分析

```sql
EXPLAIN SELECT * FROM user WHERE name = 'zhang';
```

| 字段 | 含义 | 重要程度 |
|------|------|---------|
| id | 查询序列号（id 越大越先执行） | ⭐⭐ |
| **select_type** | 查询类型（SIMPLE/PRIMARY/SUBQUERY...） | ⭐⭐ |
| **type** | 访问类型 | ⭐⭐⭐⭐ |
| **possible_keys** | 可能使用的索引 | ⭐⭐ |
| **key** | 实际使用的索引 | ⭐⭐⭐ |
| key_len | 索引使用长度 | ⭐⭐⭐ |
| ref | 与索引比较的列 | ⭐⭐ |
| **rows** | 预估扫描行数 | ⭐⭐⭐ |
| filtered | 按条件过滤的百分比 | ⭐⭐ |
| **Extra** | 额外信息 | ⭐⭐⭐⭐ |

### type 访问类型（从好到差）

```
system > const > eq_ref > ref > range > index > ALL
                                        ▼
                                 (全索引扫描，比全表稍好)
```

| type | 说明 |
|------|------|
| const | 主键/唯一索引等值查询 |
| eq_ref | 唯一索引关联查询（join 时） |
| ref | 非唯一索引等值查询 |
| range | 索引范围查询 |
| index | 全索引扫描 |
| ALL | 全表扫描（最差） |

### Extra 关键信息

| Extra 值 | 含义 |
|---------|------|
| Using index | 覆盖索引 ✅ |
| Using where | 使用 where 过滤 |
| Using index condition | ICP 条件过滤 |
| Using temporary | 用了临时表 ⚠️ |
| Using filesort | 文件排序 ⚠️（需优化） |
| Using join buffer | join 时用了缓冲区 |

---

## 八、索引设计原则

1. 为 `WHERE / JOIN / GROUP BY / ORDER BY` 的字段建索引
2. 区分度高的列放联合索引前面
3. 频繁更新的列不建太多索引（维护成本高）
4. 字符串类型建前缀索引
5. 避免冗余索引（`(a,b)` 和 `(a)` 只需要 `(a,b)`）

---

## 面试追问集

**Q：为什么主键推荐自增 ID？**

1. 插入时总是在 B+Tree 最后追加，不会引发页分裂
2. 非自增（UUID）会导致随机插入，频繁页分裂，碎片多
3. 聚簇索引叶子节点物理有序，自增 ID 利用磁盘顺序读写

**Q：索引下推（ICP）是什么？**

MySQL 5.6+ 的特性。将 `WHERE` 条件中能用索引过滤的部分提前在存储引擎层过滤，减少回表次数。

```sql
-- 联合索引 (name, age)
SELECT * FROM user WHERE name LIKE '张%' AND age = 18;
-- 无 ICP：name 过滤后全部回表，再过滤 age
-- 有 ICP：索引中直接过滤 name AND age，再回表
```
