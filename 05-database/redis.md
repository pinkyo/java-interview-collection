# Redis 核心知识

---

## 一、五种基本数据结构及底层实现

| 结构 | 用途 | 底层实现 |
|------|------|---------|
| **String** | 缓存、计数、分布式锁 | SDS（简单动态字符串） |
| **Hash** | 用户信息、购物车 | ziplist（小）/ hashtable（大） |
| **List** | 消息队列、最新列表 | ziplist（小）/ quicklist（默认） |
| **Set** | 好友/粉丝集合、共同关注 | intset（小且全整数）/ hashtable |
| **ZSet** | 排行榜、延时队列 | ziplist（小）/ skiplist+hashtable |

**追问：为什么 ZSet 要同时使用跳表和哈希表？**

- 跳表保证范围查询和排序效率（O(log n)）
- 哈希表保证按 member 查找分数的效率（O(1)）

---

## 二、特殊数据结构

| 结构 | 用途 | 说明 |
|------|------|------|
| **Bitmap** | 签到、布隆过滤器 | 位图，极省内存 |
| **HyperLogLog** | UV 统计 | 误差 0.81%，极小内存（12KB） |
| **GEO** | 附近的人、距离计算 | ZSet 封装 |
| **Stream** | 消息队列 | 类似 Kafka，消费者组 |

---

## 三、持久化 🔥

### 3.1 RDB（快照）

```bash
# 配置：N 秒内至少 M 次修改则触发
save 900 1      # 900 秒内 1 次修改
save 300 10     # 300 秒内 10 次修改
save 60 10000   # 60 秒内 10000 次修改
```

**优点**：恢复快、文件紧凑、适合备份
**缺点**：可能丢失最后一次快照后的数据、大内存下 fork 耗时长

### 3.2 AOF（追加日志）

```bash
appendonly yes
appendfsync always   # 每次写入都 fsync（最安全，性能最差）
appendfsync everysec # 每秒 fsync（推荐，最多丢 1s 数据）
appendfsync no       # 交给 OS（性能最好，安全性低）
```

**AOF 重写**：合并冗余命令，减小文件体积（BGREWRITEAOF）。

**追问：RDB 和 AOF 如何选择？**

- 追求数据安全：**RDB + AOF 混合**（Redis 4.0+）
- 可以接受少量数据丢失：只用 RDB
- 建议：生产环境两者都开启

### 3.3 混合持久化（Redis 4.0+）

```
AOF 文件结构：
[RDB 数据快照] + [增量 AOF 日志]
恢复：先加载 RDB 快照，再重放增量日志
```

**优点**：兼顾 RDB 的恢复速度和 AOF 的数据安全性。

---

## 四、高可用架构

### 4.1 主从复制

```
     ┌────────┐
     │ Master │  ─ 读写
     └───┬────┘
    ┌────┴────┐
    ▼         ▼
┌────────┐ ┌────────┐
│ Slave1 │ │ Slave2 │  ─ 只读
└────────┘ └────────┘
```

- 异步复制（Redis 默认）
- 全量同步（RDB） + 增量同步（offset 偏移量）
- **Replication ID + offset** 判断同步位置

### 4.2 哨兵模式（Sentinel）

```
          ┌──────────┐  ┌──────────┐  ┌──────────┐
          │ Sentinel│  │ Sentinel│  │ Sentinel│
          │    1    │  │    2    │  │    3    │
          └────┬─────┘  └────┬─────┘  └────┬─────┘
               │              │              │
               └──────────────┼──────────────┘
                              │ 监控 + 选举
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
         ┌────────┐     ┌────────┐      ┌────────┐
         │ Master │────►│ Slave  │─────►│ Slave  │
         └────────┘     └────────┘      └────────┘
```

- **监控**：Sentinel 向所有节点发 PING
- **判定**：多个 Sentinel 都认为 Master 挂了（quorum）→ 主观下线 → 客观下线
- **选举**：Raft 算法选举 Sentinel Leader → 选出新 Master
- **通知**：通知 Slave 和新 Master 同步，客户端切换连接

### 4.3 集群模式（Cluster）

```
           ┌────────────────────────────┐
           │        Redis Cluster       │
           │   ┌──────┐ ┌──────┐ ┌──────┐│
           │   │Slot  │ │Slot  │ │Slot  ││
           │   │0-5460│ │5461- │ │10923-││
           │   │      │ │10922 │ │16383 ││
           │   └──────┘ └──────┘ └──────┘│
           │   ┌──────┐ ┌──────┐ ┌──────┐│
           │   │Slave │ │Slave │ │Slave ││  ← 从节点（高可用）
           │   └──────┘ └──────┘ └──────┘│
           └────────────────────────────┘
```

- 16384 个 Hash Slot（CRC16(key) % 16384）
- 去中心化，节点间 Gossip 协议通信
- 每个分片有 Master+Slave

---

## 五、缓存策略 🔥

### 5.1 缓存穿透

**问题**：查询不存在的数据，请求直达数据库。

```
客户端 → Redis(无数据) → MySQL(无数据)
```

**解决方案：**

```java
// 1. 缓存空值（最简单）
if (dbValue == null) {
    redis.set(key, "null", 60, TimeUnit.SECONDS);  // 短期缓存空值
}

// 2. 布隆过滤器
BloomFilter<String> filter = BloomFilter.create(
    Funnels.stringFunnel(Charset.defaultCharset()),
    1000000,  // 预计元素数
    0.01      // 误判率
);
// 查询前先过布隆过滤器（可能存在但一定不会漏）
if (!filter.mightContain(key)) return null;
```

