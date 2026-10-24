---
title: "Java 线程池：ctl、Worker 与生产参数治理"
date: 2026-10-18T09:00:00+08:00
summary: "ctl 的位运算怎么同时表达状态与线程数、addWorker 的 CAS 占位、Worker 为什么继承 AQS、getTask 的六种退出条件，以及核心线程数、队列与拒绝策略在生产里到底怎么定。"
categories: ["Java 并发"]
tags: ["线程池", "并发", "性能优化"]
author: ZLLiang
ShowToc: true
TocOpen: false
draft: false
---

## 一、基本概念

### 1.1 为什么需要线程池

直接 `new Thread()` 有四个问题：

| 问题 | 后果 |
| :--- | :--- |
| 创建/销毁开销 | 每次创建都要申请栈内存、注册到 OS，频繁创建会显著拉高 CPU |
| 数量不可控 | 无限制创建会耗尽内存与线程句柄，最终 `OutOfMemoryError: unable to create native thread` |
| 缺乏任务管理 | 没有排队、拒绝、超时、监控能力 |
| 无法复用 | 无法控制并发度，也无法做隔离 |

线程池的本质是一个**生产者-消费者模型**：

```text
提交任务的线程（生产者）
        │ execute(Runnable)
        ▼
   ┌─────────────┐
   │  workQueue  │  缓冲区
   └──────┬──────┘
          │ getTask()
          ▼
   Worker 线程（消费者，循环取任务执行）
```

它解决三件事：**复用线程**、**控制并发度**、**管理任务生命周期**。

### 1.2 核心术语

| 术语 | 含义 |
| :--- | :--- |
| corePoolSize | 核心线程数，常驻（除非允许超时） |
| maximumPoolSize | 线程总数上限 |
| keepAliveTime | 非核心线程空闲存活时间 |
| workQueue | 任务缓冲队列 |
| handler | 拒绝策略，过载时的兜底 |
| ctl | 一个 `AtomicInteger`，同时保存运行状态与线程数 |
| Worker | 线程池内部的工作线程封装，继承 AQS |
| 核心线程 / 非核心线程 | 逻辑区分，都存放在同一个 `workers` 集合里，靠创建时是否 `core=true` 标记 |

### 1.3 体系定位

```text
Executor（顶层接口，只有 execute）
  └── ExecutorService（提交、关闭、批量）
        └── AbstractExecutorService（submit/invokeAll 模板）
              └── ThreadPoolExecutor（本文主角）
                    └── ScheduledThreadPoolExecutor（定时任务）
Executor 的另一分支：ForkJoinPool（分治 + 工作窃取）
```

`Executors` 只是工厂，生产环境不应直接使用它的四个快捷方法（原因见 2.5）。

---

## 二、基础知识点

### 2.1 七个构造参数

```java
public ThreadPoolExecutor(
    int corePoolSize,
    int maximumPoolSize,
    long keepAliveTime,
    TimeUnit unit,
    BlockingQueue<Runnable> workQueue,
    ThreadFactory threadFactory,
    RejectedExecutionHandler handler
)
```

| 参数 | 职责 | 配错的表现 |
| :--- | :--- | :--- |
| `corePoolSize` | 常驻线程数 | 太小：任务排队；太大：空闲资源浪费 |
| `maximumPoolSize` | 扩容上限 | 配合无界队列时**永不生效** |
| `keepAliveTime` | 非核心线程存活时间 | 太大：线程不回收；太小：频繁创建 |
| `workQueue` | 缓冲策略 | 无界队列 → OOM；太小 → 频繁拒绝 |
| `threadFactory` | 线程命名与属性 | 不命名 → 线上排查无法定位 |
| `handler` | 过载兜底 | 默认 Abort 抛异常；Discard 静默丢任务 |
| `unit` | 时间单位 | — |

### 2.2 四种拒绝策略

