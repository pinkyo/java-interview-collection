# DDD（Domain-Driven Design，领域驱动设计）

> DDD 是一种以业务为核心的软件设计方法论，解决复杂业务的建模与落地问题。本文梳理 DDD 的核心概念、业务建模方法、充血/失血模型对比。

---

## 一、分层架构

```
表现层（Interface）   → Controller / DTO，负责协议与转换
   │
应用层（Application） → 编排领域对象、事务、安全，不含业务规则
   │
领域层（Domain）       ← 核心：实体/值对象/聚合/领域服务/领域事件
   │
基础设施层（Infra）    → DB/Cache/MQ/外部接口实现
```

### 各层职责

| 层 | 职责 | 关键约束 |
|----|------|---------|
| 表现层 | 协议转换、参数校验、DTO 装配 | 不含业务规则 |
| 应用层 | 用例编排、事务、安全、发事件 | 不含业务规则，只协调 |
| 领域层 | 实体、值对象、聚合、领域服务、领域事件 | 业务规则的唯一归属 |
| 基础设施层 | 仓储实现、外部接口、MQ、DB | 依赖倒置，实现领域层接口 |

> 依赖方向：外层依赖内层，领域层不依赖任何其他层（依赖倒置）。

---

## 二、核心概念

| DDD 概念 | 说明 | 示例 |
|---------|------|------|
| **实体（Entity）** | 有唯一标识，状态可变 | `Order`（订单 ID 标识唯一） |
| **值对象（Value Object）** | 无唯一标识，属性决定相等性，不可变 | `Address`（省市区街道相同即相等） |
| **聚合（Aggregate）** | 一组相关对象的一致性边界 | `Order` + `OrderItem`（订单聚合） |
| **聚合根（Aggregate Root）** | 聚合的入口，外部只能通过它访问聚合内对象 | `Order` 是根，`OrderItem` 不能单独被外部持有 |
| **领域服务（Domain Service）** | 跨聚合或无归属的业务逻辑 | `TransferService`（转账涉及两个账户聚合） |
| **领域事件（Domain Event）** | 领域内发生的事情，解耦与异步 | `OrderPaidEvent` |
| **仓储（Repository）** | 聚合的持久化抽象，面向接口 | `OrderRepository`（接口在领域层，实现在基础设施层） |
| **限界上下文（Bounded Context）** | 一个模型的适用边界 | 订单上下文中的 `Product` ≠ 商品上下文中的 `Product` |
| **工厂（Factory）** | 复杂对象的创建逻辑封装 | `OrderFactory.create(...)` |

### 聚合的设计原则 🔥

1. **尽量小**：聚合越小，并发冲突越少，事务越轻。
2. **按一致性边界划分**：必须一起变更的才放同一聚合，否则拆开。
3. **聚合间通过 ID 引用**：`OrderItem` 里存 `productId`，而不是直接持有 `Product` 对象（避免大事务）。
4. **一次事务只修改一个聚合**：跨聚合的一致性用领域事件 + 最终一致。

---

## 三、业务建模（DDD 的起点）

DDD 的核心是**先理解业务，再写代码**。业务建模即把现实业务抽象为领域模型。

### 3.1 常用方法

1. **事件风暴（Event Storming）**：团队一起贴便签，找领域事件 → 命令 → 实体 → 聚合 → 限界上下文。
2. **统一语言（Ubiquitous Language）**：业务术语 = 代码术语，避免"业务叫订单，代码叫 OrderInfo"的翻译损耗。
3. **识别聚合**：找一致性边界——"哪些数据必须一起变更"就划到同一聚合。
4. **划分限界上下文**：一个术语在不同上下文有不同含义就拆分（如"商品"在商品域是 SKU 详情，在订单域只是快照）。

### 3.2 建模流程

```
业务场景  →  事件风暴  →  识别领域事件/命令
                              │
                              ▼
                      识别实体/聚合/限界上下文
                              │
                              ▼
                      定义统一语言 + 领域模型
                              │
                              ▼
                      代码落地（充血模型 + 仓储）
```

### 3.3 建模示例：电商下单

| 业务行为 | 领域事件 | 聚合 | 关键不变量（Invariants） |
|---------|---------|------|----------------------|
| 用户提交订单 | `OrderPlacedEvent` | `Order` 聚合 | 库存 ≥ 下单数量、金额正确 |
| 支付完成 | `OrderPaidEvent` | `Order` 聚合 | 只有待支付才能变已支付 |
| 取消订单 | `OrderCancelledEvent` | `Order` 聚合 | 已发货不能取消 |

