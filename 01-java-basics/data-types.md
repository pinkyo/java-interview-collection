# 数据类型与运算符

---

## 一、基本数据类型与包装类

### 1.1 Java 有哪几种基本数据类型？

| 类型 | 大小（字节） | 默认值 | 取值范围 | 包装类 |
|------|-------------|--------|---------|--------|
| byte | 1 | 0 | -128 ~ 127 | Byte |
| short | 2 | 0 | -2^15 ~ 2^15-1 | Short |
| int | 4 | 0 | -2^31 ~ 2^31-1 | Integer |
| long | 8 | 0L | -2^63 ~ 2^63-1 | Long |
| float | 4 | 0.0f | IEEE 754 | Float |
| double | 8 | 0.0d | IEEE 754 | Double |
| char | 2 | '\u0000' | 0 ~ 65535 | Character |
| boolean | JVM 相关 | false | true / false | Boolean |

### 1.2 基本类型和包装类有什么区别？

| 对比维度 | 基本类型 | 包装类 |
|---------|---------|--------|
| 存储位置 | 栈（局部变量） | 堆 |
| 默认值 | 0 / false 等 | null |
| 泛型支持 | 不支持 | 支持 |
| 性能 | 高 | 较低（装箱拆箱开销） |
| 用途 | 数值计算 | 集合容器、泛型 |

### 1.3 装箱和拆箱是如何实现的？

```java
Integer a = 100;         // 自动装箱 → Integer.valueOf(100)
int b = a;               // 自动拆箱 → a.intValue()
Integer c = 200;
Integer d = 200;
System.out.println(a == b);   // true，拆箱后比较值
System.out.println(c == d);   // false，> 127 不在缓存范围
```

**追问：Integer 缓存机制是什么？**

Integer 默认缓存 `-128 ~ 127` 的实例（可通过 `-XX:AutoBoxCacheMax` 调整上限）。

```java
// Integer.valueOf() 源码
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

类似地，Byte/Short/Long 缓存 -128~127，Character 缓存 0~127，Boolean 缓存 TRUE/FALSE。

---

## 二、String 家族

### 2.1 String、StringBuilder、StringBuffer 有什么区别？

| 对比维度 | String | StringBuilder | StringBuffer |
|---------|--------|--------------|--------------|
| 可变性 | 不可变（final 类） | 可变 | 可变 |
| 线程安全 | 安全（不可变） | 不安全 | 安全（synchronized） |
| 性能 | 低（频繁拼接产生新对象） | 高 | 中 |
| 适用场景 | 少量字符串操作 | 单线程大量拼接 | 多线程大量拼接 |

### 2.2 String 为什么设计为不可变？

1. **字符串常量池**：不可变才能安全共享
2. **Hash 缓存**：String 常用作 HashMap 的 Key，不可变则 hashCode 固定
3. **线程安全**：不可变对象天然线程安全
4. **类加载器**：String 作为参数传递时不可变保证安全性

```java
// String 底层使用 private final char value[] 存储（JDK 9 后为 byte[]）
// final 修饰数组只能保证引用不变，但数组内容可变
// 真正不可变的保证是：类为 final + 没有提供修改数组的方法
```

### 2.3 字符串常量池在哪里？

- **JDK 6 及之前**：放在永久代（PermGen）
- **JDK 7**：移到堆中（因为永久代空间有限，容易 OOM）
- **JDK 8**：仍在堆中（元空间替代永久代）

### 2.4 `String s = new String("abc")` 创建了几个对象？

- 如果常量池中已有 `"abc"`：创建 **1 个**对象（堆中的 String 对象）
- 如果常量池中没有 `"abc"`：创建 **2 个**对象（常量池中的 "abc" + 堆中的 String 对象）

### 2.5 intern() 方法的作用

```java
String s1 = new String("a") + new String("b");  // 堆中 "ab"，常量池没有
String s2 = s1.intern();                         // 将 s1 引用放入常量池（JDK7+）
System.out.println(s1 == s2);                    // JDK7+: true, JDK6: false
```

JDK 7+ 中，intern() 如果常量池没有，会将该字符串对象的**引用**放入常量池，而非复制一份。

---

## 三、位运算（常见但不常用）

```java
// 常用位运算
int a = 5;      // 0101
int b = 3;      // 0011
a & b;          // 0001 = 1   按位与
a | b;          // 0111 = 7   按位或
a ^ b;          // 0110 = 6   按位异或
~a;             // 1010 = -6  按位取反
a << 1;         // 1010 = 10  左移（每移一位乘2）
a >> 1;         // 0010 = 2   右移（每移一位除2，保留符号位）
a >>> 1;        // 0010 = 2   无符号右移（高位补0）
```

**应用场景**：HashMap 中计算容量为 2 的幂、权限控制（位标志）、状态判断。

---

## 面试追问：常见陷阱题

**Q：以下代码输出什么？**

```java
Integer a = 100, b = 100, c = 200, d = 200;
System.out.println(a == b);   // true（缓存）
System.out.println(c == d);   // false（超出缓存范围）

Float e = 1.0f, f = 1.0f;
System.out.println(e == f);   // false（Float 没有缓存）
```