| 策略 | 行为 | 适用 |
| :--- | :--- | :--- |
| `AbortPolicy`（默认） | 抛 `RejectedExecutionException` | 需要调用方感知失败 |
| `CallerRunsPolicy` | **提交任务的线程自己执行** | 形成背压，适合可降速的场景 |
| `DiscardPolicy` | 静默丢弃 | 几乎不用，任务会无声消失 |
| `DiscardOldestPolicy` | 丢队列最老的任务，再提交当前 | 只保留最新任务的场景 |

生产上更常用**自定义策略**：记录日志 + 落库/落 MQ 补偿 + 监控告警，避免「静默丢弃」。

### 2.3 队列选择

| 队列 | 特性 | 影响 |
| :--- | :--- | :--- |
| `ArrayBlockingQueue` | 有界、数组 | 推荐，容量可控 |
| `LinkedBlockingQueue` | 默认无界（`Integer.MAX_VALUE`） | **maximumPoolSize 失效**，堆积 OOM |
| `SynchronousQueue` | 不存储，直接交接 | 无排队，任务来了就扩容线程 |
| `PriorityBlockingQueue` | 无界、按优先级 | 需任务实现 `Comparable` |
| `DelayedWorkQueue` | 延迟队列 | `ScheduledThreadPoolExecutor` 专用 |

### 2.4 标准用法

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
        8, 32, 60L, TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(1000),
        new ThreadFactory() {
            private final AtomicInteger seq = new AtomicInteger(1);
            @Override
            public Thread newThread(Runnable r) {
                Thread t = new Thread(r, "order-export-" + seq.getAndIncrement());
                t.setDaemon(false);
                return t;
            }
        },
        (r, executor) -> {
            log.error("task rejected, poolSize={}, queueSize={}",
                    executor.getPoolSize(), executor.getQueue().size());
            throw new RejectedExecutionException("order-export pool saturated");
        });

// 提交
pool.execute(() -> handle());                 // 无返回值
Future<String> f = pool.submit(() -> "ok");   // 有返回值，异常被包装进 Future

