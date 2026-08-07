# 服务治理

---

## 一、注册中心

### 1.1 核心功能

```
1. 服务注册：服务启动时向注册中心注册
2. 服务发现：消费者从注册中心获取服务列表
3. 健康检查：定期检测服务是否存活
4. 服务下线：剔除不健康的服务节点
5. 元数据管理：服务权重、版本、分组等
```

### 1.2 注册中心对比

详见 [Spring Cloud 文档](../04-spring-ecosystem/spring-cloud.md)。

| 对比 | Nacos | Eureka | Zookeeper | Consul | Etcd |
|------|-------|--------|-----------|--------|------|
| CAP | AP/CP | AP | CP | CP | CP |
| 控制台 | ✅ | ✅ | ❌ | ✅ | ❌ |
| 配置中心 | ✅ | ❌ | ❌ | ✅ | ❌ |
| 一致性 | Raft/Distro | — | ZAB | Raft | Raft |
| 推荐度 | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| 语言 | Java | Java | Java | Go | Go |

---

## 二、负载均衡

```
负载均衡层次：
客户端负载均衡（Ribbon / LoadBalancer）
服务端负载均衡（Nginx / F5）
```

### 客户端负载均衡算法

| 算法 | 原理 | 适用场景 |
|------|------|---------|
| 随机 | Random 选一个 | 通用 |
| 轮询 | 依次选择 | 实例性能一致 |
| 加权轮询 | 按权重分配 | 实例性能不同 |
| 最小连接数 | 选连接最少的 | 长连接 |
| 一致性 Hash | 相同请求到同一实例 | 缓存命中率 |
| 响应时间权重 | 根据响应时间动态调整 | 自适应 |

---

## 三、限流（详见高可用章节）

### 快速回顾限流算法

| 算法 | 关键点 |
|------|--------|
| 固定窗口 | 简单，有临界问题 |
| 滑动窗口 | 平滑，精度高 |
| 漏桶 | 恒速，不适合突发 |
| 令牌桶 | 允许突发，推荐 |

### Sentinel vs Hystrix

| 特性 | Sentinel | Hystrix |
|------|----------|---------|
| 隔离策略 | 信号量 + 线程池 | 线程池 |
| 熔断降级 | 基于响应时间/异常 | 基于异常 |
| 限流 | 强大（QPS/线程数/关联） | 无 |
| 监控 | 控制台实时 | Dashboard |
| 维护状态 | 阿里活跃维护 | Netflix 停更 |

---

## 四、熔断

### 熔断器模式

```
         ┌─────────┐
正常请求 →│ CLOSED  │← 成功
         └────┬────┘
        失败阈值│触发熔断
              ▼
         ┌─────────┐
         │  OPEN   │→ 快速失败
         └────┬────┘
        休眠超时│
              ▼
         ┌─────────┐
         │HALF_OPEN│→ 探测请求
         └────┬────┘
      成功   │   失败
  ┌─────────┼─────────┐
  ▼                   ▼
CLOSED              OPEN
```

### Sentinel 熔断规则

```java
DegradeRule rule = new DegradeRule()
    .setGrade(RuleConstant.DEGRADE_GRADE_RT)  // 慢调用比例
    .setCount(100)     // 阈值 100ms
    .setTimeWindow(10) // 熔断 10s
    .setMinRequestAmount(5);
```

---

## 五、链路追踪

### 5.1 核心概念

```
Trace：一次完整的请求链路
Span：链路中的一个节点

请求链：Gateway → ServiceA → ServiceB → ServiceC
         Span1     Span2     Span3     Span4
         └──────── TraceID: abc123 ────────┘
```

### 5.2 追踪方案对比

| 方案 | 特点 |
|------|------|
| SkyWalking | 国内最流行，非侵入（Agent），功能全 |
| Jaeger | Uber 开源，兼容 OpenTracing |
| Zipkin | Twitter 开源，轻量 |
| Pinpoint | 韩国开源，功能强但较重 |
| Cat | 美团开源，实时监控 |

```java
// SkyWalking 使用：加 -javaagent 即可，代码无侵入
java -javaagent:skywalking-agent.jar -Dskywalking.agent.service_name=my-service -jar app.jar
```

---

## 六、健康检查与优雅上下线

### 6.1 健康检查

```yaml
# Spring Boot Actuator
management:
  endpoints:
    web:
      exposure:
        include: health
  endpoint:
    health:
      show-details: always
```

Kubernetes 中通过 **Liveness Probe** 和 **Readiness Probe**：
- Liveness：容器是否存活（挂了就重启）
- Readiness：容器是否准备好接受流量

### 6.2 优雅上下线

```
上线：
1. 启动服务 → 注册到注册中心（status=UP）
2. 注册前先预热（Prepared）
3. 启动健康检查 → OK 后才接收流量

下线：
1. 注册中心标记为 DOWN → 停止接收新请求
2. 等待处理中的请求完成（timeout）
3. Spring Boot: spring.lifecycle.timeout-per-shutdown-phase=30s
4. 关闭服务
```

```yaml
# Spring Boot 优雅关闭
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

---

## 面试追问集

**Q：如何做服务灰度发布？**

1. 注册中心元数据 + 版本标签
2. 给灰度服务打 `version=2.0` 标签
3. 网关/消费者根据 Header（如 gray=true）路由到新版本
4. 逐步放量：1% → 10% → 50% → 100%

**Q：你们的服务治理技术栈是什么？**

常见组合：Nacos（注册+配置）+ Sentinel（限流熔断）+ Gateway（网关）+ SkyWalking（链路追踪）+ Prometheus+Grafana（监控）。
