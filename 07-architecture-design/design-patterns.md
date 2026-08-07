# 设计模式

---

## 一、设计原则（SOLID）

| 原则 | 说明 |
|------|------|
| S - 单一职责 | 一个类只做一件事 |
| O - 开闭原则 | 对扩展开放，对修改关闭 |
| L - 里氏替换 | 子类可以替换父类而不影响程序正确性 |
| I - 接口隔离 | 不强迫客户端依赖不需要的接口 |
| D - 依赖倒置 | 依赖抽象而非具体实现 |

**追问：除了 SOLID，还有哪些常见设计原则？**

- **KISS**（Keep It Simple and Stupid）：保持简单
- **YAGNI**（You Aren't Gonna Need It）：不过度设计
- **DRY**（Don't Repeat Yourself）：不重复代码
- **CRP**（组合复用原则）：多用组合少用继承

---

## 二、创建型模式

### 2.1 单例模式 🔥

```java
// 1. 饿汉式 → 类加载时创建，线程安全，可能浪费内存
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() {}
    public static Singleton getInstance() { return INSTANCE; }
}

// 2. 懒汉式（双重检查锁定，DCL）→ 按需创建
public class Singleton {
    private static volatile Singleton instance;
    private Singleton() {}
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}

// 3. 静态内部类 → 利用类加载机制，懒加载 + 线程安全
public class Singleton {
    private Singleton() {}
    private static class Holder {
        private static final Singleton INSTANCE = new Singleton();
    }
    public static Singleton getInstance() { return Holder.INSTANCE; }
}

// 4. 枚举 → 最安全（防反射、防序列化破坏）
public enum Singleton {
    INSTANCE;
    public void doSomething() { }
}
```

**追问：如何破坏单例？如何防止？**

破坏方式：
- **反射**：通过 `setAccessible` 调用私有构造器
- **序列化**：反序列化创建新实例
- **克隆**：重写 clone()

防止：
- 枚举（天然防反射和序列化）
- 构造器中判断实例是否已存在，抛异常

### 2.2 工厂模式

```java
// 简单工厂
class AnimalFactory {
    public static Animal create(String type) {
        switch (type) {
            case "dog": return new Dog();
            case "cat": return new Cat();
        }
    }
}

// 工厂方法（每个产品一个工厂）
interface AnimalFactory { Animal create(); }
class DogFactory implements AnimalFactory { public Animal create() { return new Dog(); } }
class CatFactory implements AnimalFactory { public Animal create() { return new Cat(); } }

// 抽象工厂（创建一族产品）
interface UIFactory { Button createButton(); Input createInput(); }
class WinFactory implements UIFactory { /* Windows 风格 */ }
class MacFactory implements UIFactory { /* Mac 风格 */ }
```

### 2.3 建造者模式（Builder）

```java
User user = User.builder()
    .name("张三")
    .age(25)
    .email("zhang@example.com")
    .build();

// Lombok @Builder 自动生成
@Builder
public class User {
    private String name;
    private int age;
}
```

### 2.4 原型模式（Prototype）

```java
// 通过克隆创建对象，适用于创建成本高的对象
User user1 = new User("张三", 25);
User user2 = (User) user1.clone();  // 需要实现 Cloneable
```

---

## 三、结构型模式

### 3.1 代理模式 🔥

Spring AOP 是最典型的应用。详见 [Spring Core 文档](../04-spring-ecosystem/spring-core.md) 中的 AOP 章节。

```java
// 静态代理（需手动编写代理类）
// 动态代理（JDK 动态代理 / CGLIB）
// Spring AOP 正是基于动态代理实现
```

### 3.2 适配器模式

```java
// Spring MVC 的 HandlerAdapter
interface Target { void request(); }
class Adaptee { public void specificRequest() { } }

class Adapter implements Target {
    private Adaptee adaptee;
    public void request() { adaptee.specificRequest(); }  // 转换接口
}
```

### 3.3 装饰器模式

