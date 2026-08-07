# Spring Core（IoC / DI / AOP）

---

## 一、IoC（控制反转）

### 1.1 什么是 IoC？

IoC（Inversion of Control）是将对象的创建和管理权交给 Spring 容器，而不是在代码中 new 对象。实现方式为 **DI（依赖注入）**。

### 1.2 IoC 容器

| 接口 | 说明 |
|------|------|
| BeanFactory | 最基础的 IoC 容器，延迟加载 Bean |
| ApplicationContext | BeanFactory 的子接口，更强大（国际化、事件广播、AOP） |

```java
// BeanFactory：首次 getBean() 时才创建
BeanFactory factory = new XmlBeanFactory(new ClassPathResource("beans.xml"));

// ApplicationContext：启动时初始化所有单例 Bean
ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
```

### 1.3 Bean 的装配方式

```java
// 1. XML 配置（传统）
<bean id="userService" class="com.example.UserService"/>

// 2. 注解配置
@Component       // 通用组件
@Service         // 业务层
@Repository      // DAO 层
@Controller      // 控制层

// 3. Java Config（推荐）
@Configuration
public class AppConfig {
    @Bean
    public UserService userService() {
        return new UserService();
    }
}
```

---

## 二、依赖注入方式

```java
// 1. 构造器注入（推荐，强制依赖）
@Service
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {  // @Autowired 可省略
        this.userRepository = userRepository;
    }
}

// 2. Setter 注入（可选依赖）
@Autowired
public void setUserRepository(UserRepository repo) {
    this.repo = repo;
}

// 3. 字段注入（不推荐，难以测试）
@Autowired
private UserRepository userRepository;
```

**追问：为什么推荐构造器注入？**

- 依赖不可变（final）
- 保证注入的对象不为 null
- 便于单元测试（Mock 传参即可）
- 避免 Spring Framework 的耦合

---

## 三、Bean 生命周期 🔥

```
1. 实例化（反射创建对象）
2. 属性赋值（populateBean，DI 注入）
3. BeanNameAware.setBeanName()
4. BeanFactoryAware.setBeanFactory()
5. ApplicationContextAware.setApplicationContext()
6. BeanPostProcessor.postProcessBeforeInitialization()
7. @PostConstruct / InitializingBean.afterPropertiesSet()
8. 自定义 init-method
9. BeanPostProcessor.postProcessAfterInitialization()  ← AOP 在这里生成代理
10. 使用 Bean
11. @PreDestroy / DisposableBean.destroy()
12. 自定义 destroy-method
```

---

## 四、循环依赖 🔥

### 4.1 什么是循环依赖？

```java
@Service
class A {
    @Autowired private B b;
}

@Service
class B {
    @Autowired private A a;
}
// A 依赖 B，B 依赖 A → 循环依赖
```

### 4.2 三级缓存解决循环依赖

```java
// 一级缓存（singletonObjects）：存放完全初始化好的 Bean
// 二级缓存（earlySingletonObjects）：存放早期引用（原始 Bean）
// 三级缓存（singletonFactories）：存放 ObjectFactory（生成代理对象的工厂）

// 解决流程：
// 1. 创建 A → 实例化后放入三级缓存（ObjectFactory）
// 2. 填充 A 的属性 B → 发现 B 不存在 → 创建 B
// 3. 创建 B → 实例化 → 填充 B 的属性 A → 从三级缓存获取 A 的 ObjectFactory
//    → 生成 A 的早期引用（可能生成代理）→ 放入二级缓存
// 4. B 完成初始化 → 放入一级缓存
// 5. A 继续填充 B → 完成初始化 → 放入一级缓存
```

**追问：为什么需要三级缓存，二级够吗？**

核心原因是 **AOP**。如果 A 需要被代理，在实例化后还不能确定是否需要代理，需要等到 postProcessAfterInitialization 阶段。三级缓存中的 ObjectFactory 可以**延迟生成代理对象**——如果 B 注入 A 时 A 还没做 AOP，ObjectFactory 可以保证 B 拿到的是代理后的 A。

如果没有 AOP，二级缓存就够了。

**追问：构造器注入的循环依赖为什么无法解决？**

构造器注入在实例化阶段就需要依赖对象，此时对象还没被创建，三级缓存中也没有，必然失败。解决方式：
1. 改为 Setter/字段注入
2. 使用 `@Lazy` 延迟注入
3. 重新设计，消除循环依赖

