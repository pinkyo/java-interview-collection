# 分布式系统

> 分布式系统是互联网大厂面试的核心考察点，理论深度与实践广度并重。

## 📂 内容索引

| 文件 | 核心知识点 |
|------|-----------|
| [distributed-theory.md](./distributed-theory.md) | CAP 理论、BASE 理论、一致性模型、共识算法（Raft/Paxos） |
| [distributed-transaction.md](./distributed-transaction.md) | 2PC/3PC/TCC/Saga、Seata 框架、最终一致性方案 |
| [distributed-id.md](./distributed-id.md) | 雪花算法、号段模式、Leaf 框架、时钟回拨处理 |
| [service-governance.md](./service-governance.md) | 注册发现、配置中心、负载均衡、限流熔断、链路追踪 |
| [rpc.md](./rpc.md) | RPC 原理、序列化协议、Dubbo 架构、gRPC |

## 🔥 高频考点速览

1. **CAP**：为什么 P 必须选择？CA 为何不现实？
2. **BASE**：最终一致性的实现方式（可靠消息、TCC、最大努力通知）
3. **分布式事务**：TCC vs 2PC、Seata AT 模式原理
4. **雪花算法**：时钟回拨如何处理（百度 UidGenerator、美团 Leaf）
5. **服务治理**：注册中心对比（Nacos/Eureka/Zookeeper/Consul）
6. **限流熔断**：Sentinel 原理、Hystrix 熔断模式
