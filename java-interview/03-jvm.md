# JVM 与 Java 内存模型：中英文要点

用法：每个点先看中文理解，再背英文说法。回答公式：**原理 → 排查/调优思路 → 项目例子**。线程安全、happens-before 的部分见 `01-multithreading.md`。

> 版本说明：下面以 JDK 8 及以上为主，涉及版本差异的地方会标注。默认收集器：JDK 8 是 Parallel GC，**JDK 9 起是 G1**。

## 1. 运行时数据区（JVM 内存结构）

| 区域 | 线程共享？ | 存什么 | 会 OOM？ |
| --- | --- | --- | --- |
| **堆 Heap** | 共享 | 对象实例、数组 | 是（`Java heap space`） |
| **方法区 / 元空间 Metaspace** | 共享 | 类信息、常量、静态变量 | 是（`Metaspace`） |
| **虚拟机栈 Stack** | 私有 | 每个方法一个栈帧：局部变量表、操作数栈、返回地址 | `StackOverflowError`（太深）或 OOM |
| **本地方法栈** | 私有 | native 方法 | 同上 |
| **程序计数器 PC** | 私有 | 当前执行的字节码行号 | 唯一不会 OOM |

- **JDK 8 的变化：** 永久代（PermGen）被移除，换成**元空间**，使用本地内存（native memory），不在堆里，默认上限只受系统内存限制。
- **堆是 GC 的主要区域。**
- **字符串常量池**：JDK 7 起从永久代移到**堆**里。
- **直接内存（Direct Memory）**：NIO 的 `ByteBuffer.allocateDirect` 使用，不在堆内，不受堆大小限制，但会受 `-XX:MaxDirectMemorySize` 影响。

**English:** "The heap and the metaspace are shared by all threads. Each thread has its own stack, native method stack, and program counter. The heap holds objects, the metaspace holds class metadata, and each method call creates a stack frame for local variables. Since JDK 8, the permanent generation is replaced by the metaspace, which uses native memory."

## 2. 堆的分代结构

```
堆
├── 新生代 Young
│   ├── Eden
│   ├── Survivor 0 (From)
│   └── Survivor 1 (To)
└── 老年代 Old
```
默认比例：新生代 : 老年代 ≈ 1 : 2；Eden : S0 : S1 = 8 : 1 : 1。

**对象的生命周期：**
1. 新对象在 **Eden** 分配。
2. Eden 满 → **Minor GC**：存活对象复制到 Survivor，年龄 +1。
3. 在两个 Survivor 之间来回复制，每熬过一次年龄 +1。
4. 年龄达到阈值（默认 15，`-XX:MaxTenuringThreshold`）→ 晋升到**老年代**。
5. 老年代满 → **Major / Full GC**。

**特殊情况直接进老年代：**
- **大对象**（超过阈值，避免在 Eden 和 Survivor 之间来回复制）。
- Survivor 放不下。
- 动态年龄判断：同龄对象总大小超过 Survivor 的一半，大于等于该年龄的对象直接晋升。

**为什么分代：** 大部分对象"朝生夕死"（弱分代假说），分开管理，新生代用复制算法快速回收，老年代回收频率低。

**English:** "New objects are allocated in Eden. When Eden fills up, a minor GC copies the surviving objects to a survivor space and increases their age. Objects that survive enough collections are promoted to the old generation. We split the heap into generations because most objects die young, so collecting the young generation frequently is cheap."

## 3. 对象怎么死：可达性分析

- **引用计数法**：有人引用就加 1，为 0 就回收。**无法解决循环引用**，Java 不用。
- **可达性分析（Reachability analysis）**：从 **GC Roots** 出发，沿引用链遍历，**不可达的对象**就是垃圾。
- **GC Roots 包括：**
  - 栈帧中的局部变量
  - 静态变量
  - 常量
  - 活跃的线程
  - JNI 引用

**四种引用：**

| 引用 | 回收时机 | 用途 |
| --- | --- | --- |
| 强引用 | 只要有强引用就不回收 | 平时的 `new` |
| 软引用 SoftReference | 内存不足时回收 | 缓存 |
| 弱引用 WeakReference | 下一次 GC 就回收 | `ThreadLocalMap` 的 key、WeakHashMap |
| 虚引用 PhantomReference | 随时，用于通知 | 跟踪对象被回收 |

**English:** "Java uses reachability analysis. Starting from GC roots, such as local variables on the stack and static fields, the GC walks the references. Objects that can't be reached are garbage. It handles circular references, which reference counting can't."

## 4. 垃圾回收算法

| 算法 | 做法 | 优点 | 缺点 | 用在 |
| --- | --- | --- | --- | --- |
| 标记-清除 | 标记垃圾，然后清掉 | 简单 | 产生**内存碎片**，两遍扫描 | 老年代（CMS） |
| **复制** | 把存活对象复制到另一块区域，清空原区域 | 无碎片，快 | 浪费一部分空间 | **新生代** |
| 标记-整理 | 标记后把存活对象移到一端，清掉边界外的 | 无碎片 | 移动对象，**成本高** | **老年代** |

