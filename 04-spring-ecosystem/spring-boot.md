# Spring Boot

---

## 一、自动配置原理 🔥

### 1.1 @SpringBootApplication 注解

```java
@SpringBootApplication
// 等价于：
@SpringBootConfiguration   // = @Configuration，配置类
@EnableAutoConfiguration   // 启用自动配置（核心）
@ComponentScan             // 组件扫描
```

### 1.2 @EnableAutoConfiguration

```java
@EnableAutoConfiguration
// 内部通过 @Import(AutoConfigurationImportSelector.class) 实现

// AutoConfigurationImportSelector 核心逻辑：
// 1. 读取 META-INF/spring.factories 中的配置类列表
// 2. 通过 @Conditional 条件注解过滤（条件满足才生效）
// 3. 注入符合条件的配置类

// 例如自动配置 DataSource：
@Configuration
@ConditionalOnClass(DataSource.class)
@ConditionalOnMissingBean(DataSource.class)
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {
    // 自动创建 DataSource Bean
}
```

### 1.3 条件注解

| 注解 | 条件 |
|------|------|
| @ConditionalOnClass | classpath 中存在指定类 |
| @ConditionalOnMissingClass | classpath 中不存在指定类 |
| @ConditionalOnBean | 容器中存在指定 Bean |
| @ConditionalOnMissingBean | 容器中不存在指定 Bean |
| @ConditionalOnProperty | 配置文件中存在指定属性 |
| @ConditionalOnResource | classpath 中存在指定资源 |
| @ConditionalOnWebApplication | Web 应用 |
| @ConditionalOnExpression | SpEL 表达式满足 |

---

## 二、Starter 机制

```xml
<!-- Spring Boot Starter 本质是依赖聚合 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!-- 引入了 spring-boot-starter + spring-webmvc + tomcat + jackson + ... -->
```

**追问：如何自定义一个 Starter？**

```
my-starter/
├── pom.xml
├── src/main/java/
│   └── com/example/
│       ├── MyAutoConfiguration.java       # 自动配置类
│       └── MyProperties.java              # 配置属性类
└── src/main/resources/
    └── META-INF/
        └── spring.factories               # 注册自动配置类
```

```properties
# META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.MyAutoConfiguration
```

---

## 三、配置文件

### 3.1 多环境配置

```yaml
# application.yml
spring:
  profiles:
    active: dev

# application-dev.yml
server:
  port: 8080

# application-prod.yml
server:
  port: 8080
```

### 3.2 @ConfigurationProperties vs @Value

```java
// @ConfigurationProperties：批量注入，支持复杂类型
@Component
@ConfigurationProperties(prefix = "app.user")
@Data
public class UserProperties {
    private String name;
    private int age;
    private List<String> roles;
    private Map<String, String> extra;
}

// @Value：单个注入，支持 SpEL
@Value("${app.user.name}")
private String userName;

@Value("#{1 + 2}")
private int compute;  // 3
```

**追问：@ConfigurationProperties 如何开启校验？**

```java
@ConfigurationProperties(prefix = "app")
@Validated
public class AppProperties {
    @NotBlank
    private String name;
}
```

---

## 四、Actuator 监控

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
```

常用端点：

| 端点 | 说明 |
|------|------|
| /actuator/health | 健康检查 |
| /actuator/info | 应用信息 |
| /actuator/metrics | 指标数据 |
| /actuator/env | 环境变量 |
| /actuator/loggers | 日志级别管理 |
| /actuator/beans | Bean 列表 |
| /actuator/mappings | 请求映射 |

---

## 五、启动流程

```
1. 创建 SpringApplication 实例
2. 推断应用类型（SERVLET / REACTIVE / NONE）
3. 加载并触发所有 SpringApplicationRunListener
4. 准备 Environment（加载配置文件）
5. 创建 ApplicationContext
6. 执行 prepareContext（设置环境、注册单例等）
7. 执行 refreshContext（refresh() 方法，IoC 容器初始化）
8. afterRefresh（自定义扩展点）
9. 发布应用启动完成事件
```

**追问：SpringApplication.run() 做了哪些事？**

核心是调用 `context.refresh()`，包含了 Bean 的完整生命周期流程（前面 Spring Core 讲过）。额外做了：
- 打印 Banner
- 异常处理（FailureAnalyzer）
- 启动耗时统计

---

## 面试追问集

**Q：如何排除某个自动配置类？**

```java
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
// 或
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

**Q：Spring Boot 如何处理跨域？**

```java
// 全局跨域
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOriginPatterns("*")
                .allowedMethods("*")
                .allowCredentials(true);
    }
}

// 方法级别
@CrossOrigin(origins = "http://example.com")
```

**Q：Spring Boot 项目打包方式？**

- JAR：内嵌 Tomcat，`java -jar` 直接运行
- WAR：部署到外部 Servlet 容器
