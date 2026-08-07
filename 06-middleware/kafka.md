# Kafka

---

## 一、核心架构

```
                     ┌──────────────────────┐
                     │        ZooKeeper      │
                     │  (Kraft 模式可替代)    │
                     └──────────┬───────────┘
                                │
          ┌────────────────┬────┴────┬────────────────┐
          ▼                ▼         ▼                ▼
    ┌──────────┐    ┌──────────┐  ┌──────────┐  ┌──────────┐
    │  Broker  │    │  Broker  │  │  Broker  │  │  Broker  │
    │    1     │    │    2    │  │    3    │  │    4    │
    └──────────┘    └──────────┘  └──────────┘  └──────────┘
    
    ┌─────────────────────────────────────────────────────────┐
    │                   Topic: order                          │
    │  Partition 0  │  Partition 1  │  Partition 2  │  P3    │
    │  [0][1][2]...  │  [0][1][2]... │  [0][1][2]... │        │
    └─────────────────────────────────────────────────────────┘
```

### 核心术语

| 术语 | 说明 |
|------|------|
| **Broker** | Kafka 服务节点 |
| **Topic** | 消息主题（逻辑概念） |
| **Partition** | 分区（物理概念，消息有序性保证） |
| **Producer** | 消息生产者 |
| **Consumer** | 消息消费者 |
| **Consumer Group** | 消费者组（同组内竞争消费，不同组独立） |
| **ISR** | In-Sync Replicas，与 Leader 保持同步的副本集合 |
| **AR** | Assigned Replicas，全部副本 |
| **OSR** | Out-of-Sync Replicas，落后太多的副本 |
| **Controller** | 集群控制器（选举/管理分区） |

---

## 二、高吞吐原理 🔥

### 2.1 顺序写

Kafka 使用**顺序追加写入**磁盘，而非随机写。机械硬盘顺序写可达 600MB/s，随机写仅 100KB/s。

### 2.2 零拷贝（Zero Copy）

传统网络传输路径：

```
磁盘 → 内核缓冲区(read) → 用户态缓冲区 → Socket 缓冲区(write) → 网卡
```

Kafka 使用 `sendfile()` + `transferTo()`：

```
磁盘 → 内核缓冲区 → Socket 缓冲区 → 网卡
```

数据不经过用户态，减少 2 次 CPU 拷贝和 2 次上下文切换。

### 2.3 Page Cache

- 利用 OS 的 Page Cache 缓存消息
- 写：数据先写入 Page Cache，由 OS 异步刷盘
- 读：优先从 Page Cache 读，命中率高则类似内存读取

### 2.4 批量压缩

消息批量打包压缩，减少网络和磁盘 I/O。

```properties
# Producer 配置
batch.size=16384           # 批量大小
linger.ms=5                # 等待时间（凑批）
compression.type=snappy    # 压缩算法
```

---

## 三、分区与消费

### 3.1 分区策略

```java
// Producer 分区规则
// 1. 指定 partition → 直接用
// 2. 指定 key → hash(key) % 分区数
// 3. 都没指定 → 轮询（粘性分区）
```

### 3.2 Consumer Group 再平衡

```
         Topic: order (3 partitions)
    
    Consumer Group: orders-group
    ┌──────┐  ┌──────┐  ┌──────┐
    │  C1  │  │  C2  │  │  C3  │
    │ (P0) │  │ (P1) │  │ (P2) │
    └──────┘  └──────┘  └──────┘
    
    如果 C3 宕机 → Rebalance → C1(P0,P1) C2(P2)
```

**追问：再平衡有什么问题？**

1. 触发 STW：再平衡期间所有消费者暂停
2. 重复消费：拉过的消息可能重新拉取
3. 集群抖动：频繁加入/离开导致连锁再平衡

---

## 四、消息可靠性 🔥

### 4.1 生产端 ACK 机制

```properties
acks=0          # 不等确认，性能最高，可能丢消息
acks=1          # Leader 写入即确认（默认），Leader 挂了可能丢
acks=all / -1   # 所有 ISR 确认，最可靠
```

### 4.2 ISR 机制

```
ISR = {与 Leader 同步时间差 < replica.lag.time.max.ms 的副本}
```

- 副本落后太多会被踢出 ISR → 变为 OSR
- 追赶上 Leader 后重新加入 ISR
- `min.insync.replicas`：最少同步副本数，配合 acks=all 保证可靠性

### 4.3 幂等性与事务

```java
// 幂等性：同一个 Producer 的生产操作，多次重试不会产生重复消息
props.put("enable.idempotence", true);
// 原理：Producer ID + Sequence Number，Broker 根据 Sequence 去重

// 事务：跨分区原子写入
props.put("transactional.id", "tx-id");
producer.initTransactions();
producer.beginTransaction();
producer.send(record1);
producer.send(record2);
producer.commitTransaction(); // 或 abortTransaction()
```

---

## 五、消费者

### 5.1 消费模式

```java
// 自动提交（可能重复消费）
props.put("enable.auto.commit", "true");

// 手动提交（推荐）
props.put("enable.auto.commit", "false");
consumer.commitSync();   // 同步提交
// 或
consumer.commitAsync();  // 异步提交 + 回调
```

### 5.2 重复消费与解决方案

```
重复消费原因：
1. 自动提交，offset 未提交时宕机
2. 手动提交 offset，但业务处理已完成
3. Rebalance 过程中

解决方案：
1. 业务幂等（数据库唯一约束 / Redis 去重）
2. 手动提交 offset（先处理业务，再提交）
```

### 5.3 消息丢失场景

| 场景 | 原因 | 解决 |
|------|------|------|
| Producer | acks=0/1，网络异常未重试 | acks=all + retries |
| Broker | 副本不同步，Leader 宕机 | min.insync.replicas >= 2 |
| Consumer | 先提交 offset 再处理业务 | 先处理业务再提交 offset |

---

## 六、Kafka 控制器 Controller

- 集群中某个 Broker 担任 Controller
- 负责分区 Leader 选举、ISR 管理
- 通过 ZooKeeper 竞争 `/controller` 临时节点
- **Kafka 2.8+ 支持 Kraft 模式**，去除 ZooKeeper 依赖

---

## 七、常用参数

```properties
# Producer
batch.size=16384             # 批量大小
linger.ms=5                  # 凑批等待时间
buffer.memory=33554432       # 缓冲内存
max.request.size=1048576     # 最大请求大小
retries=3                    # 重试次数

# Consumer
max.poll.records=500         # 单次拉取最大消息数
max.poll.interval.ms=300000  # 两次 poll 最大间隔
session.timeout.ms=10000     # 会话超时
heartbeat.interval.ms=3000   # 心跳间隔
```

---

## 面试追问集

**Q：Kafka 和 RabbitMQ 的区别？**

| 对比 | Kafka | RabbitMQ |
|------|-------|----------|
| 设计 | 分布式日志流 | 传统消息队列 |
| 吞吐量 | 极高（百万 QPS） | 较低（万级） |
| 消息可靠性 | ISR + 持久化 | 持久化 + ACK |
| 消费模式 | Pull | Push |
| 适用场景 | 大数据、日志、流处理 | 业务消息、复杂路由 |

**Q：Kafka 的消息有序性如何保证？**

- 单个 Partition 内消息有序
- 全局有序：1 个 Producer + 1 个 Partition + 1 个 Consumer

**Q：Kafka 为什么这么快？**

顺序写磁盘 + 零拷贝 + Page Cache + 批量压缩 + 分区并行。
