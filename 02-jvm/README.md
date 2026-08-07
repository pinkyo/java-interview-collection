# JVM（Java 虚拟机）

> JVM 是 Java 高级工程师面试的必考模块，涵盖内存管理、GC 机制、类加载和执行引擎。

## 📂 内容索引

| 文件 | 核心知识点 |
|------|-----------|
| [memory-structure.md](./memory-structure.md) | 运行时数据区、堆/栈/方法区、元空间、直接内存、对象创建过程 |
| [gc.md](./gc.md) | GC 算法、垃圾收集器（CMS/G1/ZGC）、GC 调优、三色标记 |
| [class-loading.md](./class-loading.md) | 类加载流程、双亲委派模型、打破双亲委派、自定义类加载器 |
| [tuning.md](./tuning.md) | JVM 参数、OOM 排查、内存泄漏分析、Arthas/MAT 工具 |

## 🔥 高频考点速览

1. **JVM 内存模型**：堆、栈、方法区、程序计数器的分工
2. **GC 机制**：Minor GC / Full GC 触发时机、GC Roots 有哪些？
3. **垃圾收集器**：CMS 与 G1 的区别、ZGC 原理
4. **类加载**：双亲委派模型、如何打破？Tomcat 为什么打破？
5. **OOM 排查**：内存快照分析、常见 OOM 场景与解决方案
