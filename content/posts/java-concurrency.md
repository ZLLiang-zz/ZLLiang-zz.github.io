---
title: "Java 并发：从 JMM、锁升级到 AQS 的完整链路"
date: 2026-10-22T09:00:00+08:00
summary: "内存可见性为什么需要屏障、synchronized 的四个队列怎么流转、偏向锁为什么被废弃、AQS 的 park/unpark 与 PROPAGATE 各自解决什么问题——把并发知识串成一条链，而不是一堆名词。"
categories: ["Java 并发"]
tags: ["并发", "JMM", "AQS", "synchronized"]
author: ZLLiang
ShowToc: true
TocOpen: false
draft: false
---

## 一、基本概念

### 1.1 并发、并行、异步：三个词不能混用

| 概念 | 定义 | 判定标准 |
| :--- | :--- | :--- |
| **并发 Concurrency** | 多个任务**交替**推进，宏观上"同时"，微观上可能串行 | 是否有多个任务在**重叠的时间段**内推进 |
| **并行 Parallelism** | 多个任务**同一时刻**在不同 CPU 核上执行 | 是否**物理同时**执行 |
| **异步 Async** | 调用方不等结果返回就继续，结果通过回调/Future 获取 | 是否**阻塞等待** |

一句话：**并发是结构（怎么组织任务），并行是执行（谁来跑），异步是交互（要不要等）**。单核 CPU 上可以并发但不可能并行；异步可以用单线程实现（如 Netty 的 EventLoop）。

### 1.2 并发编程要解决的三件事（以及第四件）

| 问题 | 表现 | 根源 | 解决手段 |
| :--- | :--- | :--- | :--- |
| **原子性** | `i++` 丢更新 | 读-改-写三步非原子 | `synchronized`、`CAS`、`Lock`、原子类 |
| **可见性** | 一个线程改了标志位，另一个线程永远看不到 | CPU 缓存 + 编译器优化，写不回主存/读到旧值 | `volatile`、`synchronized`、`final` |
| **有序性** | 指令重排导致半初始化对象被发布 | 编译器/CPU/内存系统三重重排 | `volatile`（内存屏障）、`happens-before` |
| **活跃性**（第四件） | 死锁、活锁、饥饿、线程泄漏 | 资源竞争与调度 | 锁顺序、超时、公平锁、隔离 |

> **常见误解**："`volatile` 能保证原子性吗？" —— 不能。`volatile int i; i++` 依然丢更新，因为 `i++` 是 `read → use → assign → write` 四步，`volatile` 只保证每一步都是最新值，不保证四步整体原子。要原子就用 `AtomicInteger` 或加锁。

### 1.3 线程的生命周期：六状态与转换条件

```
                   start()
        NEW ──────────────────→ RUNNABLE ──────────────→ TERMINATED
                                   │  ↑                      ↑
              ┌────────────────────┘  └──────────┐           │
              │ 进入 synchronized 未抢到           │           │
              ↓                                   │           │
          BLOCKED ────── 抢到锁 ──────────────────┘           │
                                                              │
   RUNNABLE ── wait()/join()/park() ──→ WAITING ───────────────┤
            ←── notify()/notifyAll()/unpark() ──┘              │
   RUNNABLE ── sleep(t)/wait(t)/join(t)/parkNanos ──→ TIMED_WAITING
            ←── 超时/唤醒 ──┘                                  │
                                                              │
   RUNNABLE ── run() 返回或异常 ─────────────────────────────→ ┘
```

**五个容易答错的点**：

1. **`RUNNABLE` 包含"正在等 IO/等 CPU 时间片"**——Java 不区分 Ready 与 Running，也不区分内核态的 IO 等待（只要不是被 JVM 显式挂起）。
2. **`BLOCKED` 只针对 `synchronized`**。`ReentrantLock` 等不到锁是 `WAITING`（`LockSupport.park`），不是 `BLOCKED`。这是 `jstack` 判读的关键差别。
3. **`wait()` 必须在 `synchronized` 块内**，否则 `IllegalMonitorStateException`；它释放锁并进入对象的 WaitSet。
4. **`sleep()` 不释放锁**，`wait()` 释放锁。这是两者最本质的区别（不是"一个属于 Thread 一个属于 Object"）。
5. **`stop()`/`suspend()` 已废弃**，正确姿势是 `interrupt()` + 标志位协作退出。

### 1.4 上下文切换：并发的隐性成本

一次上下文切换大致包含：保存当前线程寄存器/PC → 更新 TCB → 调度器选下一个 → 恢复寄存器 → 刷新 TLB/缓存局部性受损。成本量级 **1~10 微秒**（不含缓存失效的间接损失）。

**什么时候切换**：

- 时间片用完（抢占式）；
- 线程主动让出（`yield`、`sleep`、`wait`、`park`）；
- 阻塞（IO、锁、`join`）；
- 更高优先级线程就绪。

**这解释了三件事**：

1. 线程数远超 CPU 核数时，切换开销可能超过业务耗时（线程池不是越大越好）；
2. 无锁不一定比加锁快——CAS 自旋失败重试也是"空转"，竞争激烈时不如直接 park；
3. 虚拟线程的价值：**把"阻塞"从"占用 OS 线程"变成"挂起 Continuation"**，切换成本从微秒级降到纳秒级。

### 1.5 核心术语对齐

| 术语 | 准确含义 | 常见误解 |
| :--- | :--- | :--- |
| **JMM** | Java 内存模型，一套并发语义规范（happens-before、可见性、有序性） | ❌ 与"堆栈方法区"混淆（那是运行时数据区） |
| **happens-before** | 两个操作的可见性偏序关系：A hb B ⇒ A 的结果对 B 可见 | ❌ 理解成"时间上先发生" |
| **as-if-serial** | 单线程内重排不能改变执行结果 | ❌ 以为多线程也保证 |
| **内存屏障** | 阻止特定类型重排的 CPU 指令/编译器约束 | ❌ 以为四种屏障都有实际指令 |
| **临界区** | 访问共享资源、必须互斥执行的代码段 | — |
| **竞态条件** | 结果依赖线程执行时序 | ❌ 等同于数据竞争（后者特指无同步的并发读写） |
| **偏向锁** | 只有一个线程反复进入时，把线程 ID 写进 Mark Word，免 CAS | ❌ 以为 JDK 17 还默认开启（JDK 15 起默认关闭并废弃） |
| **AQS** | `AbstractQueuedSynchronizer`，用 `state` + CLH 变体队列实现同步器骨架 | ❌ 以为它是锁（它是**框架**，`ReentrantLock` 才是锁） |
| **CLH 队列** | Craig-Landin-Ten-Hagersten 自旋锁队列，AQS 用的是其**变体**（双向、显式 park） | ❌ 以为 AQS 就是原版 CLH |
| **虚假唤醒** | 没有 notify 却从 wait 返回 | ❌ 忘记用 `while` 而非 `if` 判断条件 |
| **伪共享** | 两个无关变量落在同一缓存行（64B），互相使对方缓存失效 | ❌ 与"共享变量竞争"混淆 |

### 1.6 体系定位：一张图看清 JUC 的分层

```
┌──────────────────── 应用层：线程池、CompletableFuture、并发容器组合 ────────────────────┐
├──────────────────── 工具层：Semaphore / CountDownLatch / CyclicBarrier / Phaser ────────┤
├──────────────────── 锁层：ReentrantLock / ReentrantReadWriteLock / StampedLock ─────────┤
├──────────────────── 骨架层：AQS（state + 队列）  │  原子类（CAS + varHandle）──────────┤
├──────────────────── 基础层：volatile / synchronized / CAS / Unsafe / 内存屏障 ───────────┤
└──────────────────── 规范层：JMM（happens-before）+ 硬件内存模型（TSO/弱一致）───────────┘
```

**这张图的用法**：任何并发问题都能向上或向下定位。比如"`CountDownLatch` 能不能重复用？"——向上看它是工具层（一次性），向下看它是 AQS 共享模式（`state` 减到 0 就结束，无法重置）。

---

## 二、基础知识点（怎么用）

### 2.1 创建线程：四种方式与一个铁律

| 方式 | 写法 | 评价 |
| :--- | :--- | :--- |
| 继承 `Thread` | `new MyThread().start()` | Java 单继承，扩展性差，不推荐 |
| 实现 `Runnable` | `new Thread(runnable).start()` | 职责分离，推荐 |
| 实现 `Callable` + `FutureTask` | 有返回值、可抛异常 | 需要返回值时用 |
| 线程池 | `executor.execute/submit` | **生产唯一正确姿势** |

