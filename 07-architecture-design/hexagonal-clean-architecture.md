# 六边形架构与整洁架构详解

---

## 一、六边形架构（Hexagonal Architecture）

六边形架构，又称**端口与适配器架构（Ports and Adapters）**，由 Alistair Cockburn 于 2005 年提出。其核心目标是将**业务逻辑（领域核心）与外部系统（UI、数据库、第三方服务等）彻底解耦**，使得业务逻辑可以独立于任何外部技术进行测试和演进。

### 1.1 架构图示

```
              ┌─────────────────────────────────────────┐
              │           外部系统（左侧）                 │
              │  ┌──────────┐    ┌──────────┐           │
              │  │  Web UI  │    │ REST API │           │
              │  └────┬─────┘    └────┬─────┘           │
              └───────┼───────────────┼─────────────────┘
                      │               │
              ┌───────▼───────────────▼─────────────────┐
              │           输入适配器（Primary/Driving）   │
              │  ┌──────────────┐  ┌────────────────┐   │
              │  │ Controller   │  │ REST Controller│   │
              │  └──────┬───────┘  └────────┬───────┘   │
              └─────────┼───────────────────┼───────────┘
                        │                   │
              ┌─────────▼───────────────────▼───────────┐
              │                                          │
              │  ┌────────────────────────────────────┐  │
              │  │        端口（Port - 接口）           │  │
              │  │  ┌──────────────┐  ┌────────────┐  │  │
              │  │  │ Input Port   │  │ Output Port│  │  │
              │  │  └──────────────┘  └────────────┘  │  │
              │  └────────────────────────────────────┘  │
              │                                          │
              │  ┌────────────────────────────────────┐  │
              │  │       领域核心（Domain Core）        │  │
              │  │  实体 / 值对象 / 领域服务 / 用例     │  │
              │  └────────────────────────────────────┘  │
              │                                          │
              └─────────────┬───────────────┬────────────┘
                            │               │
              ┌─────────────▼───────────────▼────────────┐
              │         输出适配器（Secondary/Driven）     │
              │  ┌──────────────┐  ┌────────────────┐   │
              │  │ Repository   │  │  MQ Publisher  │   │
              │  │ Impl (MySQL) │  │ Impl (Kafka)   │   │
              │  └──────┬───────┘  └────────┬───────┘   │
              └─────────┼───────────────────┼───────────┘
                        │                   │
              ┌─────────▼───────────────────▼───────────┐
              │          外部系统（右侧）                  │
              │  ┌──────────┐    ┌──────────┐           │
              │  │  MySQL   │    │  Kafka   │           │
              │  └──────────┘    └──────────┘           │
              └─────────────────────────────────────────┘
```

### 1.2 核心概念

| 概念 | 说明 | 本质 |
|------|------|------|
| **端口（Port）** | 定义与外部交互的**接口契约**，属于领域层 | Java Interface |
| **适配器（Adapter）** | 端口的具体实现，负责技术细节转换 | Interface 的 Impl 类 |
| **左侧/主适配器（Driving Adapter）** | 驱动应用的输入方，调用输入端口 | Controller、CLI、测试用例 |
| **右侧/从适配器（Driven Adapter）** | 被应用调用的输出方，实现输出端口 | Repository、MQ、HTTP Client |
| **领域核心（Domain Core）** | 纯业务逻辑，不依赖任何外部框架 | 实体、值对象、领域服务 |

### 1.3 依赖方向（关键！）

```
外部框架 → 适配器 → 端口接口 → 领域核心
```

- **领域核心只依赖自身和端口接口**，绝不依赖任何适配器或外部框架
- 适配器依赖端口接口和外部技术，做"翻译"工作
- 通过**依赖倒置（DIP）**，使得依赖始终指向内部

### 1.4 代码结构示例

```
com.example.hexagonal
├── domain/                    # 领域核心（无任何外部依赖）
│   ├── model/
│   │   ├── Order.java         # 实体
│   │   └── OrderStatus.java   # 值对象/枚举
│   ├── service/
│   │   └── OrderDomainService.java  # 领域服务（纯业务逻辑）
│   └── port/                  # 端口（接口定义）
│       ├── input/
│       │   └── CreateOrderUseCase.java   # 输入端口
│       └── output/
│           ├── OrderRepository.java      # 输出端口-持久化
│           └── NotificationPort.java     # 输出端口-通知
│
├── adapter/                   # 适配器（依赖端口和外部技术）
│   ├── input/                 # 主适配器
│   │   ├── rest/
│   │   │   └── OrderController.java      # 实现：调用 CreateOrderUseCase
│   │   └── cli/
│   │       └── OrderCliCommand.java      # 同功能，不同输入渠道
│   └── output/                # 从适配器
│       ├── persistence/
│       │   ├── OrderRepositoryImpl.java  # 实现 OrderRepository，用 JPA/MyBatis
│       │   └── OrderJpaRepository.java
│       └── messaging/
│           └── KafkaNotificationAdapter.java  # 实现 NotificationPort
│
└── Application.java           # 组装类（Spring Boot 启动类）
```

