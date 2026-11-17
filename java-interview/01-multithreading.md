# Java 多线程与并发：中英文要点

用法：每个点先看中文理解，再背英文说法。回答公式：**原理 → 项目例子 → 取舍**。标 [项目] 的地方接你自己的故事（TasCreed、余额竞态题、FXCenter 缓存）。

## 1. 线程安全的三个问题

| 问题 | 中文 | English |
| --- | --- | --- |
| 原子性 | 一个操作不可被打断（如 `count++` 其实是读、加、写三步） | Atomicity: an operation can't be interrupted halfway. `count++` is read, add, write. |
| 可见性 | 一个线程的修改，别的线程能立刻看到 | Visibility: one thread's change is seen by others. |
| 有序性 | 编译器和 CPU 可能重排指令，多线程下会出问题 | Ordering: the compiler and CPU may reorder instructions. |

**一句话（English）：** "Thread safety means correct behavior under concurrent access. The three issues are atomicity, visibility, and ordering."

## 2. synchronized

- **作用**：互斥（同一时刻只有一个线程进入）+ 保证可见性（释放锁前的修改，对下一个获得锁的线程可见）。
- **三种用法**：修饰实例方法（锁 this）、静态方法（锁 Class 对象）、代码块（锁指定对象）。
- **特点**：可重入；JVM 自动加锁和释放，异常时也会释放。
- **锁升级（JDK 6 起）**：无锁 → 偏向锁 → 轻量级锁（CAS 自旋）→ 重量级锁（操作系统互斥）。竞争越激烈，锁越重。注意：偏向锁在较新的 JDK 中已被默认禁用或移除，不用深究。

**English:** "synchronized gives mutual exclusion and visibility. It's reentrant, and the JVM releases the lock automatically, even on exceptions. The JVM optimizes it by upgrading the lock from lightweight spinning to a heavyweight lock only when contention grows."

## 3. synchronized vs ReentrantLock（高频）

| | synchronized | ReentrantLock |
| --- | --- | --- |
| 释放 | 自动 | 必须手动 `unlock()`，放在 `finally` |
| 可中断获取 | 不支持 | `lockInterruptibly()` |
| 超时获取 | 不支持 | `tryLock(timeout)` |
| 公平锁 | 只能非公平 | 可选公平或非公平 |
| 多条件等待 | 一个 wait set | 多个 `Condition` |
| 实现 | JVM 层面 | JDK 层面，基于 AQS |

```java
lock.lock();
try {
    // critical section
} finally {
    lock.unlock();
}
```

**怎么选：** 简单场景用 synchronized；需要超时、可中断、公平性或多条件时用 ReentrantLock。

**English:** "I use synchronized by default because it's simple and safe. I switch to ReentrantLock when I need a timeout, interruptible locking, fairness, or multiple conditions. With ReentrantLock, I always unlock in a finally block."

## 4. volatile

- **保证**：可见性 + 禁止指令重排。
- **不保证**：原子性。`volatile int count; count++` 仍然不安全。
- **适用场景**：状态标志（`volatile boolean running`）、双重检查锁定的单例。

```java
private static volatile Singleton instance;   // volatile 防止重排导致拿到未初始化的对象
```

**English:** "volatile guarantees visibility and prevents reordering, but not atomicity. It's good for a status flag, but not for count++, which needs an atomic class or a lock."

## 5. CAS 与原子类

- **CAS（Compare-And-Swap）**：当前值等于期望值才更新，否则失败重试。CPU 级原子指令。
- **用在**：`AtomicInteger`、`AtomicLong`、`ConcurrentHashMap`、AQS。
- **问题**：
  - **ABA**：A 变 B 又变回 A，CAS 看不出。用 `AtomicStampedReference`（带版本号）。
  - **高竞争下自旋浪费 CPU**。高并发计数可以用 `LongAdder`（分段累加）。
  - **只能保证单个变量的原子性**。

