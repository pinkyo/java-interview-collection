# 点赞系统设计

> 点赞是典型「写多读少、峰值高、允许最终一致」的场景。本文给出基于消息队列异步化 + 多级缓存的完整落地方案，覆盖写入链路、读取优化、幂等可靠性与异常兜底。

---

## 一、需求特征与约束

| 特征 | 说明 | 设计影响 |
|------|------|---------|
| 写多读少 | 海量点赞/取消，读集中在计数与状态展示 | 写入需异步削峰，读走缓存 |
| 峰值高 | 热门内容瞬间海量点赞 | MQ 攒批 + 热点 Key 拆分 |
| 幂等要求 | 同一用户对同一内容不重复计数 | 唯一约束 + 去重结构 |
| 最终一致 | 赞数短暂差几个无感知 | 异步落库 + 对账兜底 |
| 低延迟读 | 前端实时展示赞数/是否点赞 | 多级缓存，读不碰 MySQL |

**核心约束**：点赞是「状态变更」而非单纯流水，需同时维护「计数」与「用户-内容关系」两类数据，二者都要求高并发读写。

---

## 二、技术选型

### 2.1 消息队列：Kafka

依据（见 `mq-selection.md`）：

- **原生批量**：`batch.size` + `linger.ms` 把单条点赞攒成批量，契合"单条转批量"。
- **高吞吐**：顺序写 + 零拷贝 + Page Cache，扛峰值写入（百万级 QPS）。
- **分区有序**：`key = contentId` 哈希分区，保证同一内容点赞有序，消费端可按内容聚合。
- **可回溯**：故障时按 offset 重放补数。
- 不需要 RabbitMQ 的复杂路由 / 延迟消息，故不选。

### 2.2 存储：Redis + MySQL 分层

- **Redis（实时层）**：内存读写，扛高并发，存计数与点赞关系，直接服务前端。
- **MySQL（持久层）**：存点赞明细与计数快照，`唯一索引 (userId, contentId)` 去重 + 幂等，异步批量落库。
- 超大规模明细可归档至 HBase / Cassandra，中小规模 MySQL 足够。

---

## 三、整体架构

### 3.1 分层视图

```
┌─────────────────────────────────────────────────────┐
│ 接入层   前端 / API 网关（限流、鉴权、参数校验）        │
├─────────────────────────────────────────────────────┤
│ 服务层   点赞服务：状态校验 → 写 MQ → 返回            │
│          读服务：  多级缓存查询 → 回源兜底             │
├─────────────────────────────────────────────────────┤
│ 消息层   Kafka（topic: like_event, key=contentId）    │
├─────────────────────────────────────────────────────┤
│ 存储层   Redis（实时计数+关系）  MySQL（明细+快照）    │
└─────────────────────────────────────────────────────┘
```

### 3.2 写入链路

```
用户点赞（单条）
   │
   ▼
点赞服务 ──状态校验(读 Redis 是否已赞)──┐
   │                                    │
   ▼                                    │ 已赞→直接返回(幂等)
Kafka（like_event, key=contentId）     │ 未赞→入队
   │  攒批（linger.ms + batch.size）    │
   ▼                                    │
消费者（批量拉取，按 contentId 聚合）◄──┘
   │
   ├─► Redis：HINCRBY like:count {contentId} Δ
   │         SADD / SETBIT like:users:{contentId} {userId}
   │
   └─► MySQL：批量 INSERT ... ON DUPLICATE KEY UPDATE（明细）
             批量 UPDATE content SET like_count = ?（快照）
```

**写入流程要点**：

1. **前置幂等**：API 先查 Redis「是否已赞」，命中则直接返回，避免无效入队；但这只是优化，**真正幂等靠 DB 唯一键兜底**（防并发下两请求同时判"未赞"）。
2. **异步削峰**：发 MQ 后立即返回，不阻塞写库。
3. **聚合落库**：消费者按 `contentId` 合并多条点赞，对计数做 `INCRBY Δ`、对明细做批量 upsert，把高频随机写转成低频批量写。

---

## 四、数据结构设计

### 4.1 Redis

| 用途 | 结构 | Key 示例 | 说明 |
|------|------|---------|------|
| 计数 | String / Hash | `like:count:{contentId}` | `INCR/HINCRBY`，读 `GET` |
| 是否点赞（小规模） | Set | `like:users:{contentId}` | `SADD` / `SISMEMBER`，可全量取成员 |
| 是否点赞（大规模/省内存） | Bitmap | `like:bit:{contentId}` | `SETBIT {userId} 1` / `GETBIT`，O(1)，极省内存 |
| 热点拆分 | String | `like:count:{contentId}:{shard}` | 超级热点拆多片，读时 `SUM` |

**Set vs Bitmap 取舍**：
- Set 适合「需枚举点赞用户列表」且单内容点赞数不大；
- Bitmap 适合「只需判断点没点过」且用户 ID 可映射为连续整数，内存最省。

### 4.2 MySQL