**English:** "The young generation uses copying, because most objects die and only a few need to be copied, and there is no fragmentation. The old generation uses mark-compact or mark-sweep, because most objects survive, so copying them would be expensive."

## 5. 垃圾收集器（重点：G1）

| 收集器 | 特点 | 适用 |
| --- | --- | --- |
| Serial | 单线程，Stop-The-World | 小内存、客户端 |
| Parallel（吞吐量优先） | 多线程并行回收 | 批处理，关注吞吐量（JDK 8 默认） |
| CMS | 并发标记，低停顿，**产生碎片**，JDK 9 起不推荐，JDK 14 移除 | 老的低延迟方案 |
| **G1** | 把堆分成许多 **Region**，优先回收垃圾最多的区域，**可预测停顿**（`-XX:MaxGCPauseMillis`） | **JDK 9 起默认**，大堆、服务端 |
| ZGC / Shenandoah | 亚毫秒级停顿，堆大小几乎不影响停顿 | 超低延迟（JDK 15+ 的 ZGC 可用于生产） |

**G1 要点：**
- 堆被划成大小相等的 **Region**（1 到 32MB），每个 Region 动态扮演 Eden、Survivor、Old 或 Humongous（大对象）。
- 优先回收**回收价值最高**的 Region（Garbage-First 名字的由来）。
- 回收流程：**Young GC** → **并发标记** → **Mixed GC**（回收新生代 + 部分老年代 Region）。
- 回收时复制存活对象到别的 Region，所以**没有碎片**。
- 目标：在设定的停顿时间内，尽可能多地回收。

**English:** "G1 splits the heap into equal-sized regions and collects the regions with the most garbage first. It lets me set a pause-time goal, and it compacts while collecting, so there's no fragmentation. It has been the default since JDK 9. For very low latency, I'd consider ZGC."

## 6. Stop-The-World 与 GC 触发

- **STW（Stop-The-World）**：GC 时暂停所有应用线程。停顿越长，请求延迟越高。
- **Minor GC**：新生代，频繁，快。
- **Major GC / Old GC**：老年代。
- **Full GC**：整个堆 + 元空间，最慢，**要尽量避免**。
- **Full GC 的常见触发：** 老年代空间不足、元空间不足、`System.gc()`、晋升失败、大对象无法分配。

## 7. 类加载机制

**过程（五步）：** 加载 → 验证 → 准备 → 解析 → 初始化（前面三个合称"链接"）。
- **准备**：为静态变量分配内存并设**默认值**（`int` 是 0）。
- **初始化**：执行静态代码块，赋**真正的值**。

**双亲委派（Parent Delegation）：**
- 类加载器收到请求，**先交给父加载器**，父加载不了才自己加载。
- 层次：Bootstrap（核心类库）→ Platform/Extension → Application（你的代码）→ 自定义。
- **好处：**
  1. **安全**：不能自己写一个 `java.lang.String` 替换核心类。
  2. **避免重复加载**：同一个类只会被加载一次，保证唯一。
- **打破双亲委派的例子：** Tomcat（每个应用有独立的类加载器，隔离不同应用的类）、JDBC 的 SPI。

**English:** "Class loading has loading, linking, which includes verification, preparation, and resolution, and initialization. With parent delegation, a class loader first asks its parent to load the class. This protects core classes, such as java.lang.String, from being replaced, and avoids loading a class twice."

## 8. Java 内存模型（JMM）

> 注意：**JMM 和 JVM 内存结构是两回事。** JVM 内存结构讲堆、栈、元空间；JMM 讲**多线程如何通过内存交互**。

- 每个线程有自己的**工作内存**（对应 CPU 缓存和寄存器），共享变量存放在**主内存**。
- 线程对变量的读写都在工作内存里进行，再同步回主内存 → 所以会有**可见性**问题。
- JMM 通过 **happens-before** 规则和**内存屏障**，规定什么时候一个线程的修改对另一个线程可见，以及哪些重排是允许的。
- `volatile`、`synchronized`、`final` 都是 JMM 提供的保证。

**English:** "The Java Memory Model is different from the JVM memory layout. It defines how threads see each other's writes to shared variables, using happens-before rules. volatile, synchronized, and final give specific visibility and ordering guarantees."

（详细的 volatile、happens-before 见 `01-multithreading.md`。）

## 9. OOM 与排查（高频，结合实战）

**常见 OOM 类型：**

| 报错 | 原因 | 常见场景 |
| --- | --- | --- |
| `Java heap space` | 堆满了，且 GC 回收不了 | **内存泄漏**，或一次加载太多数据 |
| `GC overhead limit exceeded` | 花了大部分时间做 GC，却回收很少 | 同上，快撑满了 |
| `Metaspace` | 加载的类太多 | 动态生成类（反射、CGLIB）、类加载器泄漏 |
| `unable to create new native thread` | 创建的线程太多 | 线程池无限增长 |
| `Direct buffer memory` | 直接内存不足 | NIO、Netty 使用不当 |
| `StackOverflowError` | 栈太深 | 无限递归 |