**铁律：`start()` 而不是 `run()`**。`run()` 只是普通方法调用，会在当前线程执行；`start()` 才走 `start0()` native → JVM 创建 OS 线程 → 回调 `run()`。而且**同一个 Thread 对象只能 `start()` 一次**，第二次抛 `IllegalThreadStateException`。

### 2.2 synchronized：三种用法与锁对象

| 用法 | 锁对象 | 字节码 |
| :--- | :--- | :--- |
| 实例方法 `synchronized void m()` | **当前实例 `this`** | 方法标志 `ACC_SYNCHRONIZED` |
| 静态方法 `static synchronized void m()` | **`Class` 对象** | 同上，锁的是类 |
| 代码块 `synchronized(obj)` | 括号里的对象 | `monitorenter` / `monitorexit` ×2 |

三个必须知道的细节：

1. **代码块的 `monitorexit` 有两条**：正常路径一条，异常路径一条（编译器自动生成 `athrow` 前的释放），所以 `synchronized` **不会因为异常而泄漏锁**。
2. **锁的是对象，不是代码**。两个线程用不同实例调用同一个 `synchronized` 实例方法，**不互斥**。
3. **`synchronized` 可重入**，靠 `ObjectMonitor._recursions` 计数，重入 N 次要退出 N 次才真正释放。

**构造方法不能加 `synchronized`**：构造期间对象还未发布，锁它没有意义（编译器也不允许这个修饰符语义成立）。要保证构造期间的可见性，用 `final` 字段或安全发布。

### 2.3 volatile：两个语义、三种用法、一个禁区

**语义**：

- **可见性**：写立即刷回主内存，读强制从主内存读（准确说是**禁止缓存到寄存器/工作内存**，靠 `lock` 前缀 + 缓存一致性协议）；
- **有序性**：禁止与前后指令的特定重排（插入内存屏障）。

**三种正确用法**：

```java
// ① 状态标志（最典型）
private volatile boolean running = true;
while (running) { ... }            // 无 volatile 可能永远不退出

// ② 双重检查锁单例（防止半初始化对象被发布）
private static volatile Singleton instance;
if (instance == null) {
    synchronized (Singleton.class) {
        if (instance == null) instance = new Singleton();   // 少了 volatile 会出问题
    }
}

// ③ 一次性安全发布（读多写少）
private volatile Config config;
void reload() { config = load(); }   // 其他线程读 config 一定能看到完整对象
```

**一个禁区**：**不能保证原子性**。`volatile++`、`volatile` 修饰的复合判断都不行。

**为什么 DCL 必须 `volatile`**：`instance = new Singleton()` 编译成三步——① 分配内存 ② 初始化对象 ③ 引用指向内存。②③ 可能重排，导致另一个线程看到**非 null 但字段全是默认值**的对象。`volatile` 的 StoreStore 屏障禁止了这个重排。

### 2.4 线程协作：wait/notify、join、interrupt

#### 2.4.1 标准范式（必须背下来）

```java
synchronized (lock) {
    while (!condition) {          // 必须 while，不能 if
        lock.wait();              // 释放锁并挂起
    }
    // 条件满足，执行
}

synchronized (lock) {
    condition = true;
    lock.notifyAll();             // 优先 notifyAll，notify 可能唤醒错的对象
}
```

**为什么必须 `while`**：① **虚假唤醒**（JVM 允许无 notify 返回）；② 被唤醒后条件可能又被别的线程改回去了。用 `if` 是并发 bug 的经典来源。

**为什么优先 `notifyAll`**：`notify` 随机唤醒一个等待者，如果它等的不是这个条件，就会"唤醒丢失"导致其他线程永久等待。

#### 2.4.2 四种"等待"的区别

| 方法 | 释放锁 | 可被中断 | 需要 notify | 属于 |
| :--- | :--- | :--- | :--- | :--- |
| `wait()` | ✅ | ✅ | ✅ | Object |
| `sleep(ms)` | ❌ | ✅ | ❌ | Thread |
| `join()` | ✅（内部 wait） | ✅ | 线程结束时自动 notify | Thread |
| `park()` | ❌（但可先释放锁再调用） | ✅ | `unpark` | LockSupport |

#### 2.4.3 `interrupt()` 的正确理解

`interrupt()` **不是强制停止线程**，而是设置中断标志：

- 线程处于 `sleep/wait/join/park` → 抛 `InterruptedException` 并**清除中断标志**；
- 线程在正常运行 → 只是置位，需要代码自己检查 `Thread.currentThread().isInterrupted()`。

正确范式：

```java
try {
    while (!Thread.currentThread().isInterrupted()) {
        doWork();
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();   // 恢复中断标志，交给上层处理
    // 清理资源后退出
}
```

**吞掉 `InterruptedException` 不恢复标志**，是线程池里"任务取消不生效"的最常见原因。

### 2.5 死锁：四个必要条件与破解

| 条件 | 含义 | 破解 |
| :--- | :--- | :--- |
| 互斥 | 资源独占 | 尽量用无锁结构 |
| 持有并等待 | 拿着 A 等 B | **一次性申请所有资源** |
| 不可剥夺 | 不能强制抢 | `tryLock(timeout)` 超时放弃 |
| 循环等待 | A 等 B，B 等 A | **全局锁顺序**（最实用） |

**排查命令**：

```bash
jstack <pid> | grep -A 20 "Found one Java-level deadlock"
jcmd <pid> Thread.print -l          # -l 会打印锁信息
```

**代码层检测**：`ThreadMXBean.findDeadlockedThreads()`，可在监控里定时跑。

> **真实场景**：两个 Service 互相调用，各自持有数据库行锁再请求对方接口，形成"数据库锁 + 应用锁"的混合死锁。`jstack` 只能看到应用锁，数据库侧要看 `SHOW ENGINE INNODB STATUS` 的 `LATEST DETECTED DEADLOCK`——**跨系统死锁要两边一起看**。

### 2.6 原子类：从 CAS 到分段

| 类别 | 代表 | 适用 |
| :--- | :--- | :--- |
| 基本类型 | `AtomicInteger` / `AtomicLong` / `AtomicBoolean` | 单变量计数 |
| 引用类型 | `AtomicReference` / `AtomicStampedReference`（带版本，解 ABA）/ `AtomicMarkableReference` | 状态机、无锁栈 |
| 数组 | `AtomicIntegerArray` / `AtomicLongArray` | 按索引原子更新 |
| 字段更新器 | `AtomicIntegerFieldUpdater` | 已有类不想改造成原子类 |
| 累加器（JDK8） | `LongAdder` / `LongAccumulator` / `DoubleAdder` | **高并发计数**，比 `AtomicLong` 快数倍 |

**选型判据**：读多写少用 `AtomicLong`；**高并发写多**用 `LongAdder`（`sum()` 时把 `base + Cell[]` 累加，弱一致）。

### 2.7 AQS 家族：六个组件的对照

| 组件 | `state` 含义 | 模式 | 是否可重用 | 典型场景 |
| :--- | :--- | :--- | :--- | :--- |
| `ReentrantLock` | 重入次数（0=自由） | 独占 | ✅ | 替代 `synchronized`，需要 `tryLock`/公平/中断 |
| `ReentrantReadWriteLock` | 高 16 位读锁数 + 低 16 位写锁重入数 | 共享 + 独占 | ✅ | 读多写少（注意写饥饿） |
| `Semaphore` | 剩余许可数 | 共享 | ✅ | 限流、资源池 |
| `CountDownLatch` | 剩余计数 | 共享 | ❌ 一次性 | 主线程等多个子任务完成 |
| `CyclicBarrier` | 到达线程数（用 ReentrantLock+Condition 实现，非 AQS） | — | ✅ 可循环 | 分批并行计算 |
| `StampedLock` | 版本号 stamp（非 AQS） | 乐观读/悲观读写 | ✅ | 读极多写极少，注意**不可重入** |

**`CountDownLatch` 与 `CyclicBarrier` 的本质区别**：

- `CountDownLatch` 是"**一个线程等多个线程**"（主等子），计数到 0 后永远为 0；
- `CyclicBarrier` 是"**多个线程互相等**"（一批到齐再一起走），有 `generation` 概念可循环，还能注册 `barrierAction`。

### 2.8 并发容器：选型表

