---
title: "Java 并发编程全量深度解析：从 JMM、锁升级、AQS 到线程池与 ThreadLocal 工程"
date: 2026-11-09T23:00:00+08:00
summary: "一篇把 Java 并发讲透的长文：并发三要素与活跃性、线程生命周期、synchronized 的 Monitor 与锁升级全路径、volatile 与内存屏障、AQS 的 state+队列骨架与 Condition 搬运、CAS 与 LongAdder、ConcurrentHashMap 的三处关键设计、虚拟线程，以及生产视角下的线程池七参数、ThreadLocal 内存泄露、可见性 Bug 排查与锁优化六层次。每个结论都带成立条件和代价。"
categories: ["Java 并发"]
tags: ["Java", "并发", "JMM", "AQS", "synchronized", "volatile", "CAS", "ConcurrentHashMap", "线程池", "ThreadLocal", "虚拟线程", "LongAdder"]
author: ZLLiang
ShowToc: true
TocOpen: false
draft: false
---

线上某个线程池里的任务突然"串号"了——同一个 traceId 出现在两个线程，一个请求拿到了另一个请求的用户上下文。加锁也加了，`volatile` 也标了，可问题还是复现。绝大多数并发问题都不是"不知道 AQS"，而是**没把内存可见性、锁的升级、队列的排队与唤醒串成一条连贯的链路**，于是排查时不知道先看哪。

这篇文章把 Java 并发从概念到源码链路、从 JMM 到线程池工程全量展开，默认以 OpenJDK 17 为源码锚点，JDK 8 / 15 / 21 / 24 的差异单独标注。读完你能做到三件事：定位线程堆积/死锁时知道先抓哪几个指标；说清锁升级与 AQS 排队唤醒的完整过程；以及写并发代码时能预判每个选择的**代价**——而不是只会背"用 ConcurrentHashMap 就对了"。

## 一、为什么会有并发问题：三个词、四件事与一张分层图

### 1.1 并发、并行、异步：三个词不能混用

| 概念 | 定义 | 判定标准 |
| :--- | :--- | :--- |
| **并发 Concurrency** | 多个任务**交替**推进，宏观上"同时"，微观上可能串行 | 是否有多个任务在**重叠的时间段**内推进 |
| **并行 Parallelism** | 多个任务**同一时刻**在不同 CPU 核上执行 | 是否**物理同时**执行 |
| **异步 Async** | 调用方不等结果返回就继续，结果通过回调/Future 获取 | 是否**阻塞等待** |

一句话：**并发是结构（怎么组织任务），并行是执行（谁来跑），异步是交互（要不要等）**。单核 CPU 上可以并发但不可能并行；异步可以用单线程实现（如 Netty 的 EventLoop）。

### 1.2 并发编程要解决的四件事

| 问题 | 表现 | 根源 | 解决手段 |
| :--- | :--- | :--- | :--- |
| **原子性** | `i++` 丢更新 | 读-改-写三步非原子 | `synchronized`、`CAS`、`Lock`、原子类 |
| **可见性** | 一个线程改了标志位，另一个线程永远看不到 | CPU 缓存 + 编译器优化，写不回主存/读到旧值 | `volatile`、`synchronized`、`final` |
| **有序性** | 指令重排导致半初始化对象被发布 | 编译器/CPU/内存系统三重重排 | `volatile`（内存屏障）、`happens-before` |
| **活跃性** | 死锁、活锁、饥饿、线程泄漏 | 资源竞争与调度 | 锁顺序、超时、公平锁、隔离 |

一个高频陷阱：`volatile int i; i++` 依然丢更新。因为 `i++` 是 `read → use → assign → write` 四步，`volatile` 只保证每一步读到最新值，不保证四步整体原子。要原子就用 `AtomicInteger` 或加锁。

### 1.3 进程与线程：谁私有、谁共享

线程是进程里更小的执行单位。同一进程的多个线程**共享堆和方法区**（JDK 8 起方法区落到元空间），但每个线程有自己**私有的程序计数器、虚拟机栈、本地方法栈**。

- **程序计数器私有**：记录当前线程执行到哪条字节码指令，线程切换回来能恢复正确位置。执行 native 方法时记录的是 undefined。
- **虚拟机栈/本地方法栈私有**：每个方法一个栈帧，存局部变量表、操作数栈等，私有才能保证局部变量不被别的线程读到。
- **堆/方法区共享**：堆存对象实例，方法区存类信息、常量、静态变量——这两块是并发问题的重灾区，因为所有线程都往这里读写。

这个"私有/共享"的划分，是理解后面一切可见性问题的起点：**共享的数据才需要同步，私有的数据天然安全**。

### 1.4 线程的生命周期：六状态与转换条件

```
                   start()
        NEW ──────────────────→ RUNNABLE ──────────────→ TERMINATED
                                   │  ↑
              ┌────────────────────┘  └──────────┐
              │ 进入 synchronized 未抢到           │
              ↓                                   │
          BLOCKED ────── 抢到锁 ──────────────────┘
   RUNNABLE ── wait()/join()/park() ──→ WAITING
            ←── notify()/notifyAll()/unpark() ──┘
   RUNNABLE ── sleep(t)/wait(t)/join(t) ──→ TIMED_WAITING
            ←── 超时/唤醒 ──┘
   RUNNABLE ── run() 返回或异常 ─────────────────→ TERMINATED
```

五个容易搞错的点：

1. **`RUNNABLE` 包含"正在等 IO/等 CPU 时间片"**——Java 不区分 Ready 与 Running，也不区分内核态的 IO 等待。
2. **`BLOCKED` 只针对 `synchronized`**。`ReentrantLock` 等不到锁是 `WAITING`（`LockSupport.park`），不是 `BLOCKED`。这是 `jstack` 判读的关键差别。
3. **`wait()` 必须在 `synchronized` 块内**，否则 `IllegalMonitorStateException`；它释放锁并进入对象的 WaitSet。
4. **`sleep()` 不释放锁，`wait()` 释放锁**——这是两者最本质的区别，比"一个属于 Thread 一个属于 Object"更重要。
5. **`stop()`/`suspend()` 已废弃**，正确姿势是 `interrupt()` + 标志位协作退出。

### 1.5 上下文切换：并发的隐性成本

一次上下文切换大致包含：保存当前线程寄存器/PC → 更新 TCB → 调度器选下一个 → 恢复寄存器 → 刷新 TLB/缓存局部性受损。成本量级 **1~10 微秒**（不含缓存失效的间接损失）。

这解释了三件事：

1. 线程数远超 CPU 核数时，切换开销可能超过业务耗时——**线程池不是越大越好**；
2. 无锁不一定比加锁快——CAS 自旋失败重试也是"空转"，竞争激烈时不如直接 park；
3. 虚拟线程的价值：把"阻塞"从"占用 OS 线程"变成"挂起 Continuation"，切换成本从微秒级降到纳秒级。

