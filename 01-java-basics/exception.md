# 异常机制

---

## 一、异常体系

```
                    Throwable
                        │
         ┌──────────────┴──────────────┐
       Error                        Exception
         │                               │
    OutOfMemoryError          ┌──────────┴──────────┐
    StackOverflowError    RuntimeException      CheckedException
    NoClassDefFoundError      │                     │
                          NullPointerException   IOException
                          IndexOutOfBoundsException SQLException
                          ClassCastException      ClassNotFoundException
                          IllegalArgumentException  InterruptedException
```

### 1.1 Error vs Exception

| 对比维度 | Error | Exception |
|---------|-------|-----------|
| 严重性 | 严重，通常无法恢复 | 可捕获处理 |
| 处理方式 | 不应 try-catch | 应当处理 |
| 检查性 | 非受检 | 分受检和非受检 |
| 典型例子 | OOM、SOF | NPE、IOException |

### 1.2 受检异常 vs 非受检异常

| 对比维度 | Checked Exception | Unchecked Exception |
|---------|------------------|-------------------|
| 父类 | Exception（非 RuntimeException） | RuntimeException |
| 编译器检查 | 必须处理（try-catch 或 throws） | 不强制 |
| 代表异常 | IOException、SQLException | NPE、IllegalArgumentException |
| 设计理念 | 可预见的、可恢复的错误 | 程序 Bug、不可预见的错误 |

---

## 二、try-catch-finally 机制

### 2.1 finally 一定会执行吗？

不一定，以下情况 finally 不会执行：
1. 在 try/catch 中调用 `System.exit()`
2. 守护线程中所有非守护线程退出
3. try/catch 中无限循环
4. JVM 崩溃或强制杀死进程

### 2.2 经典面试题：return 和 finally 的执行顺序

```java
// 情况 1：finally 无 return
public static int test() {
    try {
        return 1;
    } finally {
        System.out.println("finally");
    }
}
// 输出 "finally" 然后返回 1
// 原因：执行 return 前会先执行 finally

// 情况 2：finally 有 return
public static int test() {
    try {
        return 1;
    } finally {
        return 2;
    }
}
// 返回 2，finally 的 return 会覆盖 try 的 return

// 情况 3：返回引用类型
public static List<String> test() {
    List<String> list = new ArrayList<>();
    try {
        list.add("try");
        return list;
    } finally {
        list.add("finally");
    }
}
// 返回 ["try", "finally"]
// 原因：return 返回的是引用，finally 修改了引用指向的对象
```

**追问：catch 中 return 后 finally 还执行吗？**

会执行。无论 try 还是 catch 中有 return，finally 都会在方法返回前执行。

---

## 三、try-with-resources（JDK 7+）

```java
// 传统方式
BufferedReader br = null;
try {
    br = new BufferedReader(new FileReader("test.txt"));
    // 使用 br
} catch (IOException e) {
    e.printStackTrace();
} finally {
    if (br != null) {
        try { br.close(); } catch (IOException e) { }
    }
}

// try-with-resources 方式
try (BufferedReader br = new BufferedReader(new FileReader("test.txt"))) {
    // 使用 br
} catch (IOException e) {
    e.printStackTrace();
}
// 资源会自动关闭，需实现 AutoCloseable 接口
```

**追问：多个资源的关闭顺序？**

后声明的先关闭（类似栈，后进先出）。

---

## 四、自定义异常

```java
// 受检异常
public class BusinessException extends Exception {
    private String errorCode;

    public BusinessException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }
}

// 运行时异常
public class ServiceException extends RuntimeException {
    public ServiceException(String message) {
        super(message);
    }
}
```

---

## 面试追问集

**Q：异常处理的最佳实践有哪些？**

1. 不要吞掉异常（空 catch 块）
2. 抛出具体异常，不要 throw Exception
3. 优先使用标准异常
4. 捕获时保持异常链（throw new XxxException(msg, e)）
5. 不要用异常控制业务流程（影响性能）
6. finally 中不要有 return 语句

**Q：NoClassDefFoundError 和 ClassNotFoundException 的区别？**

- **ClassNotFoundException**：运行时动态加载类时找不到（反射、Class.forName）
- **NoClassDefFoundError**：编译时存在，运行时找不到；通常是 jar 版本冲突导致
