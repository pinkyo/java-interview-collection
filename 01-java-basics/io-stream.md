# IO 流与 NIO

---

## 一、Java IO 体系概览

```
                      InputStream                 OutputStream
                          │                           │
        ┌─────────────────┼──────────────┐
   FileInputStream  FilterInputStream  ByteArrayInputStream
                          │
                   BufferedInputStream
                   DataInputStream

                      Reader                       Writer
                          │                           │
        ┌─────────────────┼──────────────┐
   FileReader     BufferedReader      InputStreamReader
                                      (字节→字符桥梁)
```

---

## 二、BIO / NIO / AIO

### 2.1 核心对比

| 对比维度 | BIO (Blocking IO) | NIO (Non-blocking IO) | AIO (Async IO) |
|---------|-------------------|----------------------|----------------|
| 模型 | 同步阻塞 | 同步非阻塞（多路复用） | 异步非阻塞 |
| 线程模型 | 一个连接一个线程 | 一个线程管理多个连接 | 回调/事件驱动 |
| 复杂度 | 简单 | 较复杂 | 复杂 |
| 吞吐量 | 低 | 高 | 高 |
| 适用场景 | 连接数少的小型应用 | 高并发连接（如聊天服务器） | 超高并发读（鲜少使用） |
| Java 实现 | 传统 IO API | Channel + Buffer + Selector | AsynchronousChannel |

### 2.2 同步 vs 异步、阻塞 vs 非阻塞

| 概念 | 含义 | 关注点 |
|------|------|--------|
| 同步 | 调用者主动等待结果 | 消息通信机制 |
| 异步 | 被调用者通过回调通知结果 | 消息通信机制 |
| 阻塞 | 线程被挂起等待数据 | 调用者线程状态 |
| 非阻塞 | 线程不等待，立即返回 | 调用者线程状态 |

---

## 三、NIO 核心组件

### 3.1 Buffer（缓冲区）

```java
// 核心属性
position   // 当前读写位置
limit      // 可读写边界
capacity   // 缓冲区总容量
mark       // 标记位置

// 操作
flip()     // 切换到读模式（limit=position, position=0）
clear()    // 切换到写模式（position=0, limit=capacity）
rewind()   // 重新读取（position=0, limit 不变）
compact()  // 将未读数据移到开头（压缩）
```

**追问：直接内存（Direct Buffer）和堆内存（Heap Buffer）的区别？**

| 对比 | Heap Buffer | Direct Buffer |
|------|-------------|---------------|
| 分配位置 | JVM 堆 | 堆外（直接内存） |
| IO 操作 | 需要从堆复制到堆外 | 无需复制，零拷贝基础 |
| 分配速度 | 快 | 较慢 |
| 回收 | GC 管理 | Full GC 或手动释放 |
| 适用场景 | 小数据、频繁分配 | 大数据、长生命周期 |

### 3.2 Channel（通道）

- FileChannel：文件读写
- SocketChannel：TCP 连接
- ServerSocketChannel：TCP 服务端监听
- DatagramChannel：UDP

Channel 是双向的，而传统 Stream 是单向的。

### 3.3 Selector（多路复用器）

```java
// 核心流程
Selector selector = Selector.open();
ServerSocketChannel ssc = ServerSocketChannel.open();
ssc.configureBlocking(false);               // 必须设为非阻塞
ssc.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select();                      // 阻塞，直到有事件就绪
    Set<SelectionKey> keys = selector.selectedKeys();
    Iterator<SelectionKey> it = keys.iterator();
    while (it.hasNext()) {
        SelectionKey key = it.next();
        if (key.isAcceptable()) { /* 处理连接 */ }
        else if (key.isReadable()) { /* 处理读 */ }
        else if (key.isWritable()) { /* 处理写 */ }
        it.remove();
    }
}
```

**追问：Selector 底层实现？**

Linux 使用 epoll（JDK 1.8+），基于事件驱动，支持水平触发（LT）和边沿触发（ET）。

---

## 四、零拷贝（Zero Copy）

### 4.1 传统 IO 的数据传输流程

```
磁盘 → 内核缓冲区 → 用户缓冲区 → Socket 缓冲区 → 网卡
      （DMA复制）    （CPU复制）    （CPU复制）    （DMA复制）
```

4 次上下文切换 + 4 次数据拷贝（2 次 CPU + 2 次 DMA）

### 4.2 mmap 优化

```
磁盘 → 内核缓冲区 → 用户缓冲区 → 内核缓冲区映射到 Socket 缓冲区 → 网卡
```

减少 1 次 CPU 拷贝，仍需要 4 次上下文切换。

### 4.3 sendfile（Kafka 使用的方式）

```java
// Java NIO 中的使用
FileChannel.transferTo(position, count, socketChannel);
```

```
磁盘 → 内核缓冲区 → Socket 缓冲区（只传递数据描述信息） → 网卡
      （DMA）       （少量CPU）                        （DMA）
```

2 次上下文切换 + 2 次数据拷贝（1 次 DMA + 1 次少许 CPU 拷贝），高效！

---

## 五、序列化

### 5.1 Java 原生序列化

```java
// 实现 Serializable 接口
public class User implements Serializable {
    private static final long serialVersionUID = 1L;  // 版本号
    private transient String password;  // transient 不序列化
}
```

缺点：不支持跨语言、序列化后体积大、性能差。

### 5.2 常见序列化框架

| 框架 | 特点 | 性能 |
|------|------|------|
| Java 原生 | 简单，Java 生态 | 低 |
| JSON（Jackson/Fastjson） | 可读性好，跨语言 | 中 |
| Protobuf | 体积小，跨语言 | 高 |
| Hessian | 二进制，跨语言 | 中高 |
| Kryo | 高性能，Java 专用 | 高 |
| FST | 高性能，Java 专用 | 高 |

**追问：serialVersionUID 的作用？**

用于验证序列化对象的版本一致性。如果未显式定义，JVM 会自动生成，但不同 JVM 可能生成不同的值，导致反序列化失败。所以建议显式定义。

**追问：transient 关键字的作用？**

标记的字段不会被序列化。常用于密码、临时变量等敏感或不需持久化的数据。

---

## 面试追问集

**Q：如何实现对象的深拷贝？**

1. Cloneable + clone() 递归克隆
2. 序列化 + 反序列化
3. 拷贝构造器
4. Apache Commons BeanUtils / Spring BeanUtils

**Q：NIO 中 Selector 的 select、poll、epoll 有什么区别？**

| 方式 | 数据结构 | 特点 |
|------|---------|------|
| select | bitmap | 有 fd 数量限制（1024），O(n) 轮询 |
| poll | 链表 | 无 fd 限制，但仍 O(n) 轮询 |
| epoll | 红黑树 + 双向链表 | 事件驱动，O(1)，无 fd 限制 |
