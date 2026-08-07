# RPC 原理与实践

---

## 一、什么是 RPC？

RPC（Remote Procedure Call）远程过程调用，像调用本地方法一样调用远程服务。

```
调用方                    被调用方
┌──────────┐            ┌──────────┐
│ Client   │  ─网络──►  │ Server   │
│ Stub     │            │ Stub     │
└──────────┘            └──────────┘
     │                       │
     ▼                       ▼
 序列化请求               反序列化请求
     │                       │
     ▼                       ▼
  网络传输    ──────────►   执行方法
     │                       │
     ▼                       ▼
 反序列化响应             序列化响应
```

---

## 二、RPC 核心流程

```
1. Client 调用本地接口
2. Client Stub 将方法、参数序列化
3. 通过网络发送到 Server
4. Server Stub 反序列化请求
5. Server Stub 调用本地实现
6. Server 将结果序列化
7. 通过网络返回给 Client
8. Client Stub 反序列化结果
```

---

## 三、序列化协议

| 协议 | 格式 | 性能 | 可读性 | 语言支持 | 特点 |
|------|------|------|--------|---------|------|
| JSON | 文本 | 中 | ✅ | 跨语言 | 通用、可读 |
| Hessian | 二进制 | 中 | ❌ | 跨语言 | Dubbo 默认 |
| Protobuf | 二进制 | 高 | ❌ | 跨语言 | Google、需定义 schema |
| Kryo | 二进制 | 高 | ❌ | Java | 高性能、需注册类 |
| Thrift | 二进制 | 高 | ❌ | 跨语言 | Facebook、完整 RPC 框架 |
| MessagePack | 二进制 | 中高 | ❌ | 跨语言 | 类 JSON 的二进制 |

**追问：序列化协议怎么选？**

- 内部高性能调用：Protobuf / Kryo
- 外部 API：JSON
- 跨语言：Protobuf

---

## 四、Dubbo 架构

```
   ┌──────────┐               ┌──────────┐
   │  Registry │               │  Monitor │
   │ (注册中心) │               │ (监控中心) │
   └─────┬─────┘               └─────┬─────┘
         │                           │
    ┌────┴───────────────────────────┴────┐
    │                                     │
┌───▼────┐                        ┌───────▼──┐
│Provider│◄──── RPC 调用 ────────│ Consumer │
│(服务端) │                        │ (客户端)  │
└────────┘                        └──────────┘
```

**分层架构：**

```
┌────────────────────────────────────────────┐
│                Service 层                    │ ← 业务接口层
├────────────────────────────────────────────┤
│                Config 层                     │ ← 配置、注解
├────────────────────────────────────────────┤
│               Proxy 层                       │ ← 动态代理（Javassist/JDK）
├────────────────────────────────────────────┤
│              Registry 层                     │ ← 服务注册与发现
├────────────────────────────────────────────┤
│               Cluster 层                     │ ← 路由、负载均衡
├────────────────────────────────────────────┤
│               Monitor 层                     │ ← 监控统计
├────────────────────────────────────────────┤
│              Protocol 层                     │ ← RPC 协议
├────────────────────────────────────────────┤
│              Exchange 层                     │ ← 请求响应模型
├────────────────────────────────────────────┤
│              Transport 层                    │ ← 网络传输（Netty）
├────────────────────────────────────────────┤
│              Serialize 层                    │ ← 序列化
└────────────────────────────────────────────┘
```

### Dubbo 负载均衡策略

| 策略 | 说明 |
|------|------|
| Random（默认） | 加权随机 |
| RoundRobin | 加权轮询 |
| LeastActive | 最小活跃数（性能好的机器响应快，活跃数少） |
| ConsistentHash | 一致性 Hash（相同参数到同一机器） |

### Dubbo 集群容错

| 策略 | 说明 |
|------|------|
| Failover（默认） | 失败自动切换，重试其他机器 |
| Failfast | 快速失败，直接抛异常 |
| Failsafe | 失败安全，忽略异常 |
| Failback | 失败后后台定时重试 |
| Forking | 并行调用多个，一个成功就返回 |
| Broadcast | 广播调用所有提供者 |

---

## 五、gRPC（Google RPC）

```protobuf
// 定义 proto 文件
service GreetingService {
  rpc SayHello (HelloRequest) returns (HelloResponse);
}

message HelloRequest {
  string name = 1;
}

message HelloResponse {
  string message = 1;
}
```

```java
// Java 使用
// 基于 HTTP/2 + Protobuf
// 支持四种调用方式：Unary、Server Streaming、Client Streaming、Bidirectional Streaming
```

---

## 面试追问集

**Q：Dubbo 的连接方式和通讯协议？**

- 默认：单一长连接 + NIO 异步通讯（Netty）
- 协议：dubbo（默认，单长连接）/ rmi / hessian / http / thrift / gRPC
- 为什么是单长连接？服务提供者较少时，一个连接就能满足，降低连接管理成本

**Q：Dubbo 和 Spring Cloud 的区别？**

| 对比 | Dubbo | Spring Cloud |
|------|-------|-------------|
| 定位 | RPC 框架 | 微服务整体解决方案 |
| 通信 | TCP（高性能） | HTTP REST（更通用） |
| 社区生态 | 阿里+Apache | Pivotal+Spring |
| 注册中心 | 多支持（Nacos/ZK/Redis） | 主要为 Nacos/Eureka |
| 适用场景 | 高性能内部服务调用 | 完整的微服务架构 |

**Q：RPC 调用和 HTTP 调用的区别？**

- RPC：面向方法，性能高（二进制序列化、TCP），耦合度高
- HTTP：面向资源，通用（跨语言），RESTful 风格，性能稍低

现在很多项目两者都在用：对内 RPC（Dubbo），对外 HTTP REST。