顺带回答"为什么要用多线程"：单核时代是为了**把 CPU 计算和 IO 等待叠起来**（一个线程算、一个线程等 IO，综合利用率从 50% 提到接近 100%）；多核时代是为了**让多个核同时干活**。代价就是上面的四件事（原子性/可见性/有序性/活跃性）。

### 1.6 核心术语对齐

| 术语 | 准确含义 | 常见误解 |
| :--- | :--- | :--- |
| **JMM** | Java 内存模型，一套并发语义规范（happens-before、可见性、有序性） | 与"堆栈方法区"混淆（那是运行时数据区） |
| **happens-before** | 两个操作的可见性偏序关系：A hb B ⇒ A 的结果对 B 可见 | 理解成"时间上先发生" |
| **as-if-serial** | 单线程内重排不能改变执行结果 | 以为多线程也保证 |
| **内存屏障** | 阻止特定类型重排的 CPU 指令/编译器约束 | 以为四种屏障都有实际指令 |
| **偏向锁** | 只有一个线程反复进入时，把线程 ID 写进 Mark Word，免 CAS | 以为 JDK 17 还默认开启（JDK 15 起默认关闭并废弃） |
| **AQS** | `AbstractQueuedSynchronizer`，用 `state` + CLH 变体队列实现同步器骨架 | 以为它是锁（它是**框架**，`ReentrantLock` 才是锁） |
| **CLH 队列** | Craig-Landin-Ten-Hagersten 自旋锁队列，AQS 用的是其**变体**（双向、显式 park） | 以为 AQS 就是原版 CLH |
| **虚假唤醒** | 没有 notify 却从 wait 返回 | 忘记用 `while` 而非 `if` 判断条件 |
| **伪共享** | 两个无关变量落在同一缓存行（64B），互相使对方缓存失效 | 与"共享变量竞争"混淆 |

### 1.7 体系定位：一张图看清 JUC 的分层

```
┌──────────────────── 应用层：线程池、CompletableFuture、并发容器组合 ────────────────────┐
├──────────────────── 工具层：Semaphore / CountDownLatch / CyclicBarrier / Phaser ────────┤
├──────────────────── 锁层：ReentrantLock / ReentrantReadWriteLock / StampedLock ─────────┤
├──────────────────── 骨架层：AQS（state + 队列）  │  原子类（CAS + varHandle）──────────┤
├──────────────────── 基础层：volatile / synchronized / CAS / Unsafe / 内存屏障 ───────────┤
└──────────────────── 规范层：JMM（happens-before）+ 硬件内存模型（TSO/弱一致）───────────┘
```

这张图的用法：任何并发问题都能向上或向下定位。比如"`CountDownLatch` 能不能重复用？"——向上看它是工具层（一次性），向下看它是 AQS 共享模式（`state` 减到 0 就结束，无法重置）。

---

## 二、基础工具：锁、volatile、原子类、容器怎么用

### 2.1 创建线程：四种方式与一个铁律

| 方式 | 写法 | 评价 |
| :--- | :--- | :--- |
| 继承 `Thread` | `new MyThread().start()` | Java 单继承，扩展性差，不推荐 |
| 实现 `Runnable` | `new Thread(runnable).start()` | 职责分离，推荐 |
| 实现 `Callable` + `FutureTask` | 有返回值、可抛异常 | 需要返回值时用 |
| 线程池 | `executor.execute/submit` | **生产唯一正确姿势** |

铁律：**`start()` 而不是 `run()`**。`run()` 只是普通方法调用，会在当前线程执行；`start()` 才走 `start0()` native → JVM 创建 OS 线程 → 回调 `run()`。而且**同一个 Thread 对象只能 `start()` 一次**，第二次抛 `IllegalThreadStateException`。

`Runnable` 与 `Callable` 的区别：`Callable` 是 JDK 1.5 引入，`call()` 有返回值、能抛检查异常；`Runnable` 的 `run()` 两者都没有。任务不需要返回值就用 `Runnable`。

### 2.2 synchronized：三种用法与锁对象

| 用法 | 锁对象 | 字节码 |
| :--- | :--- | :--- |
| 实例方法 `synchronized void m()` | **当前实例 `this`** | 方法标志 `ACC_SYNCHRONIZED` |
| 静态方法 `static synchronized void m()` | **`Class` 对象** | 同上，锁的是类 |
| 代码块 `synchronized(obj)` | 括号里的对象 | `monitorenter` / `monitorexit` ×2 |

三个必须知道的细节：

1. **代码块的 `monitorexit` 有两条**：正常路径一条，异常路径一条，所以 `synchronized` **不会因为异常而泄漏锁**。
2. **锁的是对象，不是代码**。两个线程用不同实例调用同一个 `synchronized` 实例方法，**不互斥**。
3. **`synchronized` 可重入**，靠 `ObjectMonitor._recursions` 计数，重入 N 次要退出 N 次才真正释放。

**构造方法不能加 `synchronized`**：构造期间对象还未发布，锁它没有意义。要保证构造期间的可见性，用 `final` 字段或安全发布。

一个容易踩的坑：**不要锁 `String` 常量或包装类**。`Integer` 有 -128~127 缓存、字符串有常量池，锁的是共享对象，锁范围会失控。

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

标准范式：

```java
synchronized (lock) {
    while (!condition) {          // 必须 while，不能 if
        lock.wait();              // 释放锁并挂起
    }
}

synchronized (lock) {
    condition = true;
    lock.notifyAll();             // 优先 notifyAll，notify 可能唤醒错的对象
}
```

**为什么必须 `while`**：① **虚假唤醒**（JVM 允许无 notify 返回）；② 被唤醒后条件可能又被别的线程改回去了。用 `if` 是并发 bug 的经典来源。

**为什么优先 `notifyAll`**：`notify` 随机唤醒一个等待者，如果它等的不是这个条件，就会"唤醒丢失"导致其他线程永久等待。

四种"等待"的区别：

| 方法 | 释放锁 | 可被中断 | 需要 notify | 属于 |
| :--- | :--- | :--- | :--- | :--- |
| `wait()` | ✅ | ✅ | ✅ | Object |
| `sleep(ms)` | ❌ | ✅ | ❌ | Thread |
| `join()` | ✅（内部 wait） | ✅ | 线程结束时自动 notify | Thread |
| `park()` | ❌（但可先释放锁再调用） | ✅ | `unpark` | LockSupport |

**`interrupt()` 不是强制停止线程**，而是设置中断标志：

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

排查命令：

```bash
jstack <pid> | grep -A 20 "Found one Java-level deadlock"
jcmd <pid> Thread.print -l          # -l 会打印锁信息
```

代码层检测：`ThreadMXBean.findDeadlockedThreads()`，可在监控里定时跑。

真实场景：两个 Service 互相调用，各自持有数据库行锁再请求对方接口，形成"数据库锁 + 应用锁"的混合死锁。`jstack` 只能看到应用锁，数据库侧要看 `SHOW ENGINE INNODB STATUS` 的 `LATEST DETECTED DEADLOCK`——**跨系统死锁要两边一起看**。