| 容器 | 线程安全机制 | 读性能 | 写性能 | 适用 |
| :--- | :--- | :--- | :--- | :--- |
| `ConcurrentHashMap` | CAS + `synchronized` 锁桶头 | 无锁 | 桶级锁 | 通用缓存/映射（**首选**） |
| `CopyOnWriteArrayList` | 写时复制整个数组 | 无锁、极快 | O(n) 复制 | **读极多写极少**（监听器列表、白名单） |
| `ConcurrentLinkedQueue` | CAS 无锁链表 | 快 | 快 | 高并发无界队列 |
| `ArrayBlockingQueue` | `ReentrantLock` + 两个 Condition | — | — | 有界、需要阻塞（线程池） |
| `LinkedBlockingQueue` | 两把锁（putLock/takeLock） | — | — | 默认无界（**慎用**，易 OOM） |
| `SynchronousQueue` | 配对传递 | — | — | 不存储，直接交接（`newCachedThreadPool`） |
| `PriorityBlockingQueue` | 堆 + 锁 | — | — | 优先级任务 |
| `DelayQueue` | 优先队列 + 时间 | — | — | 延时任务、订单超时取消 |

**三个高频追问**：

1. **`ConcurrentHashMap` 为什么不允许 null 键值？** 因为并发下 `get(k)==null` 无法区分"不存在"和"值就是 null"，两者需要不同处理（前者可能要加锁加载），歧义无法消除。`HashMap` 单线程可以用 `containsKey` 区分，并发下这个判断已失效。
2. **`CopyOnWriteArrayList` 的坑**：只保证**最终一致**，迭代器是快照（看不到新元素）；写操作要复制整个数组，写多时 CPU/GC 压力大。
3. **`Collections.synchronizedList` 与 `CopyOnWriteArrayList` 的区别**：前者是**装饰器 + 单锁**，读写都互斥；后者读不加锁。前者适合写多的场景。

### 2.9 CompletableFuture：异步编排的正确姿势

```java
CompletableFuture<Order> f = CompletableFuture
    .supplyAsync(() -> queryOrder(id), bizPool)              // 必须指定线程池！
    .thenCompose(o -> CompletableFuture.supplyAsync(         // 串联另一个异步任务
            () -> queryUser(o.getUserId()), bizPool))
    .thenCombine(CompletableFuture.supplyAsync(              // 并行合并
            () -> queryLimit(id), bizPool), OrderVO::of)
    .orTimeout(800, TimeUnit.MILLISECONDS)                   // 超时（JDK9+）
    .exceptionally(e -> OrderVO.fallback(id));               // 兜底
```

**四条铁律**：

1. **绝不省略线程池**。默认用 `ForkJoinPool.commonPool()`，它是**全 JVM 共享、线程数 = CPU-1**，一个慢任务能拖垮所有依赖它的业务。
2. **`thenApply` vs `thenCompose`**：前者是同步映射（`T→U`），后者是异步扁平化（`T→CompletableFuture<U>`）。返回 `CompletableFuture` 时用错 `thenApply` 会得到嵌套的 `CompletableFuture<CompletableFuture<U>>`。
3. **`thenApply` 与 `thenApplyAsync`**：前者可能在上一个任务的线程执行（甚至是调用线程），后者一定提交到线程池。
4. **异常处理三件套**：`exceptionally`（只处理异常，返回同类型）、`handle`（正常/异常都处理）、`whenComplete`（不改变结果，只做副作用）。

### 2.10 十二条入门误区

1. "`synchronized` 锁的是代码块"——锁的是**对象**。
2. "`volatile` 保证原子性"——只保证可见性 + 有序性。
3. "`wait` 和 `sleep` 都是让线程等待，差不多"——`wait` 释放锁，`sleep` 不释放。
4. "`notify` 比 `notifyAll` 高效"——除非只有一个等待者且条件相同，否则会丢唤醒。
5. "线程越多吞吐越高"——受 CPU 核数、锁竞争、上下文切换限制。
6. "`ConcurrentHashMap` 完全无锁"——JDK 8 是"读无锁 + 写桶级 `synchronized`"。
7. "`AtomicInteger` 一定比 `synchronized` 快"——低竞争快，高竞争时 CAS 自旋失败率高，`LongAdder` 更好。
8. "`ReentrantLock` 比 `synchronized` 好"——`synchronized` 有锁消除/锁粗化/偏向锁（历史）等 JVM 优化，简单场景更省心。
9. "线程池 `submit` 和 `execute` 一样"——`submit` 会把异常吞进 `Future`，不 `get` 就永远看不到。
10. "`ThreadLocal` 是给线程存副本"——数据存在 `Thread.threadLocals` 里，`ThreadLocal` 只是 key。
11. "`BLOCKED` 就是等锁"——只有 `synchronized` 是 `BLOCKED`，`Lock` 是 `WAITING`。
12. "虚拟线程能提升 CPU 密集任务性能"——它解决的是**阻塞导致的线程占用**，CPU 密集场景没有任何收益。

---

## 三、技术原理（怎么工作的）

### 3.1 JMM：一套"允许你少想一点"的规范

#### 3.1.1 主内存与工作内存

JMM 抽象出两层：

```
线程A                线程B
┌──────────┐        ┌──────────┐
│ 工作内存  │        │ 工作内存  │   ← 抽象概念，对应 CPU 寄存器/缓存/写缓冲
│ (副本)    │        │ (副本)    │
└────┬─────┘        └────┬─────┘
     └────────┬──────────┘
        ┌─────┴─────┐
        │  主内存    │              ← 所有变量（实例字段、静态字段、数组元素）
        └───────────┘
```

**注意**：工作内存不是"JVM 里的一块内存"，而是**CPU 缓存 + 寄存器 + Store Buffer 的抽象**。所以 JMM 的问题本质上是**硬件内存模型的问题**，JMM 只是给了统一的编程契约。

#### 3.1.2 八种原子操作（规范层面的"指令集"）

| 操作 | 作用 |
| :--- | :--- |
| `lock` | 把主内存变量标记为线程独占 |
| `unlock` | 释放独占 |
| `read` | 从主内存读到工作内存（传输） |
| `load` | 把 read 的值放入工作内存副本 |
| `use` | 把副本值传给执行引擎 |
| `assign` | 把执行引擎结果赋给副本 |
| `store` | 把副本值传到主内存（传输） |
| `write` | 把 store 的值写入主内存变量 |

规范要求 `read+load`、`store+write` **必须成对顺序出现**，且 `assign` 后必须 `store+write` 才同步回主存——这就是"可见性问题的规范级解释"。JDK 9 之后 JMM 章节改用"变量引用/读取/写入"的表述，八种操作成为历史提法，但很多资料仍在用，看到时可以补一句"新规范用更抽象的术语表述"。

#### 3.1.3 happens-before：八条规则

**定义**：若 A happens-before B，则 A 的结果对 B 可见，且 A 在 B 之前执行（对程序员可见的顺序）。**它不是"时间先后"，而是"可见性保证"**——两个没有 hb 关系的操作，JVM 可以任意重排。

| # | 规则 | 说明 |
| :--- | :--- | :--- |
| 1 | **程序顺序** | 单线程内，前面的操作 hb 后面的（as-if-serial 允许重排但不改变结果） |
| 2 | **监视器锁** | 对同一个锁的 `unlock` hb 后续的 `lock` |
| 3 | **volatile** | 对 volatile 变量的写 hb 后续的读 |
| 4 | **线程启动** | `Thread.start()` hb 该线程内的任何操作 |
| 5 | **线程终止** | 线程内所有操作 hb 其他线程从 `join()` 返回 |
| 6 | **中断** | `interrupt()` hb 被中断线程检测到中断 |
| 7 | **对象终结** | 构造函数结束 hb `finalize()` 开始 |
| 8 | **传递性** | A hb B，B hb C ⇒ A hb C |

**用法示例（传递性最常考）**：

```java
int x = 0;                       // 普通变量
volatile boolean flag = false;

// 线程A
x = 42;                          // ①
flag = true;                     // ② volatile 写

// 线程B
if (flag) {                      // ③ volatile 读
    print(x);                    // ④ 一定输出 42
}
```

② hb ③（volatile 规则），① hb ②（程序顺序），③ hb ④（程序顺序）→ ① hb ④。**这就是 volatile 的"捎带可见性"**：它不只保证自己可见，还保证"之前的所有写"对"之后的所有读"可见。

#### 3.1.4 as-if-serial 与顺序一致性

- **as-if-serial**：单线程内怎么重排都行，只要结果不变。这是编译器优化的合法性来源。
- **顺序一致性模型**：多线程下所有操作按一个全局顺序执行，且每个线程内的顺序与程序顺序一致。**JMM 不保证顺序一致性**（否则性能无法接受），只保证正确同步的程序具有顺序一致性（**DCL 少了 volatile 就属于"未正确同步"**）。

### 3.2 内存屏障：JMM 落到 CPU 上的四种指令

