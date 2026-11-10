---
title: "JVM 全量深度解析：内存分区、对象生命周期、GC 与线上调优"
date: 2026-11-10T21:00:00+08:00
summary: "一篇把 JVM 讲透的长文：运行时数据区为什么这样划分、对象从 new 到回收的完整源码链路（TLAB 指针碰撞、动态年龄判定）、类加载五阶段与双亲委派的三种打破（SPI/TCCL 的真相）、GC 四种算法与七种收集器、安全点与 OopMap、卡表写屏障、CMS/G1/ZGC 三条低停顿路线、JIT 分层编译与内存屏障、八类 OOM 与四类排查路径，以及四个真实调优案例。每个结论带成立条件和代价。"
categories: ["JVM"]
tags: ["JVM", "GC", "G1", "ZGC", "类加载", "双亲委派", "TLAB", "OOM", "内存屏障", "JIT"]
author: ZLLiang
ShowToc: true
TocOpen: false
draft: false
---

"JVM 内存结构"大概是每个 Java 后端面试的第一题，也往往是最答不深的一题。多数人能背出"堆、栈、方法区、程序计数器"，但再往下问一句"为什么 `-Xmx` 设了 4G，容器却因为 5G 内存被 OOMKilled"，或者"为什么 G1 下明明内存没满还会 Full GC"，很多人就接不上了——因为背名词背得住，但没搞清**内存到底分给谁、对象到底怎么活、GC 到底在停什么**。

这篇文章把 JVM 从内存分区到对象生命周期、从类加载到 GC 工程取舍、从 JIT 到线上排查全量展开，默认 HotSpot，主线 JDK 17、兼容 JDK 8/21 差异。每个结论都落到"为什么只能这么设计"和"代价是什么"。读完你应该能：看懂 `jstat`/GC 日志、讲清 G1 和 ZGC 的本质区别、并在 OOM 或 CPU 飙高时 10 分钟内定位。

## 一、JVM 到底解决了什么：内存分区与术语口径

JVM 做了三件事：**屏蔽平台差异**（`.java` 编译成平台无关的 `.class`，各平台 JVM 翻译成机器指令）、**自动内存管理**（分配走 TLAB、回收走 GC，但泄漏与停顿的责任转移给了你）、**运行时支撑**（类加载、反射、JIT、线程调度）。关键认知：JVM 不是 Java 专属——Kotlin、Scala、Groovy 都编译成 class 文件，GraalVM 还能 AOT 编译成本地镜像。**JVM 规范（JVMS）才是标准，HotSpot 只是最主流的一份实现。**

### 1.1 运行时数据区：五块内存与"为什么这样分"

```
┌───────────────────────── 线程私有（随线程生灭） ─────────────────────────┐
│  程序计数器 PC   │  虚拟机栈（栈帧×N）  │  本地方法栈（Native）            │
└──────────────────────────────────────────────────────────────────────────┘
┌───────────────────────── 线程共享（随进程生灭） ─────────────────────────┐
│  堆 Heap：新生代(Eden+S0+S1) + 老年代        │  方法区 Metaspace（元空间）  │
│  ├ 对象实例、数组                            │  ├ 类元信息、方法字节码       │
│  └ 字符串常量池（JDK7 起移到堆）              │  └ 运行时常量池、静态变量     │
└──────────────────────────────────────────────────────────────────────────┘
┌───────────────────────── 堆外（不受 -Xmx 管） ───────────────────────────┐
│  直接内存 Direct Memory（NIO DirectByteBuffer） │  JNI / 线程栈 / 元空间      │
└──────────────────────────────────────────────────────────────────────────┘
```

三条设计依据：**生命周期对齐**（栈帧与方法调用同生共死，放线程私有区就无需 GC）；**并发成本**（私有区不需要同步，堆和元空间必须处理并发分配）；**可预测性**（每块有独立容量参数和独立 OOM 错误，出问题能指着某一块说"是它满了"）。

| 区域 | 存什么 | 线程 | 溢出错误 | 参数 |
| :--- | :--- | :--- | :--- | :--- |
| 程序计数器 PC | 当前线程执行的字节码行号 | 私有 | **不会 OOM**（唯一） | — |
| 虚拟机栈 | 栈帧：局部变量表、操作数栈、动态链接、返回地址 | 私有 | `StackOverflowError` / `OutOfMemoryError` | `-Xss`（默认 1M） |
| 本地方法栈 | Native 方法栈帧 | 私有 | 同上 | `-Xss` |
| 堆 | 对象实例、数组、字符串常量池（7+） | 共享 | `OutOfMemoryError: Java heap space` | `-Xms -Xmx -Xmn` |
| 方法区（元空间） | 类元信息、字节码、运行时常量池、静态变量 | 共享 | `OutOfMemoryError: Metaspace` | `-XX:MetaspaceSize` |
| 直接内存 | DirectByteBuffer、Netty 池化内存 | 堆外共享 | `OutOfMemoryError: Direct buffer memory` | `-XX:MaxDirectMemorySize`（默认≈`-Xmx`） |

