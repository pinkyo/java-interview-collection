# Java 面试知识宝典

> 系统化、分模块的 Java 面试知识点集合，涵盖基础、JVM、并发、框架、数据库、中间件、架构设计与分布式系统。

---

## 📚 知识模块总览

| 序号 | 模块 | 描述 |
|------|------|------|
| 01 | [Java 基础](./01-java-basics/README.md) | 数据类型、OOP、集合框架、异常、IO/NIO、反射、泛型与注解 |
| 02 | [JVM](./02-jvm/README.md) | 内存结构、垃圾回收、类加载、调优工具 |
| 03 | [并发编程](./03-concurrency/README.md) | 线程基础、锁与同步、线程池、JUC 容器、并发模式 |
| 04 | [Spring 生态](./04-spring-ecosystem/README.md) | Spring Core/MVC/Boot/Cloud、MyBatis |
| 05 | [数据库](./05-database/README.md) | MySQL 架构/索引/事务/优化、分库分表、Redis |
| 06 | [中间件](./06-middleware/README.md) | Kafka、RabbitMQ、ES、Nginx、Zookeeper |
| 07 | [架构设计](./07-architecture-design/README.md) | 设计模式、系统架构、微服务、高可用、安全设计 |
| 08 | [分布式系统](./08-distributed-system/README.md) | 分布式理论、分布式事务、分布式 ID、服务治理、RPC |

---

## 🔥 高频必考知识点

### Java 基础
- `==` 与 `equals()` 的区别
- HashMap 底层原理（1.7 vs 1.8）
- ArrayList vs LinkedList
- String、StringBuilder、StringBuffer 区别
- 深拷贝与浅拷贝

### JVM
- JVM 内存模型与运行时数据区
- GC 算法与垃圾收集器（CMS、G1、ZGC）
- 类加载过程与双亲委派模型
- OOM 排查与 JVM 调优

### 并发编程
- synchronized 锁升级过程（偏向锁→轻量级锁→重量级锁）
- volatile 的可见性与有序性
- AQS 原理与 ReentrantLock
- 线程池核心参数与拒绝策略
- ThreadLocal 原理与内存泄漏

### Spring 生态
- Spring IoC 与依赖注入原理
- Spring AOP 底层实现（JDK 动态代理 vs CGLIB）
- Spring Bean 生命周期
- Spring Boot 自动配置原理
- Spring 事务传播机制

### 数据库
- MySQL 索引底层结构（B+Tree）
- 聚簇索引 vs 非聚簇索引
- MySQL 事务隔离级别与 MVCC
- SQL 优化步骤与 Explain 解读
- 分库分表方案（ShardingSphere）

### 中间件
- Redis 五种数据结构及应用场景
- Redis 持久化（RDB vs AOF）
- Redis 缓存穿透/击穿/雪崩
- Kafka 高吞吐原理与零拷贝
- RabbitMQ 消息可靠性保证

### 架构设计
- CAP 理论与 BASE 理论
- 分布式事务方案（2PC、TCC、Seata）
- 限流算法（令牌桶、漏桶、滑动窗口）
- 微服务拆分原则
- 雪花算法与分布式 ID

---

## 📖 使用建议

1. **按模块系统学习**：从 Java 基础开始，逐步深入
2. **重点突破**：标有 🔥 的高频考点优先掌握
3. **理解原理**：不要死记硬背，每个知识点都要理解"为什么"
4. **动手实践**：代码示例可以实际运行验证效果
5. **定期复习**：利用目录结构快速定位薄弱环节

---

## 🗺 知识图谱

```
                        ┌─────────────────────────────────────┐
                        │          Java 面试知识体系            │
                        └─────────────────────────────────────┘
                                        │
      ┌─────────────┬─────────────┬─────┴─────┬─────────────┬─────────────┐
      │             │             │           │             │             │
   ┌──┴──┐      ┌──┴──┐      ┌──┴──┐    ┌──┴──┐      ┌──┴──┐      ┌──┴──┐
   │Java │      │ JVM │      │并发编程│    │Spring│      │数据存储│     │架构设计│
   │基础 │      │原理 │      │      │    │生态  │      │      │      │      │
   └──┬──┘      └──┬──┘      └──┬──┘    └──┬──┘      └──┬──┘      └──┬──┘
      │             │             │           │             │             │
   ┌──┴──┐      ┌──┴──┐      ┌──┴──┐    ┌──┴──┐      ┌──┴──┐      ┌──┴──┐
   │OOP  │      │内存 │      │线程 │    │IoC/ │      │MySQL│      │分布式│
   │集合 │      │GC   │      │锁   │    │AOP  │      │Redis│      │微服务│
   │IO   │      │调优 │      │线程池│    │Boot │      │MQ   │      │高可用│
   └─────┘      └─────┘      └─────┘    └─────┘      └─────┘      └─────┘
```

---

> **持续更新中** — 欢迎补充和完善知识点。