**English:** "CAS updates a value only if it still equals what I read, otherwise it retries. It's lock-free, but it has the ABA problem and wastes CPU under high contention."

[项目] 数据库里 `UPDATE ... WHERE version = ?` 本质也是一次 CAS，是乐观锁。

## 6. AQS（AbstractQueuedSynchronizer）

- **核心**：一个 `volatile int state`（同步状态）+ 一个 FIFO 等待队列。
- **流程**：线程尝试用 CAS 修改 state，成功就获得锁；失败就进入队列并挂起（park）；释放时唤醒队首线程（unpark）。
- **基于 AQS**：ReentrantLock、Semaphore、CountDownLatch、ReentrantReadWriteLock。
- **ReentrantLock**：state 表示重入次数。

**English:** "AQS keeps a volatile state and a FIFO queue of waiting threads. A thread tries to change the state with CAS. If it fails, it queues and parks. ReentrantLock, Semaphore, and CountDownLatch are all built on it."

## 7. 线程池（必考）

**为什么用：** 复用线程，降低创建销毁成本；控制并发数；统一管理。

**ThreadPoolExecutor 七个参数：**
`corePoolSize`、`maximumPoolSize`、`keepAliveTime`、`unit`、`workQueue`、`threadFactory`、`handler`

**任务提交流程（背下来）：**
1. 线程数 < core → 创建新线程执行。
2. 否则放入队列。
3. 队列满，且线程数 < max → 创建非核心线程。
4. 线程数已达 max，队列也满 → 执行**拒绝策略**。

**四种拒绝策略：**
- `AbortPolicy`（默认）：抛 `RejectedExecutionException`
- `CallerRunsPolicy`：调用者线程自己执行（天然限流）
- `DiscardPolicy`：直接丢弃
- `DiscardOldestPolicy`：丢弃队列里最老的

**为什么不推荐 `Executors` 的快捷方法：**
- `newFixedThreadPool`、`newSingleThreadExecutor`：队列是无界的，任务堆积会 OOM。
- `newCachedThreadPool`：最大线程数是 `Integer.MAX_VALUE`，可能创建大量线程。

**线程数怎么定：**
- **CPU 密集**：约 CPU 核数 + 1
- **IO 密集**：约 CPU 核数 × (1 + 等待时间 / 计算时间)，实际靠压测调整

**English:** "When a task is submitted, the pool first creates core threads, then queues the task. If the queue is full, it adds threads up to the max, and after that it applies the rejection policy. I avoid Executors factory methods because they use unbounded queues or unbounded thread counts, which can cause OOM. I size CPU-bound pools near the core count, and IO-bound pools larger, then tune by load testing."

[项目] TasCreed 里 `maxWorkerCountOverall` 和单实例 worker 上限，本质是控制并发度，防止节点被压垮（节点内存不够时 kill 的例子）。

## 8. ConcurrentHashMap（并发角度）

- **JDK 7**：分段锁（Segment），每段一把锁。
- **JDK 8+**：数组 + 链表/红黑树，**CAS + synchronized 锁住单个桶的头节点**，锁粒度更细。
- **读操作**：基本不加锁（value 和 next 是 volatile）。
- **不允许 null**：因为无法区分"key 不存在"和"value 是 null"。
- **size()** 是近似值（用 `baseCount` + `CounterCell`，类似 LongAdder）。

**English:** "In JDK 8, ConcurrentHashMap uses CAS to insert into empty buckets and synchronized on the head node of a bucket otherwise. Reads are mostly lock-free. It doesn't allow null keys or values."

（HashMap 的数据结构和扩容放在下一份文档。）

## 9. 死锁

**四个必要条件：** 互斥、持有并等待、不可剥夺、循环等待。

**避免：**
- **按固定顺序加锁**（破坏循环等待），如转账时按账户 id 从小到大加锁
- 使用 `tryLock(timeout)`
- 减少锁的范围和嵌套

