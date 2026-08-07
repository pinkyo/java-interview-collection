# 线程基础

---

## 一、线程的生命周期

```java
// Java 中线程有 6 种状态（Thread.State 枚举）
NEW              → 线程已创建，但未调用 start()
RUNNABLE         → 可运行（包括运行中 + 等待 CPU 调度）
BLOCKED          → 等待获取锁（进入 synchronized 区域）
WAITING          → 无限等待（wait() / join() / park()）
TIMED_WAITING    → 限时等待（sleep() / wait(timeout) / join(timeout)）
TERMINATED       → 线程执行完毕
```

```
                    ┌──────┐
                    │ NEW  │
                    └──┬───┘
                    start()
                       │
                 ┌─────▼──────┐
          ┌──────┤  RUNNABLE  ├────────┐
          │      └─────┬──────┘        │
          │            │               │
    获取锁成功      等待锁         sleep/join/wait
          │            │            (with timeout)
          │      ┌─────▼──────┐        │
          │      │  BLOCKED   │  ┌─────▼──────────┐
          │      └────────────┘  │ TIMED_WAITING   │
          │                      └────────┬────────┘
          │           等待锁           超时/唤醒
          │           成功                 │
          │            │                  │
          │      ┌─────▼──────┐    ┌──────▼──────┐
          │      │  RUNNABLE  │    │   WAITING   │
          │      └────────────┘    └──────┬───────┘
          │                               │notify/interrupt
          └───────────────────────────────┘
                       │
                 ┌─────▼──────┐
                 │ TERMINATED │
                 └────────────┘
```

---

## 二、创建线程的方式

```java
// 1. 继承 Thread 类
class MyThread extends Thread {
    public void run() { System.out.println("Hello"); }
}
new MyThread().start();

// 2. 实现 Runnable 接口（推荐）
new Thread(() -> System.out.println("Hello")).start();

// 3. 实现 Callable（有返回值）
FutureTask<String> task = new FutureTask<>(() -> "Result");
new Thread(task).start();
String result = task.get();  // 阻塞获取结果

// 4. 线程池（实际开发唯一推荐）
ExecutorService pool = Executors.newFixedThreadPool(5);
pool.execute(() -> System.out.println("Hello"));
```

**追问：Runnable 和 Callable 的区别？**

| 对比 | Runnable | Callable |
|------|----------|----------|
| 返回值 | 无 | 有 |
| 异常 | 不能抛受检异常 | 可以抛异常 |
| 方法 | run() | call() |
| 配合 | Thread / Executor | FutureTask / ExecutorService |

---

## 三、线程常用方法

| 方法 | 说明 |
|------|------|
| start() | 启动线程（只能调用一次） |
| run() | 线程执行体（直接调用不启动新线程） |
| sleep(long) | 静态方法，当前线程休眠，**不释放锁** |
| yield() | 让出 CPU，从运行态回到就绪态 |
| join() | 等待该线程执行完毕 |
| interrupt() | 中断线程（设置中断标志） |
| isInterrupted() | 检查中断标志（不清除） |
| interrupted() | 静态方法，检查中断标志（清除） |

**追问：sleep() 和 wait() 的区别？**

| 对比 | sleep() | wait() |
|------|---------|--------|
| 所属类 | Thread | Object |
| 锁释放 | **不释放锁** | **释放锁** |
| 使用位置 | 任意 | 只能在同步块中 |
| 唤醒方式 | 自动超时 | notify/notifyAll |

**追问：interrupt() 如何工作？**

```java
// 场景 1：打断 sleep/wait/join → 抛 InterruptedException，清除中断标志
Thread t = new Thread(() -> {
    try { Thread.sleep(10000); }
    catch (InterruptedException e) { /* 中断标志被清除 */ }
});
t.interrupt();

// 场景 2：打断正常运行的线程 → 设置中断标志，线程需自行检查
Thread t2 = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        // 业务逻辑
    }
});
```

---

## 四、守护线程（Daemon Thread）

- 当所有非守护线程结束时，守护线程自动退出
- 典型场景：GC 线程、后台日志线程
- 设置方式：`thread.setDaemon(true)`（必须在 start() 前设置）

---

## 面试追问集

**Q：线程和进程的区别？**

| 维度 | 进程 | 线程 |
|------|------|------|
| 定义 | 程序的一次执行，资源分配基本单位 | CPU 调度的基本单位 |
| 资源 | 独立内存空间 | 共享进程内存（私有的栈和 PC） |
| 开销 | 创建/切换开销大 | 创建/切换开销小 |
| 通信 | IPC（管道/共享内存/消息队列） | 共享内存（需同步） |

**Q：什么是上下文切换？**

CPU 从一个线程切换到另一个线程时，需要保存当前线程的状态（PC、寄存器等）并加载下一个线程的状态。频繁上下文切换影响性能。

**Q：如何减少上下文切换？**

- 无锁并发（CAS）
- 协程（如 Kotlin 协程、Project Loom 虚拟线程）
- 合理的线程池大小（CPU 密集型：N+1，IO 密集型：2N）