**两个必须记住的细节**：`-Xmx` **不包含**元空间和直接内存，所以"堆设 4G、容器因 5G 被杀"是常态；栈溢出（递归过深 `StackOverflowError`）与栈内存溢出（线程太多 `unable to create native thread`）是两回事，后者本质是 OS 线程数/内存上限，与堆无关。

### 1.2 术语辨析：先把口径对齐，否则全是歧义

| 术语 | 准确含义 | 常见误用 |
| :--- | :--- | :--- |
| **JMM** | 一套**并发语义规范**（happens-before、可见性、有序性） | ❌ 被当成"堆栈方法区"，那是**运行时数据区** |
| **GC Roots** | 可达性分析起点集合（栈引用、静态变量、JNI 引用、锁对象等） | ❌ 只记"栈和静态变量" |
| **安全点 Safepoint** | 线程可安全暂停的**代码位置**，不是时间点 | ❌ 以为任意时刻都能 STW |
| **OopMap** | 安全点上"栈/寄存器哪些位置是对象引用"的映射表 | — |
| **卡表 Card Table** | 堆按 512B 分卡，记录"哪些卡可能有跨代引用" | ❌ 与 RSet 混为一谈 |
| **RSet** | G1 中每个 Region 记录"谁引用了我"（points-into） | — |
| **TLAB** | 线程私有的 Eden 分配缓冲区 | ❌ 以为它是"新生代" |
| **SATB** | G1 并发标记方案：写前记录旧值快照 | ❌ 与 CMS 增量更新混淆 |
| **染色指针** | ZGC 把标记位直接编进 64 位指针里 | — |
| **逃逸分析** | JIT 判断对象是否逃出方法，进而栈上分配/标量替换/锁消除 | ❌ 以为解释执行阶段就有 |

### 1.3 永久代为什么换成了元空间

JDK 8 起方法区从"永久代（PermGen，堆内固定上限）"换成"元空间（Metaspace，本地内存）"，三个理由：① 永久代有 JVM 设定的固定上限无法调整，元空间受本机内存限制，溢出概率更小；② 类元数据不再受 `MaxPermSize` 控制，能加载更多类；③ HotSpot 与 JRockit 合并时，JRockit 从来没有永久代，合并后没必要保留。**代价**：元空间默认无上限，一旦动态生成类失控，会吃光物理内存——所以容器里**必须显式设 `-XX:MaxMetaspaceSize`**。

## 二、对象的一生：从 new 到回收

### 2.1 创建五步

```java
Object o = new Object();
```

| 步骤 | 做什么 | 关键点 |
| :--- | :--- | :--- |
| 1. 类加载检查 | 常量池能否定位符号引用？类是否已加载？ | 没加载就先触发类加载 |
| 2. 分配内存 | 划出确定大小的空间 | 规整→指针碰撞；不规整→空闲列表；**是否规整由 GC 是否压缩决定** |
| 3. 并发安全 | 多线程同时分配 | 方案 A：CAS 重试；方案 B：**TLAB**（每线程先预划一块） |
| 4. 零值初始化 | 实例字段全部置零（不含对象头） | 这就是"字段有默认值"的来源，发生在 `<init>` 之前 |
| 5. 设置对象头 + `<init>` | 写 Mark Word、类型指针；调构造方法 | 此时对象才真正"可用" |

**为什么零值初始化必须在 `<init>` 之前**：构造方法里可能出现"把 this 传出去"的代码，若字段还是随机值，别的线程会读到脏数据。JMM 的 `final` 语义也靠这条顺序 + 构造完成前的 StoreStore 屏障保证。

### 2.2 对象的内存布局与 Mark Word

```
┌──────── 对象头 Header（12B） ────────┬── 实例数据 ──┬── 对齐填充 ──┐
│ Mark Word 8B        │ Klass Pointer 4B │ int/long/ref  │ 补齐到 8 的倍数│
└────────────────────────────────────────┴───────────────┴───────────────┘
```

**Mark Word 的复用**（64 位，最后 2 bit 是锁标志位）：

| 状态 | 62 bit 内容 | 锁标志 |
| :--- | :--- | :--- |
| 无锁 | unused:25 \| hashCode:31 \| unused:1 \| 分代年龄:4 \| 0 | `01` |
| 偏向锁 | 线程 ID:54 \| epoch:2 \| unused:1 \| 分代年龄:4 \| 1 | `01` |
| 轻量级锁 | 指向栈中 Lock Record 的指针:62 | `00` |
| 重量级锁 | 指向 ObjectMonitor 的指针:62 | `10` |
| GC 标记 | 空（标记阶段复用） | `11` |

这张表是理解 `synchronized` 升级的钥匙：**Mark Word 只有 64 bit，hashCode、分代年龄、锁状态、GC 标记全在抢同一块空间**，所以"对象调用 `hashCode()` 之后偏向锁会失效"、"GC 期间不能保留 hashCode"都是这个约束的必然结果。

**压缩指针**：`-XX:+UseCompressedOops`（堆 < 32G 默认开）把引用从 8B 压到 4B，靠"8 字节对齐 + 左移 3 位"寻址，最大寻址 32G——**这就是"堆不要超过 32G"的真实原因**，超了压缩失效，指针反而变 8B，有效容量可能倒退。`-XX:+UseCompressedClassPointers` 默认开，Klass Pointer 4B，对应独立的 `Compressed Class Space`（默认 1G）。