| 屏障 | 作用 | x86 是否需要指令 |
| :--- | :--- | :--- |
| `LoadLoad` | 禁止上面的读与下面的读重排 | 否（TSO 天然保证） |
| `StoreStore` | 禁止上面的写与下面的写重排 | 否 |
| `LoadStore` | 禁止上面的读与下面的写重排 | 否 |
| **`StoreLoad`** | 禁止上面的写与下面的读重排 | **是**（`lock addl` / `mfence`） |

#### 3.2.1 volatile 的屏障插入策略

| 位置 | 插入的屏障 | 目的 |
| :--- | :--- | :--- |
| volatile 写之前 | `StoreStore` | 保证前面的普通写先刷出 |
| volatile 写之后 | **`StoreLoad`** | 保证写对后续读可见（代价最高） |
| volatile 读之后 | `LoadLoad` + `LoadStore` | 保证后面的读写不被提到读之前 |

**这就解释了为什么 `volatile` 写比读贵**：写要付 `StoreLoad`（`lock` 前缀会锁缓存行、刷写缓冲），读在 x86 上几乎零成本。

#### 3.2.2 `lock` 前缀到底做了什么

x86 的 `lock addl $0, (%rsp)`（HotSpot 用于实现 StoreLoad）会：

1. 把当前 CPU 的**写缓冲（Store Buffer）全部刷到缓存**；
2. 通过缓存一致性协议（MESI）**使其他核的对应缓存行失效**；
3. 禁止该指令前后的读写重排。

**成本量级**：无竞争时约 20~50 周期，跨 NUMA 节点可达数百周期。这也是为什么"高频 `volatile` 写"会成为性能瓶颈——不是逻辑问题，是缓存行来回失效。

#### 3.2.3 `final` 的两个额外屏障

JMM 对 `final` 字段额外规定：

- **写 final 字段**：构造函数内对 final 的写，与"把 this 引用赋给其他变量"之间插入 `StoreStore`；
- **读 final 字段**：首次读对象引用与首次读 final 字段之间插入 `LoadLoad`。

**效果**：只要对象在构造期间没有逸出（this 没被 `this` 逃逸），其他线程看到引用时一定能看到 final 字段的正确值。这是**不可变对象天然线程安全**的底层依据。

### 3.3 synchronized 的底层：Monitor 与锁升级

#### 3.3.1 字节码层

```java
synchronized (lock) { doSomething(); }
```

编译为：

```
monitorenter
  ... doSomething() 的字节码 ...
monitorexit          // 正常路径
goto done
astore_2             // 异常路径
aload_3
monitorexit          // 异常路径也必须释放
athrow
done:
```

方法级 `synchronized` 不加指令，靠方法访问标志 `ACC_SYNCHRONIZED`，JVM 在方法调用时隐式 `monitorenter/exit`。

#### 3.3.2 ObjectMonitor 的结构

重量级锁对应的 C++ 对象（`objectMonitor.hpp`）：

```cpp
class ObjectMonitor {
    volatile markOop   _header;      // 原 Mark Word（Displaced Header）
    void* volatile     _object;      // 关联的 Java 对象
    void* volatile     _owner;       // 持有锁的线程
    volatile intx      _recursions;  // 重入次数
    volatile intx      _count;       // 近似竞争度（用于自旋决策）
    ObjectWaiter* volatile _WaitSet;  // 调用 wait() 的线程（双向链表）
    ObjectWaiter* volatile _EntryList;// 等待锁的线程（blocked）
    ObjectWaiter* volatile _cxq;      // 竞争队列（新来的线程先入这里）
};
```

**四个队列的关系**（这是"能讲清 monitor"的分水岭）：

```
线程调用 synchronized
   ↓ 抢锁失败
  _cxq（单向，LIFO 压栈）── 被唤醒时批量转移到 ──→ _EntryList（双向，可调顺序）
                                                      ↓ 抢到锁
                                                    _owner
   wait() → 释放锁 → 进入 _WaitSet ── notify 后 ──→ 转移到 _EntryList 重新竞争
```

- **`_cxq` 是 LIFO**（后进先出，对刚释放锁的线程有利——缓存还热）；
- **`_EntryList` 可调整**（用于实现近似公平，避免饥饿）；
- **`_WaitSet` 只存 `wait()` 的线程**，`notify` 必须先把节点从 `_WaitSet` 搬到 `_EntryList`。

`ObjectMonitor::enter` 的路径：

```cpp
void ObjectMonitor::enter(TRAPS) {
    if (CAS(&_owner, NULL, Self)) { _recursions = 1; return; }  // ① 无竞争直接 CAS
    if (_owner == Self) { _recursions++; return; }               // ② 重入
    // ③ 复杂路径：自适应自旋 → 失败则 park 排队
}
```

#### 3.3.3 锁升级的完整路径

```
无锁（Mark Word: hash + age + 01）
  │ 第一个线程进入，CAS 写入 ThreadID
  ↓
偏向锁（ThreadID + epoch + 01）        ← JDK 15 起默认禁用，JDK 18 移除
  │ 第二个线程来了，偏向撤销（需 safepoint）
  ↓
轻量级锁（指向栈中 Lock Record 的指针 + 00）
  │ CAS 失败 → 自适应自旋 → 仍失败
  ↓
重量级锁（指向 ObjectMonitor 的指针 + 10）→ 内核 park/unpark
```

**逐级细节**：

| 阶段 | 关键动作 | 成本 |
| :--- | :--- | :--- |
| 偏向 | 首次 CAS 把线程 ID 写进 Mark Word；之后同一线程进入**零指令开销**（只比对 ID） | 一次 CAS（仅首次） |
| 偏向撤销 | 需在**安全点**进行（要修改栈上 Lock Record）；`-XX:BiasedLockingBulkRebiasThreshold=20` 次撤销后批量重偏向，`BulkRevokeThreshold=40` 次后批量撤销（该类彻底禁用偏向） | 高（STW） |
| 轻量级 | 在当前栈帧分配 `Lock Record`，把 Mark Word **复制**进去（Displaced Mark Word），再 CAS 把对象头指向该 Record | 一次 CAS |
| 轻量级解锁 | CAS 把 Displaced Mark Word 换回去；**失败说明有竞争 → 膨胀为重量级** | 一次 CAS |
| 重量级 | 进入 `ObjectMonitor`，`park` 挂起，涉及用户态→内核态切换 | 微秒级 |

**为什么 JDK 15 废弃偏向锁（JEP 374）**：撤销需要 safepoint、与 `hashCode`/`wait` 语义冲突、现代应用几乎都有多线程竞争（偏向命中率低），维护成本大于收益。**关键原因是「撤销代价高 + 现代场景命中率低」，而不是"因为 synchronized 变快了"这类含糊说法。**

#### 3.3.4 锁消除与锁粗化

JIT 的两个反向优化：

- **锁消除**：逃逸分析发现锁对象不逃逸（如方法内 `new Object()` 加锁），直接去掉 `monitorenter`。`StringBuffer` 的 `append` 在局部变量里就是这样被优化的。
- **锁粗化**：连续的加解锁合并成一次（如循环内 `synchronized`），减少反复进入的代价。

### 3.4 CAS：无锁的基石与它的三个缺陷

#### 3.4.1 底层链路

```java
// AtomicInteger
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

```
Unsafe.compareAndSwapInt (native)
  → Atomic::cmpxchg (C++，平台相关)
      → x86: lock cmpxchg [addr], new     ← lock 前缀保证多核原子
      → ARM: ldaxr / stlxr (LL/SC 循环)
