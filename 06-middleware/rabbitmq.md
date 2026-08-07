# RabbitMQ

---

## 一、核心概念

```
        ┌───────────────────────────────────────────┐
        │               RabbitMQ Broker              │
        │                                            │
        │  Producer ──► Exchange ──(routing)──► Queue ──► Consumer
        │                 │          key       │        ▲
        │                 │                     │        │
        │                 └─► Queue ────────────┘        │
        │                       ▲                        │
        │                       └────── Consumer ────────┘
        └───────────────────────────────────────────┘
```

| 概念 | 说明 |
|------|------|
| **Broker** | RabbitMQ 服务节点 |
| **Exchange** | 交换机，接收消息并按 routing key 路由到队列 |
| **Queue** | 消息队列，存储消息 |
| **Binding** | Exchange 和 Queue 的绑定关系 |
| **Routing Key** | 路由键，Exchange 根据它决定转发到哪个 Queue |
| **Channel** | 信道（轻量级连接，复用 TCP 连接） |

---

## 二、交换机类型

| 类型 | 路由规则 |
|------|---------|
| **Direct** | Routing Key 精确匹配 |
| **Fanout** | 广播到所有绑定的 Queue |
| **Topic** | Routing Key 模式匹配（`*` 匹配一个词，`#` 匹配零个或多个） |
| **Headers** | 通过消息 Headers 匹配（很少用） |

```java
// 示例：Topic 交换机的 binding key
// Queue A: "order.#"   → 匹配 order.create, order.update, order.pay
// Queue B: "order.*"   → 只匹配 order.create, order.update（不匹配 order.pay.success）
```

---

## 三、消息可靠性 🔥

### 3.1 可靠性保障全景

```
1. 生产者确认（Publisher Confirm）：消息到达 Broker 的确认
2. 消息持久化（Durable）：Queue、Exchange、Message 都设置为持久化
3. 消费者确认（Consumer ACK）：消费完成后手动 ACK
4. 高可用（镜像队列/仲裁队列）：Broker 级别的冗余
```

### 3.2 生产者确认

```java
// 开启生产者确认
channel.confirmSelect();
channel.basicPublish(exchange, routingKey, properties, body);

// 异步确认
channel.addConfirmListener(new ConfirmListener() {
    @Override
    public void handleAck(long deliveryTag, boolean multiple) {
        // 消息成功送达
    }
    @Override
    public void handleNack(long deliveryTag, boolean multiple) {
        // 消息发送失败，重试/告警
    }
});
```

### 3.3 消费者确认

```java
// 手动 ACK（推荐）
boolean autoAck = false;
channel.basicConsume(queueName, autoAck, (consumerTag, delivery) -> {
    try {
        // 处理消息
        processMessage(delivery);
        // 处理成功 → 确认
        channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
    } catch (Exception e) {
        // 处理失败 → 重回队列 或 拒绝+死信
        // channel.basicNack(deliveryTag, false, true);   // 重回队列
        channel.basicNack(deliveryTag, false, false);     // 拒绝
    }
}, consumerTag -> {});
```

---

## 四、死信队列

```
消息变成死信的情况：
1. 手动 reject / nack（requeue=false）
2. 消息 TTL 过期
3. 队列达到最大长度

              普通队列                死信交换机
    消息 ──► [普通队列] ──(超时/被拒)──► [DLX Exchange] ──► [死信队列] ──► 告警/重试处理
```

### 延迟消息实现

```java
// RabbitMQ 没有内置延迟功能 → 通过死信队列 + TTL 实现
// 1. 设置队列消息 TTL，或消息级别 TTL
// 2. 消息过期后投递到死信队列
// 3. 死信队列消费者处理延迟消息

// 也可用 RabbitMQ 延迟插件（rabbitmq-delayed-message-exchange）
```

---

## 五、高可用

### 5.1 镜像队列（MQ 3.9 前）

```
        ┌────────┐
        │ Master │  ← 读写都从 Master
        └───┬────┘
    ┌───────┼───────┐
    ▼       ▼       ▼
┌────────┐┌────────┐┌────────┐
│ Slave  ││ Slave  ││ Slave  │  ← 只做备份
└────────┘└────────┘└────────┘
```

slave 不提供服务，Master 挂了需重新选举。

### 5.2 仲裁队列（Quorum Queue，MQ 3.8+，推荐）

基于 **Raft 协议**，比镜像队列更可靠：
- 写入需多数节点确认
- 自动故障转移
- 性能优于镜像队列

---

## 六、常见问题

### 6.1 消息堆积

```
原因：生产速度 > 消费速度
排查：看监控（队列长度、消费速率）
解决：
  1. 增加消费者数量（水平扩容）
  2. 优化消费逻辑（批量处理、异步）
  3. 消息转储到磁盘/其他队列
  4. 设置队列最大长度 + 死信
```

### 6.2 重复消费

```
解决方案：
  1. 业务幂等（唯一键/MVCC 版本号）
  2. 消息去重表（消息 ID 插入 + 业务操作，事务保证）
```

### 6.3 消息顺序性

```
单队列 + 单消费者 = 消息有序
需要全局有序 → 消息按顺序号发送，避免并发
```

---

## 面试追问集

**Q：RabbitMQ 的消费模式（Push vs Pull）？**

RabbitMQ 使用 **Push 模式**（Broker 主动推给 Consumer），通过 `basic.qos`（prefetch count）控制推送速率，实现消费者负载保护。

Kafka 使用 Pull 模式（Consumer 主动拉）。

**Q：如何保证消息 100% 不丢失？**

1. 生产者：Publisher Confirm + 重试机制（需要去重/幂等）
2. Broker：持久化 + 镜像/仲裁队列（多副本）
3. 消费者：手动 ACK + 幂等处理

**Q：RabbitMQ 的连接（Connection）和信道（Channel）是什么关系？**

一个 TCP Connection 上可以有多个 Channel，Channel 是轻量级的虚拟连接，减少 TCP 建立开销。类似一条物理网线上跑多个逻辑会话。