### 2.3 访问定位：句柄 vs 直接指针

| 方式 | 结构 | 优点 | 缺点 |
| :--- | :--- | :--- | :--- |
| 句柄 | 栈引用 → 句柄池[实例指针, 类型指针] → 对象 | 对象移动只改句柄，引用稳定 | 两次间接寻址 |
| 直接指针 | 栈引用 → 对象（对象头内放类型指针） | 一次寻址，**HotSpot 采用** | 对象移动需改栈引用（靠 OopMap 修正） |

HotSpot 用直接指针的理由：**访问频率极高，省一次寻址的收益远大于 GC 时修正引用的成本**（反正 GC 也要遍历 OopMap）。

### 2.4 分配策略与晋升：什么时候进老年代

| 策略 | 规则 | 参数 |
| :--- | :--- | :--- |
| **优先 Eden** | 新对象在 Eden 分配（TLAB） | — |
| **大对象直接进老年代** | 超阈值直接进，避免 Eden/Survivor 反复复制 | `-XX:PretenureSizeThreshold` |
| **长期存活晋升** | 每熬过 Minor GC 年龄 +1，达标晋升 | `-XX:MaxTenuringThreshold` |
| **动态年龄判定** | Survivor 中同龄对象总和 > 一半，则 ≥ 该年龄的直接晋升 | `-XX:TargetSurvivorRatio`（默认 50） |
| **空间分配担保** | Minor GC 前检查老年代连续空间是否够 | — |

**动态年龄判定的源码**（`ageTable::compute_tenuring_threshold`）：

```cpp
uint ageTable::compute_tenuring_threshold(size_t survivor_capacity) {
  size_t desired_survivor_size = (size_t)((((double)
    survivor_capacity)*TargetSurvivorRatio)/100);
  size_t total = 0;
  uint age = 1;
  while (age < table_size) {
    total += sizes[age];                 // sizes 数组是每个年龄段对象大小
    if (total > desired_survivor_size) break;
    age++;
  }
  uint result = age < MaxTenuringThreshold ? age : MaxTenuringThreshold;
  ...
}
```

一个常被问的冷知识：**"默认晋升年龄 15"只对 Parallel（吞吐）收集器成立，CMS 的默认值是 6**（`MaxTenuringThreshold` 最大值 15 是因为 Mark Word 只有 4 bit 存年龄）。**生产含义**："明明没多少长生命周期对象，老年代却涨得快"——先查 Survivor 是否太小导致**动态年龄提前晋升**，再看大对象。

### 2.5 四种引用与回收时机

| 引用 | 回收时机 | 典型用途 |
| :--- | :--- | :--- |
| 强引用 | 永不可达才回收 | 普通对象；**集合里长期持有就是泄漏源头** |
| 软引用 `SoftReference` | 内存不足时回收 | 本地缓存（慎用，行为不可控） |
| 弱引用 `WeakReference` | **下次 GC 必回收** | `ThreadLocalMap.Entry` 的 key、`WeakHashMap` |
| 虚引用 `PhantomReference` | 随时回收，仅用于回收通知 | `DirectByteBuffer` 的 Cleaner 释放堆外内存 |

`WeakReference` + `ReferenceQueue` 是理解 NIO 堆外内存释放的关键：`DirectByteBuffer` 被回收时，其 `Cleaner`（虚引用子类）被入队，由 `ReferenceHandler` 线程调用 `unsafe.freeMemory` 真正释放。所以"堆外内存泄漏"往往是 `DirectByteBuffer` 对象本身还被引用着。

### 2.6 判断死亡：可达性分析，不是引用计数

HotSpot 用的是**可达性分析**（从 GC Roots 向下搜索引用链），不是引用计数。引用计数法的问题在于**循环引用**：`a.next = b; b.next = a`，两个对象互相引用、计数永不为 0，但已无法被外部访问，导致泄漏——所以 JVM 放弃引用计数。GC Roots 包括：栈帧中的引用、静态变量、常量、JNI 引用、活跃线程、被 synchronized 持有的锁对象。

## 三、类加载：五个阶段与双亲委派的打破

### 3.1 五个阶段

```
加载 → 验证 → 准备 → 解析 → 初始化 → 使用 → 卸载
```

| 阶段 | 做的事 | 易错点 |
| :--- | :--- | :--- |
| 加载 | 取二进制流 → 生成 `Klass` → 堆里生成 `Class` 对象 | 数组类不通过类加载器创建，JVM 直接生成 |
| 验证 | 文件格式、元数据、字节码、符号引用验证 | 可用 `-Xverify:none` 关闭（生产不建议） |
| 准备 | 静态变量分配内存并赋**零值** | `static int a = 1` 此阶段 a=0；但 `static final int A = 1`（编译期常量）直接为 1 |
| 解析 | 符号引用 → 直接引用 | 可在初始化之后（动态绑定/反射） |
| 初始化 | 执行 `<clinit>`（静态赋值 + 静态块） | **JVM 保证 `<clinit>` 线程安全**，这是静态内部类单例线程安全的原因 |

