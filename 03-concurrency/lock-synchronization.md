# 锁与同步

---

## 一、synchronized 🔥

### 1.1 三种使用方式

```java
// 1. 修饰实例方法 → 锁当前对象
public synchronized void method() { }

// 2. 修饰静态方法 → 锁 Class 对象
public static synchronized void staticMethod() { }

// 3. 修饰代码块
synchronized(obj) { }      // 锁指定对象
synchronized(MyClass.class) { }  // 锁 Class 对象
```

### 1.2 底层原理

```java
// 代码块 → monitorenter + monitorexit 指令
synchronized(obj) {
    // 编译为：
    // monitorenter
    // ... 代码 ...
    // monitorexit
}

// 方法 → ACC_SYNCHRONIZED 标志
// JVM 通过方法标志识别，自动加锁
```

每个对象关联一个 Monitor，线程通过争抢 Monitor 所有权来获得锁。

**追问：Monitor 对象的结构？**

```cpp
ObjectMonitor() {
    _header       = NULL;   // Mark Word
    _count        = 0;      // 重入计数
    _waiters      = 0;      // 等待线程数
    _recursions   = 0;      // 重入次数
    _owner        = NULL;   // 持有锁的线程
    _WaitSet      = NULL;   // wait() 的线程队列
    _EntryList    = NULL;   // 等待获取锁的线程队列
}
```

### 1.3 锁升级过程 🔥

```
无锁 → 偏向锁 → 轻量级锁 → 重量级锁
                  ↑_____________|
                   (不可降级，只能升级)
```

**偏向锁：**
- 适用于**只有一个线程**访问同步块
- 在 Mark Word 记录线程 ID，该线程再次进入时无需 CAS 加锁
- 当另一线程竞争时，撤销偏向锁 → 升级为轻量级锁

**轻量级锁：**
- 适用于**交替执行**（无竞争，但也不能长时间持有）
- 在线程栈帧中创建 Lock Record，通过 CAS 将 Mark Word 替换为 Lock Record 指针
- 竞争时自旋获取，自旋失败则升级

**重量级锁：**
- 依赖操作系统的互斥量（Mutex）
- 线程阻塞挂起，涉及系统调用和上下文切换
- 适合高竞争场景

**追问：为什么要引入锁升级？**

因为大多数场景下：
- 同步块只有一个线程访问（偏向锁够用）
- 即使有竞争，也很快释放（轻量级自旋即可）
- 真正高竞争才需要重量级锁

不加区分直接使用重量级锁会导致不必要的性能开销。

### 1.4 锁的优化

| 优化技术 | 说明 |
|---------|------|
| 锁粗化 | 多个连续加锁合并为一个大锁 |
| 锁消除 | JIT 分析确定不需要锁时去掉 |
| 自适应自旋 | 根据历史自旋成功/失败动态调整自旋次数 |

---

## 二、volatile

### 2.1 三大特性

```java
volatile int flag = 0;

// 1. 可见性：一个线程修改，其他线程立即可见
// 2. 有序性：禁止指令重排序（内存屏障）
// 3. 不保证原子性：flag++ 不是原子操作
```

### 2.2 内存屏障（Memory Barrier）

| 屏障类型 | 说明 |
|---------|------|
| LoadLoad | Load1; LoadLoad; Load2 → 保证 Load1 在 Load2 前执行 |
| StoreStore | Store1; StoreStore; Store2 → 保证 Store1 在 Store2 前执行 |
| LoadStore | Load1; LoadStore; Store2 → 保证 Load1 在 Store2 前执行 |
| StoreLoad | Store1; StoreLoad; Load2 → 保证 Store1 在 Load2 前执行（最重量级） |

### 2.3 DCL（双重检查锁定）为什么需要 volatile？

```java
public class Singleton {
    private static volatile Singleton instance;  // 必须 volatile！

    public static Singleton getInstance() {
        if (instance == null) {                    // 第一次检查
            synchronized (Singleton.class) {
                if (instance == null) {            // 第二次检查
                    instance = new Singleton();    // 不是原子操作！
                    // 1. 分配内存
                    // 2. 初始化对象
                    // 3. instance 指向内存地址
                    // 步骤 2 和 3 可能被重排序！
                }
            }
        }
        return instance;
    }
}
```

如果没有 volatile，线程 A 执行时步骤 3 在步骤 2 前执行，线程 B 读到 instance != null 但对象未初始化，导致使用了不完整的对象。

