# 垃圾回收（GC）

---

## 一、如何判断对象已死？

### 1.1 引用计数法

给对象添加引用计数器，引用为 0 则回收。但**无法解决循环引用**：

```java
A a = new A();
B b = new B();
a.b = b;
b.a = a;
a = null;
b = null;
// 引用计数仍不为 0，但对象已不可达
```

### 1.2 可达性分析（Java 使用）

通过一系列 **GC Roots** 作为起点，向下搜索引用链，不在链上的对象即可回收。

**GC Roots 有哪些？**

1. 虚拟机栈（栈帧中的局部变量表）中引用的对象
2. 方法区中静态属性引用的对象
3. 方法区中常量引用的对象
4. Native 方法栈中 JNI 引用的对象
5. 所有被 synchronized 持有的对象
6. Java 虚拟机内部的引用（Class 对象、异常对象等）

---

## 二、引用类型 🔥

Java 提供了四种引用类型，从强到弱依次为：**强引用 > 软引用 > 弱引用 > 虚引用**。

### 2.1 四种引用对比

| 类型 | 回收时机 | 是否影响 GC | 使用场景 |
|------|---------|------------|---------|
| 强引用 | 永不回收（除非不可达） | 不会被 GC | 默认，new 对象 |
| 软引用 | 内存不足时回收 | 内存不足才回收 | 缓存（适合图片缓存） |
| 弱引用 | 下次 GC 必定回收 | GC 时立即回收 | ThreadLocal、WeakHashMap |
| 虚引用 | 任何时候都可能 | 无法通过它获取对象 | 管理直接内存、跟踪对象回收 |

### 2.2 强引用（StrongReference）

最常用的引用类型，`Object obj = new Object()` 就是强引用。**只要强引用还在，GC 永远不会回收它**。

```java
Object obj = new Object();
// 只要 obj 还指向这个对象，它就不会被 GC
obj = null;  // 去掉强引用后，对象才会被回收
```

### 2.3 软引用（SoftReference）

**内存足够时不回收，内存不足时回收**。适合做缓存，JVM 在抛出 OOM 前会尝试回收所有软引用。

```java
SoftReference<byte[]> cache = new SoftReference<>(new byte[1024 * 1024 * 10]);

byte[] data = cache.get();
if (data == null) {
    // 已被回收，需要重新加载
    data = loadFromDisk();
    cache = new SoftReference<>(data);
}
```

### 2.4 弱引用（WeakReference）

**只要发生 GC，无论内存是否充足，都会被回收**。生命周期比软引用更短。

```java
WeakReference<String> weakRef = new WeakReference<>(new String("hello"));

System.out.println(weakRef.get());  // hello
System.gc();
System.out.println(weakRef.get());  // null（被回收了）
```

**典型应用：ThreadLocal**

```java
// ThreadLocalMap 的 Entry 继承 WeakReference
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;
    Entry(ThreadLocal<?> k, Object v) {
        super(k);  // Key 是弱引用
        value = v;
    }
}
```

**为什么用弱引用？** 如果 Key 是强引用，即使外部不再使用 ThreadLocal 对象，只要线程存活，ThreadLocal 就无法被 GC。弱引用让 ThreadLocal 对象可以被回收（但留下了 Value 泄漏的隐患，需要手动 `remove()`）。

**另一个典型应用：WeakHashMap**

```java
WeakHashMap<Object, String> map = new WeakHashMap<>();
Object key = new Object();
map.put(key, "value");
key = null;
System.gc();
// key 被回收后，对应的 Entry 也会被自动移除
```

### 2.5 虚引用（PhantomReference）

**最弱的引用，`get()` 永远返回 null**。唯一作用：配合 `ReferenceQueue` 在对象被回收时收到通知。

```java
ReferenceQueue<Object> queue = new ReferenceQueue<>();
PhantomReference<Object> phantom = new PhantomReference<>(new Object(), queue);

// phantom.get() 永远返回 null！

// 当对象被 GC 回收后，虚引用会被加入队列
Reference<?> ref = queue.poll();  // 非阻塞
// 或 Reference<?> ref = queue.remove();  // 阻塞等待
```

**典型应用：管理直接内存（DirectByteBuffer）**

```java
// DirectByteBuffer 内部使用 Cleaner（继承 PhantomReference）
// 当 DirectByteBuffer 对象被 GC 时，Cleaner 被加入队列
// 后台线程从队列中取出 Cleaner，调用 unsafe.freeMemory() 释放堆外内存
```

### 2.6 引用队列（ReferenceQueue）

软引用和弱引用也可以配合 `ReferenceQueue` 使用，在对象被回收后异步清理资源：

```java
ReferenceQueue<Object> queue = new ReferenceQueue<>();

SoftReference<Object> softRef = new SoftReference<>(new Object(), queue);
WeakReference<Object> weakRef = new WeakReference<>(new Object(), queue);

// 当对象被回收后，引用对象会被放入队列
// 可以从队列中取出，做后续清理工作
```

### 2.7 生命周期总结

```
强引用 → 软引用 → 弱引用 → 虚引用
（强）                          （弱）

GC 回收优先级：虚引用 > 弱引用 > 软引用 > 强引用

当对象不可达时，不一定会被立即回收，而是经历以下阶段：
1. 如果还有软引用，等内存不足时回收
2. 如果只有弱引用，下次 GC 回收
3. 如果有虚引用，GC 回收后放入 ReferenceQueue 通知
```

---

## 三、GC 算法