六种"主动引用"触发初始化：`new`/`getstatic`/`putstatic`/`invokestatic`、反射 `Class.forName`（注意 `loadClass` 不触发）、初始化子类先初始化父类、启动主类、`MethodHandle`/`VarHandle` 目标类、接口 default 方法。

### 3.2 双亲委派与 loadClass 模板方法

```java
protected Class<?> loadClass(String name, boolean resolve) {
    synchronized (getClassLoadingLock(name)) {      // ① 按类名分段锁，不是锁整个加载器
        Class<?> c = findLoadedClass(name);          // ② 已加载直接返回
        if (c == null) {
            try {
                if (parent != null) c = parent.loadClass(name, false);  // ③ 委派父
                else                c = findBootstrapClassOrNull(name); // ④ 到顶
            } catch (ClassNotFoundException ignored) { }
            if (c == null) c = findClass(name);      // ⑤ 父加载不到才自己找
        }
        if (resolve) resolveClass(c);
        return c;
    }
}
```

四个必须理解的细节：**锁粒度是 `getClassLoadingLock(name)`**（JDK 7 起按类名分段，并发加载不同类不互相阻塞）；**`findLoadedClass` 查的是 `SystemDictionary`**，键是 `(类名, 加载器)`——所以同一个 class 文件被两个加载器加载，得到两个不同的类（`instanceof` 为 false），Tomcat 应用隔离、热部署 `ClassCastException` 全源于此；**`Class.forName` 默认 `initialize=true`**（执行 `<clinit>`），`loadClass` 不初始化；**`resolve` 通常传 false**（惰性解析，启动优化）。

双亲委派的价值有两条：**避免重复加载** + **防止核心类被篡改**（你写个 `java.lang.String` 永远加载不到自己的）。

### 3.3 三次破坏：SPI/TCCL、Tomcat、热部署

| 场景 | 为什么必须破坏 | 怎么做 |
| :--- | :--- | :--- |
| **SPI**（JDBC、Logback） | 接口在 JDK 侧（父侧），实现在 classpath（子侧），父看不见子 | `Thread.currentThread().getContextClassLoader()`，即 TCCL |
| **Tomcat** | 一个容器跑多个 WebApp，需应用隔离 + 共享公共库 | 每个 webapp 一个 `WebappClassLoader`，**先自己找、再委派父** |
| **热部署 / OSGi** | 类要能卸载、替换 | 自定义加载器 + 模块化依赖图 |
| **字节码增强**（Arthas、SkyWalking） | 替换已加载类的字节码 | `Instrumentation.retransformClasses` |

**这里有个几乎人人都说错的点：SPI 并没有重写 `loadClass`，双亲委派一行都没改**。TCCL 是 `Thread` 上一个普通字段（创建时从父线程继承一次），它提供的是**另一个加载器引用**，让父侧的代码可以拿子侧的 loader 去 `Class.forName(name, false, loader)`。委派链约束的是"某个 loader 自己怎么找类"，不是"你不能用别的 loader 找类"——所以准确说法是**规则不动，通道另开**。真正靠重写 `loadClass` 改变委派顺序的只有 Tomcat 的 `WebappClassLoader`。两者机制完全不同，混为一谈会在追问时露馅。

`ServiceLoader.load(Driver.class)` 内部取的就是 TCCL，`LazyIterator` 用构造时存下的 loader 去 `Class.forName(cn, false, loader)`。反证：如果把这里换成 `service.getClassLoader()`，对 `java.sql.Driver` 拿到的是 `null`（Bootstrap）或 Platform loader，**永远找不到 classpath 下的实现**——TCCL 是这条链路上唯一的出口。注意 JDK 9 后 `java.sql` 归 Platform ClassLoader 管（不是 Bootstrap），版本差异要能分清。

### 3.4 元空间为什么会满 + 无用类的卸载条件

元空间在本地内存，理论上只受物理内存限制，但三个泄漏源：**动态生成类**（CGLIB/JDK Proxy/JSON 序列化/Groovy）、**热部署旧加载器未回收**、**`MaxMetaspaceSize` 设得过小**。判定方法：`jstat -gcutil` 看 `M` 列持续上涨且 Full GC 不降 → `jmap -clstats` 看类加载器数量是否只增不减。

类要被卸载（"无用类"）需**同时满足三个条件**：① 该类的所有实例已被回收；② 加载它的 ClassLoader 已被回收；③ 对应的 `Class` 对象无任何引用（无法反射访问）。满足也只是"**可以**被回收"，不是必然。G1 下类卸载发生在并发标记周期的 `ClassUnloading` 阶段。

## 四、GC 的工程取舍：算法、分代与跨代引用

### 4.1 四种算法与分代假设

| 算法 | 做法 | 优点 | 缺点 | 使用者 |
| :--- | :--- | :--- | :--- | :--- |
| 标记-清除 | 标记存活 → 清除垃圾 | 简单、不移动对象 | **碎片化** | CMS |
| 标记-复制 | 存活对象复制到另一半 | 无碎片、分配快 | 浪费一半、复制成本随存活率上升 | 新生代（8:1:1） |
| 标记-整理 | 存活对象向一端移动 | 无碎片、不浪费 | 移动成本高、需 STW | 老年代 |
| 分代收集 | 按存活周期分区，各区用最合适算法 | 综合成本最低 | 需处理跨代引用 | 几乎所有现代收集器 |

