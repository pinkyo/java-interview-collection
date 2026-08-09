# Elasticsearch

---

## 一、核心概念对比

| ES 概念 | 类比关系型数据库 |
|---------|----------------|
| Index（索引） | Database |
| Type（已废弃） | Table |
| Document | Row |
| Field | Column |
| Mapping | Schema |
| DSL | SQL |

---

## 二、倒排索引 🔥

### 2.1 什么是倒排索引

```
文档1：Java is great
文档2：Java is awesome
文档3：Python is great

正排索引（文档 → 词）：
  文档1: [Java, is, great]
  文档2: [Java, is, awesome]
  文档3: [Python, is, great]

倒排索引（词 → 文档）：
  Java   → [文档1, 文档2]
  is     → [文档1, 文档2, 文档3]
  great  → [文档1, 文档3]
  awesome → [文档2]
  Python → [文档3]
```

### 2.2 倒排索引的组成

```
Term Dictionary（词典）→ 按顺序存储所有词

Term    │  Posting List（倒排列表）
────────┼──────────────────
Java    │ [Doc1(offset=0), Doc2(offset=0)]
great   │ [Doc1(offset=2), Doc3(offset=2)]

             ↑
   ┌─────────┴──────────┐
   │ Term Index (FST)   │  ← 快速定位到词典中某个词的位置
   │ 前缀索引，内存中     │
   └────────────────────┘
```

**FST（Finite State Transducer）**：内存中前缀压缩，快速定位到 Term Dictionary，再到 Posting List。

---

## 三、写入与检索原理

### 3.1 写入流程

```
1. 文档写入 → 写入内存 Buffer（不可搜）
2. 同时写入 Translog（WAL，防止丢失）
3. refresh（默认 1s）→ Buffer 刷到 Segment（可搜）
4. flush → Segment 持久化到磁盘 → 清空 Translog
```

### 3.2 Segment 合并

```
多个小 Segment → Merge → 一个大 Segment
     ↑                      ↑
   碎片多                   更高效
```

### 3.3 检索流程

```
1. 查询 → 分发到所有 shard
2. 每个 shard 内：
   Query Phase：使用倒排索引检索，返回 doc ID + 排序值
   Fetch Phase：根据 doc ID 获取完整文档
3. 协调节点（Coordinating Node）汇总结果并排序返回
```

---

## 四、分片机制

```
             ┌───────────────────────────┐
             │        Index: blog        │
             │  5 primary shards         │
             │  1 replica per shard      │
             └───────────────────────────┘
    
    ┌─────────────────────────────────────────────┐
    │                 ES Cluster                   │
    │                                              │
    │  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
    │  │  Node 1  │  │  Node 2  │  │  Node 3  │   │
    │  │ P0 P1 R2 │  │ R0 P2 R3 │  │ R1 P3 R4 │   │
    │  └──────────┘  └──────────┘  └──────────┘   │
    │  P = Primary  R = Replica                     │
    └─────────────────────────────────────────────┘
```

| 分片 | 说明 |
|------|------|
| Primary Shard | 主分片，写入时先写到 Primary |
| Replica Shard | 副本分片，从 Primary 同步，提高可用性和读吞吐 |

**追问：写索引的路由规则？**

```
shard = hash(_routing) % number_of_primary_shards
// _routing 默认 = _id，可自定义
```

这也是为什么**主分片数创建后不能修改**（路由公式变了）。

---

## 五、聚合分析

```json
// 桶聚合（Bucket）→ 按条件分组
{
  "aggs": {
    "by_category": {
      "terms": { "field": "category" }
    }
  }
}

// 指标聚合（Metric）→ 计算数值
{
  "aggs": {
    "avg_price": {
      "avg": { "field": "price" }
    }
  }
}

// 管道聚合 → 对聚合结果再聚合
```

---

## 六、性能优化

### 6.1 写入优化

```json
// 批量写入
POST _bulk
{"index":{"_index":"blog","_id":"1"}}
{"title":"Hello","content":"World"}

// 调整 refresh 间隔（对实时性要求不高时）
PUT /blog/_settings
{ "refresh_interval": "30s" }

// 首次批量导入前关闭 refresh 和 replica
PUT /blog/_settings
{ "refresh_interval": "-1", "number_of_replicas": 0 }
// 导入完成后恢复
```

### 6.2 查询优化

- 使用 Filter Context（不需要评分，走缓存，更快）
- 避免 `script` 查询
- 使用 `routing` 减少搜索的分片数
- 深度分页用 `search_after` 替代 `from/size`

```json
// search_after 需要排序字段唯一
GET /blog/_search
{
  "size": 100,
  "sort": [{"id": "asc"}],
  "search_after": [1000]
}
```

---

## 七、集群相关

### 集群状态

```
Yellow：所有 Primary 分配好了，部分 Replica 未分配
Green：全部 Primary + Replica 分配好了（健康）
Red：部分 Primary 未分配（丢失数据！）
```

---

## 八、性能指标与特性

### 8.1 吞吐量与延迟（经验值，3 节点集群）

| 指标 | 量级 | 说明 |
|------|------|------|
| 写入吞吐 | **万级 doc/s**（bulk 批量更高） | refresh 间隔、副本数影响明显 |
| 查询 QPS | **千 ~ 万级 QPS** | 取决于复杂度、数据量、缓存命中 |
| 查询延迟 | **毫秒 ~ 百毫秒级** | 简单 term 查询毫秒级，聚合/深翻页更高 |
| 数据持久化 | 分片 + 副本（近实时，refresh 默认 1s） | 近实时搜索 |

> 注：量级受分片数、mapping 设计、堆内存、查询复杂度影响；bulk 写入可显著提升吞吐。

### 8.2 核心特性速查

| 特性 | Elasticsearch |
|------|---------------|
| 设计定位 | 分布式搜索 / 分析引擎 |
| 数据模型 | 文档（JSON）+ 倒排索引 |
| 读扩展 | Replica 分片提升读吞吐 |
| 实时性 | 近实时（默认 1s refresh） |
| 聚合分析 | 内置 Bucket/Metric/Pipeline |
| 适用场景 | 全文检索、日志分析、聚合统计 |

---

## 面试追问集

**Q：ES 为什么快？**

1. 倒排索引（核心）
2. 分布式的 Lucene
3. Segment 和缓存机制
4. FST 快速定位 Term

**Q：ES 和 MySQL 如何保证数据一致性？**

1. Canal 监听 MySQL Binlog → MQ → ES 更新
2. 定时任务同步（有延迟）
3. 业务双写（需要事务/补偿）

**Q：ES 如何处理深分页？**

- from+size：限制 10000（`max_result_window`），深度分页性能差
- scroll：生成快照，持续遍历（非实时、占用资源）
- search_after：实时，高性能，需要业务配合