### 2.6 原子类：从 CAS 到分段

| 类别 | 代表 | 适用 |
| :--- | :--- | :--- |
| 基本类型 | `AtomicInteger` / `AtomicLong` / `AtomicBoolean` | 单变量计数 |
| 引用类型 | `AtomicReference` / `AtomicStampedReference`（带版本，解 ABA）/ `AtomicMarkableReference` | 状态机、无锁栈 |
| 数组 | `AtomicIntegerArray` / `AtomicLongArray` | 按索引原子更新 |
| 字段更新器 | `AtomicIntegerFieldUpdater` | 已有类不想改造成原子类 |
| 累加器（JDK8） | `LongAdder` / `LongAccumulator` / `DoubleAdder` | **高并发计数**，比 `AtomicLong` 快数倍 |

选型判据：读多写少用 `AtomicLong`；**高并发写多**用 `LongAdder`（`sum()` 时把 `base + Cell[]` 累加，弱一致）。

### 2.7 AQS 家族：六个组件的对照

| 组件 | `state` 含义 | 模式 | 是否可重用 | 典型场景 |
| :--- | :--- | :--- | :--- | :--- |
| `ReentrantLock` | 重入次数（0=自由） | 独占 | ✅ | 替代 `synchronized`，需要 `tryLock`/公平/中断 |
| `ReentrantReadWriteLock` | 高 16 位读锁数 + 低 16 位写锁重入数 | 共享 + 独占 | ✅ | 读多写少（注意写饥饿） |
| `Semaphore` | 剩余许可数 | 共享 | ✅ | 限流、资源池 |
| `CountDownLatch` | 剩余计数 | 共享 | ❌ 一次性 | 主线程等多个子任务完成 |
| `CyclicBarrier` | 到达线程数（用 ReentrantLock+Condition 实现，非 AQS） | — | ✅ 可循环 | 分批并行计算 |
| `StampedLock` | 版本号 stamp（非 AQS） | 乐观读/悲观读写 | ✅ | 读极多写极少，注意**不可重入** |

`CountDownLatch` 与 `CyclicBarrier` 的本质区别：前者是"**一个线程等多个线程**"（主等子），计数到 0 后永远为 0；后者是"**多个线程互相等**"（一批到齐再一起走），有 `generation` 概念可循环，还能注册 `barrierAction`。

### 2.8 并发容器：选型表

| 容器 | 线程安全机制 | 读性能 | 写性能 | 适用 |
| :--- | :--- | :--- | :--- | :--- |
| `ConcurrentHashMap` | CAS + `synchronized` 锁桶头 | 无锁 | 桶级锁 | 通用缓存/映射（**首选**） |
| `CopyOnWriteArrayList` | 写时复制整个数组 | 无锁、极快 | O(n) 复制 | **读极多写极少**（监听器列表、白名单） |
| `ConcurrentLinkedQueue` | CAS 无锁链表 | 快 | 快 | 高并发无界队列 |
| `ArrayBlockingQueue` | `ReentrantLock` + 两个 Condition | — | — | 有界、需要阻塞（线程池） |
| `LinkedBlockingQueue` | 两把锁（putLock/takeLock） | — | — | 默认无界（**慎用**，易 OOM） |
| `SynchronousQueue` | 配对传递 | — | — | 不存储，直接交接（`newCachedThreadPool`） |
| `DelayQueue` | 优先队列 + 时间 | — | — | 延时任务、订单超时取消 |

三个高频追问：

1. **`ConcurrentHashMap` 为什么不允许 null 键值？** 因为并发下 `get(k)==null` 无法区分"不存在"和"值就是 null"，两者需要不同处理（前者可能要加锁加载），歧义无法消除。`HashMap` 单线程可以用 `containsKey` 区分，并发下这个判断已失效。
2. **`CopyOnWriteArrayList` 的坑**：只保证**最终一致**，迭代器是快照（看不到新元素）；写操作要复制整个数组，写多时 CPU/GC 压力大。
3. **`Collections.synchronizedList` 与 `CopyOnWriteArrayList` 的区别**：前者是**装饰器 + 单锁**，读写都互斥；后者读不加锁。前者适合写多，后者适合读多。

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

四条铁律：

1. **绝不省略线程池**。默认用 `ForkJoinPool.commonPool()`，它是**全 JVM 共享、线程数 = CPU-1**，一个慢任务能拖垮所有依赖它的业务。
2. **`thenApply` vs `thenCompose`**：前者是同步映射（`T→U`），后者是异步扁平化（`T→CompletableFuture<U>`）。返回 `CompletableFuture` 时用错 `thenApply` 会得到嵌套的 `CompletableFuture<CompletableFuture<U>>`。
3. **`thenApply` 与 `thenApplyAsync`**：前者可能在上一个任务的线程执行（甚至是调用线程），后者一定提交到线程池。
4. **异常处理三件套**：`exceptionally`（只处理异常，返回同类型）、`handle`（正常/异常都处理）、`whenComplete`（不改变结果，只做副作用）。

### 2.10 ThreadLocal：线程私有变量与内存泄露

`ThreadLocal` 解决的是"每个线程一份私有副本"，避免共享变量的竞争。它的原理藏在一个反直觉的事实里：**数据不是存在 `ThreadLocal` 上，而是存在 `Thread` 对象里**。

```java
// Thread 类里有两个字段
ThreadLocal.ThreadLocalMap threadLocals = null;           // 当前线程的 ThreadLocal 值
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null; // 可被子线程继承的

// ThreadLocal.set() 的真实逻辑
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);       // 就是 t.threadLocals
    if (map != null) map.set(this, value);
    else createMap(t, value);
}
```

`ThreadLocal` 只是一个 key，真正的容器是每个 `Thread` 里的 `ThreadLocalMap`（`ThreadLocal` 的静态内部类）。`ThreadLocalMap` 和 `HashMap` 不同，它**用线性探测解决哈希冲突**，初始容量 16、负载因子 2/3、扩容翻倍。

