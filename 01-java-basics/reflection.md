# 反射机制

---

## 一、什么是反射？

反射是在运行时动态获取类信息并操作类属性的机制。通过反射可以在运行时构造对象、调用方法、访问字段。

```java
// 核心入口
Class<?> clazz = Class.forName("com.example.User");

// 创建实例
Object obj = clazz.getDeclaredConstructor().newInstance();

// 调用方法
Method method = clazz.getDeclaredMethod("sayHello", String.class);
method.setAccessible(true);            // 绕过访问权限
method.invoke(obj, "World");

// 访问字段
Field field = clazz.getDeclaredField("name");
field.setAccessible(true);
field.set(obj, "张三");
```

---

## 二、获取 Class 对象的三种方式

```java
// 1. Class.forName（需要全限定名，会触发类初始化）
Class<?> clazz1 = Class.forName("java.lang.String");

// 2. 类名.class（安全，不触发初始化）
Class<?> clazz2 = String.class;

// 3. 对象.getClass()
String s = "hello";
Class<?> clazz3 = s.getClass();
```

**追问：三种方式的区别？**

| 方式 | 是否触发初始化 | 使用场景 |
|------|-------------|---------|
| Class.forName() | 是（执行静态代码块） | 加载数据库驱动等需要初始化的场景 |
| 类名.class | 否 | 获取已知类的 Class 对象 |
| obj.getClass() | 已初始化 | 运行时获取对象类型 |

---

## 三、反射常用 API

```java
Class<?> clazz = User.class;

// 获取所有构造方法
Constructor<?>[] constructors = clazz.getConstructors();          // 所有 public
Constructor<?>[] declared = clazz.getDeclaredConstructors();      // 所有（含 private）

// 获取方法
Method[] methods = clazz.getMethods();                            // public + 继承的
Method[] declaredMethods = clazz.getDeclaredMethods();            // 所有（不含继承）

// 获取字段
Field[] fields = clazz.getFields();                               // public + 继承的
Field[] declaredFields = clazz.getDeclaredFields();               // 所有（不含继承）

// 获取注解
Annotation[] annotations = clazz.getAnnotations();
```

---

## 四、动态代理

### 4.1 JDK 动态代理

```java
// 1. 定义接口
public interface UserService {
    void addUser(String name);
}

// 2. 实现类
public class UserServiceImpl implements UserService {
    public void addUser(String name) {
        System.out.println("添加用户: " + name);
    }
}

// 3. InvocationHandler
public class LogInvocationHandler implements InvocationHandler {
    private Object target;

    public LogInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before: " + method.getName());
        Object result = method.invoke(target, args);
        System.out.println("after: " + method.getName());
        return result;
    }
}

// 4. 生成代理对象
UserService proxy = (UserService) Proxy.newProxyInstance(
    target.getClass().getClassLoader(),
    target.getClass().getInterfaces(),
    new LogInvocationHandler(target)
);
```

**追问：JDK 动态代理为什么必须基于接口？**

生成的代理类 `$Proxy0` 已经继承了 `Proxy` 类，而 Java 是单继承，所以只能通过实现接口的方式来代理。

### 4.2 CGLIB 动态代理

```java
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(UseServiceImpl.class);  // 直接代理类
enhancer.setCallback(new MethodInterceptor() {
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("before");
        Object result = proxy.invokeSuper(obj, args);
        System.out.println("after");
        return result;
    }
});
UserService proxy = (UserService) enhancer.create();
```

CGLIB 通过继承目标类生成子类来做代理，因此不能代理 final 类和方法。

### 4.3 JDK 动态代理 vs CGLIB

| 对比维度 | JDK 动态代理 | CGLIB |
|---------|------------|-------|
| 原理 | 实现接口 | 生成子类（ASM 字节码） |
| 要求 | 必须有接口 | 不能是 final 类 |
| 性能 | 创建快、调用慢 | 创建慢、调用快（JDK 8+ 差距缩小） |
| Spring 默认 | 有接口时使用 | 无接口时使用（Spring Boot 2.x 默认 CGLIB） |

---

## 五、反射的优缺点

**优点：**
- 运行时动态操作，灵活性强
- 框架开发的基石（Spring、MyBatis 等）

**缺点：**
- 性能较低（查找方法、安全检查等开销）
- 破坏封装性（绕过访问控制）
- 代码可读性降低

---

## 六、SPI 机制

SPI（Service Provider Interface）是 JDK 内置的服务发现机制。

```java
// META-INF/services/com.example.Driver 文件内容：
com.example.MysqlDriver
com.example.OracleDriver

// 加载所有实现
ServiceLoader<Driver> drivers = ServiceLoader.load(Driver.class);
for (Driver driver : drivers) {
    driver.connect();
}
```

**追问：SPI 有什么缺点？**

1. 不能按需加载，会加载所有实现
2. 不支持 AOP 或依赖注入
3. 线程不安全

Dubbo 对 SPI 做了增强，支持按名称获取和 IOC/AOP。