### 3.1 标记-清除（Mark-Sweep）

```
标记：找出所有存活对象
清除：回收未标记的对象
缺点：产生内存碎片、效率低
```

### 3.2 标记-复制（Mark-Copy）

```
将内存分为两块，只使用一块
GC 时将存活对象复制到另一块，然后清空当前块
优点：无碎片、效率高
缺点：内存利用率仅 50%
用于：新生代（Eden + Survivor 的设计）
```

### 3.3 标记-整理（Mark-Compact）

```
标记存活对象 → 将存活对象移到一端 → 清理边界外内存
优点：无碎片
缺点：移动对象需要 STW，耗时较长
用于：老年代（CMS 除外）
```

---

## 四、垃圾收集器 🔥

### 4.1 七种收集器组合

```
新生代                   老年代
Serial      ──────────► Serial Old
ParNew      ──────────► CMS
Parallel Scavenge ────► Parallel Old
G1（逻辑分代，物理不分）
ZGC / Shenandoah（JDK 11+）
```

### 4.2 各收集器对比

| 收集器 | 区域 | 算法 | 线程 | STW | 目标 |
|--------|------|------|------|-----|------|
| Serial | 新生代 | 复制 | 单线程 | 长 | 客户端 |
| ParNew | 新生代 | 复制 | 多线程 | 较长 | 配合 CMS |
| Parallel Scavenge | 新生代 | 复制 | 多线程 | 长 | 吞吐量优先 |
| Serial Old | 老年代 | 整理 | 单线程 | 长 | 客户端 |
| Parallel Old | 老年代 | 整理 | 多线程 | 长 | 吞吐量优先 |
| CMS | 老年代 | 清除 | 多线程 | 短 | 低延迟 |
| G1 | 全部 | 复制+整理 | 多线程 | 可控 | 低延迟 |
| ZGC | 全部 | 染色指针 | 多线程 | <10ms | 超低延迟 |

### 4.3 CMS（Concurrent Mark Sweep）

```
1. 初始标记（STW）       → 标记 GC Roots 直接关联的对象（快）
2. 并发标记              → 从 GC Roots 向下遍历（与用户线程并发，耗时长）
3. 重新标记（STW）       → 修正并发标记期间变化的对象（比初始标记稍长）
4. 并发清除              → 清除垃圾（与用户线程并发）
```

**追问：CMS 有什么缺点？**

1. 并发阶段占用 CPU 资源
2. 无法处理浮动垃圾（并发清除期间新产生的垃圾）
3. 标记-清除算法产生碎片（可开启 `-XX:+UseCMSCompactAtFullCollection`）
4. 对新生代配合的 ParNew 也是需要 STW 的

### 4.4 G1（Garbage First）

```
区域划分：将堆划分为大小相同的 Region（1~32MB），每个 Region 可以是 Eden/Survivor/Old/Humongous

主要阶段：
1. 初始标记（STW）        → 伴随 Minor GC
2. 并发标记（并发）        → 标记存活对象
3. 最终标记（STW）         → 修正并发标记
4. 筛选回收（STW 可控）    → 按用户设定的停顿时间，选择回收收益最高的 Region
```

**追问：G1 相比 CMS 的优势？**

| 对比 | CMS | G1 |
|------|-----|-----|
| 内存布局 | 连续分代 | 分区 Region |
| 碎片 | 有（标记-清除） | 无（复制+整理） |
| 停顿时间 | 不可控 | 可预测（-XX:MaxGCPauseMillis） |
| 巨型对象 | 直接老年代 | 专门 Humongous 区 |

### 4.5 ZGC

- JDK 11 引入，JDK 15 正式生产可用
- 染色指针技术（Colored Pointers）
- 停顿时间 < 10ms，与堆大小无关
- 支持 TB 级堆

---

## 五、三色标记算法

CMS 和 G1 的并发标记阶段使用三色标记：

| 颜色 | 含义 |
|------|------|
| 白色 | 未访问（潜在垃圾） |
| 灰色 | 自身访问了，但子引用未全部访问 |
| 黑色 | 自身和子引用都已访问 |

**追问：漏标问题及解决方案？**

并发标记期间，如果在已标记的黑色对象中新增了对白色对象的引用，且该引用未被扫描：

- **CMS**：增量更新（Incremental Update），黑色对象新增引用时记录，重新标记时再次扫描
- **G1**：原始快照（SATB），标记开始时保存快照，新增引用不影响当前 GC
- **ZGC**：读屏障

---

## 六、Full GC 的触发条件

1. System.gc() 调用（不一定立即执行）
2. 老年代空间不足（Minor GC 前检查）
3. 方法区/元空间不足
4. CMS GC 时，并发模式失败（Concurrent Mode Failure）
5. 空间分配担保失败（老年代放不下晋升对象）

---

## 面试追问集

**Q：为什么 GC 需要 STW（Stop The World）？**

保证可达性分析的准确性。如果用户线程和 GC 线程同时运行，引用关系会不断变化，无法得到一致的分析结果。

**Q：说一下 JVM 默认的垃圾收集器？**

- JDK 8：Parallel Scavenge + Parallel Old
- JDK 9-16：G1
- JDK 17+：G1（大堆场景建议 ZGC）

**Q：什么情况下会产生 Concurrent Mode Failure？**

CMS 并发清除时，老年代即将被填满，剩余空间不足以容纳从新生代晋升的对象。此时 CMS 会退化为 Serial Old 进行 Full GC，停顿时间变长。