// 关闭
pool.shutdown();
pool.awaitTermination(30, TimeUnit.SECONDS);
```

### 2.5 为什么阿里规约禁用 Executors

| 工厂方法 | 底层队列 | 风险 |
| :--- | :--- | :--- |
| `newFixedThreadPool` | `LinkedBlockingQueue`（无界） | 队列无限堆积 → OOM；`maximumPoolSize` 失效 |
| `newSingleThreadExecutor` | 同上 | 同上 |
| `newCachedThreadPool` | `SynchronousQueue` | `maximumPoolSize = Integer.MAX_VALUE`，线程无限增长 |
| `newScheduledThreadPool` | `DelayedWorkQueue`（无界） | 队列无界 |

规约要求：**手动 new、有界队列、自定义线程工厂、有意义的拒绝策略**。

### 2.6 入门误区

| 误区 | 事实 |
| :--- | :--- |
| 先扩到 maximum 再排队 | 实际是**先入队，队列满才扩容** |
| 设了 maximum 就一定会用到 | 无界队列下永远不会触发扩容 |
| `submit` 的异常会被线程池抛出 | 异常被封装进 `Future`，不 `get()` 就永远看不到 |
| `shutdown()` 会立即终止任务 | 它是平滑关闭，队列中的任务仍会执行 |
| 核心线程一定不回收 | 设置 `allowCoreThreadTimeOut(true)` 后会回收 |
| 线程池越大吞吐越高 | 超过瓶颈后上下文切换与锁竞争会让吞吐下降 |

---

## 三、技术原理

### 3.1 ctl：一个 int 管两件事

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;      // 29
private static final int CAPACITY   = (1 << COUNT_BITS) - 1; // 约 5.36 亿

// 高 3 位：运行状态；低 29 位：线程数
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

五种状态（高位取值）：

| 状态 | 值 | 含义 | 是否接受新任务 | 是否处理队列 |
| :--- | :--- | :--- | :--- | :--- |
| `RUNNING` | `-1 << 29` | 正常运行 | 是 | 是 |
| `SHUTDOWN` | `0 << 29` | 调用了 `shutdown()` | 否 | **是** |
| `STOP` | `1 << 29` | 调用了 `shutdownNow()` | 否 | 否，且中断在途任务 |
| `TIDYING` | `2 << 29` | 所有任务结束、worker 数为 0 | 否 | 否 |
| `TERMINATED` | `3 << 29` | `terminated()` 钩子执行完 | 否 | 否 |

状态只能单向前进：`RUNNING → SHUTDOWN → TIDYING → TERMINATED`，或 `RUNNING → STOP → TIDYING → TERMINATED`。

**为什么用一个 int 而不是两个字段**：状态与线程数必须**原子地一起读写**。`execute` 里要先读状态再读线程数，分两个 Atomic 变量会出现「读到的状态与线程数不属于同一时刻」的竞态。

### 3.2 execute 的四步优先级

```java
public void execute(Runnable command) {
    if (command == null) throw new NullPointerException();
    int c = ctl.get();

    // ① 线程数 < core：直接建核心线程
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true)) return;
        c = ctl.get();
    }

    // ② 入队
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (!isRunning(recheck) && remove(command))   // 入队后池已关闭 → 回滚并拒绝
            reject(command);
        else if (workerCountOf(recheck) == 0)        // 池在运行但没有 worker → 补一个
            addWorker(null, false);
    }
    // ③ 队列满 → 建非核心线程
    else if (!addWorker(command, false))
        // ④ 达到 maximum → 拒绝
        reject(command);
}
```

| 步骤 | 条件 | 动作 |
| :--- | :--- | :--- |
| ① | `workerCount < corePoolSize` | 新建核心线程执行 |
| ② | 线程数已达标 | `workQueue.offer` 入队 |
| ③ | 队列满且 `workerCount < maximumPoolSize` | 新建非核心线程 |
| ④ | 队列满且达到上限 | 拒绝策略 |

**这个顺序是最容易答错的地方**：线程池不是「先扩到最大再排队」，而是「**先排队，排不下才扩容**」。直接后果是：用 `LinkedBlockingQueue` 不传容量时，队列永远不满，`maximumPoolSize` 形同虚设。

### 3.3 addWorker：如何安全地加一个线程

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 池已关闭：除「SHUTDOWN 且无首任务且队列非空」外，一律拒绝
        if (rs >= SHUTDOWN &&
            !(rs == SHUTDOWN && firstTask == null && !workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            if (compareAndIncrementWorkerCount(c))   // CAS 占位，成功才继续
                break retry;
            c = ctl.get();                            // 失败重读状态
            if (runStateOf(c) != rs) continue retry;  // 状态变了，回外层重判
        }
    }

    Worker w = new Worker(firstTask);
    final Thread t = w.thread;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        int rs = runStateOf(ctl.get());
        if (rs < SHUTDOWN || (rs == SHUTDOWN && firstTask == null)) {
            workers.add(w);          // HashSet，必须加锁
            int s = workers.size();
            if (s > largestPoolSize) largestPoolSize = s;
            t.start();               // 启动线程
            workerStarted = true;
        }
    } finally {
        mainLock.unlock();
    }
    if (!workerStarted) addWorkerFailed(w);
    return workerStarted;
}
```

三个设计点：

1. **先 CAS 增加计数占位，再真正创建线程**，避免超发。
2. **`workers` 是 `HashSet`**，必须用 `mainLock` 保护；这也是为什么线程池在高并发扩容时会有短暂的锁竞争。
3. **`t.start()` 在锁内**，保证 worker 加入集合与启动的原子性。

