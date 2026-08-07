---
name: java-interview-knowledge-collection
overview: 在空项目中创建一套结构化的 Java 面试知识点文档，按大类分目录管理，涵盖 Java 基础、JVM、并发、框架、数据库、中间件、架构设计等核心领域，采用 MD 格式方便维护和查阅。
todos:
  - id: create-structure
    content: 创建项目根目录 README 与 8 个模块目录结构
    status: completed
  - id: java-basics
    content: 编写 Java 基础模块：数据类型、OOP、集合框架、异常、IO、反射、泛型与注解
    status: completed
    dependencies:
      - create-structure
  - id: jvm-module
    content: 编写 JVM 模块：内存结构、垃圾回收、类加载机制、调优工具
    status: completed
    dependencies:
      - create-structure
  - id: concurrency-module
    content: 编写并发编程模块：线程基础、锁与同步、线程池、JUC 容器、并发模式
    status: completed
    dependencies:
      - create-structure
  - id: spring-database
    content: 编写 Spring 生态与数据库模块：Spring 全家桶、MyBatis、MySQL 全套知识点
    status: completed
    dependencies:
      - create-structure
  - id: middleware-redis
    content: 编写中间件模块与 Redis：Kafka、RabbitMQ、Elasticsearch、Nginx、Zookeeper、Redis 核心知识
    status: completed
    dependencies:
      - create-structure
  - id: architecture-distributed
    content: 编写架构设计与分布式系统模块：设计模式、微服务、高可用、分布式理论、事务、服务治理
    status: completed
    dependencies:
      - create-structure
---

## 用户需求
在空项目中创建一套系统化的 Java 面试知识点文档集合，按分类大纲形式管理，全面覆盖 Java 面试核心领域。

## 产品概述
一个结构清晰、内容全面的 Java 面试知识库，以 Markdown 文件 + 层级目录的方式组织，便于查阅、维护和持续更新。

## 核心功能
- **模块化分类**：按知识领域拆分为独立模块，每个模块含自述索引
- **大纲层级管理**：采用多级目录结构，从概览到具体知识点层层递进
- **架构设计专题**：包含系统架构、设计模式、微服务架构等架构相关内容
- **中间件专题**：覆盖 Redis、Kafka、RabbitMQ、Elasticsearch、Nginx、Zookeeper 等常见中间件
- **分布式系统专题**：含分布式理论、一致性协议、分布式事务、服务治理等内容

## 技术方案

### 实现方式
采用静态 Markdown 文档集合方案，通过层级目录结构实现知识点分类管理。所有内容以 `.md` 文件形式存储，无需构建工具，可直接用任何 Markdown 阅读器浏览。

### 目录结构设计
项目采用**领域驱动**的模块划分方式，将 Java 面试知识分为 8 大模块：

```
java-interview-collection/
├── README.md                          # 项目总索引，含知识图谱与使用指南
├── 01-java-basics/                    # Java 基础模块
│   ├── README.md                      # 模块索引
│   ├── data-types.md                  # 数据类型与运算符
│   ├── oop.md                         # 面向对象（封装/继承/多态）
│   ├── collection-framework.md        # 集合框架（List/Set/Map/Queue）
│   ├── exception.md                   # 异常处理机制
│   ├── io-stream.md                   # IO 流与 NIO
│   ├── reflection.md                  # 反射机制
│   └── generics-annotation.md         # 泛型与注解
├── 02-jvm/                            # JVM 模块
│   ├── README.md                      # 模块索引
│   ├── memory-structure.md            # 内存结构（堆/栈/方法区/元空间）
│   ├── gc.md                          # 垃圾回收机制（算法/收集器）
│   ├── class-loading.md               # 类加载机制
│   └── tuning.md                      # JVM 调优与排查工具
├── 03-concurrency/                    # 并发编程模块
│   ├── README.md                      # 模块索引
│   ├── thread-basics.md               # 线程基础（生命周期/创建方式）
│   ├── lock-synchronization.md        # 锁与同步（synchronized/Lock/AQS）
│   ├── thread-pool.md                 # 线程池与 Executor 框架
│   ├── juc-collections.md             # JUC 并发容器
│   └── concurrent-patterns.md         # 并发编程模式与最佳实践
├── 04-spring-ecosystem/               # Spring 生态模块
│   ├── README.md                      # 模块索引
│   ├── spring-core.md                 # Spring IoC/DI/AOP 核心
│   ├── spring-mvc.md                  # Spring MVC 原理
│   ├── spring-boot.md                 # Spring Boot 自动配置与 Starter
│   ├── spring-cloud.md                # Spring Cloud 微服务组件
│   └── mybatis.md                     # MyBatis/MyBatis-Plus
├── 05-database/                       # 数据库模块
│   ├── README.md                      # 模块索引
│   ├── mysql-architecture.md          # MySQL 架构与存储引擎
│   ├── mysql-index.md                 # 索引原理与优化
│   ├── mysql-transaction-lock.md      # 事务与锁机制
│   ├── mysql-optimization.md          # SQL 优化与分库分表
│   └── redis.md                       # Redis 核心知识（数据结构/持久化/集群）
├── 06-middleware/                     # 中间件模块
│   ├── README.md                      # 模块索引
│   ├── kafka.md                       # Kafka 架构/消息模型/可靠性
│   ├── rabbitmq.md                    # RabbitMQ 核心概念与实践
│   ├── elasticsearch.md               # Elasticsearch 索引/检索/聚合
│   ├── nginx.md                       # Nginx 反向代理/负载均衡
│   └── zookeeper.md                   # Zookeeper 分布式协调
├── 07-architecture-design/            # 架构设计模块
│   ├── README.md                      # 模块索引
│   ├── design-patterns.md             # 常见设计模式（创建型/结构型/行为型）
│   ├── system-architecture.md         # 系统架构设计原则与范式
│   ├── microservice.md                # 微服务架构设计
│   ├── high-availability.md           # 高可用/高并发/高性能设计
│   └── security.md                    # 系统安全设计
└── 08-distributed-system/             # 分布式系统模块
    ├── README.md                      # 模块索引
    ├── distributed-theory.md          # CAP/BASE/一致性模型
    ├── distributed-transaction.md     # 分布式事务（2PC/TCC/Seata）
    ├── distributed-id.md              # 分布式 ID 生成方案
    ├── service-governance.md          # 服务治理（注册发现/限流/熔断）
    └── rpc.md                         # RPC 原理与实践
```

### 文档编写规范
- 每个模块以 `README.md` 作为索引，列出子主题及核心考点摘要
- 每个知识点文件采用统一结构：**核心概念** → **原理分析** → **面试高频问题** → **参考答案** → **延伸思考**
- 架构设计中包含 Mermaid 图表辅助理解系统拓扑与交互流程
- 每个文件控制在适量篇幅，注重核心要点提炼而非冗长堆砌
