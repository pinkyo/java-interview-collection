# 分布式事务

---

## 一、分布式事务的挑战

单体应用中本地事务（ACID）很简单，但微服务架构下，一个业务操作可能横跨多个服务和数据库：

```
创建订单 → 订单服务（DB1）
扣减库存 → 库存服务（DB2）
扣减余额 → 账户服务（DB3）

这三个操作需要原子性保证！
```

---

## 二、两阶段提交（2PC）

```
            ┌─────────────┐
            │ Coordinator │
            └──────┬──────┘
                   │
      ┌────────────┼────────────┐
      ▼            ▼            ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│Participant│ │Participant│ │Participant│
│     1    │ │     2    │ │     3    │
└──────────┘ └──────────┘ └──────────┘

Phase 1（准备阶段）：
  Coordinator → 发送 Prepare → 所有 Participant
  Participant → 执行操作，写 Undo/Redo 日志 → Prepare OK / Fail

Phase 2（提交阶段）：
  全部 OK → Coordinator 发送 Commit → Participant 提交
  任何 Fail → Coordinator 发送 Rollback → Participant 回滚
```

**缺点：**
- **同步阻塞**：等待所有参与者响应，最慢的拖累整个系统
- **单点故障**：Coordinator 宕机，参与者一直阻塞
- **数据不一致**：第二阶段部分节点收到 Commit 后宕机

---

## 三、TCC（Try-Confirm-Cancel）🔥

TCC 是补偿型事务，每个服务提供三个接口：

```java
// 转账示例
public interface TransferService {
    // Try：预留资源
    boolean tryFreeze(long userId, BigDecimal amount);  // 冻结资金

    // Confirm：确认提交（Try 全部成功后才执行）
    boolean confirmDeduct(long userId, BigDecimal amount); // 扣减冻结资金

    // Cancel：取消回滚（Try 部分失败时执行）
    boolean cancelUnfreeze(long userId, BigDecimal amount); // 解冻资金
}
```

**执行流程：**

```
Try 阶段：
  A 服务 tryFreeze(A, 100) → OK
  B 服务 tryFreeze(B, -100) → Fail

Cancel 阶段：（B 失败，所有 Cancel）
  A 服务 cancelUnfreeze(A, 100)  ← 回滚
  B 服务 cancelUnfreeze(B, -100) ← 回滚

或全部 Try OK → Confirm 阶段：
  A 服务 confirmDeduct(A, 100)
  B 服务 confirmDeduct(B, -100)
```

**TCC vs 2PC：**

| 对比 | 2PC | TCC |
|------|-----|-----|
| 层次 | 资源层（数据库） | 业务层 |
| 阻塞 | 同步阻塞 | 非阻塞 |
| 性能 | 低 | 较高 |
| 实现复杂度 | 低（中间件实现） | 高（需编写补偿逻辑） |
| 适用场景 | 数据库层面 | 业务层面、跨系统 |

---

## 四、可靠消息最终一致性

```
本地事务 + MQ 事务消息

流程：
1. 发送 Half 消息（Prepare）
2. 执行本地事务
3. 本地事务成功 → Commit 消息 → 消费者可见
4. 本地事务失败 → Rollback 消息 → 消费者不可见
5. 生产者宕机未提交 → MQ 定时回查 → 确认状态
```

**RocketMQ 事务消息示例：**

```java
TransactionMQProducer producer = new TransactionMQProducer("group");
producer.setTransactionListener(new TransactionListener() {
    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        try {
            // 执行本地事务
            orderService.createOrder(order);
            return LocalTransactionState.COMMIT_MESSAGE;
        } catch (Exception e) {
            return LocalTransactionState.ROLLBACK_MESSAGE;
        }
    }

    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        // 回查本地事务状态
        if (orderService.isOrderCreated(msg.getKeys())) {
            return LocalTransactionState.COMMIT_MESSAGE;
        }
        return LocalTransactionState.ROLLBACK_MESSAGE;
    }
});
```

---

## 五、Seata（阿里分布式事务框架）

### 5.1 Seata 架构

```
              ┌──────────┐
              │  TC (Seata│ 事务协调器
              │  Server)  │
              └────┬──────┘
                   │
     ┌─────────────┼─────────────┐
     ▼             ▼             ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│   TM    │  │   TM    │  │   TM    │
│(启动事务)│  │         │  │         │
├─────────┤  ├─────────┤  ├─────────┤
│   RM    │  │   RM    │  │   RM    │
│(资源管理)│  │(资源管理)│  │(资源管理)│
└─────────┘  └─────────┘  └─────────┘
   服务A       服务B        服务C
```

### 5.2 AT 模式（最常用，无侵入）

```java
@GlobalTransactional
public void createOrder() {
    orderService.create();          // 本地事务（RM1）
    storageService.deduct();        // RPC 调用（RM2）
    accountService.debit();         // RPC 调用（RM3）
}
```

两阶段：
- **一阶段**：执行 SQL → 记录 Undo Log（用于回滚）→ 注册 Branch 到 TC
- **二阶段**：
  - 全部成功 → 异步删除 Undo Log
  - 有失败 → 根据 Undo Log 回滚（生成反向 SQL）

### 5.3 Seata 模式对比

| 模式 | 隔离性 | 侵入性 | 性能 | 适用场景 |
|------|--------|--------|------|---------|
| AT | 弱 | 无 | 中 | 通用，对一致性要求不是极高 |
| TCC | 强 | 高（需实现三接口） | 高 | 对一致性要求高 |
| Saga | 弱 | 中 | 高 | 长事务、无法改造的老系统 |
| XA | 强 | 无 | 低 | 需要强一致性的短事务 |

---

## 六、最大努力通知

```
1. 业务方调用通知接口失败 → 异步重试 N 次 → 仍失败 → 人工介入
2. 适用于对一致性要求不高的场景（短信通知、数据同步）
```

---

## 面试追问集

**Q：你们的项目用的是什么分布式事务方案？**

根据实际回答。常见组合：
- 不需要强一致：**RocketMQ 事务消息 + 最终一致性**
- 跨服务写操作：**Seata AT 模式**（简单场景）/ **TCC**（一致性要求高）
- 简单通知类：**本地消息表 + 定时补偿**

**Q：TCC 的空回滚、悬挂和幂等问题？**

- **空回滚**：Try 未执行，Cancel 被执行 → 判断 Try 是否执行过
- **悬挂**：Cancel 在 Try 之前到达 → 拒绝操作
- **幂等**：Confirm/Cancel 支持重试 → 事务状态记录表

**Q：Seata AT 模式的写隔离问题？**

全局锁：在分支事务提交前，TC 会尝试获取相关行的全局锁，防止其他分布式事务修改同一行数据。
