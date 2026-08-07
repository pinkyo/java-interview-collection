# JVM 调优与排查工具

---

## 一、常见 JVM 参数

### 1.1 内存相关

```bash
-Xms2g                          # 初始堆大小
-Xmx2g                          # 最大堆大小
-Xmn1g                          # 新生代大小（不推荐，建议用比例）
-XX:NewRatio=2                  # 老年代/新生代比例（默认 2）
-XX:SurvivorRatio=8             # Eden/Survivor 比例（默认 8）

-XX:MetaspaceSize=256m          # 元空间初始大小
-XX:MaxMetaspaceSize=256m       # 元空间最大大小
-XX:MaxDirectMemorySize=512m    # 直接内存上限
-Xss256k                        # 线程栈大小
```

### 1.2 GC 相关

```bash
# 选择垃圾收集器
-XX:+UseG1GC                    # 使用 G1
-XX:+UseZGC                     # 使用 ZGC（JDK 11+）

# GC 日志
-XX:+PrintGCDetails             # 打印 GC 详情
-XX:+PrintGCDateStamps          # 打印日期时间戳
-Xloggc:/path/to/gc.log         # 日志输出位置

# G1 调优
-XX:MaxGCPauseMillis=200        # 最大停顿时间目标
-XX:G1HeapRegionSize=4m         # Region 大小

# OOM 时输出堆 dump
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/path/to/dump
```

---

## 二、常见 OOM 场景

| 类型 | 原因 | 排查方向 |
|------|------|---------|
| java.lang.OutOfMemoryError: Java heap space | 堆空间不足 | 分析堆 dump，查大对象 |
| java.lang.OutOfMemoryError: Metaspace | 类太多 | 检查动态代理/反射生成的大量类 |
| java.lang.OutOfMemoryError: GC overhead limit exceeded | GC 效果太差 | GC 频繁且回收少 |
| java.lang.OutOfMemoryError: Direct buffer memory | 直接内存不足 | NIO 使用不当 |
| java.lang.OutOfMemoryError: unable to create new native thread | 线程数超限 | 检查线程泄漏/操作系统限制 |

---

## 三、常用排查工具

### 3.1 命令行工具

| 工具 | 作用 | 示例 |
|------|------|------|
| jps | 查看 Java 进程 | `jps -l` |
| jstat | 监控 JVM 统计信息 | `jstat -gc pid 1000` |
| jmap | 生成堆 dump / 查看堆信息 | `jmap -heap pid` / `jmap -dump:format=b,file=heap.hprof pid` |
| jstack | 线程栈信息 | `jstack pid` |
| jinfo | JVM 参数信息 | `jinfo -flags pid` |
| jhat | 分析堆 dump（已过时） | — |

### 3.2 jstat 示例

```bash
jstat -gc pid 1000 10
# S0C  S1C  S0U  S1U  EC   EU   OC    OU    MC    MU   YGC YGCT FGC FGCT
# S0C：survivor0 容量    S1U：survivor1 使用量
# EC：Eden 容量       EU：Eden 使用量
# OC：老年代容量       OU：老年代使用量
# MC：元空间容量       MU：元空间使用量
# YGC：Young GC 次数   YGCT：Young GC 耗时
# FGC：Full GC 次数    FGCT：Full GC 耗时
```

### 3.3 jstack 排查死锁

```bash
jstack pid | grep -A 20 "deadlock"
# 或
jstack pid > thread.log
# 查看 BLOCKED / WAITING 状态的线程
```

### 3.4 可视化工具

| 工具 | 用途 |
|------|------|
| **MAT** | 堆 dump 分析，查内存泄漏、大对象 |
| **Arthas** | 在线诊断（最推荐），无需重启 |
| **JProfiler** | 性能分析，收费 |
| **VisualVM** | 综合监控，JDK 自带 |
| **GCViewer** | GC 日志可视化分析 |
| **Async Profiler** | 低开销 CPU/内存采样 |

### 3.5 Arthas 常用命令

```bash
dashboard       # 实时面板（线程/内存/GC）
thread          # 查看线程，thread -b 查找死锁
jad 类名        # 反编译类
watch 类名 方法  # 观测方法调用（入参/返回值）
trace 类名 方法  # 追踪方法调用链
heapdump        # 生成堆 dump
logger          # 动态修改日志级别
```

---

## 四、内存泄漏排查思路

```
1. 确认现象（OOM 日志 / 内存持续增长）
2. 查看 GC 日志 → FGC 频繁且回收效果差
3. jmap 导出 heap dump → jmap -dump:format=b,file=heap.hprof pid
4. MAT 分析：
   → Leak Suspects Report（泄露疑点报告）
   → Dominator Tree（支配树）找到大对象
   → GC Roots 追溯引用链
5. 定位泄漏代码并修复
```

---

## 五、常见调优案例

### 案例一：频繁 Full GC

```
现象：响应慢，GC 日志中 FGC 频繁
原因：老年代被快速填满（大对象直接进入 / Survivor 太小 / 晋升阈值太低）
解决：
  - 增大年轻代（-Xmn）
  - 调整 Survivor 比例（-XX:SurvivorRatio）
  - 增大晋升阈值（-XX:MaxTenuringThreshold）
  - 提前对象（大对象缓存/复用）
```

### 案例二：Young GC 耗时过长

```
原因：新生代太大，单次复制对象太多
解决：适当减小新生代，或使用 G1
```

### 案例三：元空间 OOM

```
原因：动态生成了大量类（反射、CGLIB、动态代理、Groovy 脚本）
解决：
  - 增大 -XX:MaxMetaspaceSize
  - 减少动态类生成
  - 使用 -XX:+MetaspaceSize 提前触发 GC
```

---

## 面试追问集

**Q：线上服务 CPU 飙升，如何排查？**

1. `top` 找到高 CPU 进程 PID
2. `top -H -p PID` 找到高 CPU 线程 TID
3. `printf "%x\n" TID` 转十六进制
4. `jstack PID | grep -A 20 hex_TID` 定位具体代码
5. Arthas：`thread -n 3` 直接查看最忙的 3 个线程

**Q：内存泄漏和内存溢出的区别？**

- 内存泄漏是原因：无用对象未被回收
- 内存溢出是结果：内存不足

**Q：有没有遇到过 GC 问题？怎么解决的？**

结合自己的项目经验回答，核心思路：看日志 → 看堆 → 定位问题 → 调整参数。
