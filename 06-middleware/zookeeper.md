# Zookeeper

---

## 一、核心概念

| 概念 | 说明 |
|------|------|
| **ZNode** | 数据节点，类似文件系统中的文件/目录 |
| **Session** | 客户端会话，心跳维持 |
| **Watcher** | 事件监听机制，数据变化时通知客户端 |
| **ACL** | 访问控制列表 |
| **Leader** | 集群主节点，处理写请求 |
| **Follower** | 从节点，参与选举和投票 |
| **Observer** | 观察者，不参与投票，只提供读服务 |

---

## 二、ZAB 协议（ZooKeeper Atomic Broadcast）

### 2.1 消息广播（原子广播）

```
Leader 收到写请求：
1. Leader 生成提案（Proposal），发送给所有 Follower
2. Follower 收到提案，写入本地日志，回复 ACK
3. Leader 收到过半 ACK → 发送 Commit
4. Leader + 所有 Follower 提交该操作
```

类似 **两阶段提交（2PC）**，但不需要所有 Follower 都确认，过半即可。

### 2.2 崩溃恢复（Leader 选举）

```
1. 选举阶段：选出一个有最新数据的 Leader
2. 发现阶段：Leader 确定最新的 Zxid
3. 同步阶段：Leader 将差异数据同步给 Follower
4. 广播阶段：恢复正常服务
```

**追问：为什么 Zxid 是 64 位？**

```
Zxid 结构（64 位）：
┌───────────────────┬───────────────────┐
│  高 32 位: epoch   │  低 32 位: 计数器  │
│  （选举周期）        │  （本周期内自增）    │
└───────────────────┴───────────────────┘
```

epoch 越大表示数据越新，竞选时 epoch 大的优先成为 Leader。

---

## 三、ZNode 节点类型

| 类型 | 创建方式 | 说明 |
|------|---------|------|
| **持久节点（PERSISTENT）** | 默认 | 创建后一直存在 |
| **持久顺序节点** | PERSISTENT_SEQUENTIAL | 带自增序号 |
| **临时节点（EPHEMERAL）** | EPHEMERAL | Session 断开后自动删除 |
| **临时顺序节点** | EPHEMERAL_SEQUENTIAL | 临时 + 自增序号 |
| **容器节点** | CONTAINER | 子节点为空时删除 |
| **TTL 节点** | PERSISTENT_WITH_TTL | 到期自动删除 |

---

## 四、Watcher 机制

```java
ZooKeeper zk = new ZooKeeper("localhost:2181", 3000, null);

// 注册 Watcher
zk.exists("/path", event -> {
    System.out.println("节点变化: " + event.getType());
});

// Watcher 特点：
// 1. 一次性触发（触发后需重新注册）
// 2. 轻量级通知（只告知事件类型，不含变化内容）
// 3. 先通知 Watcher，再通知客户端
// 4. Watcher 异步通知
```

---

## 五、Leader 选举

```
服务器启动时的选举过程：
1. 每个服务器都投自己一票，发送给其他服务器
2. 收到投票后比较：
   a. 先比较 Zxid（大的优先）
   b. Zxid 相同比较 myid（大的优先）
3. 更新自己的投票为更好的服务器 → 再次广播
4. 统计票数，如果某台服务器获得 > 半数票 → 成为 Leader
```

---

## 六、典型应用场景

### 6.1 分布式锁

```java
// 方案一：临时节点（非公平锁）
// 多客户端创建同一个名称的临时节点，创建成功的获得锁
// 问题：羊群效应（所有客户端都被唤醒）

// 方案二：临时顺序节点（公平锁，推荐）
// 1. 在 /lock 下创建临时顺序节点
// 2. 获取 /lock 下所有子节点，排序
// 3. 如果自己是最小序号 → 获得锁
// 4. 否则 → watch 前一个节点 → 等待

// Curator 框架封装了这些逻辑
InterProcessMutex lock = new InterProcessMutex(client, "/lock");
lock.acquire();
// ... 业务逻辑
lock.release();
```

### 6.2 注册中心

```
/registry/service-name/
    ├── 192.168.1.10:8080 (临时节点)
    ├── 192.168.1.11:8080 (临时节点)
    └── 192.168.1.12:8080 (临时节点)

服务启动 → 创建临时节点
服务下线 → 临时节点自动删除 → Watch 通知
```

### 6.3 配置管理

```
/config/app/
    ├── timeout = "5000"
    ├── maxPoolSize = "100"
    └── dbUrl = "jdbc:mysql://..."

应用启动时读取配置 + Watch 配置变化 → 动态刷新
```

### 6.4 选主

```
/master_lock
创建临时节点的客户端成为 Master
Master 宕机 → 临时节点删除 → 其他客户端竞争
```

### 6.5 分布式 Barrier

```
/barrier/node_001
/barrier/node_002
...
所有节点创建完毕 → 触发后续操作
```

---

## 面试追问集

**Q：Zookeeper 为什么是 CP 系统中的代表？**

ZAB 协议要求多数节点写入成功才返回，在 Leader 选举期间集群不可用（无 A）。数据强一致性（C） + 分区容忍性（P）优先。

**Q：Zookeeper 集群数量为什么建议奇数？**

- 容忍故障数 = (N-1)/2
- 3 台：容忍 1 台故障；4 台：也容忍 1 台故障
- 4 台比 3 台多一个故障点，但不增加容错能力
- 所以 3 台性价比最高

**Q：Zookeeper 和 Nacos 的区别？**

| 对比 | ZooKeeper | Nacos |
|------|-----------|-------|
| CAP | CP | AP/CP 可切换 |
| 配置中心 | 需自建 | 内置 |
| 健康检查 | 基于 Session 心跳 | 多种（TCP/HTTP/MySQL） |
| 一致性协议 | ZAB | Raft（CP）/ Distro（AP） |
| 推荐度 | Dubbo 老项目 | 新项目首选 |