```java
// Java IO 中的 BufferedInputStream(FileInputStream) 就是装饰器
InputStream in = new BufferedInputStream(new FileInputStream("file.txt"));
// 不改变原有类，动态增强功能
```

### 3.4 外观模式（Facade）

```java
// 复杂的子系统 → 简单的统一接口
class Computer {
    private CPU cpu;
    private Memory memory;
    public void start() {
        cpu.start();
        memory.load();
    }
}
```

### 3.5 桥接模式

将抽象部分与实现部分分离，使它们可以独立变化。

### 3.6 组合模式

树状结构（如文件系统、UI 组件树），整体和部分统一对待。

### 3.7 享元模式（Flyweight）

复用不可变对象（如 String 常量池、Integer 缓存、数据库连接池）。

---

## 四、行为型模式

### 4.1 策略模式 🔥

```java
// 支付策略
interface PaymentStrategy { void pay(int amount); }
class AliPay implements PaymentStrategy { public void pay(int amount) { /* 支付宝 */ } }
class WeChatPay implements PaymentStrategy { public void pay(int amount) { /* 微信 */ } }

class PaymentContext {
    private PaymentStrategy strategy;
    public void setStrategy(PaymentStrategy s) { this.strategy = s; }
    public void execute(int amount) { strategy.pay(amount); }
}

// Spring 中大量使用：Resource 接口、TransactionManager
```

**追问：策略模式和if-else的区别？**

| 对比 | 策略模式 | if-else |
|------|---------|---------|
| 扩展性 | ✅ 新增策略类即可 | ❌ 修改原有逻辑 |
| 代码复杂度 | 类数量多 | 逻辑集中但冗长 |
| 可测试性 | ✅ 每策略独立测试 | ❌ 需覆盖所有分支 |

### 4.2 观察者模式

Spring 事件机制：`ApplicationEvent` + `ApplicationListener` / `@EventListener`。

### 4.3 模板方法模式

```java
abstract class AbstractTemplate {
    public final void template() {
        step1();
        step2();   // 子类实现
        step3();
    }
    protected abstract void step2();
}

// Spring: JdbcTemplate, RestTemplate, RedisTemplate
```

### 4.4 责任链模式

```java
// Java Web: Filter Chain, Interceptor Chain
// GateWay 过滤器链, Sentinel 责任链路
```

### 4.5 其他行为型模式

| 模式 | 场景 |
|------|------|
| 迭代器 | Java Collection Iterator |
| 状态模式 | 订单状态流转 |
| 命令模式 | 撤销/重做、消息队列 |
| 备忘录 | 快照/回滚 |
| 中介者 | 微服务编排（Orchestrator） |
| 解释器 | SQL 解析、EL 表达式 |
| 访问者 | 编译器的 AST 遍历 |

---

## 五、Spring 中用到了哪些设计模式？

| 模式 | Spring 中的应用 |
|------|---------------|
| 单例 | Bean 默认单例（singleton scope） |
| 工厂 | BeanFactory / ApplicationContext |
| 代理 | AOP（JDK 动态代理 / CGLIB） |
| 模板方法 | JdbcTemplate / RestTemplate / RedisTemplate |
| 观察者 | ApplicationEvent 事件机制 |
| 适配器 | HandlerAdapter（将不同 Controller 适配给 DispatcherServlet） |
| 策略 | Resource 接口（不同资源加载策略） |
| 责任链 | Filter Chain / Interceptor Chain |
| 装饰器 | BeanWrapper |
| 建造者 | BeanDefinitionBuilder / ApplicationContext 构造 |

---

## 面试追问集

**Q：代理模式和装饰器模式的区别？**

- 代理模式：控制访问，通常由框架生成（权限控制、延迟加载）
- 装饰器模式：动态增强功能，使用时显式包裹

**Q：单例模式在系统中的实际应用？**

Spring Bean、数据库连接池、配置管理器、日志工厂、线程池。
