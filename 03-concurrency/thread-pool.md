# 线程池

---

## 一、为什么需要线程池？

- **降低资源消耗**：复用线程，减少创建/销毁开销
- **提高响应速度**：线程已创建好，直接使用
- **统一管理线程**：控制最大并发数、监控状态
- **提供附加功能**：定时执行、周期执行

---

## 二、ThreadPoolExecutor 核心参数 🔥

```java
public ThreadPoolExecutor(
    int corePoolSize,        // 核心线程数
    int maximumPoolSize,     // 最大线程数
    long keepAliveTime,      // 非核心线程空闲存活时间
    TimeUnit unit,           // 时间单位
    BlockingQueue<Runnable> workQueue,  // 任务队列
    ThreadFactory threadFactory,        // 线程工厂
    RejectedExecutionHandler handler    // 拒绝策略
)
```

### 2.1 任务提交流程

```
                     提交任务
                        │
                        ▼
              ┌─────────────────┐
              │ 线程数 < coreSize? │──是──→ 创建核心线程执行
              └────────┬────────┘
                       │否
                       ▼
              ┌─────────────────┐
              │ 任务队列满了吗？  │──否──→ 放入任务队列等待
              └────────┬────────┘
                       │满
                       ▼
              ┌─────────────────┐
              │ 线程数 < maxSize? │──是──→ 创建非核心线程执行
              └────────┬────────┘
                       │否
                       ▼
              ┌─────────────────┐
              │   执行拒绝策略    │
              └─────────────────┘
```

### 2.2 拒绝策略

| 策略 | 行为 |
|------|------|
| AbortPolicy（默认） | 抛 RejectedExecutionException |
| CallerRunsPolicy | 由提交任务的线程执行 |
| DiscardPolicy | 直接丢弃（无异常） |
| DiscardOldestPolicy | 丢弃最旧任务，重试提交 |

---

## 三、常见线程池

```java
// 固定大小线程池 → 核心=最大，无界队列（可能 OOM）
Executors.newFixedThreadPool(10);

// 单线程池 → 串行执行任务
Executors.newSingleThreadExecutor();

// 缓存线程池 → 0 核心，无限 max，SynchronousQueue
Executors.newCachedThreadPool();

// 定时任务池
Executors.newScheduledThreadPool(5);
```

**追问：为什么阿里规约不允许使用 Executors 创建线程池？**

1. FixedThreadPool / SingleThreadPool 使用 **无界队列**（LinkedBlockingQueue），可能 OOM
2. CachedThreadPool 使用 **SynchronousQueue + Integer.MAX_VALUE** 作为最大线程数，可能 OOM
3. ScheduledThreadPool 使用 **无界延迟队列**，可能 OOM

**正确方式**：直接使用 `new ThreadPoolExecutor()`，根据业务场景合理配置参数。

---

## 四、线程池参数配置

### CPU 密集型

```
核心线程数 = CPU 核心数 + 1
```

### IO 密集型

```
核心线程数 = CPU 核心数 * 2  （经验值）
或 = CPU 核心数 / (1 - 阻塞系数)  （阻塞系数通常在 0.8~0.9）
```

**更科学的公式（考虑 QPS 和 RT）**：

```
线程数 = QPS × RT (秒) × 冗余系数
例如：QPS=100, RT=0.1s, 冗余系数=1.5
线程数 = 100 × 0.1 × 1.5 = 15
```

---

## 五、线程池状态

```
         RUNNING ──shutdown()──► SHUTDOWN
            │                      │
            │shutdownNow()         │队列为空
            ▼                      ▼
          STOP                    TIDYING
                                    │
                          terminated() 执行完毕
                                    ▼
                                TERMINATED
```

| 状态 | 是否接收新任务 | 是否处理队列任务 | 说明 |
|------|-------------|----------------|------|
| RUNNING | 是 | 是 | 正常运行 |
| SHUTDOWN | 否 | 是 | shutdown() 后进入 |
| STOP | 否 | 否 | shutdownNow() 后进入 |
| TIDYING | 否 | 否（正在过渡） | 所有任务终止 |
| TERMINATED | 否 | 否（已终止） | terminated() 已执行 |

---

## 六、ForkJoinPool

适用于**分而治之**的任务（如大数据量并行计算、归并排序）。

```java
ForkJoinPool pool = new ForkJoinPool();
pool.invoke(new RecursiveTask<Integer>() {
    @Override
    protected Integer compute() {
        // 任务太大？拆分为子任务
        // subTask1.fork();
        // subTask2.fork();
        // return subTask1.join() + subTask2.join();
    }
});
```

- **工作窃取算法**：空闲线程从繁忙线程的任务队列尾部"偷"任务执行
- **双端队列**：线程从头部取自己的任务，其他线程从尾部窃取

---

## 七、CompletableFuture

JDK 8 引入的异步编程利器，相当于增强版的 Future。

```java
// 创建
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> "Hello");

// 链式处理
cf.thenApply(String::toUpperCase)         // 转换结果
  .thenAccept(System.out::println)        // 消费结果（无返回）
  .thenRun(() -> System.out.println("Done")); // 完成后不关心结果

// 组合
CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> "World");
cf1.thenCombine(cf2, (a, b) -> a + " " + b);  // "Hello World"

// 异常处理
cf.exceptionally(e -> "fallback");        // 异常恢复
cf.handle((result, e) -> e == null ? result : "fallback");

// 任意一个完成
CompletableFuture.anyOf(cf1, cf2).thenAccept(System.out::println);

// 全部完成
CompletableFuture.allOf(cf1, cf2).join();
```

**追问：CompletableFuture 的线程池默认是什么？**

不指定线程池时，使用 `ForkJoinPool.commonPool()`（并行度 = CPU 核心数 - 1）。生产环境建议指定自定义线程池。

---

## 面试追问集

**Q：线程池的核心线程可以回收吗？**

默认不会。但调用 `allowCoreThreadTimeOut(true)` 后，核心线程空闲超过 keepAliveTime 也会被回收。

**Q：如何监控线程池？**

```java
pool.getActiveCount();    // 活跃线程数
pool.getPoolSize();       // 当前池大小
pool.getQueue().size();   // 任务队列大小
pool.getCompletedTaskCount(); // 已完成任务数
pool.getTaskCount();      // 总提交任务数
```

**Q：submit() 和 execute() 的区别？**

- execute()：提交 Runnable，无返回值
- submit()：提交 Runnable 或 Callable，返回 Future，可获取结果和异常