```sql
-- 点赞明细表（幂等核心）
CREATE TABLE like_record (
  id         BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id    BIGINT NOT NULL,
  content_id BIGINT NOT NULL,
  status     TINYINT NOT NULL DEFAULT 1,  -- 1=赞 0=取消
  create_time DATETIME,
  update_time DATETIME,
  UNIQUE KEY uk_user_content (user_id, content_id)  -- 唯一约束防重复
);

-- 计数快照（冗余，便于按计数排序/分页）
ALTER TABLE content ADD COLUMN like_count INT DEFAULT 0;
```

---

## 五、读取优化（降低 MySQL 压力）🔥

核心原则：**读请求尽量不碰 MySQL**，热数据放内存层，MySQL 只做兜底与持久化。

### 5.1 多级缓存读路径

```
前端读点赞数 / 是否点过赞
        │
        ▼
   ┌──────────┐  命中
   │ 本地缓存  │ ──────► 返回   （Caffeine，TTL 秒级，存超热点）
   └────┬─────┘
        │ 未命中
        ▼
   ┌──────────┐  命中
   │  Redis   │ ──────► 返回 + 回写本地缓存
   └────┬─────┘
        │ 未命中
        ▼
   ┌──────────┐  回源(加互斥锁)
   │ MySQL 从库│ ──────► 回写 Redis + 本地缓存
   └──────────┘
```

### 5.2 具体手段

1. **计数读走 Redis**：`GET like:count:{contentId}`，完全绕开 MySQL。
2. **本地缓存（L1）**：Caffeine 缓存超热点计数，TTL 短（几秒~几十秒），热点内容近乎零远程调用（允许短暂误差）。
3. **状态读走 Redis**：`SISMEMBER` / `GETBIT` 判断是否点过赞，O(1)。
4. **批量聚合读**：列表页一次 `MGET` 计数 + `SMISMEMBER`/多条 `GETBIT` 批量判状态，避免 N+1。
5. **计数异步落 MySQL**：MySQL 不做实时读，周期/阈值触发把 Redis 计数批量刷盘，随机读转低频批量写。
6. **热点 Key 拆分**：超级热点拆 `like:count:{contentId}:{shard}`，读 `SUM`、写分散，防单 key 瓶颈。
7. **读写分离**：回源走 MySQL 从库，主库仅承接 Kafka 批量写。

### 5.3 一致性保障

- **缓存更新策略**：写操作「先更 DB，再删缓存」（Cache-Aside 删除模式）；但本场景中计数主要由 MQ 消费者统一维护，更推荐**消费者先更 Redis 再异步刷 MySQL**，读始终读 Redis。
- **短暂不一致可接受**：计数最终一致即可。
- **防缓存击穿**：热点 key 失效用 `SETNX` 互斥锁，只放一个请求回源。
- **防缓存雪崩**：TTL 加随机抖动。
- **防缓存穿透**：不存在的 contentId 缓存空值（短 TTL）或布隆过滤器拦截。

---

## 六、幂等与可靠性

| 问题 | 方案 |
|------|------|
| 重复点赞 | MySQL 唯一索引 `(user_id, content_id)` + Redis Set/Bitmap 去重（API 前置判断 + DB 兜底） |
| 消息重复消费 | Kafka 手动提交 offset（处理成功后再提交）+ 唯一键 upsert 幂等 |
| 计数少计/多计 | 取消点赞对称 `DECR`；定期对账（MySQL 明细 count vs Redis 计数）修复 |
| 消息丢失 | `acks=all` + ISR + 重试；消费端处理成功再提交 offset |
| Redis 宕机丢数据 | 开启 AOF；关键计数定期快照到 MySQL，故障后从快照 + Kafka 重放恢复 |
| 消费失败 | 死信队列（DLQ）兜底，告警 + 人工/自动重试 |

**取消点赞**：与点赞对称，状态置 0、`SREM`/`SETBIT 0`、计数 `DECR`，同样走 MQ 异步，靠唯一键 + status 字段保证幂等。

---

## 七、异常与兜底设计

- **MQ 积压**：监控 lag，超阈值自动扩容消费者分区并行度。
- **Redis 与 MySQL 不一致**：定时对账任务比对，以 MySQL 明细为准校准 Redis。
- **热点突发**：本地缓存 + 热点拆分 + 限流（令牌桶）三道防线。
- **降级**：Redis 不可用时，读降级直查 MySQL 从库 + 限流；写降级可暂存本地队列异步补发 MQ。
- **监控指标**：MQ lag、消费速率、Redis 命中率、缓存击穿次数、对账差异量、接口 P99。

---

## 八、面试要点速记

1. **为什么 Kafka 不用 RabbitMQ**：要高吞吐 + 原生批量攒写，而非复杂路由/延迟任务。
2. **为什么 Redis + MySQL**：Redis 扛实时高并发读写，MySQL 做持久化 + 唯一去重。
3. **怎么降 MySQL 读压力**：多级缓存（本地 + Redis），读路径绕开 MySQL，MySQL 仅异步批量写。
4. **怎么保证不重复计数**：唯一索引 + Set/Bitmap 去重 + 消费幂等 + API 前置判断。
5. **计数与关系如何一致**：消费者统一维护 Redis，MySQL 定期快照 + 对账校准。
6. **缓存三大问题**：击穿（互斥锁）、雪崩（TTL 抖动）、穿透（空值/布隆）。