```

**为什么必须 `lock`**：`cmpxchg` 单条指令只保证"比较+交换"不被打断，但多核下仍需 `lock` 保证缓存一致性（其他核看到的是同一份数据）。

#### 3.4.2 三个缺陷

| 缺陷 | 表现 | 解法 |
| :--- | :--- | :--- |
| **ABA 问题** | A→B→A，CAS 以为没变过 | `AtomicStampedReference`（加版本号）、`AtomicMarkableReference` |
| **自旋开销** | 高竞争下大量 CAS 失败空转，CPU 飙升 | `LongAdder` 分段、退化为锁、`-XX:PreSpin` 类自适应自旋 |
| **只能保证单变量原子** | 多个变量要一起原子更新 | `AtomicReference` 打包成对象、加锁、`StampedLock` |

**伪代码看清 ABA 的危害**：

```java
// 无锁栈：head = A → B → C
Thread1: pop() 读到 head=A，准备 CAS(A, B) 前被挂起
Thread2: pop A, pop B, push A   // 栈变回 A → C
Thread1: CAS(A, B) 成功 —— 但 B 已经被弹出过，B.next 可能已被复用
```

#### 3.4.3 `LongAdder` 为什么快

```java
// 简化结构
class LongAdder {
    volatile long base;              // 无竞争时直接用
    Cell[] cells;                    // 竞争时按线程哈希分散到不同 Cell
    // Cell 内部：@Contended 填充，独占缓存行（避免伪共享）
}
```

- 无竞争：只更新 `base`（一次 CAS）；
- 有竞争：每个线程按 `probe` 值选一个 `Cell` 更新，**把一次热点 CAS 分散成 N 次**；
- `sum()`：`base + Σcells`，**不加锁，所以是弱一致**（读取瞬间可能有线程正在写）。

**代价**：`sum()` 不是精确值，且内存占用更大。适合"计数展示"，不适合"必须精确的序号生成"。

### 3.5 AQS：一个 state 加一条队列撑起半个 JUC

#### 3.5.1 骨架

```java
public abstract class AbstractQueuedSynchronizer {
    private volatile int state;          // 同步状态，含义由子类定义
    private transient volatile Node head; // 队列头（哨兵，不代表等待线程）
    private transient volatile Node tail; // 队列尾
    // Node: prev, next, thread, waitStatus
    //   CANCELLED=1, SIGNAL=-1, CONDITION=-2, PROPAGATE=-3
}
```

AQS 只做三件事，把"能不能拿到资源"留给子类：

| 留给子类（模板方法） | AQS 负责（骨架逻辑） |
| :--- | :--- |
| `tryAcquire` / `tryRelease` | 入队、出队、park/unpark |
| `tryAcquireShared` / `tryReleaseShared` | 中断处理、超时处理 |
| `isHeldExclusively` | 状态维护与传播 |

#### 3.5.2 `acquire` 全链路（独占模式）

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&                        // ① 先试一次（非公平锁在这里插队）
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))  // ② 失败才入队
        selfInterrupt();                           // ③ 补中断标志
}
```

**① `tryAcquire` 先试一次**——这就是非公平锁的"插队"来源：新来的线程不管队列里有没有人，先抢一把。

**② `addWaiter`**：

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {       // CAS 抢尾
            pred.next = node;                      // 成功后串上（这一步非原子，所以 next 可能短暂不可靠）
            return node;
        }
    }
    enq(node);                                     // 队列为空或 CAS 失败 → 自旋初始化
    return node;
}
```

**③ `acquireQueued`**：

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean interrupted = false;
    for (;;) {
        final Node p = node.predecessor();
        if (p == head && tryAcquire(arg)) {        // 只有前驱是 head 才有资格再试（避免无谓竞争）
            setHead(node);                          // 出队：自己变 head，thread=null
            p.next = null;
            return interrupted;
        }
        if (shouldParkAfterFailedAcquire(p, node) &&  // 前驱状态设为 SIGNAL 后
            parkAndCheckInterrupt())                  // 才真正 park
            interrupted = true;
    }
}
```

**`shouldParkAfterFailedAcquire` 的三分支**（很能体现"会不会读源码"）：

| 前驱 `waitStatus` | 处理 |
| :--- | :--- |
| `SIGNAL(-1)` | 返回 true → 可以安全 park（前驱释放时会 unpark 我） |
| `> 0`（CANCELLED） | 往前跳过所有取消节点（`do-while` 找有效前驱） |
| 其他（0 / PROPAGATE） | CAS 设为 `SIGNAL`，返回 false（**下一轮再 park**，避免"刚设完就被唤醒"的竞态） |

**④ 释放 `release`**：

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);       // 唤醒 head 的后继
        return true;
    }
    return false;
}
```

**`unparkSuccessor` 为什么从 tail 往前找**：`addWaiter` 里 `pred.next = node` 不是原子的，`next` 可能断链；而 `prev` 一定可靠（先赋值再 CAS）。所以找后继要用 `prev` 反向遍历——**这是 AQS 里最容易被追问的细节**。

#### 3.5.3 共享模式：`PROPAGATE` 的意义

`Semaphore`/`CountDownLatch` 用共享模式：

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)          // 返回值 <0 失败，=0 成功但无剩余，>0 成功且有剩余
        doAcquireShared(arg);
}
```

**`PROPAGATE` 的作用**：解决"释放时唤醒一个线程，但多个线程都该被唤醒"的竞态。`releaseShared` 后把 head 设为 `PROPAGATE`，让后续 `shouldParkAfterFailedAcquire` 继续传播唤醒，直到所有能拿到资源的线程都醒来。

### 3.6 Condition：两个队列之间的搬运

`ConditionObject` 内部维护一条**单向条件队列**（`firstWaiter`/`lastWaiter`，节点复用 AQS 的 `Node`，`waitStatus=CONDITION`）。

#### 3.6.1 `await()` 的四步

```java
public final void await() throws InterruptedException {
    int savedState = fullyRelease(node);        // ① 完全释放锁（重入 N 次就释放 N 次）
    while (!isOnSyncQueue(node)) {              // ② 只要还在条件队列
        LockSupport.park(this);                 //    就 park
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0) break;
    }
    if (acquireQueued(node, savedState) && ...) // ③ 被 signal 后重新排队抢锁
        ...
    // ④ 恢复中断状态
}
```

**关键点**：`await` 必须 `fullyRelease`（释放全部重入计数），否则其他线程永远拿不到锁；恢复时再 `acquireQueued(node, savedState)` 把重入次数补回来。

#### 3.6.2 `signal()` 的搬运

```java
public final void signal() {
    if (!isHeldExclusively()) throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null) doSignal(first);
}
// doSignal → transferForSignal：
//   1) CAS 把节点 waitStatus 从 CONDITION 改为 0
//   2) 节点从条件队列摘下，enq 到同步队列
//   3) 若前驱 CANCELLED 或 CAS 设 SIGNAL 失败 → 直接 unpark（避免丢失唤醒）
```

**注意**：`signal` 只是"搬家"（条件队列 → 同步队列），**不释放锁，也不立即唤醒执行**；被唤醒的线程要重新竞争锁，抢到后才从 `await` 返回。这解释了"`signal` 之后必须释放锁，被等待的线程才有机会跑"。

### 3.7 公平锁与非公平锁的源码差异

```java
// NonfairSync
final boolean nonfairTryAcquire(int acquires) {
    if (compareAndSetState(0, acquires)) {     // 直接抢，不看队列
        setExclusiveOwnerThread(current); return true;
    }
    ...
}

// FairSync
protected final boolean tryAcquire(int acquires) {
    if (getState() == 0) {
        if (!hasQueuedPredecessors() &&        // 必须队列里没有更早的等待者
            compareAndSetState(0, acquires)) { ... }
    }
    ...
}
```

`hasQueuedPredecessors()` 的逻辑：`h != t && ((s = h.next) == null || s.thread != current)`。**注意 `h.next == null` 也算"有前驱"**——这是为了处理"正在入队、next 还没连上"的中间态。

**性能差异**：非公平锁吞吐更高（减少唤醒后的上下文切换、利用刚释放锁线程的缓存热度），但可能饥饿；公平锁保证 FIFO，代价是更多切换。**默认用非公平**。

### 3.8 ConcurrentHashMap：JDK 8 的三处关键设计

#### 3.8.1 读为什么无锁

```java
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectAcquire(tab, ((long)i << ASHIFT) + ABASE);  // volatile 语义读
}
```

- `Node.val` 是 `volatile`，`next` 也是 `volatile`；
- 扩容时旧表桶头会被替换为 `ForwardingNode`（`hash = MOVED = -1`），读线程看到它就**去新表读**（`helpTransfer` 的读版本）。

所以读操作**完全无锁**，且扩容期间也能读到正确数据。

#### 3.8.2 写为什么用桶级 `synchronized`

```java
// putVal 核心
for (Node<K,V>[] tab = table;;) {
    if (tab == null || (n = tab.length) == 0) tab = initTable();
    else if ((f = tabAt(tab, i = (n-1) & hash)) == null) {
        if (casTabAt(tab, i, null, new Node<K,V>(hash, key, val)))  // ① 空桶 CAS 直接放
            break;
    }
    else if ((fh = f.hash) == MOVED) tab = helpTransfer(tab, f);     // ② 帮扩容
    else {
        synchronized (f) {                                            // ③ 只锁桶头节点
            // 链表插入 / 红黑树插入
        }
    }
}
```

**为什么是 `synchronized(f)` 而不是 `ReentrantLock`**：JDK 8 时 `synchronized` 已优化得足够好（锁消除、轻量级），而且**每个桶一个锁对象**天然分段，比维护一堆 Lock 对象省内存。JDK 7 的 Segment 分段锁（默认 16 段）则被废弃——**段数固定导致并发度上限固定**。