---

## 三、AQS（AbstractQueuedSynchronizer）🔥

### 3.1 核心思想

```java
// AQS 维护一个 volatile int state + FIFO 等待队列（CLH 变体）
private volatile int state;

// 需要子类实现的方法：
tryAcquire(int)    // 独占获取
tryRelease(int)    // 独占释放
tryAcquireShared(int)  // 共享获取
tryReleaseShared(int)  // 共享释放
isHeldExclusively()    // 是否独占
```

### 3.2 AQS 的工作流程

```
1. 线程尝试 CAS 修改 state 获取锁
2. 获取失败 → 加入 CLH 队列尾部（CAS 入队）
3. 前驱节点是头节点 → 再次尝试获取锁
4. 获取失败 → 挂起（park），等待前驱唤醒（unpark）
5. 释放锁 → state 归零 → 唤醒后继节点
```

### 3.3 基于 AQS 的常见实现

| 类 | 锁类型 | state 含义 |
|----|--------|-----------|
| ReentrantLock | 独占可重入 | 0 未锁，1 锁定，>1 重入次数 |
| CountDownLatch | 共享 | 0 可以执行，>0 需要等待 |
| Semaphore | 共享 | 剩余许可数 |
| ReentrantReadWriteLock | 写独占读共享 | 高 16 位读锁，低 16 位写锁 |
| CyclicBarrier | 重用 Barrier | 等待线程数 |

---

## 四、ReentrantLock 🔥

### 4.1 synchronized 和 ReentrantLock 的区别

| 对比 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 实现 | JVM 层面（Monitor） | JDK 层面（AQS + CAS） |
| 锁释放 | 自动释放（代码块结束/异常） | 必须在 finally 中 unlock() |
| 锁类型 | 非公平（默认） | 可公平也可非公平 |
| 可中断 | 不可中断 | lockInterruptibly() |
| 超时获取 | 不支持 | tryLock(timeout, unit) |
| 条件变量 | 一个条件队列（wait/notify） | 多个 Condition |
| 性能 | JDK 6+ 差距不大 | 相差不大 |

**追问：什么时候使用 ReentrantLock？**

1. 需要可中断的锁获取
2. 需要超时获取锁
3. 需要公平锁
4. 需要多个条件变量

### 4.2 公平锁 vs 非公平锁

```java
// 公平锁
ReentrantLock fairLock = new ReentrantLock(true);

// 非公平锁（默认，吞吐量更高）
ReentrantLock unfairLock = new ReentrantLock(false);
```

| 对比 | 公平锁 | 非公平锁 |
|------|--------|---------|
| 获取顺序 | FIFO | 抢占式 |
| 上下文切换 | 多 | 少 |
| 吞吐量 | 低 | 高 |
| 饥饿 | 不会 | 可能 |

---

## 五、CAS（Compare And Swap）

```java
// CAS 的三个操作数：内存地址 V、期望值 A、新值 B
// 如果 V == A，则将 V 改为 B，返回 true；否则返回 false
// CPU 层面由 cmpxchg 指令保证原子性
```

**追问：CAS 的 ABA 问题及解决？**

```
线程1：A → 修改中...
线程2：A → B → A（已经改回来了）
线程1：CAS 发现还是 A，修改成功（但中间已经被改过了！）
```

解决：**AtomicStampedReference**（加版本号）

```java
AtomicStampedReference<Integer> ref = new AtomicStampedReference<>(100, 0);
ref.compareAndSet(100, 200, 0, 1);  // 期望值 + 期望版本号 + 新值 + 新版本号
```

---

## 六、synchronized vs Lock 底层总结

```
JVM 监视器 → ObjectMonitor → 操作系统的 Mutex Lock（重量级）

JDK 锁    → AQS + CAS + LockSupport.park/unpark（更灵活）

锁升级    → 偏向锁（无竞争）→ 轻量级锁（交替执行）→ 重量级锁（高竞争）
              ↑ 基于 Mark Word + 自旋 + CAS，无需内核态切换
```

---

## 面试追问集

**Q：ReentrantLock 如何实现可重入？**

AQS 的 state 记录重入次数，每次 lock() state+1，unlock() state-1，state=0 时释放锁。

**Q：Condition 的 await/signal 与 Object 的 wait/notify 有什么区别？**

- Condition 可以创建多个，可以实现精确唤醒
- wait/notify 只有一个等待队列，notify 随机唤醒
