# 中间件

> 中间件是大型分布式系统的血脉，面试中重点关注其架构原理和应用实践。

## 📂 内容索引

| 文件 | 核心知识点 |
|------|-----------|
| [kafka.md](./kafka.md) | Topic/Partition 模型、高吞吐原理、零拷贝、ISR、幂等与事务 |
| [rabbitmq.md](./rabbitmq.md) | 交换机类型、消息确认机制、死信队列、延迟消息、集群 |
| [mq-selection.md](./mq-selection.md) | RabbitMQ vs Kafka 选型标准、对比维度、决策树 |
| [elasticsearch.md](./elasticsearch.md) | 倒排索引、分片机制、写入/检索原理、聚合分析 |
| [nginx.md](./nginx.md) | 反向代理与正向代理、负载均衡算法、限流配置、Lua 扩展 |
| [zookeeper.md](./zookeeper.md) | ZAB 协议、节点类型、Watcher 机制、选举 Leader、典型应用 |

## 🔥 高频考点速览

1. **Kafka 高吞吐**：顺序写、零拷贝（sendfile）、Page Cache、批量压缩
2. **Kafka 可靠性**：ISR、ACK 机制、幂等与事务
3. **RabbitMQ 可靠性**：生产者确认、消费者确认、持久化策略
4. **ES 倒排索引**：Term Dictionary、Term Index、FST、段合并
5. **Nginx**：负载均衡算法（轮询/IP Hash/最少连接）、高并发模型
6. **Zookeeper**：CP 实现、选举机制、分布式锁实现