**内存泄露是 `ThreadLocal` 最需要警惕的坑**。`ThreadLocalMap` 的 `Entry` 继承了 `WeakReference`：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;                    // value 是强引用
    Entry(ThreadLocal<?> k, Object v) { super(k); value = v; }
}
```

`key`（`ThreadLocal`）是弱引用，`value` 是强引用。当 `ThreadLocal` 对象本身没有外部强引用、被 GC 回收后，`Entry` 的 key 变成 null，但 **value 还挂在 `ThreadLocalMap` 里**。而 `ThreadLocalMap` 的生命周期和线程一样长（尤其是线程池里的常驻线程），于是 value 永远无法回收，就是内存泄露。

JDK 的缓解：`get/set/remove` 时会顺带清理 key 为 null 的 entry（`expungeStaleEntry`）。**但依赖这个不够，铁律是用完 `remove()`**——尤其线程池场景，线程复用会让上一个任务的 `ThreadLocal` 值泄漏给下一个任务，造成"串号"。

`InheritableThreadLocal` 的代价：子线程能继承父线程的值，但**只在创建子线程那一刻拷贝一次**，之后父线程的修改对子线程不可见；线程池里因为线程是提前创建、复用的，继承语义基本失效。

### 2.11 线程池：七个参数与执行流程

**为什么用线程池**，三个好处：① 复用线程，降低创建/销毁开销；② 任务到了不用等创建线程，响应更快；③ 统一管理、调优、监控稀缺的线程资源。

**`Executors` 为什么被阿里规范禁用**：

- `FixedThreadPool` / `SingleThreadExecutor` 用的是 `LinkedBlockingQueue`（长度 `Integer.MAX_VALUE`）→ 任务堆积 OOM；
- `CachedThreadPool` / `ScheduledThreadPool` 允许线程数 `Integer.MAX_VALUE` → 线程爆炸 OOM。

生产必须用 `ThreadPoolExecutor` 自建，七个参数：

| 参数 | 含义 |
| :--- | :--- |
| `corePoolSize` | 核心线程数（默认常驻，除非 `allowCoreThreadTimeOut`） |
| `maximumPoolSize` | 最大线程数 |
| `keepAliveTime` + `unit` | 非核心线程空闲多久回收 |
| `workQueue` | 任务队列 |
| `threadFactory` | 线程工厂（建议自定义线程名，方便排查） |
| `handler` | 拒绝策略 |

**执行流程（`execute` 三步，对应 `ctl` 的高 3 位状态 + 低 29 位线程数）**：

```java
public void execute(Runnable command) {
    int c = ctl.get();
    // 1. 线程数 < corePoolSize → 新建核心线程
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true)) return;
        c = ctl.get();
    }
    // 2. 线程数 >= core，先入队（队列没满）
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (!isRunning(recheck) && remove(command)) reject(command);
        else if (workerCountOf(recheck) == 0) addWorker(null, false);
    }
    // 3. 队列满了 → 尝试扩到 maximumPoolSize，再失败 → 拒绝策略
    else if (!addWorker(command, false))
        reject(command);
}
```

**核心线程、队列、最大线程的优先级**：先核心线程 → 再队列 → 队列满了才扩到最大线程 → 再满才拒绝。很多人以为"满了就扩线程"，其实**先排队，队列满了才扩**，这是最容易搞错的点。

四种拒绝策略：

| 策略 | 行为 | 代价 |
| :--- | :--- | :--- |
| `AbortPolicy`（默认） | 抛 `RejectedExecutionException` | 任务丢失，调用方感知异常 |
| `CallerRunsPolicy` | 调用者线程自己执行 | 反压提交方，降低吞吐 |
| `DiscardPolicy` | 静默丢弃 | 任务无感知丢失，最危险 |
| `DiscardOldestPolicy` | 丢弃队头最老任务 | 丢的是最老而非最新 |

线程池五状态：`RUNNING` → `SHUTDOWN`（`shutdown()`，不再接新任务但跑完队列）→ `STOP`（`shutdownNow()`，中断正在跑的）→ `TIDYING` → `TERMINATED`。`shutdownNow()` 靠 `interrupt()` 中断，如果任务吞了 `InterruptedException` 就停不下来——这和 2.4 节的坑是连着的。

### 2.12 十二条入门误区

1. "`synchronized` 锁的是代码块"——锁的是**对象**。
2. "`volatile` 保证原子性"——只保证可见性 + 有序性。
3. "`wait` 和 `sleep` 都是让线程等待，差不多"——`wait` 释放锁，`sleep` 不释放。
4. "`notify` 比 `notifyAll` 高效"——除非只有一个等待者且条件相同，否则会丢唤醒。
5. "线程越多吞吐越高"——受 CPU 核数、锁竞争、上下文切换限制。
6. "`ConcurrentHashMap` 完全无锁"——JDK 8 是"读无锁 + 写桶级 `synchronized`"。
7. "`AtomicInteger` 一定比 `synchronized` 快"——低竞争快，高竞争时 CAS 自旋失败率高，`LongAdder` 更好。
8. "`ReentrantLock` 比 `synchronized` 好"——`synchronized` 有锁消除/锁粗化等 JVM 优化，简单场景更省心。
9. "线程池 `submit` 和 `execute` 一样"——`submit` 会把异常吞进 `Future`，不 `get` 就永远看不到。
10. "`ThreadLocal` 是给线程存副本"——数据存在 `Thread.threadLocals` 里，`ThreadLocal` 只是 key。
11. "`BLOCKED` 就是等锁"——只有 `synchronized` 是 `BLOCKED`，`Lock` 是 `WAITING`。
12. "虚拟线程能提升 CPU 密集任务性能"——它解决的是**阻塞导致的线程占用**，CPU 密集场景没有任何收益。

---

## 三、机制拆解：JMM、锁升级、AQS、CHM 到底怎么工作

### 3.1 JMM：一套"允许你少想一点"的规范

JMM 抽象出主内存与工作内存两层：所有变量（实例字段、静态字段、数组元素）存在主内存，线程操作变量时先拷贝到自己的工作内存。**工作内存不是"JVM 里的一块内存"，而是 CPU 缓存 + 寄存器 + Store Buffer 的抽象**——所以 JMM 的问题本质上是硬件内存模型的问题，JMM 只是给了统一的编程契约。

#### 3.1.1 happens-before：八条规则

**定义**：若 A happens-before B，则 A 的结果对 B 可见，且 A 在 B 之前执行（对程序员可见的顺序）。**它不是"时间先后"，而是"可见性保证"**——两个没有 hb 关系的操作，JVM 可以任意重排。

| # | 规则 | 说明 |
| :--- | :--- | :--- |
| 1 | **程序顺序** | 单线程内，前面的操作 hb 后面的 |
| 2 | **监视器锁** | 对同一个锁的 `unlock` hb 后续的 `lock` |
| 3 | **volatile** | 对 volatile 变量的写 hb 后续的读 |
| 4 | **线程启动** | `Thread.start()` hb 该线程内的任何操作 |
| 5 | **线程终止** | 线程内所有操作 hb 其他线程从 `join()` 返回 |
| 6 | **中断** | `interrupt()` hb 被中断线程检测到中断 |
| 7 | **对象终结** | 构造函数结束 hb `finalize()` 开始 |
| 8 | **传递性** | A hb B，B hb C ⇒ A hb C |

**传递性最常考**，也是 volatile"捎带可见性"的来源：

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

② hb ③（volatile 规则），① hb ②（程序顺序），③ hb ④（程序顺序）→ ① hb ④。volatile 不只保证自己可见，还保证"之前的所有写"对"之后的所有读"可见。

#### 3.1.2 as-if-serial 与顺序一致性

- **as-if-serial**：单线程内怎么重排都行，只要结果不变。这是编译器优化的合法性来源。
- **顺序一致性模型**：多线程下所有操作按一个全局顺序执行，且每个线程内顺序与程序顺序一致。**JMM 不保证顺序一致性**（否则性能无法接受），只保证正确同步的程序具有顺序一致性——DCL 少了 volatile 就属于"未正确同步"。

### 3.2 内存屏障：JMM 落到 CPU 上的四种指令

| 屏障 | 作用 | x86 是否需要指令 |
| :--- | :--- | :--- |
| `LoadLoad` | 禁止上面的读与下面的读重排 | 否（TSO 天然保证） |
| `StoreStore` | 禁止上面的写与下面的写重排 | 否 |
| `LoadStore` | 禁止上面的读与下面的写重排 | 否 |
| **`StoreLoad`** | 禁止上面的写与下面的读重排 | **是**（`lock addl` / `mfence`） |

volatile 的屏障插入策略：写之前插 `StoreStore`（保证前面的普通写先刷出），写之后插 **`StoreLoad`**（保证写对后续读可见，代价最高），读之后插 `LoadLoad` + `LoadStore`。**这就解释了为什么 `volatile` 写比读贵**：写要付 `StoreLoad`，读在 x86 上几乎零成本。

x86 的 `lock addl $0, (%rsp)`（HotSpot 实现 StoreLoad 的方式）会做三件事：把写缓冲全部刷到缓存；通过 MESI 使其他核的对应缓存行失效；禁止该指令前后的读写重排。成本：无竞争约 20~50 周期，跨 NUMA 可达数百周期——这就是"高频 volatile 写"成为性能瓶颈的原因。

`final` 字段还有两个额外屏障：构造函数内对 final 的写，与"把 this 引用赋给其他变量"之间插 `StoreStore`；首次读对象引用与首次读 final 字段之间插 `LoadLoad`。只要对象在构造期间没有逸出，其他线程看到引用时一定能看到 final 字段的正确值——这是**不可变对象天然线程安全**的底层依据。

### 3.3 synchronized 的底层：Monitor 与锁升级

#### 3.3.1 字节码层

代码块 `synchronized` 编译成 `monitorenter` / `monitorexit`（正常路径一条 + 异常路径一条）；方法级 `synchronized` 不加指令，靠方法访问标志 `ACC_SYNCHRONIZED`，JVM 在方法调用时隐式 `monitorenter/exit`。两者本质都是对对象监视器 monitor 的获取。

#### 3.3.2 ObjectMonitor 的结构与四个队列

重量级锁对应的 C++ 对象（`objectMonitor.hpp`）：

```cpp
class ObjectMonitor {
    volatile markOop   _header;      // 原 Mark Word（Displaced Header）
    void* volatile     _owner;       // 持有锁的线程
    volatile intx      _recursions;  // 重入次数
    ObjectWaiter* volatile _WaitSet;  // 调用 wait() 的线程
    ObjectWaiter* volatile _EntryList;// 等待锁的线程（blocked）
    ObjectWaiter* volatile _cxq;      // 竞争队列（新来的线程先入这里）
};
```

四个队列的关系（这是"能讲清 monitor"的分水岭）：

```
线程调用 synchronized
   ↓ 抢锁失败
  _cxq（单向，LIFO 压栈）── 被唤醒时批量转移到 ──→ _EntryList（双向，可调顺序）
                                                      ↓ 抢到锁
                                                    _owner
   wait() → 释放锁 → 进入 _WaitSet ── notify 后 ──→ 转移到 _EntryList 重新竞争
