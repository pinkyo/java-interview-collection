# JUC 并发容器与并发模式

---

## 一、JUC 并发容器

### 1.1 ConcurrentHashMap（详见集合框架）

### 1.2 CopyOnWriteArrayList

**思想**：写时复制，适用于**读多写少**的场景。

```java
// add 方法源码（简化）
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1); // 复制新数组
        newElements[len] = e;
        setArray(newElements);  // 替换整个数组
        return true;
    } finally {
        lock.unlock();
    }
}
```

**优点**：读不需要加锁，读操作高性能。

**缺点**：写操作复制整个数组（OOM 风险）、**数据实时性不保证**（读到旧数据）。

### 1.3 BlockingQueue

| 实现 | 特点 |
|------|------|
| ArrayBlockingQueue | 有界，数组实现，一把锁 |
| LinkedBlockingQueue | 可指定容量（默认 Integer.MAX_VALUE），链表，两把锁（putLock/takeLock） |
| SynchronousQueue | 容量为 0，必须有线程等待 take 才能 put |
| PriorityBlockingQueue | 无界，按优先级出队 |
| DelayQueue | 无界，延迟出队（用于定时任务、超时订单） |
| LinkedTransferQueue | 无界，支持 transfer 直接移交 |

### 1.4 ConcurrentLinkedQueue

- 无界非阻塞队列，基于 CAS 无锁实现
- 适用于高并发、不需要阻塞的场景

---

## 二、ThreadLocal 🔥

### 2.1 原理

```java
// 每个 Thread 内部维护一个 ThreadLocalMap
Thread.threadLocals = ThreadLocal.ThreadLocalMap;

// ThreadLocalMap 是一个自定义的哈希表
// Key：ThreadLocal 对象的弱引用
// Value：存储的值

public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);  // this = ThreadLocal 对象
    else
        createMap(t, value);
}

public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) return (T) e.value;
    }
    return setInitialValue();
}
```

### 2.2 内存泄漏问题 🔥

```
ThreadLocal ←──强引用── 代码
     │
     ├───弱引用───→ Entry(Key) ──强引用──→ Value
     │                 ↑                    ↑
     └─────────────────┘                    │
                                     如果 Key 被 GC，Value 无法访问但也不会被回收
                                     （除非 Thread 结束 或 手动 remove）
```

**原因**：Entry 的 Key 是弱引用，GC 后 Key 为 null，但 Value 还被 Entry 强引用，只要线程存活就不会回收。

**追问：为什么使用弱引用？**

如果 Key 是强引用，ThreadLocal 对象无法被 GC。弱引用让 ThreadLocal 可以被回收，但留下了 Value 泄漏的隐患。

**解决方案**：`threadLocal.remove()`

```java
try {
    threadLocal.set(value);
    // 使用 value
} finally {
    threadLocal.remove();  // 必须手动清理！
}
```

另外，ThreadLocalMap 的 set/get/remove 方法中有**探测式清理**（碰到 null key 就清理 Value）。

### 2.3 使用场景

- Spring 事务管理（同一个 Connection）
- 用户上下文信息传递（拦截器 → Service）
- 链路追踪 TraceId
- 避免参数层层传递

### 2.4 InheritableThreadLocal

子线程可以继承父线程的 ThreadLocal 值：

```java
InheritableThreadLocal<String> itl = new InheritableThreadLocal<>();
// 子线程 new Thread() 时会复制父线程的 inheritableThreadLocals
```

**追问：线程池下的 InheritableThreadLocal 问题？**

线程复用导致值不会每次都重新复制。阿里开源了 **TransmittableThreadLocal** 解决线程池下的上下文传递。

---

## 三、并发工具类

### 3.1 CountDownLatch

```java
CountDownLatch latch = new CountDownLatch(3);
// 多个线程 countDown()
latch.countDown();
// 主线程等待
latch.await();
// 一次性，不能重置
```

### 3.2 CyclicBarrier

```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("所有线程到达"));
// 每个线程
barrier.await();
// 等到所有线程都到达 → 执行回调 → 自动重置 → 可重复使用
```

### 3.3 Semaphore

```java
Semaphore semaphore = new Semaphore(5); // 5 个许可
semaphore.acquire();  // 获取许可（-1）
// 临界区
semaphore.release();  // 释放许可（+1）
// 常用于：限流、连接池控制
```

### 3.4 对比

| 工具 | 核心作用 | 可重用 |
|------|---------|--------|
| CountDownLatch | N 等 1，等待其他线程完成 | 否 |
| CyclicBarrier | N 个相互等待，齐步走 | 是 |
| Semaphore | 控制并发访问数量 | 是 |

### 3.5 Exchanger

两个线程交换数据：

```java
Exchanger<String> exchanger = new Exchanger<>();
// 线程1
String dataB = exchanger.exchange("Data A");
// 线程2
String dataA = exchanger.exchange("Data B");
```

---

## 四、原子类

```java
AtomicInteger        // int 原子操作
AtomicLong           // long 原子操作
AtomicBoolean        // boolean 原子操作
AtomicReference      // 对象引用原子操作
AtomicStampedReference // 带版本号（解决 ABA）
LongAdder            // 高并发下性能优于 AtomicLong（分段累加）
LongAccumulator      // 更灵活的累加器
```

**追问：LongAdder 为什么在高并发下比 AtomicLong 性能好？**

AtomicLong 使用 CAS 自旋，高并发时大量线程自旋浪费 CPU。LongAdder 将值分散到多个 Cell（分段），最后 sum 汇总，减少竞争。

---

## 面试追问集

**Q：HashMap 为什么线程不安全，怎样变成线程安全的？**

- ConcurrentHashMap（推荐）
- Collections.synchronizedMap()
- Hashtable（不推荐）

**Q：ArrayList 有什么线程安全的替代？**

- CopyOnWriteArrayList（读多写少）
- Collections.synchronizedList()
- Vector（不推荐）