### 1.5 核心优势

1. **极致的可测试性**
   - 领域核心可以脱离数据库、Web 容器直接运行单元测试
   - 端口可以用 Mock 实现，测试速度极快

2. **技术无关性（Technology Agnostic）**
   - 业务逻辑不绑定 Spring、MyBatis 等具体框架
   - 更换数据库（MySQL → PostgreSQL）只需替换适配器，领域代码零改动
   - 同时支持多种接入方式（REST、gRPC、CLI、消息驱动）

3. **边界清晰**
   - 开发者一眼就知道哪些是核心业务，哪些是技术实现细节
   - 新人接手只需要理解领域核心，不需要掌握所有技术栈

4. **符合 DDD 理念**
   - 是 DDD 分层架构的最佳实践载体
   - 领域层真正做到了"纯净"

### 1.6 缺点与适用场景

| 缺点 | 说明 |
|------|------|
| 代码量增加 | 接口、适配器类数量增多，初期开发成本略高 |
| 学习曲线陡 | 团队需要理解端口/适配器、依赖倒置等概念 |
| 过度设计风险 | 简单 CRUD 项目没必要使用 |

**适用场景**：
- 核心业务复杂、生命周期长的项目（5年以上）
- 业务规则频繁变化、需要快速迭代的领域
- 需要极高可测试性的金融/风控/支付系统
- 预期未来可能替换技术栈的产品

---

## 二、整洁架构（Clean Architecture）

整洁架构由 **Robert C. Martin（Uncle Bob，鲍勃大叔）** 于 2012 年提出。它综合了六边形架构、洋葱架构（Onion Architecture）等思想的精华，核心是**通过严格的依赖规则实现业务逻辑与框架的解耦**。

### 2.1 架构图示（同心圆模型）

```
┌──────────────────────────────────────────────────────┐
│  第 4 层：框架与驱动层（Frameworks & Drivers）         │
│  ┌─────────────────────────────────────────────────┐ │
│  │  Spring / Vue / MySQL / Kafka / Redis / JUnit   │ │
│  └─────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────┐ │
│  │  第 3 层：接口适配器层（Interface Adapters）      │ │
│  │  ┌───────────────────────────────────────────┐  │ │
│  │  │ Controller / Presenter / Repository Impl  │  │ │
│  │  │ / Gateway / DTO 转换 / Mapper             │  │ │
│  │  └───────────────────────────────────────────┘  │ │
│  │  ┌───────────────────────────────────────────┐  │ │
│  │  │  第 2 层：用例层（Application Business     │  │ │
│  │  │  Rules / Use Cases）                       │  │ │
│  │  │  ┌─────────────────────────────────────┐  │  │ │
│  │  │  │ CreateOrderUseCase / PayOrderUC     │  │  │ │
│  │  │  │ / CancelOrderUC (编排业务流程)       │  │  │ │
│  │  │  └─────────────────────────────────────┘  │  │ │
│  │  │  ┌─────────────────────────────────────┐  │  │ │
│  │  │  │  第 1 层：实体层（Enterprise         │  │  │ │
│  │  │  │  Business Rules / Entities）         │  │  │ │
│  │  │  │  ┌───────────────────────────────┐  │  │  │ │
│  │  │  │  │  Order / User / Product       │  │  │  │ │
│  │  │  │  │  (纯业务对象 + 业务规则)       │  │  │  │ │
│  │  │  │  └───────────────────────────────┘  │  │  │ │
│  │  │  └─────────────────────────────────────┘  │  │ │
│  │  └───────────────────────────────────────────┘  │ │
│  └─────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

> **依赖规则（The Dependency Rule）**：
> **源代码依赖只能指向内圈**。外圈的任何东西都不能影响内圈，内圈完全不知道外圈的存在。

### 2.2 四层详解

| 层级 | 别称 | 职责 | 代码示例 | 依赖方向 |
|------|------|------|----------|----------|
| **第1层：实体层** | 领域层 / Enterprise Layer | 存放**通用的、最高层次的业务规则**，不随外部变化而变化 | `Order` 实体（含计算总价方法）、`User` 实体（含校验方法） | 只依赖自身，不依赖任何外层 |
| **第2层：用例层** | 应用层 / Application Layer | 编排实体完成**单个业务用例**的流程，定义输入输出端口接口 | `CreateOrderUseCase`（调用 User/Product/Order 实体，组合业务步骤）、`PayOrderUseCase` | 只能向内依赖实体层，定义接口供外层实现 |
| **第3层：接口适配器层** | 适配层 | 负责**数据格式转换**，将用例的输入输出适配到外部框架需要的格式 | `Controller`（HTTP 请求 → 用例入参 DTO）、`Presenter`（用例出参 → HTTP 响应 JSON）、`RepositoryImpl`（用例输出接口 → JPA/MyBatis 实现）、`Mapper`（DTO ↔ 实体） | 向内依赖用例层的接口，向外使用框架 |
| **第4层：框架与驱动层** | 基础设施层 | 最外层，存放具体**技术实现和工具**，是"脏"的一层 | Spring Boot、数据库驱动、Kafka、Redis、Vue、JUnit、Spring Security 等 | 向内适配所有层，自己不被任何层依赖 |

### 2.3 跨层调用：跨越边界（Crossing Boundaries）

内层对外层一无所知，所以跨层调用必须使用**依赖倒置**：

```
┌──────────────────────────────────────────────────────────┐
│ Use Case 层（内层）定义接口（Repository Interface）        │
│      ↑ 依赖方向（Inner ← Outer）                          │
│ Adapter 层（外层）实现该接口（RepositoryImpl 用 MyBatis）   │
└──────────────────────────────────────────────────────────┘
```

**具体流程示例（创建订单）**：

```
步骤 1：HTTP 请求进来（最外层）
  ↓