```

`_cxq` 是 LIFO（后进先出，对刚释放锁的线程有利——缓存还热）；`_EntryList` 可调整（实现近似公平，避免饥饿）；`_WaitSet` 只存 `wait()` 的线程。

`ObjectMonitor::enter` 的路径：先 CAS 抢 `_owner`（无竞争直接成功），`_owner` 是自己则是重入，否则走自适应自旋 → 失败则 park 排队。

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

| 阶段 | 关键动作 | 成本 |
| :--- | :--- | :--- |
| 偏向 | 首次 CAS 把线程 ID 写进 Mark Word；之后同一线程进入**零指令开销**（只比对 ID） | 一次 CAS（仅首次） |
| 偏向撤销 | 需在**安全点**进行；撤销次数超过阈值后批量重偏向/批量撤销 | 高（STW） |
| 轻量级 | 栈帧分配 Lock Record，复制 Mark Word，CAS 把对象头指向该 Record | 一次 CAS |
| 轻量级解锁 | CAS 换回 Displaced Mark Word；失败说明有竞争 → 膨胀为重量级 | 一次 CAS |
| 重量级 | 进入 `ObjectMonitor`，`park` 挂起，涉及用户态→内核态切换 | 微秒级 |

**为什么 JDK 15 废弃偏向锁（JEP 374）**：撤销需要 safepoint、与 `hashCode`/`wait` 语义冲突、现代应用几乎都有多线程竞争（偏向命中率低），维护成本大于收益。

#### 3.3.4 锁消除与锁粗化

JIT 的两个反向优化：**锁消除**——逃逸分析发现锁对象不逃逸（如方法内 `new Object()` 加锁），直接去掉 `monitorenter`；**锁粗化**——连续的加解锁合并成一次（如循环内 `synchronized`）。

### 3.4 CAS：无锁的基石与它的三个缺陷

底层链路：

```java
// AtomicInteger
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
// Unsafe.compareAndSwapInt (native) → Atomic::cmpxchg
//   x86: lock cmpxchg [addr], new     ← lock 前缀保证多核原子
//   ARM: ldaxr / stlxr (LL/SC 循环)
```

`cmpxchg` 单条指令只保证"比较+交换"不被打断，但多核下仍需 `lock` 保证缓存一致性。

三个缺陷：

| 缺陷 | 表现 | 解法 |
| :--- | :--- | :--- |
| **ABA 问题** | A→B→A，CAS 以为没变过 | `AtomicStampedReference`（加版本号） |
| **自旋开销** | 高竞争下大量 CAS 失败空转，CPU 飙升 | `LongAdder` 分段、退化为锁 |
| **只能保证单变量原子** | 多个变量要一起原子更新 | `AtomicReference` 打包、加锁 |

ABA 的危害：无锁栈里 `Thread1` 读到 head=A 准备 CAS 前被挂起，`Thread2` 弹出 A、B 又压回 A，`Thread1` 的 CAS(A, B) 仍成功——但 B 已被弹出过，`B.next` 可能已被复用，导致链表错乱。

**`LongAdder` 为什么快**：无竞争只更新 `base`；有竞争时按线程哈希分散到 `Cell[]`，把一次热点 CAS 分散成 N 次；`Cell` 用 `@Contended` 独占缓存行避免伪共享。代价：`sum()` 是 `base + Σcells` 的**不加锁累加**，弱一致，且内存占用更大。适合"计数展示"，不适合"必须精确的序号生成"。

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

**② `addWaiter`**：CAS 抢尾节点，成功后 `pred.next = node`（**这一步非原子，所以 `next` 可能短暂不可靠**）；失败则 `enq` 自旋初始化。

**③ `acquireQueued`**：

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean interrupted = false;
    for (;;) {
        final Node p = node.predecessor();
        if (p == head && tryAcquire(arg)) {        // 只有前驱是 head 才有资格再试
            setHead(node);
            p.next = null;
            return interrupted;
        }
        if (shouldParkAfterFailedAcquire(p, node) &&  // 前驱状态设为 SIGNAL 后
            parkAndCheckInterrupt())                  // 才真正 park
            interrupted = true;
    }
}
```

