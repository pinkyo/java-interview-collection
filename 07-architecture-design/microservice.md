# 微服务架构设计

---

## 一、微服务拆分原则

### 1.1 拆分原则

| 原则 | 说明 |
|------|------|
| **单一职责** | 一个微服务只做一件事 |
| **高内聚低耦合** | 服务内聚，服务间松耦合 |
| **数据独立性** | 每个微服务有自己的数据库 |
| **康威定律** | 系统架构反映组织沟通结构 |
| **边界上下文** | 按 DDD 的限界上下文划分 |

### 1.2 拆分步骤

```
1. 识别业务域（用户域、订单域、商品域、支付域...）
2. 提炼限界上下文
3. 确定服务大小（微服务不宜过大也不宜过小，一般 3-5 人团队维护）
4. 定义服务接口（API First）
5. 数据库拆分（每服务独立 DB）
```

---

## 二、服务间通信

### 2.1 同步通信

```java
// HTTP REST（Feign）
@FeignClient(name = "user-service")
public interface UserClient {
    @GetMapping("/users/{id}")
    UserDTO getUser(@PathVariable Long id);
}

// RPC（Dubbo）
@DubboReference
private UserService userService;
```

### 2.2 异步通信（MQ 解耦）

```
订单创建 ──► MQ ──► 库存扣减
                ──► 物流通知
                ──► 积分增加
```

### 2.3 通信方式对比

| 方式 | 优点 | 缺点 | 场景 |
|------|------|------|------|
| HTTP REST | 简单、跨语言 | 性能较低、同步阻塞 | 对外 API、跨团队 |
| RPC | 高性能、类型安全 | 强依赖、跨语言困难 | 内部服务调用 |
| MQ 异步 | 解耦、削峰、最终一致性 | 延迟、复杂 | 非实时业务 |

---

## 三、API 网关

```
客户端 → Gateway(认证/限流/路由/日志) → 微服务集群
```

### 网关职责

1. **路由转发**：根据 Path/Header 路由到对应服务
2. **认证鉴权**：统一 JWT 校验，透传用户信息
3. **限流熔断**：保护后端服务
4. **日志监控**：统一记录请求日志
5. **协议转换**：HTTP → gRPC、协议适配
6. **聚合**：BFF 模式下按前端定制返回

---

## 四、服务容器化

### Docker

```dockerfile
FROM openjdk:17-slim
COPY target/app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    spec:
      containers:
      - name: user-service
        image: user-service:1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
```

---

## 五、服务网格（Service Mesh）

```
                 ┌───────────────────────┐
                 │      Control Plane     │
                 │        (Istiod)        │
                 └───────────┬───────────┘
                             │
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                 ▼
   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
   │   Service A  │ │   Service B  │ │   Service C  │
   │ ┌──────────┐ │ │ ┌──────────┐ │ │ ┌──────────┐ │
   │ │Envoy Sidecar│ │ │Envoy Sidecar│ │ │Envoy Sidecar│ │
   │ │(Data Plane)│ │ │(Data Plane)│ │ │(Data Plane)│ │
   │ └──────────┘ │ │ └──────────┘ │ │ └──────────┘ │
   └──────────────┘ └──────────────┘ └──────────────┘
```

**优点**：非侵入式（业务代码无关）、内置可观测性、流量管理。
**缺点**：复杂度高、性能开销。

---

## 六、CI/CD

```
代码提交 → 编译 → 单元测试 → 代码扫描 → 构建镜像 → 部署到测试环境 → 自动化测试 → 部署到生产
```

| 工具 | 用途 |
|------|------|
| Jenkins / GitLab CI | CI/CD 引擎 |
| Maven / Gradle | 构建 |
| SonarQube | 代码质量 |
| Docker | 容器化 |
| Kubernetes | 编排部署 |
| ArgoCD | GitOps |

---

## 面试追问集

**Q：微服务的优缺点？**

**优点**：独立部署、技术异构、弹性伸缩、故障隔离、团队自治
**缺点**：分布式事务复杂、运维成本高、网络延迟、数据一致性挑战

**Q：如何保证微服务的可观测性？**

三大支柱：
1. **日志（Logging）**：ELK（Elasticsearch + Logstash + Kibana）
2. **指标（Metrics）**：Prometheus + Grafana
3. **链路追踪（Tracing）**：SkyWalking / Jaeger / Zipkin

**Q：你们项目中微服务数量有多少？如何划分？**

根据实际经验回答。示例：用户服务、订单服务、商品服务、支付服务、营销服务、搜索服务、网关服务。