**分代假设（Weak Generational Hypothesis）**：绝大多数对象朝生夕灭，熬过越多次 GC 的对象越难死——这就是"新生代用复制、老年代用标记整理/清除"的全部理由。

### 4.2 GC 术语精确分类（易混）

针对 HotSpot，GC 准确分类只有两大类：

- **部分收集（Partial GC）**：`Minor GC`（只新生代）、`Major GC`（只老年代，有的语境也指整堆）、`Mixed GC`（新生代 + 部分老年代，G1 特有）；
- **整堆收集（Full GC）**：整个 Java 堆 + 方法区。

"Major GC"和"Full GC"经常被混用，严格说 Major GC 只指老年代，Full GC 才是整堆 + 方法区——面试被追问时能分清这一点是加分项。

### 4.3 安全点与 OopMap：STW 到底停在哪

GC 要找到所有引用，就得知道**栈和寄存器里哪些位置是引用、哪些是 int**。逐个扫描栈帧极其昂贵，所以 HotSpot 在**安全点**（方法返回前、循环回边、异常抛出点、native 调用返回后）记录一张 OopMap：某偏移处是 Oop 还是非 Oop。**主动式中断**：GC 设标志位，线程在安全点轮询（`test` 指令读内存页）发现标志就挂起。

两个真实生产现象：**长循环导致 STW 变长**——非 counted loop 内部没有安全点轮询，线程迟迟到不了安全点，GC 只能干等（`-XX:+UseCountedLoopSafepoints` 就是为此存在）；**TTSP（Time To Safepoint）比 GC 时间长**——日志里 `Total time for which application threads were stopped` 远大于各阶段耗时之和，差额就是等线程到安全点的时间。阻塞线程（sleep/blocked）用**安全区域**处理：进入区域时标记自己，离开时必须检查 GC 是否完成。

### 4.4 卡表与写屏障：跨代引用怎么解

Minor GC 只扫新生代，但老年代对象可能持有新生代对象的引用——若只扫 Eden 的 GC Roots，会把活对象误判为垃圾。解法：**把堆按 512B 切卡，脏卡记录"可能有跨代引用"**，Minor GC 只扫脏卡。

写屏障是每次引用赋值时 JIT 插入的代码：

```c
// 伪代码（oop_store_at）
if (is_in_young(new_value) && !is_in_young(store_addr)) {
    card_table[card_of(store_addr)] = dirty;   // 标脏，不立即扫描
}
```

**开销**：每次引用赋值多几条指令——这是"GC 不是免费的"的直接体现，G1 的写屏障比 CMS 更重（还要维护 SATB 队列与 RSet）。**G1 里卡表升级为 RSet**：每个 Region 记录"谁引用了我"（points-into），粒度是"其他 Region 的卡"。**RSet 占内存可达堆的 10%~20%**——这是 G1 不适合超小堆、也不适合"堆几乎全满"场景的原因之一。一句话对比：卡表解决"老→新"单向跨代；RSet 解决"任意 Region→我"的多对一；SATB 解决并发标记期间引用被改导致漏标。

### 4.5 七种收集器一览

| 收集器 | 分代 | 算法 | 停顿 | 适用 |
| :--- | :--- | :--- | :--- | :--- |
| Serial / Serial Old | 新生代/老年代 | 复制 / 标记整理 | 长 STW | 客户端、小堆（<100M） |
| ParNew | 新生代 | 复制（多线程） | STW | 曾与 CMS 搭配 |
| Parallel Scavenge / Old | 新生代/老年代 | 复制 / 标记整理 | STW，**吞吐优先** | 批处理（JDK 8 默认） |
| CMS | 老年代 | 标记清除（并发） | 低停顿、有碎片 | JDK 9 废弃、14 移除 |
| **G1** | 整堆分区 | 复制 + 标记整理 | 可预测停顿 | **JDK 9+ 默认**，4G~32G |
| **ZGC** | 整堆 | 并发标记-复制（染色指针） | **<1ms** | 超大堆、极低延迟 |
| Shenandoah | 整堆 | 并发标记-整理 | 低停顿 | 低延迟，非 Oracle 主线 |

G1/ZGC/Shenandoah 是**整堆收集器**，不能再和新生代/老年代收集器自由搭配；CMS 只能配 Serial Old 或 ParNew。

## 五、三条低停顿路线：CMS、G1、ZGC

### 5.1 CMS：并发标记的代价（历史包袱，但要能讲）

| 阶段 | 停顿 | 关键动作 |
| :--- | :--- | :--- |
| 初始标记 | STW | 只标 GC Roots 直接关联对象（快） |
| 并发标记 | 并发 | 从初始标记集合遍历对象图 |
| 重新标记 | STW | **增量更新**：修正并发期间被修改的引用 |
| 并发清除 | 并发 | 清除垃圾，**不压缩** |

三个致命问题：**浮动垃圾**（并发清除期间新产生的垃圾本轮收不掉，需预留空间，预留不足 → `Concurrent Mode Failure` → 退化为 Serial Old 单线程 Full GC，秒级停顿）；**碎片**（不整理，大对象分配失败 → `promotion failed` → 提前 Full GC）；**对 CPU 敏感**（并发阶段占用线程）。