`shouldParkAfterFailedAcquire` 的三分支：

| 前驱 `waitStatus` | 处理 |
| :--- | :--- |
| `SIGNAL(-1)` | 返回 true → 可以安全 park（前驱释放时会 unpark 我） |
| `> 0`（CANCELLED） | 往前跳过所有取消节点 |
| 其他（0 / PROPAGATE） | CAS 设为 `SIGNAL`，返回 false（**下一轮再 park**，避免"刚设完就被唤醒"的竞态） |

**④ `release`** → `unparkSuccessor`。**为什么从 tail 往前找后继**：`addWaiter` 里 `pred.next = node` 不是原子的，`next` 可能断链；而 `prev` 一定可靠（先赋值再 CAS）。所以找后继要用 `prev` 反向遍历——**这是 AQS 里最容易被追问的细节**。

#### 3.5.3 共享模式：`PROPAGATE` 的意义

`Semaphore`/`CountDownLatch` 用共享模式。`PROPAGATE` 解决"释放时唤醒一个线程，但多个线程都该被唤醒"的竞态：`releaseShared` 后把 head 设为 `PROPAGATE`，让后续 `shouldParkAfterFailedAcquire` 继续传播唤醒，直到所有能拿到资源的线程都醒来。

### 3.6 Condition：两个队列之间的搬运

`ConditionObject` 内部维护一条单向条件队列（节点复用 AQS 的 `Node`，`waitStatus=CONDITION`）。

`await()` 四步：① `fullyRelease` 完全释放锁（重入 N 次就释放 N 次，否则其他线程永远拿不到锁）；② 在条件队列里 park；③ 被 signal 后重新 `acquireQueued` 抢锁；④ 恢复中断状态。

`signal()` 三步：① CAS 把节点 `waitStatus` 从 CONDITION 改为 0；② 节点从条件队列摘下，enq 到同步队列；③ 若前驱 CANCELLED 或 CAS 设 SIGNAL 失败 → 直接 unpark（避免丢失唤醒）。

关键点：`signal` 只是"搬家"（条件队列 → 同步队列），**不释放锁，也不立即唤醒执行**；被唤醒的线程要重新竞争锁，抢到后才从 `await` 返回。这解释了"`signal` 之后必须释放锁，被等待的线程才有机会跑"。

### 3.7 公平锁与非公平锁的源码差异

```java
// NonfairSync：直接抢，不看队列
final boolean nonfairTryAcquire(int acquires) {
    if (compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current); return true;
    }
    ...
}

// FairSync：必须队列里没有更早的等待者
protected final boolean tryAcquire(int acquires) {
    if (getState() == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) { ... }
    }
    ...
}
```

`hasQueuedPredecessors()` 的逻辑：`h != t && ((s = h.next) == null || s.thread != current)`。注意 `h.next == null` 也算"有前驱"——处理"正在入队、next 还没连上"的中间态。

性能差异：非公平锁吞吐更高（减少唤醒后的上下文切换、利用刚释放锁线程的缓存热度），但可能饥饿；公平锁保证 FIFO，代价是更多切换。**默认用非公平**。

### 3.8 ConcurrentHashMap：JDK 8 的三处关键设计

#### 3.8.1 读为什么无锁

`tabAt` 用 `getObjectAcquire`（volatile 语义读）；`Node.val` 和 `next` 都是 `volatile`；扩容时旧表桶头被替换为 `ForwardingNode`（`hash = MOVED = -1`），读线程看到它就**去新表读**。所以读操作完全无锁，且扩容期间也能读到正确数据。

#### 3.8.2 写为什么用桶级 `synchronized`

```java
for (Node<K,V>[] tab = table;;) {
    if ((f = tabAt(tab, i = (n-1) & hash)) == null) {
        if (casTabAt(tab, i, null, new Node<K,V>(...)))  // ① 空桶 CAS 直接放
            break;
    }
    else if ((fh = f.hash) == MOVED) tab = helpTransfer(tab, f);  // ② 帮扩容
    else {
        synchronized (f) { ... }                          // ③ 只锁桶头节点
    }
}
```

为什么是 `synchronized(f)` 而不是 `ReentrantLock`：JDK 8 时 `synchronized` 已优化得足够好，而且**每个桶一个锁对象**天然分段，比维护一堆 Lock 对象省内存。JDK 7 的 Segment 分段锁（默认 16 段）被废弃——段数固定导致并发度上限固定。

#### 3.8.3 `size()` 为什么是弱一致

本质是 `LongAdder` 同款思路：`baseCount` + `CounterCell[]` 分散热点，无锁读取，所以返回的是**近似值**（Javadoc 明确写 "may be inaccurate"）。`size()` 只能用于监控，**不能用于业务判断**（如"容量到 100 就拒绝"）。

#### 3.8.4 扩容：多线程协助

`sizeCtl` 的语义：`>0` 是下次扩容阈值，`=0` 未初始化，`=-1` 正在初始化，`<-1` 正在扩容（值为 `-(1+参与线程数)`）。`transfer` 把旧表按 stride 分段，每个线程认领一段搬迁，搬完的桶放 `ForwardingNode`。这就是"多线程协助扩容"——也是 CHM 在扩容期间仍能读写的关键。

树化阈值：链表长度 ≥ 8 尝试树化、树节点 ≤ 6 退化、**表容量 < 64 只扩容不树化**。阈值取 8 是因为泊松分布下哈希均匀时链表到 8 的概率约 `0.00000006`，正常几乎不会树化——树化是"防哈希碰撞攻击"的兜底。

### 3.9 虚拟线程（JDK 21）：把阻塞变便宜

模型：虚拟线程（千万级）mount/unmount 到载体线程（= CPU 核数个平台线程，ForkJoinPool 调度），载体线程再落到 OS 线程。栈存在堆上的 `Continuation` 里，按需增长，所以能创建百万级。

遇到 IO/`park`/`sleep` 时，虚拟线程**卸载**，载体线程去跑别的虚拟线程；IO 就绪后再**挂载**回来。

**`pin`（钉住）是虚拟线程的软肋**——以下情况无法卸载，会一直占着载体线程：