**排查：** `jstack <pid>` 会直接提示 "Found one Java-level deadlock"。

**English:** "Deadlock needs four conditions: mutual exclusion, hold and wait, no preemption, and circular wait. The easiest fix is to always acquire locks in a consistent order. I diagnose it with jstack."

[项目] 余额转账题：两个账户加锁要按 id 排序。

## 10. ThreadLocal

- **作用**：每个线程有自己的变量副本，不共享，所以线程安全。
- **原理**：每个 `Thread` 内部有一个 `ThreadLocalMap`，key 是 ThreadLocal（弱引用），value 是值。
- **内存泄漏**：线程池中线程长期存活，value 不会被回收（key 是弱引用，被回收后 value 仍被强引用）。所以用完要 **`remove()`**。
- **常见用途**：用户上下文、请求 id、`SimpleDateFormat`（非线程安全）。

**English:** "ThreadLocal gives each thread its own copy. The values live in the thread's own map. In a thread pool, threads live long, so I always call remove() to avoid leaks and stale data."

## 11. wait/notify 与线程状态

- **线程状态**：NEW、RUNNABLE、BLOCKED（等 synchronized 锁）、WAITING（`wait()`、`join()`、`park()`）、TIMED_WAITING（`sleep(t)`）、TERMINATED。
- **sleep vs wait**：
  - `sleep` 是 Thread 的方法，**不释放锁**。
  - `wait` 是 Object 的方法，**释放锁**，必须在 synchronized 里调用，用 `notify/notifyAll` 唤醒。
- **`wait` 要放在 while 循环里**，防止虚假唤醒。

**English:** "sleep doesn't release the lock, while wait does. wait must be called inside synchronized and in a while loop, to guard against spurious wakeups."

## 12. 常用并发工具

| 工具 | 用途 | English |
| --- | --- | --- |
| `CountDownLatch` | 一个或多个线程等待其他线程完成，一次性 | Wait until N tasks finish. One-time use. |
| `CyclicBarrier` | 一组线程互相等待到齐，可重复使用 | Threads wait for each other at a barrier. Reusable. |
| `Semaphore` | 限制同时访问资源的线程数 | Limit concurrent access with permits. |
| `CompletableFuture` | 异步编排（串联、并行、组合） | Compose async tasks without blocking. |

## 13. Java 内存模型与 happens-before（和 JVM 一起深入）

- JMM 规定线程如何通过主内存和工作内存交互。
- **happens-before** 保证：前一个操作的结果对后一个操作可见。常见规则：
  - 同一线程内，程序顺序
  - 解锁 happens-before 之后对同一把锁的加锁
  - volatile 写 happens-before 之后对它的读
  - `Thread.start()` happens-before 该线程中的操作

## 14. 面试高频问题清单（自测）

1. `count++` 为什么不是线程安全的？怎么修？
2. synchronized 和 ReentrantLock 的区别？
3. volatile 能保证什么，不能保证什么？
4. 线程池的工作流程？参数怎么设？为什么不用 Executors？
5. 什么是死锁？怎么避免和排查？
6. CAS 是什么？有什么问题？
7. ThreadLocal 为什么会内存泄漏？
8. ConcurrentHashMap 在 JDK 7 和 8 的区别？
9. sleep 和 wait 的区别？
10. 余额更新的竞态问题怎么解决？（原子 SQL、乐观锁、悲观锁、幂等）

## 15. 和你项目的连接

| 知识点 | 你的故事 |
| --- | --- |
| 线程池、并发度控制 | TasCreed：单实例 worker 上限，节点内存不够时 kill 让别的节点接手 |
| CAS、乐观锁 | 余额更新、etcd 的 "key 不存在才创建" |
| 锁与超时 | etcd lease 和心跳，类似带超时的分布式锁 |
| 幂等 | 报表分片输出固定文件名，重跑覆盖 |
| 有序加锁避免死锁 | 转账按账户 id 排序 |