---

## 五、AOP（面向切面编程）

### 5.1 核心概念

| 概念 | 说明 |
|------|------|
| Aspect（切面） | 横切关注点的模块化（如日志、事务） |
| JoinPoint（连接点） | 程序执行的某个点（通常为方法调用） |
| Advice（通知） | 在连接点执行的动作（Before/After/Around） |
| Pointcut（切点） | 匹配连接点的表达式 |
| Weaving（织入） | 将切面应用到目标对象的过程 |

### 5.2 通知类型

```java
@Aspect
@Component
public class LogAspect {

    @Before("execution(* com.example.*.*(..))")
    public void before() { }

    @After("execution(* com.example.*.*(..))")
    public void after() { }  // 无论正常还是异常都会执行

    @AfterReturning(value = "execution(* com.example.*.*(..))", returning = "result")
    public void afterReturning(Object result) { }  // 正常返回后

    @AfterThrowing(value = "execution(* com.example.*.*(..))", throwing = "e")
    public void afterThrowing(Exception e) { }  // 异常后

    @Around("execution(* com.example.*.*(..))")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        // before
        Object result = pjp.proceed();
        // after
        return result;
    }
}
```

### 5.3 JDK 动态代理 vs CGLIB

| 对比 | JDK 动态代理 | CGLIB |
|------|------------|-------|
| 原理 | 反射 + 接口实现 | ASM 字节码 + 子类继承 |
| 依赖接口 | 必须有接口 | 不需要接口 |
| 限制 | 只能代理接口方法 | 不能代理 final 类/方法 |
| 性能 | 创建快，JDK 8+ 反射优化后调用也快 | 创建稍慢，直接调用快 |

Spring 默认：有接口用 JDK 代理，无接口用 CGLIB。

**Spring Boot 2.x 默认使用 CGLIB**（`spring.aop.proxy-target-class=true`）。

---

## 六、Spring 事务

### 6.1 事务传播机制（7 种）

| 传播行为 | 说明 |
|---------|------|
| REQUIRED（默认） | 有事务则加入，无则新建 |
| REQUIRES_NEW | 总是新建事务，挂起当前事务 |
| SUPPORTS | 有则加入，无则非事务执行 |
| NOT_SUPPORTED | 非事务执行，挂起当前事务 |
| MANDATORY | 必须有事务，无则抛异常 |
| NEVER | 必须无事务，有则抛异常 |
| NESTED | 嵌套事务（savepoint 回滚） |

### 6.2 事务失效的场景

```java
// 1. 非 public 方法
@Transactional
void privateMethod() { }  // 失效

// 2. 自调用（同类方法调用）
public void methodA() {
    this.methodB();  // 直接调用，不走代理 → 事务失效！
}

@Transactional
public void methodB() { }

// 解决：注入自身，或拆分到不同类
@Autowired
private UserService self;
self.methodB();

// 3. 异常被捕获
@Transactional
public void save() {
    try {
        // ... 抛异常
    } catch (Exception e) {
        // 吞掉异常 → 事务不回滚
    }
}
// 解决：catch 中重新抛出，或手动事务回滚
TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();

// 4. 非 RuntimeException（默认回滚规则）
@Transactional
public void save() throws Exception {  // 受检异常默认不回滚
}
// 解决：@Transactional(rollbackFor = Exception.class)

// 5. 数据库引擎不支持（MyISAM 不支持事务）
```

---

## 面试追问集

**Q：BeanFactory 和 ApplicationContext 的区别？**

- BeanFactory：IoC 基础容器，延迟加载
- ApplicationContext：BeanFactory 子接口，启动时初始化单例 Bean，支持国际化、事件机制、资源加载、Web 环境

**Q：@Autowired 和 @Resource 的区别？**

- @Autowired：Spring 提供，默认 byType，配合 @Qualifier 实现 byName
- @Resource：JSR-250 提供，默认 byName，找不到再 byType

**Q：Spring 如何解决循环依赖？**

三级缓存（singletonObjects / earlySingletonObjects / singletonFactories），关键在 ObjectFactory 延迟生成代理对象。

**Q：Spring 用到了哪些设计模式？**

- 单例模式（Bean 默认单例）、工厂模式（BeanFactory）、代理模式（AOP）
- 模板模式（JdbcTemplate）、观察者模式（事件监听）
- 适配器模式（HandlerAdapter）、策略模式（Resource 接口）