| 场景 | 原因 | 规避 |
| :--- | :--- | :--- |
| `synchronized` 块内阻塞 | monitor 与载体线程绑定（JDK 21 限制，JDK 24 起改善） | 改用 `ReentrantLock` |
| native 方法 / JNI 调用 | 无法保存 native 栈 | 减少 JNI，或用平台线程池隔离 |
| 类初始化（`<clinit>`）阻塞 | 类初始化锁与载体线程绑定 | 避免静态块里做 IO |

三个必须说清的边界：

1. **不提升 CPU 密集性能**：没有阻塞就没有卸载收益，反而多一层调度开销。
2. **不要池化虚拟线程**：创建成本极低，池化反而引入排队与状态复用问题（`ThreadLocal` 串号）。"每个请求一个虚拟线程"是正确姿势。
3. **`ThreadLocal` 仍可用但要谨慎**：百万虚拟线程 × 每个 ThreadLocal = 内存爆炸；JDK 21 提供 `ScopedValue`（预览）作为替代。

---

## 四、生产实战：可见性 Bug、排查手册与优化

### 4.1 四个真实可见性 Bug 的形态

#### 4.1.1 无锁退出标志（最经典）

```java
boolean running = true;
new Thread(() -> { while (running) { doWork(); } }).start();
running = false;   // 子线程可能永远看不到
```

**为什么线上难复现**：JIT 把 `running` 提升到寄存器（循环不变量外提），只有 C2 编译后才暴露，测试环境（解释执行/C1）一切正常。这类 bug 的特征是"**上线跑一段时间才出现**"。修复：`volatile` 或 `AtomicBoolean`。

#### 4.1.2 对象逸出（半初始化对象被发布）

```java
public class UnsafePublish {
    private int x;
    public UnsafePublish() {
        new Thread(() -> System.out.println(this.x)).start();  // 可能打印 0
        x = 42;
    }
}
```

修复：不要在构造方法里启动线程 / 注册监听器 / 把 `this` 传给外部方法；必须这样做时用工厂方法（构造完成后发布）。

#### 4.1.3 非线程安全的延迟初始化

两个线程可能都判断 `instance == null`，创建两个实例。修复三选一：DCL + volatile；静态内部类（靠 `<clinit>` 的线程安全，推荐）；枚举（防反射/反序列化破坏）。

#### 4.1.4 集合的"并发修改"与"弱一致迭代"

`fail-fast`（如 `ArrayList`）检测到 `modCount` 变化就抛 `ConcurrentModificationException`，属于**尽力而为**（不保证一定抛）；`fail-safe`（COW、CHM）是迭代快照/弱一致，不抛异常但可能读到旧数据。遍历删除用 `Iterator.remove` 或 `removeIf`。

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
| 大量 `WAITING (parking)` 在 `LinkedBlockingQueue.take` | 队列空、消费跟不上 |
| 大量 `BLOCKED` 在同一把锁 | 锁竞争，需看锁持有者在干嘛 |
| 线程数持续增长不回收 | 用了 `CachedThreadPool` 或自建无界池 |

关键指标（必须监控）：`activeCount`、`queue.size`、`completedTaskCount`、`largestPoolSize`。**只看 CPU 是发现不了线程池打满的**。

#### 4.2.2 死锁

应用锁死锁 `jstack` 直接报告；数据库死锁要看 `SHOW ENGINE INNODB STATUS`；类初始化死锁线程栈显示 `in class loading`（静态块互相依赖）。

#### 4.2.3 活锁与饥饿

**活锁**：线程都在运行、都在重试，但没人能推进（如两个线程互相礼让重试）。特征：CPU 高、吞吐为零，加随机退避。**饥饿**：低优先级线程永远抢不到锁（非公平锁 + 高频短任务）。特征：某些请求 P99 极高但平均正常，改公平锁或限并发。

#### 4.2.4 线程泄漏

常见原因：`new Thread()` 未结束、`Executors.newCachedThreadPool()` 突发流量后 60s 才回收、`Timer` 不 `cancel`、自定义线程池 `shutdown` 未调用。判定：连续多次 `jstack` 抓线程名，同名线程数量单调递增即泄漏。

### 4.3 锁性能优化：六个层次

| 层次 | 手段 | 收益 | 代价 |
| :--- | :--- | :--- | :--- |
| 1. 消除共享 | 无状态设计、ThreadLocal、每线程独立对象 | 最大 | 内存/一致性 |
| 2. 缩小临界区 | 只锁必要语句，IO/远程调用移出锁外 | 大 | 需仔细验证 |
| 3. 降低锁粒度 | 锁分段（CHM）、按 key 哈希取锁 | 大 | 复杂度上升 |
| 4. 读写分离 | `ReentrantReadWriteLock` / `StampedLock` 乐观读 | 中（读多写少） | 写饥饿、不可重入 |
| 5. 无锁化 | CAS、`LongAdder`、`ConcurrentLinkedQueue` | 中（低竞争） | ABA、自旋开销 |
| 6. 避免伪共享 | `@Contended`、字段填充 | 中（高频写） | 内存浪费 |

伪共享实例：两个 volatile 变量落在同一缓存行（64B），两个线程分别写，导致缓存行来回失效。修复：JDK 8 用 `@Contended`（需 `-XX:-RestrictContended`），或手工 padding 到 64B。`LongAdder` 的 `Cell` 就带 `@Contended`。

一个反直觉的结论：**低竞争时 `synchronized` 可能比 `AtomicLong` 更快**——`synchronized` 无竞争下只有一次偏向/轻量级 CAS，而 `AtomicLong` 每次都要 `lock cmpxchg`。**不要盲目"去锁化"，要用 JMH 测**。

### 4.4 并发正确性怎么验证

| 手段 | 用途 | 说明 |
| :--- | :--- | :--- |
| **jcstress** | 验证 JMM 层面的可见性/原子性 | OpenJDK 官方工具，能构造极端交错 |
| **JMH** | 性能对比（吞吐/延迟） | 手写 `currentTimeMillis` 测不准（JIT 预热、死代码消除） |
| **压测**（JMeter/Gatling） | 端到端验证 | 关注 P99 而非平均值 |
| **故障注入** | 超时、异常、下游慢 | 验证降级与线程池隔离 |
| **静态检查** | SpotBugs（`IS2_INCONSISTENT_SYNC`） | 发现"一半方法加锁一半不加" |

并发 bug 不能靠测试覆盖，只能靠**设计约束**（不可变、线程封闭、同步策略统一）+ 代码评审 + 极端场景工具（jcstress）。

### 4.5 框架里的并发设计（能对上号就说明真用过）

| 框架 | 并发模型 | 关键点 |
| :--- | :--- | :--- |
| **Tomcat** | `ThreadPoolExecutor`（`maxThreads` 默认 200，`acceptCount` 100） | IO 线程与业务线程分离；打满后请求进 `acceptCount` 队列，再满则拒绝 |
| **Dubbo** | 默认 `fixed` 线程池（200）+ `ThreadlessExecutor` | `actives` 限流、`executes` 限并发 |
| **RocketMQ** | 消费线程池 + 长轮询 | 消费位点提交与业务处理解耦 |
| **Sentinel** | 滑动窗口（`LeapArray`）+ `LongAdder` 计数 | 统计线程安全靠 `AtomicLong` 数组分段 |
| **HikariCP** | `ConcurrentBag`（无锁借还）+ `SynchronousQueue` | 连接借还走 `ThreadLocal` 列表 |
| **Netty** | 主从 `EventLoopGroup`（`NioEventLoop` 单线程绑定多 Channel） | **一个 Channel 的所有事件在同一线程**，业务 Handler 里不能阻塞 |

