# 泛型与注解

---

## 一、泛型（Generics）

### 1.1 什么是泛型？

泛型是 JDK 5 引入的类型参数化机制，将类型作为参数传递，在编译时进行类型检查。

```java
// 不使用泛型（类型不安全）
List list = new ArrayList();
list.add("hello");
list.add(123);
String s = (String) list.get(1);  // ClassCastException

// 使用泛型（编译时类型安全）
List<String> list = new ArrayList<>();
list.add("hello");
// list.add(123);  // 编译错误
String s = list.get(0);   // 无需强转
```

### 1.2 泛型擦除（Type Erasure）

Java 泛型通过**类型擦除**实现：编译后泛型信息被擦除，替换为原始类型（Object 或限定类型）。

```java
List<String> list1 = new ArrayList<>();
List<Integer> list2 = new ArrayList<>();
System.out.println(list1.getClass() == list2.getClass()); // true
// 运行时都是 List.class
```

**追问：泛型擦除后，如何获取泛型信息？**

通过反射获取`签名（Signature）`属性：

```java
public class GenericClass<T> {
    // 1. 子类指定具体类型
    public static class SubClass extends GenericClass<String> {}
    // 可以获取到 String

    // 2. 字段的泛型
    List<String> list; // 可以获取到 String

    // 3. 方法的参数/返回值泛型
    public <T> T getValue() { return null; }
}
```

**追问：泛型擦除有哪些影响？**

1. 不能使用基本类型作为参数（如 `List<int>` ❌）
2. 不能实例化泛型类型（`new T()` ❌）
3. 不能创建泛型数组（`new T[]` ❌）
4. 不能用 instanceof 检查泛型类型（`obj instanceof List<String>` ❌）
5. 不能重载相同原始类型的方法

```java
// 编译错误：方法签名冲突
public void print(List<String> list) { }
public void print(List<Integer> list) { }  // 擦除后相同
```

### 1.3 通配符

```java
// ? 无限定通配符：只读，不能添加（null 除外）
List<?> list = new ArrayList<String>();
// list.add("hello");  // 编译错误

// ? extends T 上界通配符：生产者（只能取，不能存）
List<? extends Number> numbers = new ArrayList<Integer>();
Number n = numbers.get(0);      // OK，get 返回 Number
// numbers.add(1);               // 编译错误，不知道具体类型

// ? super T 下界通配符：消费者（只能存，取出来是 Object）
List<? super Integer> integers = new ArrayList<Number>();
integers.add(1);                // OK，可以添加 Integer 及其子类
Object obj = integers.get(0);   // get 只能返回 Object
```

**PECS 原则**：Producer Extends, Consumer Super。

### 1.4 泛型方法

```java
// 泛型方法：类型参数在返回类型前声明
public static <T> T getFirst(List<T> list) {
    return list.get(0);
}

// 多类型参数
public static <K, V> Map<K, V> of(K k, V v) {
    return new HashMap<>();
}
```

---

## 二、注解（Annotation）

### 2.1 元注解

| 注解 | 说明 |
|------|------|
| @Target | 注解使用范围（TYPE/METHOD/FIELD 等） |
| @Retention | 注解生命周期（SOURCE/CLASS/RUNTIME） |
| @Documented | 是否包含在 JavaDoc 中 |
| @Inherited | 是否被子类继承 |
| @Repeatable | 是否可重复使用（JDK 8+） |

### 2.2 Retention 三种策略

| 策略 | 保留位置 | 使用场景 |
|------|---------|---------|
| SOURCE | 源码，编译时丢弃 | @Override、@SuppressWarnings |
| CLASS | class 文件，运行时不可获取 | Lombok（通过 APT 处理） |
| RUNTIME | 运行时保留 | Spring 注解、自定义运行时注解 |

### 2.3 自定义注解

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Log {
    String value() default "";
    boolean recordParams() default true;
    boolean recordResult() default true;
}

// 使用
@Log(value = "用户登录", recordParams = true)
public void login(String username) { }
```

### 2.4 注解处理器

```java
// 运行时获取注解
Method method = clazz.getMethod("login", String.class);
if (method.isAnnotationPresent(Log.class)) {
    Log log = method.getAnnotation(Log.class);
    System.out.println(log.value());
}
```

### 2.5 组合注解 vs 继承

```java
// @SpringBootApplication 就是组合注解
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(...)
public @interface SpringBootApplication { }
```

组合是 Java 注解的设计哲学（@Inherited 不常用）。

### 2.6 常见面试问题

**Q：@Autowired 和 @Resource 的区别？**

| 维度 | @Autowired | @Resource |
|------|-----------|-----------|
| 来源 | Spring | JSR-250（JDK 标准） |
| 注入方式 | 默认 byType | 默认 byName |
| 配合 | @Qualifier | name 属性 |
| 修饰 | 构造器/方法/字段/参数 | 类/方法/字段 |

**Q：@ConfigurationProperties 和 @Value 的区别？**

- @ConfigurationProperties：批量注入，适合一组配置，支持校验
- @Value：单个注入，支持 SpEL 表达式，不能批量绑定