**内存泄漏的典型原因：**
- 静态集合不断添加，从不清理（如静态 `HashMap` 做缓存没有淘汰）
- **ThreadLocal 用完没 `remove()`**（线程池场景）
- 监听器、回调注册后没注销
- 未关闭的连接、流
- 可变对象做 HashMap 的 key，修改后无法删除

**排查步骤（背下来）：**
1. **看现象**：监控内存曲线、GC 日志，确认是**持续上涨**（泄漏）还是**瞬间飙升**（一次性大对象）。
2. **保留现场：** 开启 `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=...`，OOM 时自动生成堆转储；或用 `jmap -dump:live,format=b,file=heap.hprof <pid>` 手动导出。
3. **分析堆转储：** 用 **MAT（Eclipse Memory Analyzer）**，看 **Dominator Tree**（谁占内存最多）和 **Path to GC Roots**（为什么没被回收）。
4. **实时观察：** `jstat -gcutil <pid> 1000` 看各区域使用率和 GC 次数；`jstack <pid>` 看线程；`jcmd`、`jmap -histo` 看对象分布。
5. **定位代码，修复，验证**（压测后看内存曲线是否平稳）。

**English:** "First I look at the memory curve and GC logs to see whether it's a leak or a spike. I enable heap dump on OOM, then analyze the dump with MAT. The dominator tree shows what takes the most memory, and path to GC roots shows why it isn't collected. I also use jstat for live GC stats and jstack for threads. Then I fix the code and verify with a load test."

## 10. 常用 JVM 参数

| 参数 | 含义 |
| --- | --- |
| `-Xms` / `-Xmx` | 初始 / 最大堆。生产建议**设成相同**，避免运行中动态扩缩容 |
| `-Xmn` | 新生代大小 |
| `-Xss` | 每个线程的栈大小 |
| `-XX:MaxMetaspaceSize` | 元空间上限 |
| `-XX:+UseG1GC` | 使用 G1 |
| `-XX:MaxGCPauseMillis` | G1 的目标停顿时间 |
| `-XX:+HeapDumpOnOutOfMemoryError` | OOM 时生成堆转储 |
| `-Xlog:gc*`（JDK 9+）/ `-XX:+PrintGCDetails`（JDK 8） | 打印 GC 日志 |

## 11. 调优思路（不要背参数，要讲方法）

1. **先确认有问题：** 有监控数据（GC 频率、停顿时间、延迟）证明问题，不要凭感觉调。
2. **明确目标：** 低延迟（停顿短）还是高吞吐（总处理量大）。
3. **常见优化手段：**
   - 增大堆或调整新生代比例，减少对象过早晋升
   - 减少对象创建（对象复用、避免大对象、避免装箱）
   - 避免 Full GC：不要手动调用 `System.gc()`，控制大对象
   - 选合适的收集器（延迟敏感用 G1 或 ZGC）
4. **一次只改一个参数，改完压测对比。**
5. **多数性能问题其实是代码问题**（泄漏、不合理的缓存、慢 SQL），不是参数问题。

**English:** "I don't tune by guessing. I start from monitoring data, set a goal such as latency or throughput, change one parameter at a time, and verify with a load test. Often the real fix is in the code, like removing a leak or reducing allocations, not in the flags."

## 12. 面试高频问题清单（自测）

1. JVM 内存区域有哪些？哪些线程私有，哪些共享？
2. JDK 8 的永久代和元空间有什么区别？
3. 对象的创建和在堆中的流转过程？什么时候进入老年代？
4. 怎么判断对象可以回收？GC Roots 有哪些？
5. 三种垃圾回收算法的区别？新生代和老年代各用哪个？为什么？
6. G1 的原理？和 CMS 的区别？为什么 G1 能做到可预测停顿？
7. Minor GC、Major GC、Full GC 的区别？什么时候触发 Full GC？
8. 类加载过程？什么是双亲委派？为什么需要它？
9. JMM 和 JVM 内存结构有什么区别？
10. 线上 OOM 怎么排查？内存泄漏的常见原因？
11. ThreadLocal 为什么会内存泄漏？（见 01 文档）
12. 软、弱、虚引用的区别和使用场景？

## 13. 和你项目的连接

| 知识点 | 你的故事 |
| --- | --- |
| 内存不足 / OOM 风险 | TasCreed：节点同时 pick 了太多 task，内存不够，kill 让别的节点接手，之后调 max workers 参数 |
| 堆内缓存的内存占用 | FXCenter：客户端缓存全部汇率并保留前一天，要估算占用，快照替换时新旧两份会短暂共存 |
| 软引用做缓存 / 内存敏感 | 缓存淘汰与内存压力的权衡 |
| 大数据量报表 | Spark 报表：大对象、数据倾斜与内存，Executor 内存配置 |
| 线程栈与线程数 | 线程池大小和每个线程栈内存会占用内存 |

[待补] 你们实际遇到的内存问题（OOM、GC 停顿、Full GC）、使用的 JVM 参数和收集器、以及如何排查，按真实经历填写。**没有的话不要编造**，面试时可以讲一般方法，并说明自己主要是在……场景下接触过。