这六个例子是最好的"框架认知"素材：面试问"你用过哪些并发实践"，答"我们 xx 场景用了 Sentinel 的滑动窗口 + LongAdder 做 QPS 统计"比空谈 AQS 强得多。

### 4.6 十二条生产高频坑

1. **`Executors.newFixedThreadPool` 用无界队列** → 任务堆积到 OOM（必须自建 + 有界队列 + 拒绝策略）。
2. **`submit` 吞异常** → 任务失败无日志，必须 `future.get()` 或包 `try-catch`。
3. **`ThreadLocal` 不 `remove`** → 线程池串号 + 内存泄漏。
4. **`SimpleDateFormat` 共享** → 日期错乱/`NumberFormatException`（用 `DateTimeFormatter`）。
5. **`HashMap` 并发写** → JDK 8 下可能丢数据/树化异常（不再死循环，但仍不安全）。
6. **`ConcurrentHashMap` 复合操作用 `get` + `put`** → 必须 `computeIfAbsent`/`merge`/`putIfAbsent`。
7. **`synchronized` 锁 `String` 常量/包装类** → 常量池共享对象，锁范围失控。
8. **锁内做远程调用/IO** → 临界区被放大到几百毫秒，拖垮吞吐。
9. **`CountDownLatch` 在 `finally` 外 `countDown`** → 异常时永久等待。
10. **`CompletableFuture` 不指定线程池** → 挤占 `commonPool`，影响全 JVM 的并行流。
11. **`wait` 用 `if` 判断条件** → 虚假唤醒导致越界/空指针。
12. **`interrupt` 被吞** → 线程池 `shutdownNow` 无法终止任务。

### 4.7 版本演进

| 版本 | 变化 | 影响 |
| :--- | :--- | :--- |
| JDK 5 | `java.util.concurrent` 引入；`synchronized` 锁升级 | 进入 JUC 时代 |
| JDK 6 | 偏向锁、自适应自旋、锁消除 | `synchronized` 性能大幅提升 |
| JDK 8 | `CompletableFuture`、`LongAdder`、`StampedLock`、CHM 改 CAS+synchronized | 异步编排质变；CHM 并发度不再受段数限制 |
| JDK 9 | `VarHandle`（替代部分 `Unsafe`） | 更安全的内存访问 |
| **JDK 15** | **偏向锁默认禁用并废弃（JEP 374）** | 锁升级只剩"轻量级 → 重量级" |
| JDK 17 | 强封装内部 API | 反射 `Unsafe` 受限 |
| **JDK 21** | **虚拟线程正式（JEP 444）**、`ScopedValue`（预览） | "一请求一线程"复兴 |
| JDK 24+ | `synchronized` 内阻塞不再 pin（JEP 491） | 虚拟线程最大软肋被修复 |

---

## 五、收口：一张全景图 + 选型决策表

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
  CAS（Unsafe.cmpxchg + lock）→ ABA / 自旋 / 单变量
  原子类：AtomicLong（低竞争） vs LongAdder（高并发，base + Cell[] + @Contended）

框架层
  AQS：state + CLH 变体双向队列
    acquire → tryAcquire → addWaiter(CAS tail) → acquireQueued(前驱是 head 才重试) → park
    release → unparkSuccessor（从 tail 反向找，因为 next 可能断链）
    Condition：条件队列 ⇄ 同步队列（signal 只搬家不释放锁）

容器层
  CHM：读无锁（volatile + ForwardingNode）/ 写桶级 synchronized / size 弱一致 / 协助扩容
  COW：写时复制，读极快，只适合读多写极少

线程层
  ThreadPoolExecutor：core → queue → max 三步 + 四拒绝策略
  ThreadLocal：数据在 Thread.threadLocals，key 弱引用，用完必须 remove
  虚拟线程：Continuation + ForkJoinPool 载体，pin 在 synchronized/native/classinit
```

### 5.2 选型与排查速查表

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
| 线程私有数据 | `ThreadLocal` + **finally remove** | 线程池场景必须 remove |
| 线程数持续增长 | 查线程池类型与 `shutdown` | `jstack` 线程名计数 |
| 大量线程卡 `socketRead0` | 查超时配置 | `connectTimeout` / `readTimeout` |
| 线程全 `BLOCKED` | 找锁持有者 | `jstack -l` 的 `locked ownable synchronizers` |

### 5.3 全篇结论的边界与代价（哪些话不能乱套）

把本文最容易被"误用"的结论再钉一遍边界：

1. **"ConcurrentHashMap 线程安全" ≠ "操作原子"**：单个 `get`/`put` 安全，但 `get` 后 `put` 的复合操作依然要 `computeIfAbsent` 等原子方法，否则照样丢更新。
2. **"volatile 保证可见性" ≠ "volatile 保证一致"**：它只保证单次读写看到最新值，`i++` 这种读改写照样丢，且它不解决"多个变量之间的原子性"。
3. **"LongAdder 快" 有前提**：只在**高并发写**场景快；低竞争下 `AtomicLong` 足够，且 `LongAdder` 的 `sum()` 是弱一致，不能用于精确序号。
4. **"无锁一定比加锁快" 是错的**：低竞争下 `synchronized` 的一次轻量级 CAS 可能比 `AtomicLong` 的 `lock cmpxchg` 更快；高竞争下 CAS 自旋空转也不如 park。**一切以 JMH 实测为准**。
5. **"虚拟线程能提速" 有前提**：只对 IO 密集/阻塞密集生效，CPU 密集零收益，且 `synchronized` 内阻塞、native、类初始化会 pin 住载体线程，收益直接归零。
6. **"ThreadLocal 隔离线程变量" 有代价**：线程池里不 `remove` 会串号 + 内存泄露，虚拟线程海量场景下 ThreadLocal 会放大内存占用。
7. **"happens-before 是时间先后" 是错的**：它是可见性契约，两个没有 hb 关系的操作，编译器可以任意重排——这也是为什么"没同步的代码跑着没 bug"只是运气。

把这些边界记牢，比背下全部源码更有价值：**源码告诉你"它怎么工作"，边界告诉你"它什么时候会骗你"**。

---

> 本文整合自《Java 并发编程深度知识文档（高级开发版）》与《Java 并发编程深度知识文档（JavaGuide 归档版）》两份文档，去重后以高级开发版为主干，补入归档版的进程线程、ThreadLocal、线程池内容，并去掉面试专用的速答清单与追问链，改写为面向阅读的技术博客。