聚合根 `Order` 负责**维护这些不变量**，而非让外部随意 set 字段。

### 3.4 限界上下文示例

```
商品上下文            订单上下文
┌────────────┐       ┌────────────────┐
│ Product    │       │ OrderItem      │
│  - sku     │  ───► │  - productId   │  ← 只存 ID，是商品快照
│  - detail  │       │  - snapshot    │
│  - stock   │       │  - qty         │
└────────────┘       └────────────────┘

同一个"商品"，在两个上下文里是不同的模型：
  - 商品域：完整的 SKU + 详情 + 库存
  - 订单域：下单时的快照（防商品后续改名导致订单错乱）
```

---

## 四、充血模型 vs 失血模型 🔥

这是面试高频考点，核心区别是**业务逻辑放在哪里**。

### 4.1 失血模型（Anemic Domain Model）

对象只有 getter/setter，业务逻辑全在 Service。

```java
// ❌ 失血模型：Order 只是数据袋，没有行为
public class Order {
    private Long id;
    private BigDecimal amount;
    private String status;  // UNPAID / PAID / CANCELLED
    // 纯 getter/setter，无业务逻辑
    public void setStatus(String status) { this.status = status; }
    public String getStatus() { return status; }
    // ...
}

// Service 承载所有业务规则 → 越来越胖，难维护
public class OrderService {
    public void pay(Order order) {
        if (!"UNPAID".equals(order.getStatus())) {
            throw new BizException("订单状态不正确");
        }
        order.setStatus("PAID");
        order.setAmount(order.getAmount());
        // 校验、扣库存、发事件... 全堆在这里
        orderRepository.save(order);
    }

    public void cancel(Order order) {
        if ("PAID".equals(order.getStatus())) {
            throw new BizException("已支付不能取消");
        }
        order.setStatus("CANCELLED");
        orderRepository.save(order);
    }
}
```

**问题**：
- 业务规则散落在 Service，`Order` 任何字段都能被随意 set，**不变量无人守护**。
- Service 膨胀成"上帝类"，逻辑复用难。
- 对象退化为数据结构，违背面向对象。

---

### 4.2 充血模型（Rich Domain Model）

对象既有数据又有行为，业务规则内聚在实体/聚合根中。

```java
// ✅ 充血模型：Order 自带业务规则，用枚举和方法保护不变量
public class Order {
    private Long id;
    private BigDecimal amount;
    private OrderStatus status;   // 枚举而非裸 String
    private List<OrderItem> items; // 聚合内的值对象/实体

    // 私有构造，只能通过工厂创建，保证初始状态合法
    private Order() {}
    public static Order place(List<OrderItem> items) {
        Order o = new Order();
        o.status = OrderStatus.UNPAID;
        o.items = items;
        o.amount = items.stream()
                        .map(i -> i.getPrice().multiply(BigDecimal.valueOf(i.getQty())))
                        .reduce(BigDecimal.ZERO, BigDecimal::add);
        return o;
    }

    // 行为：支付 —— 校验 + 状态变更 + 发事件，都在聚合根内
    public void pay() {
        if (this.status != OrderStatus.UNPAID) {
            throw new BizException("只有待支付订单才能支付");
        }
        this.status = OrderStatus.PAID;
        // 可触发领域事件：DomainEvents.publish(new OrderPaidEvent(this.id));
    }

    // 行为：取消 —— 规则内聚
    public void cancel() {
        if (this.status == OrderStatus.SHIPPED) {
            throw new BizException("已发货订单不能取消");
        }
        this.status = OrderStatus.CANCELLED;
    }

    // 状态变更只能通过行为，不暴露 setter → 不变量不会被破坏
    // 无 setStatus()、setAmount()
}

public enum OrderStatus {
    UNPAID, PAID, SHIPPED, CANCELLED
}

// Service 变薄：只做编排 + 持久化 + 事务，不写业务规则
public class OrderApplicationService {
    public void pay(Long orderId) {
        Order order = orderRepository.findById(orderId)
                       .orElseThrow(() -> new BizException("订单不存在"));
        order.pay();                    // ← 业务规则在 Order 内
        orderRepository.save(order);    // ← 仓储持久化
        domainEventPublisher.publish(new OrderPaidEvent(orderId));
    }
}
```

**优点**：
- 不变量（只有待支付才能付、已发货不能取消）由聚合根守护，**外部无法绕过**。
- 业务逻辑内聚，复用、测试容易；Service 变薄。
- 符合面向对象："对象 = 数据 + 行为"。

