# 架构设计

> 架构设计能力是区分初中级和高级工程师的核心标准，面试中通常结合场景进行考察。

## 📂 内容索引

| 文件 | 核心知识点 |
|------|-----------|
| [design-patterns.md](./design-patterns.md) | 创建型/结构型/行为型模式、设计原则（SOLID）、单例/工厂/代理 |
| [system-architecture.md](./system-architecture.md) | 架构演进（单体→SOA→微服务）、分层架构、技术选型 |
| [ddd.md](./ddd.md) | DDD 分层架构、业务建模、聚合设计、充血/失血模型、仓储与领域事件 |
| [microservice.md](./microservice.md) | 微服务拆分原则、服务间通信、服务网格 |
| [high-availability.md](./high-availability.md) | 高并发/高可用/高性能设计、限流/降级/熔断、全链路压测 |
| [security.md](./security.md) | 认证与授权（OAuth2/JWT）、SQL 注入、XSS/CSRF、API 安全 |
| [like-system.md](./like-system.md) | 点赞系统：Kafka 异步批量、Redis+MySQL、多级缓存读优化、幂等 |

## 🔥 高频考点速览

1. **设计模式**：单例模式（饿汉/懒汉/枚举/双重检查）、工厂模式、策略模式
2. **SOLID 原则**：单一职责、开闭原则、里氏替换、接口隔离、依赖倒置
3. **微服务拆分**：业务边界、数据独立性、康威定律
4. **高可用设计**：多活架构、灰度发布、蓝绿部署、故障演练
5. **限流算法**：计数器、滑动窗口、令牌桶、漏桶及对比
6. **DDD**：领域建模、聚合根、充血模型 vs 失血模型