#### 3.8.3 `size()` 为什么是弱一致

```java
long sumCount() {
    CounterCell[] as = counterCells;
    long sum = baseCount;
    if (as != null) for (CounterCell a : as) if (a != null) sum += a.value;
    return sum;
}
```

- 本质是 `LongAdder` 的同款思路：`baseCount` + `CounterCell[]` 分散热点；
- 无锁读取，所以**返回的是近似值**（Javadoc 明确写"may be inaccurate"）；
- `size()` 在并发场景只能用于监控，**不能用于业务判断**（如"容量到 100 就拒绝"必须用 `mappingCount()` 配合其他机制或直接加锁）。

#### 3.8.4 扩容：多线程协助

```
sizeCtl 的语义：
  > 0  ：下一次扩容的阈值（初始为容量 × 0.75）
  = 0  ：未初始化
  = -1 ：正在初始化
  < -1 ：正在扩容，值为 -(1 + 参与扩容的线程数)
```

`transfer` 把旧表按 stride 分段，每个线程认领一段搬迁；搬完的桶放 `ForwardingNode`。**这就是"多线程协助扩容"**（`helpTransfer`）——也是 CHM 在扩容期间仍能读写的关键。

#### 3.8.5 树化阈值

| 阈值 | 值 | 说明 |
| :--- | :--- | :--- |
| `TREEIFY_THRESHOLD` | 8 | 链表长度 ≥ 8 尝试树化 |
| `UNTREEIFY_THRESHOLD` | 6 | 树节点 ≤ 6 退化回链表 |
| `MIN_TREEIFY_CAPACITY` | 64 | **表容量 < 64 时只扩容不树化** |

**为什么阈值是 8**：泊松分布下，哈希均匀时链表长度达到 8 的概率约 `0.00000006`，正常情况几乎不会树化；树化是"防哈希碰撞攻击"的兜底（`String` 的 hash 可被构造碰撞）。

### 3.9 虚拟线程（JDK 21）：把阻塞变便宜

#### 3.9.1 模型

```
虚拟线程 V1..Vn  (千万级)
      ↓ mount / unmount
载体线程 Carrier Thread（= CPU 核数个平台线程，ForkJoinPool 调度）
      ↓
      OS 线程
```

- **创建**：`Thread.ofVirtual().start(runnable)`，或 `Executors.newVirtualThreadPerTaskExecutor()`；
- **调度**：JVM 自己的 `ForkJoinPool`（默认并行度 = CPU 核数），**不是 OS 调度**；
- **阻塞**：遇到 IO/`park`/`sleep` 时，虚拟线程**卸载（unmount）**，载体线程去跑别的虚拟线程；IO 就绪后再**挂载（mount）**回来；
- **栈**：存在堆上的 `Continuation` 里，按需增长（不是固定 1M），所以能创建百万级。

#### 3.9.2 `pin`（钉住）：虚拟线程的软肋

以下情况虚拟线程**无法卸载**，会一直占着载体线程：

| 场景 | 原因 | 规避 |
| :--- | :--- | :--- |
| `synchronized` 块内阻塞 | monitor 与载体线程绑定（JDK 21 限制，JDK 24 起改善） | 改用 `ReentrantLock` |
| native 方法 / JNI 调用 | 无法保存 native 栈 | 减少 JNI，或用平台线程池隔离 |
| 类初始化（`<clinit>`）阻塞 | 类初始化锁与载体线程绑定 | 避免静态块里做 IO |

**诊断**：`-Djdk.tracePinnedThreads=full` 打印被钉住的栈。

#### 3.9.3 三个必须说清的边界

1. **不提升 CPU 密集性能**：没有阻塞就没有卸载收益，反而多一层调度开销。
2. **不要池化虚拟线程**：创建成本极低，池化反而引入排队与状态复用问题（`ThreadLocal` 串号）。**"每个请求一个虚拟线程"是正确姿势**。
3. **`ThreadLocal` 仍可用但要谨慎**：百万虚拟线程 × 每个 ThreadLocal = 内存爆炸；JDK 21 提供 `ScopedValue`（预览）作为替代。

**一句话总结**：虚拟线程把"线程"从**稀缺的 OS 资源**变成**廉价的 Java 对象**，让"一请求一线程"的同步阻塞写法重新变得可行，代价是 pin 场景与调试复杂度。

---

## 四、深入与高级延伸（生产怎么用）

### 4.1 四个真实可见性 Bug 的形态

#### 4.1.1 无锁退出标志（最经典）

```java
// 错误
boolean running = true;
new Thread(() -> { while (running) { doWork(); } }).start();
running = false;   // 子线程可能永远看不到，进程不退出

// 正确：volatile 或 AtomicBoolean
```

**为什么线上难复现**：JIT 把 `running` 提升到寄存器（循环不变量外提），只有在 C2 编译后才暴露，测试环境（解释执行/C1）一切正常。**这类 bug 的特征是"上线跑一段时间才出现"**。

#### 4.1.2 对象逸出（半初始化对象被发布）

```java
// 错误：this 在构造完成前逸出
public class UnsafePublish {
    private int x;
    public UnsafePublish() {
        new Thread(() -> System.out.println(this.x)).start();  // 可能打印 0
        x = 42;
    }
}
```

**修复**：不要在构造方法里启动线程 / 注册监听器 / 把 `this` 传给外部方法；必须这样做时用工厂方法（构造完成后发布）。

#### 4.1.3 非线程安全的延迟初始化

```java
// 错误：两个线程可能都判断为 null，创建两个实例
if (instance == null) instance = new Singleton();

// 正确 A：DCL + volatile
// 正确 B：静态内部类（推荐，靠 <clinit> 的线程安全）
// 正确 C：枚举（防反射/反序列化破坏）
```

#### 4.1.4 集合的"并发修改"与"弱一致迭代"

```java
// ConcurrentModificationException（fail-fast）
for (String s : list) if (cond) list.remove(s);

// 正确：Iterator.remove / removeIf（JDK8）
list.removeIf(s -> cond);

// 注意：ConcurrentHashMap 的迭代器是弱一致的，不会抛 CME，但可能看不到新元素
```

> **判据**：`fail-fast` 是"检测到 modCount 变化就抛异常"，属于**尽力而为**（不保证一定抛）；`fail-safe`（COW、CHM）是"迭代快照/弱一致"，不抛异常但可能读到旧数据。

### 4.2 四类"线程不动了"的排查手册

#### 4.2.1 线程池打满 / 任务堆积

```bash
jstack <pid> | grep -c "java.lang.Thread.State"          # 线程总数
jstack <pid> | grep -A 5 "pool-1-thread" | head -50      # 看池内线程在干嘛
jcmd <pid> Thread.print -l                               # 带锁信息
```

| 栈特征 | 结论 |
| :--- | :--- |
| 大量线程卡在 `socketRead0` / `HttpClient` | 下游超时，**没设超时**或超时过长 |
| 大量 `WAITING (parking)` 在 `LinkedBlockingQueue.take` | 队列空、消费跟不上（也可能是线程数太少） |
| 大量 `BLOCKED` 在同一把锁 | 锁竞争，需看锁持有者在干嘛 |
| 线程数持续增长不回收 | 线程池用了 `CachedThreadPool` 或自建无界池 |

**关键指标**（必须监控）：`activeCount`、`queue.size`、`completedTaskCount`、`largestPoolSize`。**只看 CPU 是发现不了线程池打满的**。

#### 4.2.2 死锁

```bash
jstack <pid> | grep -B 2 -A 25 "Found one Java-level deadlock"
```

要区分三种：

| 类型 | 特征 | 排查 |
| :--- | :--- | :--- |
| 应用锁死锁 | `jstack` 直接报告 | 锁顺序 |
| 数据库死锁 | 应用线程卡在 JDBC，`jstack` 看不出 | `SHOW ENGINE INNODB STATUS` |
| 类初始化死锁 | 线程栈显示 `in class loading` | 静态块互相依赖 |

#### 4.2.3 活锁与饥饿

- **活锁**：线程都在运行、都在重试，但没人能推进（如两个线程互相礼让重试）。特征：CPU 高、吞吐为零。
- **饥饿**：低优先级线程永远抢不到锁（非公平锁 + 高频短任务）。特征：某些请求 P99 极高但平均正常。

**处置**：活锁加随机退避；饥饿改公平锁或限制并发。

#### 4.2.4 线程泄漏

```
线程数 = 基线 + 请求数 × 每请求新建线程数 × 未回收比例
```

常见原因：`new Thread()` 未 `start`/未结束、`Executors.newCachedThreadPool()` 在突发流量后 60s 才回收、`Timer` 不 `cancel`、自定义线程池 `shutdown` 未调用。

