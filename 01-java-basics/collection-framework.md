# 集合框架

---

## 一、整体架构

```
                    Collection                         Map
                       │                               │
        ┌──────────────┼──────────────┐          ┌─────┴─────┐
       List           Set            Queue      HashMap   TreeMap
        │              │               │            │
   ArrayList      HashSet      PriorityQueue   LinkedHashMap
   LinkedList     TreeSet       ArrayDeque     Hashtable
   Vector         LinkedHashSet  ConcurrentLinkedQueue  ConcurrentHashMap
   CopyOnWriteArrayList        └──BlockingQueue──┘
```

---

## 二、List

### 2.1 ArrayList vs LinkedList

| 对比维度 | ArrayList | LinkedList |
|---------|-----------|------------|
| 底层结构 | 动态数组 Object[] | 双向链表 |
| 随机访问 | O(1) | O(n) |
| 头部插入 | O(n)（需要移动元素） | O(1) |
| 尾部插入 | O(1) 均摊 | O(1) |
| 中间插入 | O(n) | O(n)（需要遍历定位） |
| 内存占用 | 紧凑（只存数据） | 较大（额外存前后指针） |
| 扩容 | 1.5 倍扩容 | 无扩容概念 |

**追问：ArrayList 扩容机制？**

```java
// JDK 8 源码
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);  // 1.5 倍
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

### 2.2 Vector vs ArrayList

- Vector 线程安全（synchronized 方法），性能较低
- ArrayList 线程不安全，性能更高
- Vector 扩容为 2 倍，ArrayList 扩容为 1.5 倍

---

## 三、Set

| 实现类 | 底层 | 有序性 | 线程安全 |
|--------|------|--------|---------|
| HashSet | HashMap | 无序 | 否 |
| LinkedHashSet | LinkedHashMap | 插入顺序 | 否 |
| TreeSet | TreeMap（红黑树） | 自然排序/比较器 | 否 |
| CopyOnWriteArraySet | CopyOnWriteArrayList | 插入顺序 | 是 |

---

## 四、HashMap 深度解析 🔥

### 4.1 底层数据结构

- **JDK 1.7**：数组 + 链表（头插法）
- **JDK 1.8**：数组 + 链表/红黑树（尾插法），链表长度 > 8 且数组长度 >= 64 时转红黑树

### 4.2 put 流程

```
1. 计算 key 的 hashCode，然后 hash = (h = key.hashCode()) ^ (h >>> 16)
2. (n-1) & hash 定位桶索引
3. 如果桶为空，直接放入
4. 如果桶非空：
   a. 判断第一个节点 key 是否相等 → 相等则替换 value
   b. 判断是否为红黑树节点 → 调用树插入方法
   c. 否则遍历链表 → 找到则替换，找不到则尾插，检查是否要树化
5. ++size，判断是否需要扩容
```

### 4.3 为什么容量必须是 2 的幂？

- `(n-1) & hash` 等价于 `hash % n`，位运算更快
- 减少哈希冲突，使分布更均匀

### 4.4 为什么 hash 需要高低位异或？

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

让高位也参与索引计算，减少碰撞。因为当容量较小时，只用到了 hashCode 的低位。

### 4.5 扩容机制

- 负载因子默认 0.75，当 `size > capacity * loadFactor` 时触发扩容
- 新容量 = 旧容量 × 2
- 1.8 转移时使用高位判断：`(e.hash & oldCap) == 0` 则在原位置，否则在原位置 + oldCap

### 4.6 为什么线程不安全？

1. JDK 1.7 头插法在扩容时可能形成循环链表（死循环）
2. put 时可能丢失数据（多线程同时 put 到同一位置）
3. size++ 非原子操作

### 4.7 为什么链表转红黑树是 8，退化是 6？

- 泊松分布：链表长度达到 8 的概率极低（0.00000006）
- 退化阈值 6 而非 8，是为了避免在 7-8 之间频繁转换

### 4.8 为什么选择红黑树而非 AVL 树？

- 红黑树插入/删除效率更高（旋转次数更少）
- AVL 查询效率略高但插入/删除需要更多旋转，不适合写多场景

---

## 五、ConcurrentHashMap

### 5.1 JDK 1.7 分段锁

- Segment 数组 + HashEntry 链表
- 并发度 = Segment 数量（默认 16）
- put 需要对 Segment 加锁（ReentrantLock）

### 5.2 JDK 1.8 CAS + synchronized

```java
// put 大体流程
1. 计算 hash，定位桶索引
2. 如果桶为空：CAS 尝试放入
3. 如果桶正在扩容：帮助扩容（helpTransfer）
4. 否则：synchronized 锁住桶的头节点
   a. 链表 → 遍历/插入
   b. 红黑树 → 树操作
5. 检查是否需要树化和扩容
```

**追问：1.8 为什么用 synchronized 而非 ReentrantLock？**

- JDK 1.8 对 synchronized 做了大量优化（偏向锁、轻量级锁、锁粗化）
- synchronized 由 JVM 管理，无需手动释放
- 实际场景中锁粒度更细，竞争不激烈时 synchronized 性能更好

---

## 六、LinkedHashMap

- 在 HashMap 基础上加了双向链表维护顺序
- accessOrder = true 时按访问顺序（LRU 缓存）
- accessOrder = false 时按插入顺序（默认）

```java
// 实现 LRU 缓存的简单方式
LinkedHashMap<String, String> lru = new LinkedHashMap<String, String>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > 100;  // 超过 100 个删除最老的
    }
};
```

---

## 七、TreeMap

- 红黑树实现
- Key 必须实现 Comparable 或传入 Comparator
- 查找/插入/删除均为 O(log n)
- 按键自然排序

---

## 面试追问集

**Q：HashMap 和 Hashtable 的区别？**

| 维度 | HashMap | Hashtable |
|------|---------|-----------|
| 线程安全 | 否 | 是（synchronized） |
| null | 允许 null Key/Value | 不允许 |
| 继承 | AbstractMap | Dictionary（过时） |
| 性能 | 高 | 低 |

**Q：有哪些线程安全的 Map？**

- Hashtable：全表锁，性能差
- Collections.synchronizedMap()：包装器，本质仍然是锁整个 Map
- ConcurrentHashMap：分段/CAS+synchronized，高并发首选
