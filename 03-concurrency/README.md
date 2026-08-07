# 并发编程

> 并发编程是 Java 高级开发的核心能力，也是面试重点考察模块。

## 📂 内容索引

| 文件 | 核心知识点 |
|------|-----------|
| [thread-basics.md](./thread-basics.md) | 线程生命周期、创建方式、线程中断、守护线程 |
| [lock-synchronization.md](./lock-synchronization.md) | synchronized 锁升级、volatile、AQS、ReentrantLock、CAS |
| [thread-pool.md](./thread-pool.md) | Executor 框架、核心参数、拒绝策略、ForkJoinPool |
| [juc-collections.md](./juc-collections.md) | ConcurrentHashMap、CopyOnWriteArrayList、BlockingQueue |
| [concurrent-patterns.md](./concurrent-patterns.md) | ThreadLocal、CompletableFuture、并发设计模式 |

## 🔥 高频考点速览

1. **synchronized**：锁升级过程（无锁→偏向锁→轻量级锁→重量级锁）
2. **volatile**：可见性、有序性（内存屏障）、不能保证原子性
3. **AQS**：CLH 队列、state 状态、公平/非公平锁实现
4. **线程池**：7 个核心参数含义、任务提交流程、拒绝策略
5. **ThreadLocal**：内部结构（ThreadLocalMap）、内存泄漏原因与预防
6. **CAS**：原理、ABA 问题、AtomicStampedReference