**判定**：连续多次 `jstack` 抓线程名，若同名线程数量单调递增 → 泄漏。

### 4.3 锁性能优化：六个层次

| 层次 | 手段 | 收益 | 代价 |
| :--- | :--- | :--- | :--- |
| 1. 消除共享 | 无状态设计、ThreadLocal、每线程独立对象 | 最大 | 内存/一致性 |
| 2. 缩小临界区 | 只锁必要语句，IO/远程调用移出锁外 | 大 | 需仔细验证 |
| 3. 降低锁粒度 | 锁分段（CHM）、按 key 哈希取锁 | 大 | 复杂度上升 |
| 4. 读写分离 | `ReentrantReadWriteLock` / `StampedLock` 乐观读 | 中（读多写少） | 写饥饿、不可重入 |
| 5. 无锁化 | CAS、`LongAdder`、`ConcurrentLinkedQueue` | 中（低竞争） | ABA、自旋开销 |
| 6. 避免伪共享 | `@Contended`、字段填充、数组按缓存行对齐 | 中（高频写） | 内存浪费 |

#### 4.3.1 伪共享实例

```java
// 两个 volatile 变量落在同一缓存行，两个线程分别写 → 缓存行来回失效
class Counter {
    volatile long a;   // 同一行
    volatile long b;   // 同一行
}

// 修复：JDK8 用 @Contended（需 -XX:-RestrictContended），或手工 padding
class PaddedCounter {
    volatile long a;
    long p1, p2, p3, p4, p5, p6, p7;   // 填充到 64B
    volatile long b;
}
```

`LongAdder` 的 `Cell` 就带 `@Contended` 注解。

#### 4.3.2 一个反直觉的结论

**低竞争时 `synchronized` 可能比 `AtomicLong` 更快**：`synchronized` 在无竞争下只有一次偏向/轻量级 CAS，而 `AtomicLong` 每次都要 `lock cmpxchg`。**不要盲目"去锁化"**，要用 JMH 测。

### 4.4 并发正确性怎么验证

| 手段 | 用途 | 说明 |
| :--- | :--- | :--- |
| **jcstress** | 验证 JMM 层面的可见性/原子性 | OpenJDK 官方工具，能构造极端交错 |
| **JMH** | 性能对比（吞吐/延迟） | 必须用，手写 `System.currentTimeMillis` 测不准（JIT 预热、死代码消除） |
| **压测**（JMeter/Gatling） | 端到端验证 | 关注 P99 而非平均值 |
| **故障注入** | 超时、异常、下游慢 | 验证降级与线程池隔离是否生效 |
| **静态检查** | SpotBugs（`IS2_INCONSISTENT_SYNC`） | 能发现"一半方法加锁一半不加" |

**一句话总结**："并发 bug 不能靠测试覆盖，只能靠**设计约束**（不可变、线程封闭、同步策略统一）+ 代码评审 + 极端场景工具（jcstress）"。

### 4.5 框架里的并发设计（能对上号就说明真用过）

| 框架 | 并发模型 | 关键点 |
| :--- | :--- | :--- |
| **Tomcat** | `ThreadPoolExecutor`（`maxThreads` 默认 200，`acceptCount` 100） | IO 线程与业务线程分离；`maxThreads` 打满后请求进 `acceptCount` 队列，再满则拒绝 |
| **Dubbo** | 默认 `fixed` 线程池（200）+ `ThreadlessExecutor` | `actives` 限流、`executes` 限并发；线程池满抛 `RejectedExecutionException` 而非阻塞 |
| **RocketMQ** | 消费线程池（`consumeThreadMin/Max`）+ 长轮询 | 消费位点提交与业务处理解耦；`ConsumeConcurrentlyStatus.RECONSUME_LATER` 进重试队列 |
| **Sentinel** | 滑动窗口（`LeapArray`）+ `LongAdder` 计数 | 统计线程安全靠 `AtomicLong` 数组分段，避免热点 |
| **HikariCP** | `ConcurrentBag`（无锁借还）+ `SynchronousQueue` | 连接借还走 `ThreadLocal` 列表（避免 CAS 竞争） |
| **Netty** | 主从 `EventLoopGroup`（`NioEventLoop` 单线程绑定多 Channel） | **一个 Channel 的所有事件在同一线程**，业务 Handler 里不能阻塞 |

**这六个例子是很好的"框架认知"素材**：讲并发实践时，"我们在 xx 场景用 Sentinel 的滑动窗口 + LongAdder 做 QPS 统计"比空谈 AQS 具体得多。

### 4.6 十二条生产高频坑

1. **`Executors.newFixedThreadPool` 用无界队列** → 任务堆积到 OOM（必须自建 + 有界队列 + 拒绝策略）。
2. **`submit` 吞异常** → 任务失败无日志，必须 `future.get()` 或包 `try-catch`。
3. **`ThreadLocal` 不 `remove`** → 线程池串号 + 内存泄漏（尤其 `InheritableThreadLocal` 在池里）。
4. **`SimpleDateFormat` 共享** → 日期错乱/`NumberFormatException`（用 `DateTimeFormatter`）。
5. **`HashMap` 并发写** → JDK 8 下可能丢数据/树化异常（不再死循环，但仍不安全）。
6. **`ConcurrentHashMap` 复合操作用 `get` + `put`** → 必须 `computeIfAbsent`/`merge`/`putIfAbsent`。
7. **`synchronized` 锁 `String` 常量/包装类** → 常量池共享对象，锁范围失控（`Integer` 缓存 -128~127）。
8. **锁内做远程调用/IO** → 临界区被放大到几百毫秒，直接拖垮吞吐。
9. **`CountDownLatch` 在 `finally` 外 `countDown`** → 异常时永久等待。
10. **`CompletableFuture` 不指定线程池** → 挤占 `commonPool`，影响全 JVM 的并行流。
11. **`wait` 用 `if` 判断条件** → 虚假唤醒导致越界/空指针。
12. **`interrupt` 被吞** → 线程池 `shutdownNow` 无法终止任务。

### 4.7 版本演进

| 版本 | 变化 | 影响 |
| :--- | :--- | :--- |
| JDK 5 | `java.util.concurrent` 引入；`synchronized` 锁升级 | 并发编程从"手写 wait/notify"进入 JUC 时代 |
| JDK 6 | 偏向锁、自适应自旋、锁消除、`ConcurrentHashMap` 分段锁 | `synchronized` 性能大幅提升 |
| JDK 7 | `ForkJoinPool`、`TransferQueue` | 分治并行 |
| **JDK 8** | `CompletableFuture`、`LongAdder`、`StampedLock`、**CHM 改为 CAS+synchronized 并引入红黑树** | 异步编排能力质变；CHM 并发度不再受段数限制 |
| JDK 9 | `VarHandle`（替代部分 `Unsafe`）、`Flow`（响应式） | 更安全的内存访问 |
| **JDK 15** | **偏向锁默认禁用并废弃（JEP 374）** | 锁升级只剩"轻量级 → 重量级" |
| JDK 17 | 密封类、强封装内部 API | 反射 `Unsafe` 受限 |
| **JDK 21** | **虚拟线程正式（JEP 444）**、`ScopedValue`（预览）、结构化并发（预览） | "一请求一线程"复兴；`ThreadLocal` 有替代方案 |
| JDK 24+ | `synchronized` 内阻塞不再 pin（JEP 491） | 虚拟线程最大软肋被修复 |

### 4.8 怎么把并发讲成"我解决过的问题"

9 年经验的答法，建议准备一个**完整的线程池治理故事**：

> "渠道中台对接了 12 家资金方，每个渠道一个 HTTP 客户端。早期共用一个 `newFixedThreadPool(50)`，2024 年某资金方接口从 200ms 劣化到 8s，50 个线程全被它占满，其他 11 个渠道一起超时——**故障扩散**。
> 排查时 `jstack` 看到 50 个线程全卡在 `socketRead0`，且 `HttpClient` 没设 `connectTimeout`。
> 改造三步：① 按渠道拆线程池（隔离），核心 8 / 最大 16 / 队列 200 / `CallerRunsPolicy`；② 所有客户端设 `connectTimeout=1s`、`readTimeout=3s`；③ 加监控，`activeCount/queueSize` 超阈值告警。
> 之后单渠道劣化只影响自己，P99 从 8s 回到 300ms 以内。"

这个故事覆盖了：**线程池参数、隔离、超时、监控、故障扩散**——比背 AQS 源码更能证明"你管过生产系统"。

---

## 五、总结整理

### 5.1 全景回顾