**增量更新 vs SATB**：CMS 用增量更新（记录**新值**，重新标记时以新值为根重扫）；G1 用 SATB（记录**旧值**，保证并发标记开始时活着的对象都被认为活着）。前者漏标风险靠"重扫新增引用"补，后者靠"多标不漏标"。

### 5.2 G1：分区化 + 停顿预测

堆被切成约 2048 个等大 Region（1~32MB），每个 Region 动态扮演 Eden/Survivor/Old/Humongous（≥Region 一半的大对象）角色——**逻辑分代、物理分区**，新生代不再是一块连续内存。

一次 Young GC 流程：初始标记（STW）→ 并发标记（SATB，统计每个 Region 存活率）→ 最终标记（STW）→ 筛选回收（STW，按"垃圾最多收益最高"排序 Region，在 `MaxGCPauseMillis` 预算内选 CSet）。

**`-XX:MaxGCPauseMillis=200` 是"目标"不是"保证"**：G1 用历史数据建立停顿预测模型（衰减均值 + 标准差），估算本次能回收多少个 Region。堆压力大时目标常被突破，日志出现 `Evacuation Failure`。**Mixed GC** 回收年轻代 + 部分老年代 Region，触发条件是老年代占用超过 `InitiatingHeapOccupancyPercent`（默认 45%）启动并发标记。**Full GC**：当 Mixed GC 追不上分配速度（`to-space exhausted`）时退化为整堆回收，停顿可达数秒——**G1 下出现 Full GC 就是事故信号**。

三个关键日志信号：`Pause Young (Normal)`（正常）；`Evacuation Failure`（晋升放不下，加 `-XX:G1ReservePercent` 或增堆）；`to-space exhausted`（复制目标耗尽，查是否有大量短命大对象）。

### 5.3 ZGC：染色指针 + 读屏障

ZGC 把 64 位指针切成：

```
┌─ 16 bit 保留 ─┬─ 42 bit 对象地址 ─┬─ 4 bit 元数据 ─┬─ 1 bit 标记 ─┐
│               │  (4TB 地址空间)   │ M0 M1 Remapped │  Finalizable│
```

**Marked0/Marked1** 两个标记位交替使用（"视图切换"，避免每次 GC 清标记）；**Remapped** 表示对象未被移动。因为标记信息在**指针里**而不是对象头里，所以对象移动后**不需要改对象头，只需要改指针**——这是并发转移能成立的前提。

应用线程读引用时触发**读屏障（Load Barrier）**：检查指针标记位，已 Remapped 直接返回，未重映射则转发到新地址并"自愈"（修好旧指针）。**代价**：每次引用读取多几条指令，牺牲吞吐换极低延迟（JDK 21 P99 < 1ms，与堆大小无关）；内存占用更高（4TB 虚拟地址空间、多重映射）、不支持压缩指针（`-XX:-UseCompressedOops` 强制）。

| 传统 GC | ZGC |
| :--- | :--- |
| STW 复制/整理 | 并发标记 + 并发转移，STW 只做根扫描 |
| 停顿随堆增大线性增长 | 停顿与堆大小、存活对象数量基本无关 |
| 需要 RSet / 卡表 | 染色指针 + 读屏障替代 |

## 六、执行引擎的另一半：JIT 与内存屏障

### 6.1 分层编译五级

| 层级 | 编译器 | 特点 |
| :--- | :--- | :--- |
| 0 | 解释器 | 启动即执行，带 profiling |
| 1 | C1（无 profiling） | 快速编译 |
| 2 | C1（有限 profiling） | 小幅优化 |
| 3 | C1（完整 profiling） | 收集分支/类型信息供 C2 用 |
| 4 | C2 | 激进优化（内联、逃逸分析、循环展开、向量化） |

路径 `0 → 3 → 4` 是常态；方法很简单时走 `0 → 1`（不收集 profiling）。热点探测靠**方法计数器**（调用次数超阈值）和**回边计数器**（OSR，长循环运行中切换到编译版）。

**逃逸分析**在 C2 阶段做三种优化：栈上分配（对象不逃出方法，随栈帧销毁，HotSpot 实际多用标量替换达到等效效果）、**标量替换**（对象可拆散则字段变局部变量，对象根本不创建）、**锁消除**（锁对象不逃逸则去掉 `synchronized`）。所以"所有对象都在堆分配"是规范允许但不必然的实现。

### 6.2 去优化（Deoptimization）

C2 的优化基于"假设"，假设被打破就退回解释执行：类型假设失败（只见过 `ArrayList` 突然来 `LinkedList` → `uncommon trap`）、类加载改变（单态变多态）、数组长度假设。日志 `-XX:+PrintCompilation` 出现 `made not entrant`，`-XX:+TraceDeoptimization` 看原因。

### 6.3 JMM 在 x86 的落地：内存屏障

JMM 是**规范**，HotSpot 在 x86 上的落地：

