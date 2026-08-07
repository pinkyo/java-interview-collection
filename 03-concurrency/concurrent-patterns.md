# 并发编程模式与最佳实践

---

## 一、并发设计模式

### 1.1 不可变模式

通过 final 和私有构造器 + 工厂方法实现不可变对象：

```java
public final class Person {
    private final String name;
    private final int age;

    private Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public static Person of(String name, int age) {
        return new Person(name, age);
    }

    // 只有 getter，没有 setter
}
```

典型的不可变类：String、Integer、BigDecimal。

### 1.2 保护性暂停（Guarded Suspension）

一个线程等待另一个线程的结果：

```java
// 使用 CompletableFuture 实现
CompletableFuture<String> future = new CompletableFuture<>();
// 线程1：等待结果
String result = future.get();
// 线程2：设置结果
future.complete("done");
```

### 1.3 Balking 模式

如果不需要执行就放弃（类似 if 判断）：

```java
volatile boolean changed = false;
if (!changed) return;  // balk
```

### 1.4 生产者-消费者

```java
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(100);
// 生产者
queue.put(task);
// 消费者
Task task = queue.take();
```

### 1.5 读写锁模式

```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();
// 读锁：可以被多个线程同时持有
rwLock.readLock().lock();
// 写锁：独占
rwLock.writeLock().lock();
```

**追问：读写锁的锁降级？**

持有写锁时可以获取读锁，释放写锁后变成读锁（写 → 读降级）。**不支持锁升级**（读 → 写），可能导致死锁。

---

## 二、死锁

### 2.1 死锁的四个必要条件

1. **互斥**：资源只能被一个线程占用
2. **占有且等待**：持有资源的同时等待其他资源
3. **不可抢占**：资源不能被强制释放
4. **循环等待**：形成等待环路

### 2.2 死锁示例

```java
Object lockA = new Object();
Object lockB = new Object();

// 线程1
synchronized(lockA) {
    Thread.sleep(100);
    synchronized(lockB) { /* ... */ }
}

// 线程2
synchronized(lockB) {
    Thread.sleep(100);
    synchronized(lockA) { /* ... */ }
}
// → 死锁！
```

### 2.3 死锁排查

```bash
# 1. jstack 查看死锁
jstack pid | grep -A 20 "deadlock"

# 2. Arthas
thread -b   # 一键查找死锁
```

### 2.4 死锁预防

1. **固定加锁顺序**：所有线程按相同顺序获取锁
2. **超时放弃**：tryLock(timeout)
3. **死锁检测**：定期检测线程状态，发现死锁后释放资源重试

---

## 三、Happens-Before 规则

JMM 规定了 8 条 Happens-Before 规则：

| 规则 | 说明 |
|------|------|
| 程序次序规则 | 同一线程内，前面的操作 HB 后面的操作 |
| 管程锁定规则 | unlock HB 后续的 lock |
| volatile 规则 | volatile 写 HB 后续 volatile 读 |
| 线程启动规则 | Thread.start() HB 线程内的所有操作 |
| 线程终止规则 | 线程所有操作 HB Thread.join() 返回 |
| 线程中断规则 | interrupt() HB 被中断线程检测到中断 |
| 对象终结规则 | 对象初始化 HB finalize() |
| 传递性 | A HB B, B HB C → A HB C |

---

## 四、并发编程最佳实践

1. **使用局部变量而非共享变量**（栈封闭）
2. **尽量使用不可变对象**
3. **优先使用并发容器（ConcurrentHashMap）而非同步包装器**
4. **锁的范围尽量小**（只锁临界区）
5. **优先使用 CAS 无锁算法**，再考虑 Lock，最后考虑 synchronized
6. **ThreadLocal 必须 finally remove**
7. **线程池须手动创建**，不要用 Executors
8. **优先使用 CompletableFuture 做异步编排**
9. **流水线/阶段式异步处理优于串行阻塞**
10. **分布式场景使用分布式锁**，本地锁只保护本地资源

---

## 面试追问集

**Q：什么是活锁？和死锁有什么区别？**

- 死锁：互相等待对方释放资源，永不进展
- 活锁：线程在不断尝试操作但始终失败（如两个人在走廊互相让路），线程没有阻塞但无法继续

**Q：什么是饥饿？**

高优先级线程一直占用资源，低优先级线程永远得不到执行。非公平锁可能导致饥饿。

**Q：Java 内存模型（JMM）是什么？**

JMM 定义了 Java 线程和主内存之间数据交互的规范，规定了变量的访问规则。核心内容：主内存、工作内存、内存间交互的 8 种原子操作、Happens-Before 规则、volatile/synchronized/final 的特殊规则。