---

### 4.3 对比总结

| 维度 | 失血模型 | 充血模型 |
|------|---------|---------|
| 业务逻辑位置 | Service 层 | 聚合根/实体内 |
| 对象职责 | 纯数据（getter/setter） | 数据 + 行为 |
| 不变量守护 | 依赖 Service 自觉 | 强制，外部无法绕过 |
| Service 复杂度 | 高（上帝类） | 低（仅编排） |
| 可测试性 | 业务规则难单测 | 可直接对领域对象单测 |
| 适用 | 简单 CRUD（Create Read Update Delete，增删改查） | 复杂业务（DDD 场景） |
| 与 DDD 关系 | 反模式，DDD 不推荐 | DDD 推荐的领域模型实现方式 |

**面试话术**：
> 失血模型是"面向过程"的面向对象——对象只剩数据，业务逻辑堆在 Service，导致 Service 膨胀、不变量无人守护。充血模型把规则收回聚合根，通过方法而非 setter 暴露变更，使对象"自己保护自己"。DDD 落地时通常采用充血模型。

---

## 五、仓储（Repository）模式

仓储是聚合持久化的抽象，接口定义在领域层，实现在基础设施层，体现**依赖倒置**。

```java
// 领域层：接口（不依赖任何 DB 框架）
public interface OrderRepository {
    Order findById(Long id);
    void save(Order order);
}

// 基础设施层：JPA 实现
@Repository
public class JpaOrderRepository implements OrderRepository {
    @Autowired private OrderJpaRepository jpa;

    @Override
    public Order findById(Long id) {
        OrderPO po = jpa.findById(id).orElseThrow(...);
        return toDomain(po);   // PO → 领域对象
    }

    @Override
    public void save(Order order) {
        OrderPO po = toPO(order);  // 领域对象 → PO
        jpa.save(po);
    }
}
```

**要点**：
- 领域层只依赖接口，不感知 JPA/MyBatis，可替换实现。
- PO（Persistent Object）与领域对象分离，避免 DB 模型污染领域模型。

---

## 六、领域事件

领域事件用于跨聚合解耦，实现最终一致性。

```java
// 定义事件
public class OrderPaidEvent {
    private Long orderId;
    private LocalDateTime paidAt;
    // ...
}

// 发布（可在聚合根内或应用层）
domainEventPublisher.publish(new OrderPaidEvent(orderId));

// 订阅（其他聚合/上下文）
@EventListener
public void onOrderPaid(OrderPaidEvent event) {
    // 扣库存、发通知、记流水...
}
```

**为什么用事件**：
- 解耦：支付聚合不直接依赖库存聚合。
- 异步：可投递 MQ 实现最终一致。
- 可扩展：新需求加订阅者即可，不改原聚合。

---

## 七、DDD 与传统三层架构对比

| 对比 | 三层架构 | DDD |
|------|---------|-----|
| 核心 | 数据驱动（以表设计为起点） | 领域驱动（以业务建模为起点） |
| 业务逻辑 | Service 层代码 | 领域模型内（充血） |
| 复杂业务 | Service 层越来越胖 | 领域模型承担 |
| 模型风格 | 失血（POJO + Service） | 充血（对象 + 行为） |
| 持久化 | Service 直接调 DAO | 仓储接口 + 依赖倒置 |
| 跨聚合协作 | 直接方法调用 | 领域事件解耦 |
| 适用 | CRUD 场景 | 复杂业务场景 |

---

## 面试追问集

**Q：DDD 适合什么场景？**

业务复杂、规则多、长期演进的项目。简单 CRUD 用 DDD 反而增加不必要的复杂度。

**Q：聚合和聚合根是什么？**

聚合是一组相关对象的一致性边界，聚合根是它的唯一入口。外部只能持有聚合根引用，不能直接操作聚合内其他对象，从而保证不变量。

**Q：充血模型一定比失血好吗？**

不一定。简单 CRUD 场景用失血模型（POJO + Service）更直接；充血模型在复杂业务下才能体现价值，否则是过度设计。

**Q：领域服务（Domain Service）和应用服务（Application Service）区别？**

- 领域服务：承载**业务规则**，跨聚合的领域逻辑（如转账）。
- 应用服务：**编排用例**，事务/安全/调用领域对象，不含业务规则。

**Q：领域事件为什么要用？**

解耦跨聚合逻辑、支持最终一致、便于扩展新需求（加订阅者不改原代码）。
