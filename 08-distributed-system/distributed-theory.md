# 分布式理论

---

## 一、CAP 定理 🔥

### 1.1 三者不可兼得

```
        C（Consistency）
        强一致性
           /\
          /  \
         /    \
        /  CA  \
       /________\
      /          \
     /    AP      \
    /  高可用+分区\  CP
   P_____________\______
   Partition        C
   Tolerance
   分区容错性
```

- **C（Consistency）**：所有节点在同一时间看到相同的数据
- **A（Availability）**：每个请求都能得到响应（不管成功或失败）
- **P（Partition Tolerance）**：系统在遇到网络分区时仍能正常工作

### 1.2 为什么 P 必须选？

分布式系统中网络分区不可避免（网络延迟、交换机故障等），所以 P 必须选。剩下的就是在 C 和 A 之间权衡。

### 1.3 典型系统分类

| 类型 | 代表 | 特性 |
|------|------|------|
| **CP** | ZooKeeper | 一致性优先，分区时牺牲可用性 |
| **AP** | Eureka、Nacos(AP模式) | 可用性优先，允许短暂不一致 |
| **CA** | 传统关系型数据库（单机） | 无分区，强一致+高可用 |
| **BASE** | 大多数互联网系统 | 最终一致性 |

---

## 二、BASE 理论 🔥

CAP 的工程实践折中方案：

| 字母 | 全称 | 含义 |
|------|------|------|
| BA | Basically Available | 基本可用（系统出现故障时允许损失部分可用性） |
| S | Soft State | 软状态（允许系统存在中间状态） |
| E | Eventually Consistent | 最终一致性（所有副本最终达到一致，但不保证实时） |

**和 ACID 的对比：**

| 维度 | ACID | BASE |
|------|------|------|
| 目标 | 强一致性 | 最终一致性 |
| 场景 | 传统数据库 | 大规模分布式系统 |
| 一致性模型 | 强一致（Strict） | 弱一致、最终一致 |
| 可用性 | 低（锁等待） | 高 |

---

## 三、一致性模型

### 3.1 一致性光谱

```
强一致性 ──────────────────────────── 弱一致性
    │                                      │
线性一致性 顺序一致性 因果一致性 最终一致性  读己之写
```

| 模型 | 说明 | 难度 |
|------|------|------|
| **线性一致性** | 全局唯一、实时顺序，最强（类似单机） | 最高 |
| **顺序一致性** | 全局顺序一致但不保证实时 | 高 |
| **因果一致性** | 有因果关系的操作按顺序 | 中 |
| **最终一致性** | 无更新时最终一致 | 低 |
| **读己之写** | 自己写入的自己能读到 | 中低 |

---

## 四、共识算法

### 4.1 Paxos

经典但难以理解，分为 Basic Paxos 和 Multi-Paxos。

```
角色：Proposer（提案者）、Acceptor（接受者）、Learner（学习者）

Basic Paxos 两阶段：
Phase 1（Prepare）：
  Proposer → 发送 Prepare(n) → Acceptor
  Acceptor：如果 n > 已承诺的最大编号，承诺不再接受 < n 的提案

Phase 2（Accept）：
  Proposer → 发送 Accept(n, value) → Acceptor（多数派）
  Acceptor：如果 n >= 已承诺的最大编号，接受该 value
```

### 4.2 Raft（易理解，工程最常用）

将共识拆分为三个子问题：

```
1. Leader 选举（Leader Election）
2. 日志复制（Log Replication）
3. 安全性（Safety）
```

#### Leader 选举

```
                         每次选举 Term +1
          Follower ──超时──► Candidate ──得多数票──► Leader
                                   │
                                   └──票不够──► 重新选举(Term+1)

每个节点在一个 Term 中只能投一次票
Term 大的优先成为 Leader
```

#### 日志复制

```
1. Leader 收到写请求 → 写入本地日志
2. 发送 AppendEntries RPC 给 Followers
3. 多数 Follower 确认 → Leader 提交 → 通知 Followers 提交
4. Leader 返回客户端成功
```

**追问：Raft 如何保证已提交的日志不被覆盖？**

1. 只有拥有最新日志的节点才能当选 Leader
2. 选举时 Candidate 的日志不够新会被拒绝投票
3. 日志比对：先比 Term，再比 Index

### 4.3 ZAB（ZooKeeper 使用）

见 [ZooKeeper 文档](../06-middleware/zookeeper.md)。

### 4.4 共识算法对比

| 算法 | 理解难度 | 工程实现 | 应用 |
|------|---------|---------|------|
| Paxos | ★★★★★ | 困难 | Chubby |
| Multi-Paxos | ★★★★ | 较难 | 少数系统 |
| Raft | ★★ | 容易 | etcd、TiKV、Consul |
| ZAB | ★★★ | 中 | ZooKeeper |

---

## 面试追问集

**Q：举一个 AP 和 CP 的实际例子？**

- **AP（Eureka）**：所有节点提供服务，即使数据不一致也返回结果。一个服务挂了不影响其他服务发现。
- **CP（ZooKeeper）**：要求一致的数据，如果 Leader 挂了集群不可用，宁可不可用也不能返回脏数据。

**Q：微服务中如何做到最终一致性？**

1. **可靠消息最终一致性**：本地事务 + MQ（RocketMQ 事务消息）
2. **TCC**：Try-Confirm-Cancel，两阶段补偿
3. **最大努力通知**：反复重试 + 人工兜底
4. **事件溯源**：只追加事件，不修改状态

**Q：分布式系统的时间问题怎么处理？**

1. NTP 同步（毫秒级）
2. 逻辑时钟（Lamport Timestamp / Vector Clock）
3. TrueTime（Google Spanner 使用原子钟+GPS，物理时间）
4. 不要依赖精确时间排序，使用版本号/序列号