| 屏障 | x86 实现 | 作用 |
| :--- | :--- | :--- |
| LoadLoad / StoreStore / LoadStore | 无需指令（x86 不重排） | 禁止对应重排 |
| **StoreLoad** | `lock addl $0, (%rsp)` 或 `mfence` | **唯一必须显式指令**，代价最高（20~100 周期） |

**x86 是 TSO（强内存模型），只有 StoreLoad 需要屏障**，所以 `volatile` 写的成本主要在 `lock` 前缀（它还顺带把写缓冲刷回主存，实现可见性）。在 ARM（弱内存模型）上四种屏障都可能有实际指令，同一段并发代码在 ARM 服务器上更容易暴露 bug。`final` 字段的特殊语义：构造函数内 final 字段写入与"把对象引用赋给其他变量"之间插入 StoreStore 屏障，保证"别人看到引用时 final 字段一定已初始化"——这是"安全发布"的底层支撑。

## 七、线上实战：OOM、排查路径与调优案例

### 7.1 八类 OOM，别再只答"堆满了"

| 错误信息 | 真正耗尽 | 典型根因 | 排查入口 |
| :--- | :--- | :--- | :--- |
| `Java heap space` | 堆 | 集合无界增长、缓存无淘汰、一次查全表 | `jmap -histo` / MAT |
| `GC overhead limit exceeded` | 堆（GC 无收益） | 98% 时间在 GC 却只回收 <2% | 看 GC 日志 |
| `Metaspace` | 元空间 | 动态生成类、热部署 | `jmap -clstats` |
| `Compressed class space` | 压缩类指针区（1G） | 类数量超限 | 加大 `CompressedClassSpaceSize` |
| `Direct buffer memory` | 堆外直接内存 | Netty 未释放、DirectByteBuffer 未回收 | `jcmd VM.native_memory` |
| `unable to create native thread` | 线程数/栈内存 | 线程池无界、`ulimit -u` | `jstack` 数线程 |
| `Requested array size exceeds VM limit` | 单数组长度 | `new byte[Integer.MAX_VALUE]` | 代码问题 |
| `Out of swap space / native memory` | 物理内存/swap | 堆外泄漏、容器 limit 太小 | `top -p`、NMT |

### 7.2 四类排查路径

**CPU 100%**：`top -Hp <pid>` 找最耗 CPU 线程 → `printf '%x\n' <tid>` 转十六进制 → `jstack <pid> | grep -A 30 "nid=0x<hex>"`。栈特征是业务方法 = 死循环；GC 线程 = 疯狂回收；`C2 CompilerThread` 大量 = JIT 编译风暴；`BLOCKED` 多但 CPU 不高 = 锁竞争（不是 CPU 问题）。

**频繁 Full GC**：`jstat -gcutil <pid> 1000` 看 `O` 列与 `FGC` 次数增长。判读：`O` 每次 FGC 后能降但涨得快 = 晋升速率过高（新生代太小/大对象多）；`O` 几乎不降 = **真泄漏**（dump + MAT 找 GC Root 路径）；`M` 持续上涨 = 元空间泄漏；FGC 少但单次极长 = 堆太大或有 `System.gc()`。

**内存缓慢上涨（最考验功力）**：往往是低频泄漏——长连接 `ThreadLocal` 未清理、本地 `HashMap` 当缓存只 put 不 remove、`static` 集合装 Listener 不注销、ClassLoader 泄漏。MAT 三个必看视图：Histogram（哪个类实例多，但 `char[]` 排第一不代表字符串泄漏，要顺着看被谁持有）、Dominator Tree（谁"独占"最多内存）、Path to GC Roots（排除弱/软引用）。

**OOM 现场保留（必须提前配）**：

```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/dump
-XX:+ExitOnOutOfMemoryError   # 容器里尤其重要：OOM 后 JVM 半死，健康检查可能还返回 200
-Xlog:gc*:file=/data/logs/gc.log:time,uptime,level,tags:filecount=10,filesize=50M
```

### 7.3 GC 日志怎么读（G1）

```
[2.341s][info][gc] GC(12) Pause Young (Normal) (G1 Evacuation Pause) 512M->96M(1024M) 12.345ms
```

三个判断基线：Young GC 后存活对象（96M）逐次上升 = 有对象持续晋升；单次 Young GC > 50ms 且堆 < 8G = 需看 Region 是否太多；`Pause Full (G1 Compaction Pause)` 出现即事故。

### 7.4 四个调优案例

**案例一：堆从 8G 加到 16G，Full GC 反而更频繁**——`MaxGCPauseMillis=200` 未变，堆变大后 Region 变大（8M/16M），单 Region 复制成本上升；`IHOP=45%` 意味着要堆到 7.2G 才开始并发标记，标记周期拉长，Mixed GC 追不上。处置：`InitiatingHeapOccupancyPercent=35` + `G1ReservePercent=15`，Full GC 归零。**结论：堆不是越大越好，G1 调优是"让并发标记更早开始"。**

**案例二：接口超时，GC 日志显示 Young GC 只有 20ms**——`jstat -gc` 发现 TTSP 很长：`Total time for which application threads were stopped` 比 `Pause Young` 大 200ms+。根因：批量对账代码里有超长非 counted loop，线程长时间不到安全点。**结论：GC 停顿 ≠ STW 总时长，差额是"等线程到安全点"。**