### 5.2 缓存击穿

**问题**：热点数据过期瞬间，大量请求打到数据库。

```java
// 1. 互斥锁
public String get(String key) {
    String value = redis.get(key);
    if (value != null) return value;
    // 只有一个线程去加载
    if (redis.setNX(lockKey, "1", 10, TimeUnit.SECONDS)) {
        try {
            value = db.query(key);
            redis.set(key, value, 60, TimeUnit.SECONDS);
        } finally {
            redis.del(lockKey);
        }
    } else {
        Thread.sleep(50);
        return get(key); // 重试
    }
    return value;
}

// 2. 热点数据永不过期（逻辑过期）
// 设置物理不过期 + 逻辑过期字段，异步刷新
```

### 5.3 缓存雪崩

**问题**：大量缓存在同一时间过期，或 Redis 宕机。

```java
// 1. 过期时间加随机值
int expire = baseExpire + new Random().nextInt(300);  // 避免同时过期

// 2. 多级缓存（本地 Caffeine + Redis）
// 3. Redis 高可用（集群/哨兵）
// 4. 限流 + 降级（Sentinel 限流）
```

### 5.4 缓存与数据库一致性问题

```
常见模式：
1. 先删缓存 → 再更新 DB → 双删延时（过时方案）
2. 先更新 DB → 再删缓存（推荐，极端情况下可能读到脏数据）
3. 订阅 Binlog → 异步更新缓存（Canal + MQ，最终一致性）
4. 缓存设置过期时间（兜底方案）
```

**追问：为什么先更新 DB 再删缓存优于先删缓存再更新 DB？**

- 先删缓存：A 删缓存 → B 读缓存 miss → B 读 DB（旧值）→ B 写缓存（旧值）→ A 更新 DB（新值）→ 缓存中永远是旧值
- 先更新 DB：B 读缓存 miss → B 读 DB（旧值）→ A 更新 DB → A 删缓存 → B 写缓存（旧值）→ 缓存中短暂旧值，但有过期时间兜底

**一致性最佳实践**：Canal 监听 Binlog → MQ → 异步刷新缓存 + 缓存设置过期时间兜底。

---

## 六、Redis 线程模型

- Redis 6.0 前：**单线程 Reactor 模型**（I/O 多路复用）
- Redis 6.0+：**多线程 I/O**（网络读写多线程，命令执行仍单线程）

**追问：为什么 Redis 单线程这么快？**

1. **纯内存操作**：纳秒级
2. **I/O 多路复用**：epoll 处理大量连接
3. **数据结构简单高效**：跳表、压缩链表等
4. **避免上下文切换和锁竞争**

---

## 七、过期策略与淘汰策略

### 7.1 过期键删除策略

| 策略 | 说明 |
|------|------|
| 定时删除 | 创建定时器，到期删除（CPU 不友好） |
| **惰性删除** | 访问 key 时检查是否过期（可能堆积过期 key） |
| **定期删除** | 每隔 100ms 随机取一批 key 删除 |

**Redis 采用：惰性删除 + 定期删除**

### 7.2 内存淘汰策略（8 种）

```bash
# 配置
maxmemory 4gb
maxmemory-policy allkeys-lru
```

| 策略 | 范围 | 行为 |
|------|------|------|
| noeviction | — | 不淘汰，写操作报错 |
| allkeys-lru | 所有 key | LRU（最近最少使用） |
| volatile-lru | 有过期时间的 key | LRU |
| allkeys-lfu | 所有 key | LFU（最不常用） |
| volatile-lfu | 有过期时间的 key | LFU |
| allkeys-random | 所有 key | 随机 |
| volatile-random | 有过期时间的 key | 随机 |
| volatile-ttl | 有过期时间的 key | 按 TTL 淘汰 |

**追问：Redis 的 LRU 和标准 LRU 有什么不同？**

标准 LRU 需要维护双向链表，开销大。Redis 使用**近似 LRU**：随机采样 N 个 key，淘汰最不活跃的那个。

---

## 八、分布式锁

```java
// Redisson 分布式锁（推荐）
RLock lock = redisson.getLock("myLock");
try {
    lock.lock(30, TimeUnit.SECONDS);  // 30s 自动续期（看门狗每 10s 续 30s）
    // 业务逻辑
} finally {
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

**追问：Redis 分布式锁需要注意什么？**

1. 加锁要设置过期时间（防止死锁）
2. 解锁必须检验是本线程加的锁（Lua 脚本保证原子性）
3. Redisson 看门狗自动续期
4. 集群模式建议用 **RedLock** 算法（争议较多，实际很少用）

---

## 面试追问集

**Q：Redis 和 Memcached 的区别？**

| 对比 | Redis | Memcached |
|------|-------|-----------|
| 数据结构 | 丰富（5种+高级） | 仅 String |
| 持久化 | RDB/AOF | 不支持 |
| 集群 | 原生支持 | 需客户端实现 |
| 线程模型 | 单线程（6.0+ IO 多线程） | 多线程 |
| 适用场景 | 缓存+消息队列+排行榜 | 纯缓存 |

**Q：Redis 管道（Pipeline）是什么？**

将多个命令打包一起发送，减少 RTT（往返时间），非原子操作。

**Q：Redis 事务（MULTI/EXEC）支持回滚吗？**

不支持。语法错误会全部不执行，运行时错误会跳过继续执行。