### 3.4 Worker 与 runWorker

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    final Thread thread;
    Runnable firstTask;
    volatile long completedTasks;

    public void run() { runWorker(this); }

    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) { setExclusiveOwnerThread(Thread.currentThread()); return true; }
        return false;
    }
    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null); setState(0); return true;
    }
}
```

**Worker 为什么要继承 AQS**：它需要一把**不可重入**的锁，用来标记「当前线程是否正在执行任务」。`shutdown()` 时通过 `tryLock()` 判断——锁被占说明正在跑任务，不能中断；锁空闲说明在等任务，可以中断。用 `ReentrantLock` 做不到「不可重入」这个语义。

```java
final void runWorker(Worker w) {
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock();                     // 允许中断
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();               // 标记正在执行
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) &&
                !Thread.currentThread().isInterrupted())
                Thread.currentThread().interrupt();
            try {
                beforeExecute(w, task);   // 钩子
                task.run();
                afterExecute(task, null); // 钩子
            } catch (Throwable ex) {
                afterExecute(task, ex);
                throw ex;
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

**执行完任务不退出**、继续 `getTask()` 取下一个——这就是「线程复用」的实现。

### 3.5 getTask：线程何时退出

```java
private Runnable getTask() {
    boolean timedOut = false;
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 池已 STOP，或 SHUTDOWN 且队列为空 → worker 退出
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount(); return null;
        }

        int wc = workerCountOf(c);
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c)) return null;  // 退出
            continue;
        }

        try {
            Runnable r = timed
                    ? workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS)  // 非核心：限时取
                    : workQueue.take();                                    // 核心：阻塞取
            if (r != null) return r;
            timedOut = true;         // 超时未取到，下轮循环退出
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

退出条件汇总：

| 条件 | 行为 |
| :--- | :--- |
| 状态 ≥ STOP | 立即退出 |
| SHUTDOWN 且队列空 | 退出 |
| 线程数 > core 且 `poll` 超时 | 退出 |
| `allowCoreThreadTimeOut=true` 且超时 | 核心线程也退出 |
| 队列 `take()` 阻塞 | 不退出，等待新任务 |

### 3.6 关闭流程

```text
shutdown()
  → 状态置 SHUTDOWN
  → 中断空闲 worker（tryLock 成功的）
  → 队列中的任务继续执行
  → 队列空 + worker 全退出 → TIDYING → terminated()

shutdownNow()
  → 状态置 STOP
  → 中断所有 worker
  → 清空队列，返回未执行任务列表
  → TIDYING → terminated()
```

---

## 四、深入与高级延伸

### 4.1 参数怎么定：从公式到压测

| 任务类型 | 经验公式 | 说明 |
| :--- | :--- | :--- |
| CPU 密集 | `N + 1` | N 为核数，多 1 个用于偶尔的页缺失 |
| IO 密集 | `N × (1 + 等待时间/计算时间)` | 常见 2N，需按实际 IO 比例算 |
| 混合型 | 拆分线程池隔离 | 避免慢 IO 拖垮 CPU 任务 |

更实用的估算：`线程数 ≈ QPS × 平均 RT(秒)`。例如 500 QPS、RT 50 ms → 约 25 个线程，再乘 1.5~2 的安全系数。

**队列容量**：不宜过大。队列是「延迟的请求」，积压越多，超时越严重。经验上按「可接受的排队时长 × QPS」估算，并配合拒绝策略与监控。

### 4.2 线程池隔离

生产上不要用一个线程池承载所有任务：

```text
order-pool     核心下单链路，独立队列 + Abort 策略
notify-pool    短信/推送，允许丢弃，Discard + 告警
export-pool    报表导出，大队列 + CallerRuns 背压
```

隔离的价值：**一个慢依赖只会打满自己的池，不会拖垮全局**。这也是 Hystrix/Sentinel 线程池隔离思想的底层实现。

### 4.3 动态线程池

`ThreadPoolExecutor` 原生支持运行时改参数：

```java
executor.setCorePoolSize(newCore);
executor.setMaximumPoolSize(newMax);
```

`setCorePoolSize` 的内部逻辑值得注意：调大时**会补建线程**，调小时**会中断多余的 idle worker**，而不是简单地改个数字。

结合配置中心可实现动态调参：

```java
@RefreshScope
@Component
public class DynamicPool {
    private ThreadPoolExecutor executor;

    @EventListener
    public void onConfigChange(ConfigChangeEvent e) {
        executor.setCorePoolSize(e.getCore());
        executor.setMaximumPoolSize(e.getMax());
    }
}
```

### 4.4 监控指标

| 指标 | 获取方式 | 告警含义 |
| :--- | :--- | :--- |
| 活跃线程数 | `getActiveCount()` | 持续接近 maximum → 容量不足 |
| 池大小 | `getPoolSize()` | 长期等于 core → 负载低或队列在堆积 |
| 队列长度 | `getQueue().size()` | 持续增长 → 消费能力不足 |
| 已完成任务数 | `getCompletedTaskCount()` | 结合时间算吞吐 |
| 最大池大小 | `getLargestPoolSize()` | 历史峰值，用于容量规划 |
| 拒绝次数 | 自定义 handler 计数 | **最关键**，直接反映过载 |

`beforeExecute` / `afterExecute` 是埋点的天然位置：

```java
@Override
protected void beforeExecute(Thread t, Runnable r) {
    startTime.set(System.nanoTime());
}

@Override
protected void afterExecute(Runnable r, Throwable ex) {
    long cost = System.nanoTime() - startTime.get();
    metrics.record(cost, ex);
}
```

### 4.5 生产高频坑

| 现象 | 根因 | 处理 |
| :--- | :--- | :--- |
| 内存持续上涨直至 OOM | 无界队列堆积任务 | 改有界队列 + 拒绝策略 + 监控 |
| `maximumPoolSize` 不生效 | 用了 `LinkedBlockingQueue` 默认无界 | 显式指定容量 |
| 线上无法定位是哪个池 | 未自定义线程名 | `ThreadFactory` 统一前缀 |
| 任务莫名消失 | `DiscardPolicy` 静默丢弃 | 自定义策略 + 计数告警 |
| 上下文串号（用户信息错乱） | 线程复用，ThreadLocal 未 `remove` | `finally` 中清理 |
| 提交任务后拿不到异常 | `submit` 把异常封进 Future | 用 `execute` 或 `Future.get()` |
| 父子任务互相等待死锁 | 同一池中父任务等子任务，池被父任务占满 | 拆分池或用 `ForkJoinPool` |
| 关闭时任务丢失 | 直接 kill 进程 | `shutdown` + `awaitTermination` |
| 定时任务堆积 | `scheduleAtFixedRate` 任务执行超过周期 | 改 `scheduleWithFixedDelay` 或加限流 |

### 4.6 与 ForkJoinPool 的分工

| 维度 | ThreadPoolExecutor | ForkJoinPool |
| :--- | :--- | :--- |
| 任务模型 | 独立任务 | 分治任务（fork/join） |
| 队列 | 每池一个共享队列 | 每线程一个双端队列 |
| 取任务 | 从队列头取 | 自己队列头取，空闲时**从别人队尾窃取** |
| 适用 | 通用异步、IO 任务 | 可拆分的计算密集型 |

`parallelStream` 默认使用 `ForkJoinPool.commonPool()`，其并行度默认为 `CPU 核数 - 1`，在 IO 场景下容易成为瓶颈——这是「parallelStream 不要用于阻塞任务」的原因。

---

## 五、总结整理

### 5.1 全景回顾

```text
execute(task)
   │
   ├─ ① workerCount < core ──► addWorker(task, true)
   ├─ ② workQueue.offer 成功 ──► 等 Worker 取
   ├─ ③ 队列满 + 未达 max ──► addWorker(task, false)
   └─ ④ 达 max ──► reject(handler)

Worker.run → runWorker
   while (task != null || (task = getTask()) != null)
       beforeExecute → task.run → afterExecute
   getTask：核心 take()，非核心 poll(keepAlive)
```

状态流转：

```text
RUNNING ──shutdown()──► SHUTDOWN ──队列空/worker=0──► TIDYING ──► TERMINATED
   └────shutdownNow()──► STOP ──────────────────────► TIDYING ──► TERMINATED
```

### 5.2 速查清单

| # | 问题 | 一句话答案 |
| :--- | :--- | :--- |
| 1 | 任务提交的四个步骤 | 核心线程 → 入队 → 非核心线程 → 拒绝 |
| 2 | 为什么无界队列让 maximum 失效 | 队列永不满，永远走不到扩容分支 |
| 3 | ctl 的作用 | 一个 int 高 3 位存状态、低 29 位存线程数，保证原子读写 |
| 4 | Worker 为什么继承 AQS | 需要不可重入锁标记「是否正在执行任务」 |
| 5 | 线程如何复用 | `runWorker` 循环调用 `getTask()`，执行完不退出 |
| 6 | 非核心线程何时退出 | `poll(keepAlive)` 超时未取到任务 |
| 7 | shutdown 与 shutdownNow 区别 | 前者执行完队列，后者中断并清空队列 |
| 8 | 默认拒绝策略 | `AbortPolicy`，抛 `RejectedExecutionException` |
| 9 | CallerRunsPolicy 的意义 | 让提交线程自己执行，形成背压 |
| 10 | submit 与 execute 区别 | submit 有返回值，异常封装在 Future 中 |
| 11 | 核心线程会回收吗 | 默认不会，`allowCoreThreadTimeOut(true)` 后可回收 |
| 12 | 线程数怎么估 | `QPS × RT`，再乘安全系数；CPU 密集 N+1，IO 密集 2N |
| 13 | 为什么禁用 Executors | 无界队列或无限线程，OOM 风险 |
| 14 | 如何监控 | activeCount / queue.size / completedTaskCount / 拒绝计数 |
| 15 | 动态调参怎么生效 | `setCorePoolSize` 会补建或中断 idle worker |

### 5.3 三条追问链

**追问链 1：maximumPoolSize 为什么不生效**

```text
明明设了 max=100，为什么线程数一直停在 core？
  → 检查队列类型
  → LinkedBlockingQueue 默认容量是 Integer.MAX_VALUE
  → 队列永远 offer 成功，走不到 addWorker(task, false)
  → 改 ArrayBlockingQueue 并显式指定容量
```

**追问链 2：线程池 OOM 怎么排查**

```text
收到 OOM，怎么定位是线程池导致？
  → dump 堆，看是否有大量 Runnable/Task 对象被队列持有
  → 看 getQueue().size() 监控曲线是否持续上升
  → 看拒绝计数是否为 0（为 0 说明从未触发过载保护）
  → 结论：无界队列 + 消费能力不足
```

**追问链 3：优雅停机为什么丢任务**

```text
发布时任务丢失，原因是什么？
  → 是否直接 kill -9（没有触发 shutdown 钩子）
  → 是否 shutdown 后没有 awaitTermination 就退出 JVM
  → 是否 shutdownNow 清空了队列
  → 正确做法：摘流量 → shutdown → awaitTermination → 强制退出兜底
```

### 5.4 决策速查

| 场景 | 建议 |
| :--- | :--- |
| 核心交易链路 | 独立池 + 有界队列 + Abort + 告警 |
| 通知/推送 | 独立池 + 允许丢弃 + 计数 |
| 批量导出 | 大队列 + CallerRuns 背压 |
| 定时任务 | `ScheduledThreadPoolExecutor`，用 `scheduleWithFixedDelay` |
| 分治计算 | `ForkJoinPool` |
| 虚拟线程（JDK 21） | IO 密集场景可用 `newVirtualThreadPerTaskExecutor`，但需注意 pinning |

---

> 本文基线：JDK 8 / 17 / 21。