**案例三：容器 OOMKilled，JVM 却没抛 OOM**——`-Xmx4g`、容器 limit 4Gi，但堆外还有元空间 256M + 直接内存（Netty，约等于 `-Xmx`）+ CodeCache 240M + 线程栈（500 线程×1M）。**结论：容器里配 JVM 参数第一原则是"所有内存加起来 < limit"，不是只写 `-Xmx`。**

**案例四：批量任务跑完老年代不释放**——代扣批量任务凌晨跑完后老年代不释放，上午开始频繁 Full GC，P99 从 100ms 涨到 2s。`jstat -gcutil` 确认 FGC 后老年代只降 5%（判断泄漏不是容量）；`jmap -dump` + MAT Dominator Tree 定位到 `static Map<String, List<Detail>>`，key 是商户号、value 是当天全量明细，跑批只 put 不 clear。改成 Guava Cache 带 `expireAfterWrite` + 跑批结束显式清理，同时把 `-Xmx` 从 4G 降到 3G（原来是为扛泄漏才加大），Full GC 从每天 6 次降到 0。**结论：内存泄漏靠加大堆治标不治本，反而掩盖根因。**

## 八、版本演进、容器化与参数清单

### 8.1 版本演进

| 版本 | 关键变化 | 影响 |
| :--- | :--- | :--- |
| JDK 7 | 字符串常量池移到堆；G1 实验 | `intern()` 不再受永久代限制 |
| **JDK 8** | **永久代 → 元空间**；G1 商用 | `PermGen space` OOM 消失，改 `Metaspace` |
| JDK 9 | **G1 默认**；CMS 废弃；模块化 | `rt.jar` 消失 |
| JDK 11 | ZGC 实验 | 亚毫秒 GC 登场 |
| JDK 14 | **CMS 移除** | 只能选 G1/ZGC/Shenandoah |
| JDK 15 | ZGC、Shenandoah 转正 | 生产可用 |
| JDK 17 | 强封装内部 API | 反射 `sun.misc.Unsafe` 受限 |
| **JDK 21** | **虚拟线程**、**分代 ZGC** | 高并发 IO 线程模型改变；ZGC 吞吐提升 |

### 8.2 容器化三个必知差异

① **内存**：JDK 8u191+ 支持 `UseContainerSupport`，按 cgroup limit 算默认堆；**JDK 8 早期版本按宿主机内存算**，这是"容器里堆开到 64G"的经典事故。② **CPU**：`Runtime.availableProcessors()` 老版本返回宿主机核数，导致 GC 线程、线程池默认大小失真，必要时显式 `-XX:ActiveProcessorCount=N`。③ **时区/字符集**：基础镜像常是 UTC + POSIX，`-Duser.timezone=Asia/Shanghai -Dfile.encoding=UTF-8` 要显式加。

### 8.3 参数速查与"不要乱动"清单

**建议显式设置的 8 个**：`-Xms = -Xmx`（相等，避免堆动态扩缩抖动）；`-XX:MaxRAMPercentage` 60~75（容器场景替代写死 `-Xmx`）；`-XX:MaxMetaspaceSize` 256~512m；`-XX:MaxDirectMemorySize` 按 Netty 用量；`-Xss` 512k~1m；`-XX:+HeapDumpOnOutOfMemoryError`（必开）；`-Xlog:gc*`（必开）；`-XX:+ExitOnOutOfMemoryError`。

**不要乱动的 6 个**：`-XX:NewRatio`/`-Xmn`（G1 下无效或有害）；`-XX:MaxTenuringThreshold`（默认合理）；关闭 `UseCompressedOops`（除非堆 >32G 被迫）；`-XX:ParallelGCThreads`（容器里已处理）；`-XX:+DisableExplicitGC`（会连带禁用 NIO 依赖的 `System.gc()` 兜底）；`-XX:+AlwaysPreTouch`（只在延迟敏感场景用）。

## 收口：一页纸记住 JVM

JVM 的核心是**自动内存管理 + 字节码执行引擎**：运行时数据区分线程私有（PC/栈/本地栈）与共享（堆/元空间），堆外还有直接内存；对象从 `new` 经 TLAB 指针碰撞分配，熬过 Minor GC 在 Survivor 累积年龄后晋升老年代；回收靠可达性分析 + 分代假设，跨代引用用卡表/写屏障解决；收集器沿 Serial → Parallel → CMS → G1 → ZGC 演进，本质是"吞吐与停顿"的取舍史；类加载靠双亲委派，SPI/Tomcat/热部署三种方式破坏它。

五条边界记住：**`-Xmx` 不等于进程内存上限**（还有元空间、直接内存、栈、CodeCache、GC 结构）；**GC 停顿 ≠ STW 总时长**（差额是等线程到安全点）；**G1 的 `MaxGCPauseMillis` 是目标不是保证**，出现 Full GC 就是事故；**内存泄漏靠加大堆治标不治本**；**"所有对象都在堆分配"不是必然**（逃逸分析可标量替换）。排查三板斧：`jstat -gcutil` 看趋势，`jmap`+MAT 找泄漏根因，`top -Hp`+`jstack` 定位 CPU 与死锁。
