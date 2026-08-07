# 类加载机制

---

## 一、类的生命周期

```
加载 → 验证 → 准备 → 解析 → 初始化 → 使用 → 卸载
│______│     │______│     │______│
   连接（Linking）           类初始化
```

### 1.1 加载（Loading）

- 通过全限定名获取类的二进制字节流
- 将字节流转换为方法区的运行时数据结构
- 在堆中生成 Class 对象

**追问：类加载的几种来源？**

1. 本地 class 文件
2. jar/war 包
3. 网络加载（Applet）
4. 动态代理生成
5. JSP 编译生成的 class
6. 数据库中的字节码

### 1.2 验证（Verification）

- 文件格式验证（魔数、版本号）
- 元数据验证（是否有父类、是否正确继承）
- 字节码验证（数据流、控制流分析）
- 符号引用验证

### 1.3 准备（Preparation）

为静态变量分配内存并设置**零值**（不是用户定义的值）。

```java
static int value = 123;   // 准备阶段 value = 0
static final int CONST = 123;  // 准备阶段 CONST = 123（final 修饰的常量）
```

### 1.4 解析（Resolution）

将常量池中的符号引用替换为直接引用。

### 1.5 初始化（Initialization）

执行类构造器 `<clinit>()` 方法，为静态变量赋初值。

---

## 二、类加载器 🔥

### 2.1 双亲委派模型

```
                 ┌─────────────────┐
                 │ Bootstrap Class  │  加载 rt.jar（%JAVA_HOME%/lib）
                 │    Loader       │  C/C++ 实现，Java 中获取为 null
                 └────────┬────────┘
                          │ 委托
                 ┌────────▼────────┐
                 │ Extension Class  │  加载 ext 目录（%JAVA_HOME%/lib/ext）
                 │    Loader       │  JDK 9+ 被 Platform ClassLoader 替代
                 └────────┬────────┘
                          │ 委托
                 ┌────────▼────────┐
                 │  Application    │  加载 classpath 下的类
                 │  ClassLoader    │
                 └────────┬────────┘
                          │ 委托（父→子）
                 ┌────────▼────────┐
                 │  Custom Class   │  用户自定义
                 │    Loader       │
                 └─────────────────┘
```

### 2.2 双亲委派的工作流程

```java
// ClassLoader.loadClass() 大致逻辑
protected Class<?> loadClass(String name, boolean resolve) {
    synchronized (getClassLoadingLock(name)) {
        // 1. 检查是否已加载
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            // 2. 委托父类加载器加载
            if (parent != null)
                c = parent.loadClass(name, false);
            else
                c = findBootstrapClassOrNull(name);
            // 3. 父类加载不到，自己加载
            if (c == null)
                c = findClass(name);
        }
        return c;
    }
}
```

**优点：**
1. 避免类重复加载（父加载过的子不再加载）
2. 防止核心类被篡改（如 java.lang.Object 始终由 Bootstrap Loader 加载）

### 2.3 打破双亲委派模型

**场景一：JDBC（SPI 机制）**

JDBC 接口在 rt.jar（Bootstrap Loader），但驱动实现由应用加载。Bootstrap Loader 需要反向委派给子加载器加载。JDK 使用 **线程上下文类加载器**（Thread Context ClassLoader）解决。

**场景二：Tomcat 类加载**

```
                 ┌─────────────────┐
                 │ Bootstrap /     │
                 │ System          │
                 └────────┬────────┘
                          │
                 ┌────────▼────────┐
                 │ Common          │  所有应用共享
                 └──┬──────────┬───┘
                    │          │
          ┌─────────▼──┐  ┌───▼─────────┐
          │ Webapp 1   │  │ Webapp 2    │  每个应用独立，隔离
          └────────────┘  └─────────────┘
```

Tomcat 打破双亲委派是为了**隔离不同 Web 应用的类**，同时共享公共类库。

**场景三：OSGi / 热部署**

每个模块独立类加载器，可动态替换。

---

## 三、类加载器的分类（JDK 9+）

```
Bootstrap ClassLoader
        │
Platform ClassLoader（替代 Extension）
        │
Application ClassLoader
```

JDK 9 引入模块化，不再有 ext 目录，Extension ClassLoader 被 Platform ClassLoader 取代。

---

## 面试追问集

**Q：如何自定义类加载器？**

```java
public class MyClassLoader extends ClassLoader {
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] data = loadClassData(name);
        return defineClass(name, data, 0, data.length);
    }

    private byte[] loadClassData(String name) {
        // 自定义字节码获取逻辑（文件、网络、加密解密等）
    }
}
```

重写 `findClass()` 而非 `loadClass()`，保留双亲委派逻辑。

**Q：两个相同全限定名的类可以共存吗？**

可以，但必须由不同的类加载器加载。同一个类加载器 + 相同全限定名的类只会加载一次。

**Q：类加载过程中什么时候触发初始化？**

1. new / getstatic / putstatic / invokestatic 指令
2. 反射调用
3. 初始化子类会先初始化父类
4. main 方法所在类
5. JDK 7+ invokedynamic 指令
6. 接口 default 方法被实现

**Q：常量在编译阶段的处理？**

```java
class A { static final String X = "hello"; }
class B { String y = A.X; }
// B 编译后 = "hello"，不引用 A，不会触发 A 初始化
```