步骤 2：Controller（第3层）接收
  - 将 RequestBody 转为 CreateOrderInputDTO
  - 调用 useCase.execute(inputDTO)
  ↓
步骤 3：CreateOrderUseCase（第2层）
  - 调用 orderRepository.findById(userId) → 这是接口，谁实现？第3层！
  - 调用 Product 实体（第1层）的业务校验方法
  - 调用 Order 实体（第1层）的构造方法
  - 调用 orderRepository.save(order) → 还是接口
  - 返回 CreateOrderOutputDTO
  ↓
步骤 4：Controller 接收 OutputDTO
  - 组装为 HTTP JSON Response 返回
  ↓
步骤 5：请求结束
```

### 2.4 代码目录结构

```
com.example.clean
│
├── entity/                        # 第1层：实体（纯净，零依赖）
│   ├── Order.java                 #    含业务方法：calculateTotal()
│   ├── Product.java               #    含业务方法：isOnSale()
│   └── User.java
│
├── usecase/                       # 第2层：用例（依赖 entity 包）
│   ├── createorder/               #    每个用例一个包，隔离性好
│   │   ├── CreateOrderUseCase.java    # 业务编排主类
│   │   ├── CreateOrderInput.java      # 入参 Data Class
│   │   ├── CreateOrderOutput.java     # 出参 Data Class
│   │   └── port/
│   │       ├── UserRepositoryPort.java     # 输出端口（接口）
│   │       ├── ProductRepositoryPort.java  # 输出端口
│   │       └── OrderRepositoryPort.java    # 输出端口
│   ├── payorder/
│   └── cancelorder/
│
├── adapter/                       # 第3层：适配器（依赖 usecase）
│   ├── input/
│   │   ├── rest/
│   │   │   ├── OrderController.java       # REST 适配器
│   │   │   └── dto/
│   │   │       ├── CreateOrderRequest.java   # HTTP 入参
│   │   │       └── CreateOrderResponse.java  # HTTP 出参
│   │   └── grpc/
│   │       └── OrderGrpcService.java     # gRPC 适配器（同用例，多入口）
│   └── output/
│       ├── persistence/
│       │   ├── OrderRepositoryAdapter.java    # 实现 OrderRepositoryPort
│       │   ├── OrderJpaDao.java               # JPA 接口
│       │   └── mapper/
│       │       └── OrderPersistenceMapper.java # DO ↔ Entity 转换
│       └── messaging/
│           └── KafkaEventPublisher.java
│
└── framework/                     # 第4层：框架配置
    ├── config/
    │   ├── SpringConfig.java
    │   ├── DataSourceConfig.java
    │   └── SecurityConfig.java
    └── Application.java           # 启动入口（依赖注入装配）