```
规范层
  JMM：原子性 / 可见性 / 有序性
  happens-before 八条（程序顺序、锁、volatile、start、join、interrupt、finalize、传递）

硬件层
  缓存 + 写缓冲 + 重排  →  内存屏障（x86 只有 StoreLoad 需要指令）
  lock 前缀 = 刷写缓冲 + 失效其他核缓存行

语言层
  synchronized：monitorenter/exit → Mark Word → 偏向(废弃) → 轻量级(栈 Lock Record + CAS) → 重量级(ObjectMonitor)
  volatile：可见性 + 有序性，不保证原子性
  final：构造完成前插入 StoreStore，保证安全发布

无锁层
  CAS（Unsafe.cmpxchg + lock）→ ABA（StampedReference）/ 自旋 / 单变量
  原子类：AtomicLong（低竞争） vs LongAdder（高并发，base + Cell[] + @Contended）

框架层
  AQS：state + CLH 变体双向队列
    acquire → tryAcquire → addWaiter(CAS tail) → acquireQueued(前驱是 head 才重试) → park
    release → unparkSuccessor（从 tail 反向找，因为 next 可能断链）
    Condition：条件队列 ⇄ 同步队列（signal 只搬家不释放锁）
  ReentrantLock（公平靠 hasQueuedPredecessors）
  Semaphore / CountDownLatch（共享，PROPAGATE 传播）/ CyclicBarrier（Lock+Condition）

容器层
  CHM：读无锁（volatile + ForwardingNode）/ 写桶级 synchronized / size 弱一致 / 协助扩容
  COW：写时复制，读极快，只适合读多写极少

线程层
  ThreadPoolExecutor：ctl 位运算、四步优先级、Worker 复用
  虚拟线程：Continuation + ForkJoinPool 载体，pin 在 synchronized/native/classinit
```

### 5.2 速查清单（35 条）

| # | 问题 | 一句话答案 |
| :--- | :--- | :--- |
| 1 | 并发三要素 | 原子性、可见性、有序性（外加活跃性） |
| 2 | `i++` 为什么线程不安全 | 读-改-写三步，非原子 |
| 3 | `volatile` 的语义 | 可见性 + 有序性，**不保证原子性** |
| 4 | `volatile` 怎么实现可见性 | 写后 `StoreLoad` 屏障（`lock` 前缀）+ 缓存一致性 |
| 5 | DCL 为什么要 `volatile` | 禁止"分配→初始化→赋值"重排，防半初始化对象 |
| 6 | `synchronized` 锁什么 | 实例方法锁 `this`，静态方法锁 `Class`，代码块锁括号对象 |
| 7 | `synchronized` 可重入吗 | 可，靠 `ObjectMonitor._recursions` |
| 8 | 锁升级路径 | 无锁 → 偏向（JDK15 废弃）→ 轻量级 → 重量级 |
| 9 | 轻量级锁怎么实现 | 栈帧 Lock Record 复制 Mark Word，CAS 替换对象头指针 |
| 10 | 偏向锁为什么废弃 | 撤销需 safepoint、与 hashCode 冲突、现代场景命中率低 |
| 11 | `wait` 与 `sleep` 区别 | `wait` 释放锁、属 Object；`sleep` 不释放、属 Thread |
| 12 | 为什么 `wait` 要用 `while` | 防虚假唤醒 + 条件被改回 |
| 13 | `notify` 与 `notifyAll` | 前者可能唤醒错对象导致丢唤醒，优先后者 |
| 14 | 线程六状态 | NEW / RUNNABLE / BLOCKED / WAITING / TIMED_WAITING / TERMINATED |
| 15 | BLOCKED 与 WAITING 区别 | BLOCKED 等 `synchronized`；`Lock` 等锁是 WAITING |
| 16 | `interrupt` 是停止线程吗 | 不是，是设置标志；阻塞时抛 `InterruptedException` |
| 17 | 死锁四条件 | 互斥、持有并等待、不可剥夺、循环等待 |
| 18 | 怎么排查死锁 | `jstack` 的 `Found one Java-level deadlock`、`ThreadMXBean` |
| 19 | CAS 三个问题 | ABA、自旋开销、只能单变量 |
| 20 | 怎么解 ABA | `AtomicStampedReference` 加版本号 |
| 21 | `LongAdder` 为什么快 | 分段 Cell 分散 CAS 热点 + `@Contended` 防伪共享；`sum()` 弱一致 |
| 22 | AQS 的核心 | `volatile state` + CLH 变体双向队列 + 模板方法 |
| 23 | AQS 为什么先 `tryAcquire` | 非公平锁的插队；避免无谓入队 |
| 24 | `acquireQueued` 为什么前驱是 head 才重试 | 避免所有等待者无谓竞争 |
| 25 | `unparkSuccessor` 为什么从 tail 找 | `next` 非原子可能断链，`prev` 一定可靠 |
| 26 | `PROPAGATE` 干什么 | 共享模式下传播唤醒，防漏唤醒 |
| 27 | `Condition` 的两个队列 | 条件队列（await）⇄ 同步队列（signal 搬运） |
| 28 | 公平锁怎么实现 | `hasQueuedPredecessors()` 检查有无更早等待者 |
| 29 | CHM 读为什么无锁 | `volatile` 读 + `ForwardingNode` 引导到新表 |
| 30 | CHM 为什么不允许 null | 并发下无法区分"不存在"和"值为 null" |
| 31 | CHM `size()` 准确吗 | 不准确，弱一致，不能用于业务判断 |
| 32 | COW 适用场景 | 读极多写极少（监听器列表），写复制整个数组 |
| 33 | `CountDownLatch` 与 `CyclicBarrier` | 前者一个等多、一次性；后者互相等、可循环 |
| 34 | 虚拟线程解决什么 | 阻塞时卸载而非占用 OS 线程，提升 IO 密集型吞吐 |
| 35 | 虚拟线程的 pin | `synchronized` 内阻塞、native、类初始化时无法卸载 |

### 5.3 三条追问链

**链一：从 `volatile` 追到 CPU**

> "volatile 怎么保证可见性？" → 写后插 `StoreLoad` 屏障 → "x86 上是什么指令？" → `lock addl $0,(%rsp)` → "它做了什么？" → 刷写缓冲 + 使其他核缓存行失效 + 禁止重排 → "为什么读不贵写贵？" → x86 是 TSO，只有 StoreLoad 需要显式屏障。

**链二：从 AQS 追到 `unparkSuccessor`**

> "AQS 怎么实现排队？" → state + CLH 变体队列 → "节点怎么入队？" → CAS 抢 tail，`pred.next` 后置赋值 → "既然 `next` 可能断，出队怎么办？" → `unparkSuccessor` 从 tail 用 `prev` 反向找 → "为什么不用 `next` 直接找？" → `next` 赋值非原子，存在空窗期。

**链三：从线程池打满追到故障隔离**

> "线程池打满会怎样？" → 队列满 → 拒绝策略（默认 `AbortPolicy` 抛异常）→ "如果多个业务共用一个池呢？" → 一个慢依赖拖垮全部（故障扩散）→ "怎么隔离？" → 按依赖拆池 + 有界队列 + 超时 + 熔断 → "隔离后池怎么定大小？" → 按 QPS × 平均耗时（Little's Law），再留 20% 余量 + 压测校准。

### 5.4 决策与排查速查表

| 场景 | 首选 | 关键判据 |
| :--- | :--- | :--- |
| 简单互斥 | `synchronized` | 无需超时/公平/中断 |
| 需要 `tryLock`/超时/公平 | `ReentrantLock` | 记得 `finally` 里 `unlock` |
| 读多写少 | `ReentrantReadWriteLock` / `StampedLock` | 写少且能接受写饥饿；StampedLock 不可重入 |
| 高并发计数 | `LongAdder` | 可接受 `sum()` 弱一致 |
| 缓存 Map | `ConcurrentHashMap` | 复合操作用 `computeIfAbsent`/`merge` |
| 监听器列表 | `CopyOnWriteArrayList` | 写极少 |
| 异步编排 | `CompletableFuture` + **自定义线程池** | 绝不省略线程池 |
| IO 密集且线程数受限 | 虚拟线程（JDK 21） | 确认无 pin（`synchronized` 内阻塞） |
| 线程数持续增长 | 查线程池类型与 `shutdown` | `jstack` 线程名计数 |
| 大量线程卡 `socketRead0` | 查超时配置 | `connectTimeout` / `readTimeout` |
| 线程全 `BLOCKED` | 找锁持有者 | `jstack -l` 的 `locked ownable synchronizers` |

---

> 本文基线：OpenJDK 17 源码；差异处标注 JDK 8 / 15 / 21 / 24。
