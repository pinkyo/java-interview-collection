# Spring Cloud 微服务组件

---

## 一、微服务架构全景

```
          ┌──────────────┐
          │    Gateway    │  (Spring Cloud Gateway)
          │    网关       │
          └──────┬───────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
┌───▼───┐   ┌───▼───┐   ┌───▼───┐
│ 服务A  │   │ 服务B  │   │ 服务C  │
└───┬───┘   └───┬───┘   └───┬───┘
    │            │            │
    └────────────┼────────────┘
                 │
  ┌──────────────┼──────────────┐
  │              │              │
┌─▼─────────┐ ┌─▼─────────┐ ┌─▼─────────┐
│   Nacos   │ │ Sentinel  │ │  Sleuth   │
│ 注册中心   │ │ 限流熔断   │ │ +Zipkin   │
│ 配置中心   │ │           │ │ 链路追踪   │
└───────────┘ └───────────┘ └───────────┘
```

---

## 二、服务注册与发现（Nacos）

### 2.1 注册中心对比

| 对比 | Nacos | Eureka | Zookeeper | Consul |
|------|-------|--------|-----------|--------|
| CAP | AP/CP 可切换 | AP | CP | CP |
| 健康检查 | TCP/HTTP/MySQL | 心跳 | 心跳 | TCP/HTTP |
| 配置中心 | ✅ 集成 | ❌ | ❌ | ✅ |
| 一致性 | Raft (CP) / Distro (AP) | 无 | ZAB | Raft |
| 社区 | 阿里开源（推荐） | Netflix（停更） | Apache | HashiCorp |

### 2.2 Nacos AP vs CP

```
AP 模式（默认）：
- 使用 Distro 协议，最终一致性
- 注册中心场景友好（可用性 > 一致性）

CP 模式：
- 使用 Raft 协议，强一致性
- 配置中心场景适合（一致性要求高）
```

---

## 三、远程调用

### 3.1 Feign（声明式 HTTP 客户端）

```java
@FeignClient(name = "user-service", fallback = UserServiceFallback.class)
public interface UserServiceClient {
    @GetMapping("/users/{id}")
    User getUser(@PathVariable("id") Long id);
}

// 使用
@Autowired
private UserServiceClient userServiceClient;

User user = userServiceClient.getUser(1L);
```

### 3.2 Feign 的负载均衡

Feign 内置 Ribbon（Spring Cloud Hoxton 及之前）或 Spring Cloud LoadBalancer（新版）。

```java
@Configuration
public class FeignConfig {
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;  // 打印完整日志
    }
}
```

### 3.3 Feign 拦截器

```java
@Component
public class FeignAuthInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        template.header("Authorization", "Bearer " + getToken());
    }
}
```

---

## 四、网关（Gateway）

### 4.1 核心概念

| 组件 | 说明 |
|------|------|
| Route（路由） | 匹配规则 + 目标 URI |
| Predicate（断言） | 匹配请求的条件 |
| Filter（过滤器） | 请求前后处理逻辑 |

### 4.2 Gateway 配置

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service   # lb = 负载均衡
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Request-Source, gateway
```

### 4.3 自定义全局过滤器

```java
@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 验证 Token
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (StringUtils.isEmpty(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -100;  // 优先级
    }
}
```

---

## 五、限流熔断（Sentinel）

### 5.1 流控规则

```java
// API 级别
@GetMapping("/users")
@SentinelResource(value = "getUsers", blockHandler = "blockHandler")
public List<User> getUsers() { }

public List<User> blockHandler(BlockException e) {
    return Collections.emptyList();  // 被限流时的降级逻辑
}
```

### 5.2 限流算法对比

| 算法 | Sentinel 实现 | 特点 |
|------|--------------|------|
| 滑动窗口 | 默认 | 统计最近 N 秒的 QPS |
| 令牌桶 | Warm Up | 预热/匀速排队 |
| 漏桶 | 排队等待 | 匀速通过 |

### 5.3 熔断降级策略

```
慢调用比例：响应时间 > 阈值（如 > 1s）→ 触发熔断
异常比例：异常数/总数 > 阈值 → 触发熔断
异常数：1 分钟内异常数 > 阈值 → 触发熔断
```

---

## 六、配置中心

```java
// Nacos 配置中心
@RefreshScope  // 配置自动刷新
@RestController
public class ConfigController {
    @Value("${app.name}")
    private String appName;
}
```

```yaml
# Nacos 中的配置（Data ID: service-name-dev.yaml）
app:
  name: my-app
  timeout: 5000
```

---

## 七、链路追踪（Sleuth + Zipkin）

```
Trace ID：全局唯一，一条请求链路的唯一标识
Span ID：每个服务节点生成一个 Span
Parent Span ID：上一个 Span 的 ID

请求 → 服务A (Span1) → 服务B (Span2) → 服务C (Span3)
      ↑                 ↑                ↑
      └─── Trace ID 统一 ─────────────────┘
```

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-zipkin</artifactId>
</dependency>
```

---

## 面试追问集

**Q：Gateway 和 Zuul 的区别？**

| 对比 | Spring Cloud Gateway | Netflix Zuul |
|------|---------------------|-------------|
| 实现 | WebFlux（响应式，Netty） | Servlet（阻塞式，Tomcat） |
| 性能 | 高（非阻塞） | 较低 |
| 维护 | Spring 官方 | Netflix 停更 |
| 推荐 | ✅ 新项目使用 | 老项目 |

**Q：Sentinel 和 Hystrix 的区别？**

- Sentinel 功能更全（限流、熔断、系统保护、热点限流、网关限流）
- Hystrix 线程池隔离更完善
- Sentinel 支持控制台实时管理，可视化更好