```

### 2.5 整洁架构的"整洁"体现在哪？

1. **测试整洁**
   - 实体层：不需要任何外部依赖，纯 JUnit 测试，毫秒级运行
   - 用例层：Mock 所有 Port 接口即可，不需要数据库、Web 容器
   - 接口适配器：单独测试转换逻辑

2. **依赖整洁**
   - 不会出现"改个数据库字段，业务逻辑跟着崩"的情况
   - 业务代码里找不到 `@Autowired`、`@Entity`、`@RestController` 等框架注解
   - 业务代码与 Spring 解耦，理论上可以无缝切到 Quarkus/Micronaut

3. **职责整洁**
   - 新人看包名就知道代码该放哪
   - Controller 绝不出现业务 `if-else`
   - Entity 绝不出现 JSON 序列化注解

4. **演进整洁**
   - 换数据库：只改 `adapter.output.persistence` 包
   - 换前端协议：只改 `adapter.input` 包（加 GraphQL 适配器）
   - 加功能：新增一个 usecase 包，旧代码零改动（开闭原则）

---

## 三、六边形架构 vs 整洁架构对比

| 维度 | 六边形架构 | 整洁架构 |
|------|-----------|----------|
| **提出者** | Alistair Cockburn（2005） | Robert C. Martin（2012） |
| **视角** | 内外分离（输入/输出端口） | 同心圆分层（四层向内依赖） |
| **层级划分** | 领域核心 + 端口 + 适配器（3层） | 实体 + 用例 + 适配器 + 框架（4层） |
| **核心强调** | 端口和适配器的抽象，左右对称 | 依赖规则的严格执行，上下分层 |
| **用例组织** | 通常作为应用服务 | 独立一层，每个用例一个类 |
| **关系** | 六边形是整洁架构的重要思想来源 | 整洁架构是六边形 + 洋葱架构的综合进化 |
| **可结合性** | **完全可以结合使用**，实践中常混合 |  |

---

## 四、落地建议与常见误区

### 4.1 落地建议

1. **不要一步到位**：先在核心模块（如订单、支付）试点，再逐步推广
2. **用例类要小**：一个用例一个类（类爆炸是正常的，清晰比少代码重要）
3. **依赖注入必须用**：Spring 就是最好的依赖注入框架，用它来组装所有层
4. **严格 Code Review**：核心是防止"偷懒"把业务逻辑写到 Controller 里

### 4.2 常见误区

- ❌ **误区1**：接口越多越整洁 → 接口只为"需要替换的点"而建，简单查询不需要 Port
- ❌ **误区2**：所有项目都用 → 纯 CRUD、生命周期 < 1 年的项目，用三层架构更高效
- ❌ **误区3**：强制四层分包，业务代码塞 Entity → Entity 放业务规则，不是 POJO 加 getter/setter
- ❌ **误区4**：DTO 满天飞 → 同层内可以复用，跨层才需要转换对象

---

## 面试追问集

**Q：为什么要用六边形/整洁架构？不就是三层架构多加了几层吗？**

关键区别在于**依赖方向和业务纯净度**：
- 三层架构中 Service 层直接依赖 DAO（数据库技术），业务逻辑被框架"污染"
- 六边形/整洁架构中，业务用例定义 Repository 接口（输出端口），DAO 层反过来实现该接口
- 结果：业务逻辑可以独立编译、独立测试、独立演进，更换数据库不需要改业务代码

**Q：端口接口会不会太多，导致类爆炸？**

- 类爆炸是**故意设计**的，目的是"一个类一个职责"
- 但 Port 接口不需要为每个查询都建，简单场景可以用"宽接口"（一个 Repository 多个方法）
- 实践中，只有**可能被替换**或**需要 Mock 的点**才建 Port

**Q：你们项目有没有用？举个实际的例子。**

根据实际经验回答，参考模板：
- 我们订单支付模块用了简化版的整洁架构
- 用例层（`CreateOrderUseCase`）不依赖 Spring，纯业务编排
- Repository 接口定义在 usecase 包里，MyBatis 的 Mapper 实现了它
- 好处：写单测不需要启 Spring 容器，直接 Mock Repository，测试跑 100% 覆盖率只要 2 秒
- 坑：团队一开始不适应，老有人把业务写到 Controller，后来加了 Checkstyle 规则限制