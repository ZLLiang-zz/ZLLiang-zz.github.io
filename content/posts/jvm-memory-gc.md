---
title: "JVM 内存与 GC：从对象分配到线上 OOM 排查"
date: 2026-10-24T09:00:00+08:00
summary: "把「堆栈方法区」还原成一条真实链路：对象怎么分配、TLAB 怎么工作、卡表与写屏障为什么存在、G1 的 Evacuation Failure 意味着什么，以及线上 OOM 与 CPU 飙高时的排查顺序。"
categories: ["Java 运行时"]
tags: ["JVM", "GC", "内存模型", "线上排查"]
author: ZLLiang
ShowToc: true
TocOpen: false
draft: false
---

## 一、基本概念

### 1.1 JVM 到底解决了什么问题

一句话：**把"一次编写，到处运行"从口号变成工程现实，同时替程序员管理内存。**

它做了三件事：

| 职责 | 具体表现 | 对你的意义 |
| :--- | :--- | :--- |
| 屏蔽平台差异 | `.java` 编译成与平台无关的 `.class` 字节码，由各平台的 JVM 翻译成机器指令 | 同一份 jar 在 Linux x86 / ARM / Windows 上都能跑 |
| 自动内存管理 | 分配由 TLAB bump-pointer 完成，回收由 GC 完成 | 不用 free，但**泄漏与停顿的责任转移给了你** |
| 运行时支撑 | 类加载、动态链接、反射、异常、线程调度、JIT 优化 | 语言特性（泛型擦除、lambda、反射）都靠它兜底 |

**关键认知**：JVM 不是"Java 专属"。Kotlin、Scala、Groovy 编译出的也是 class 文件；GraalVM 还能把字节码直接编译成本地镜像（AOT）。所以 JVM 规范（`JVMS`）才是标准，HotSpot 只是最主流的一份实现。

### 1.2 运行时数据区全景

```
┌───────────────────────── 线程私有（随线程生灭） ─────────────────────────┐
│  程序计数器 PC   │  虚拟机栈（栈帧×N）  │  本地方法栈（Native）            │
└──────────────────────────────────────────────────────────────────────────┘
┌───────────────────────── 线程共享（随进程生灭） ─────────────────────────┐
│  堆 Heap：新生代(Eden+S0+S1) + 老年代        │  方法区 Metaspace（元空间）  │
│  ├ 对象实例、数组                            │  ├ 类元信息、方法字节码       │
│  └ 字符串常量池（JDK7 起移到堆）              │  ├ 运行时常量池、静态变量     │
│                                              │  └ 即时编译代码缓存(CodeCache)│
└──────────────────────────────────────────────────────────────────────────┘
┌───────────────────────── 堆外（不受 -Xmx 管） ───────────────────────────┐
│  直接内存 Direct Memory（NIO DirectByteBuffer） │  JNI / 线程栈 / 元空间      │
└──────────────────────────────────────────────────────────────────────────┘
```

**为什么这样分？** 三条设计依据：

1. **生命周期对齐**：栈帧与一次方法调用同生共死，放线程私有区就无需 GC；对象存活时间不可预测，只能放共享堆由 GC 统一管理。
2. **并发成本**：线程私有区不需要任何同步；堆与元空间必须处理并发分配（TLAB / CAS / 类加载锁）。
3. **可预测性**：分区让每块内存有独立的容量参数与独立的 OOM 错误信息，出问题能指着某一块说"是它满了"。

### 1.3 核心术语（先对齐口径，否则后面全是歧义）

| 术语 | 准确含义 | 常见误用 |
| :--- | :--- | :--- |
| **JVM 内存模型（JMM）** | 一套**并发语义规范**：happens-before、可见性、有序性 | ❌ 被当成"堆栈方法区"，那是**运行时数据区** |
| **运行时数据区** | 本文 1.2 的内存分区 | ❌ 被叫成"JVM 内存模型" |
| **GC Roots** | 可达性分析的起点集合（栈引用、静态变量、JNI 引用、锁对象等） | ❌ 只记"栈和静态变量" |
| **安全点 Safepoint** | 线程可以安全暂停的**代码位置**，不是时间点 | ❌ 以为任意时刻都能 STW |
| **OopMap** | 在安全点上记录"栈/寄存器里哪些位置是对象引用"的映射表 | — |
| **卡表 Card Table** | 把堆按 512B 分卡，记录"哪些卡里可能有跨代引用"的字节数组 | ❌ 与 RSet 混为一谈 |
| **RSet** | G1 中每个 Region 记录"谁引用了我"（points-into） | — |
| **TLAB** | 线程私有的 Eden 分配缓冲区，分配走指针碰撞无锁 | ❌ 以为它是"新生代" |
| **PLAB** | GC 期间 Survivor 晋升用的线程私有缓冲区，TLAB 的镜像概念 | — |
| **SATB** | G1 的并发标记方案：写前记录旧值快照 | ❌ 与 CMS 的增量更新混淆 |
| **染色指针** | ZGC 把标记位直接编到 64 位指针里 | — |
| **逃逸分析** | JIT 判断对象是否逃出方法/线程，进而栈上分配、标量替换、锁消除 | ❌ 以为解释执行阶段就有 |

### 1.4 体系定位：JVM 在整个技术栈里的位置

```
业务代码  →  字节码(.class)  →  [ 类加载子系统 ]  →  运行时数据区
                                      ↓
                              [ 执行引擎：解释器 + JIT(C1/C2) + GC ]
                                      ↓
                              操作系统（线程、内存映射、信号）
```

理解这条链路的价值：**所有"玄学问题"都能落回某一段**。类冲突是类加载段；OOM 是数据区段；CPU 100% 是执行引擎段（JIT 编译线程或业务死循环）；`OutOfMemoryError: unable to create native thread` 则是 JVM 向 OS 申请线程失败——不在堆里。

---

## 二、基础知识点（怎么用）

### 2.1 五块内存：容量、异常、参数对照

| 区域 | 存什么 | 线程 | 溢出错误 | 相关参数 |
| :--- | :--- | :--- | :--- | :--- |
| 程序计数器 PC | 当前线程执行到的字节码行号 | 私有 | **不会 OOM**（唯一不抛 OOM 的区域） | — |
| 虚拟机栈 | 栈帧：局部变量表、操作数栈、动态链接、返回地址 | 私有 | `StackOverflowError` / `OutOfMemoryError` | `-Xss`（默认 1M，容器里常调 512k） |
| 本地方法栈 | Native 方法栈帧 | 私有 | 同上 | `-Xss` |
| 堆 | 对象实例、数组、字符串常量池（7+） | 共享 | `OutOfMemoryError: Java heap space` | `-Xms -Xmx -Xmn -XX:NewRatio -XX:SurvivorRatio` |
| 方法区（元空间） | 类元信息、字段/方法描述、字节码、运行时常量池、静态变量 | 共享 | `OutOfMemoryError: Metaspace` | `-XX:MetaspaceSize -XX:MaxMetaspaceSize` |
| 直接内存 | `DirectByteBuffer` 的堆外缓冲、Netty 的池化内存 | 共享（堆外） | `OutOfMemoryError: Direct buffer memory` | `-XX:MaxDirectMemorySize`（默认≈`-Xmx`） |

**两个必须记住的细节**：

- **`-Xmx` 不包含元空间和直接内存**。所以"堆设了 4G，容器却因 5G 被杀"是常态。容器里应同时约束 `-XX:MaxMetaspaceSize` 和 `-XX:MaxDirectMemorySize`。
- **栈溢出与栈内存溢出不是一回事**：递归过深 → `StackOverflowError`；线程数太多导致无法再创建栈 → `OutOfMemoryError: unable to create native thread`（本质是 OS 线程数/内存上限，与堆无关）。

### 2.2 对象创建：从 new 到可用，五步

```java
Object o = new Object();
```

| 步骤 | 做什么 | 关键点 |
| :--- | :--- | :--- |
| 1. 类加载检查 | 常量池能否定位到类的符号引用？类是否已加载/解析/初始化？ | 没加载就先触发类加载（见 2.5） |
| 2. 分配内存 | 从堆中划出确定大小的空间 | 规整→指针碰撞；不规整→空闲列表；**是否规整由 GC 是否带压缩决定** |
| 3. 并发安全 | 多线程同时分配同一块内存 | 方案 A：CAS 重试；方案 B：**TLAB**（每线程先在 Eden 预划一块，用完再换） |
| 4. 零值初始化 | 实例字段全部置零（不含对象头） | 这就是"对象字段有默认值"的来源，发生在 `<init>` 之前 |
| 5. 设置对象头 + 执行 `<init>` | 写入 Mark Word、类型指针；调用构造方法 | 此时对象才真正"可用" |

> **为什么零值初始化必须在 `<init>` 之前？** 因为构造方法里可能出现"把 this 传出去"的代码，若字段还是随机值，别的线程会读到脏数据。JMM 的 `final` 语义也是靠这条顺序 + 构造完成前的 StoreStore 屏障来保证的。

### 2.3 对象的内存布局（64 位、开启压缩指针）

```
┌──────────── 对象头 Header（12B） ────────────┬──── 实例数据 ────┬── 对齐填充 ──┐
│ Mark Word 8B          │ Klass Pointer 4B     │ int/long/ref...  │ 补齐到 8 的倍数│
└───────────────────────────────────────────────┴──────────────────┴───────────────┘
```

**Mark Word 的复用**（64 位，最后 2 bit 是锁标志位）：

| 状态 | 62 bit 内容 | 锁标志 |
| :--- | :--- | :--- |
| 无锁 | unused:25 \| hashCode:31 \| unused:1 \| 分代年龄:4 \| 0 | `01` |
| 偏向锁 | 线程 ID:54 \| epoch:2 \| unused:1 \| 分代年龄:4 \| 1 | `01` |
| 轻量级锁 | 指向栈中 Lock Record 的指针:62 | `00` |
| 重量级锁 | 指向 ObjectMonitor 的指针:62 | `10` |
| GC 标记 | 空（标记阶段复用） | `11` |

这张表是理解 `synchronized` 升级的钥匙：**Mark Word 只有 64 bit，hashCode、分代年龄、锁状态、GC 标记全在抢同一块空间**，所以"对象调用了 `hashCode()` 之后偏向锁会失效"、"GC 期间不能保留 hashCode"都是这个约束的必然结果。

- **压缩指针**：`-XX:+UseCompressedOops`（堆 < 32G 时默认开），引用从 8B 压到 4B，靠"8 字节对齐 + 左移 3 位"寻址，所以最大可寻址 32G。**这就是"堆不要超过 32G"的真实原因**——超了压缩失效，指针反而变 8B，有效容量可能倒退。
- **`-XX:+UseCompressedClassPointers`** 默认开，Klass Pointer 4B，对应独立的 `Compressed Class Space`（默认 1G），`OutOfMemoryError: Compressed class space` 就是它满了。

### 2.4 对象访问定位：句柄 vs 直接指针

| 方式 | 结构 | 优点 | 缺点 |
| :--- | :--- | :--- | :--- |
| 句柄 | 栈引用 → 句柄池[实例指针, 类型指针] → 对象 | 对象移动（GC 复制/压缩）只改句柄，引用本身稳定 | 两次间接寻址 |
| 直接指针 | 栈引用 → 对象（对象头内放类型指针） | 一次寻址，**HotSpot 采用** | 对象移动需改栈上引用（靠 OopMap 修正） |

为什么 HotSpot 用直接指针：**访问频率极高，省一次寻址的收益远大于 GC 时修正引用的成本**（反正 GC 也要遍历 OopMap）。

### 2.5 类加载：五个阶段与"什么时候才真正加载"

```
加载 → 验证 → 准备 → 解析 → 初始化 → 使用 → 卸载
  ↑                                  ↑
读取字节码生成Class对象          执行 <clinit>（静态块+静态赋值）
```

| 阶段 | 做的事 | 易错点 |
| :--- | :--- | :--- |
| 加载 | 通过全限定名取二进制流 → 生成 `Klass` 结构 → 在堆生成 `Class` 对象 | 数组类不通过类加载器创建，由 JVM 直接生成 |
| 验证 | 文件格式、元数据、字节码、符号引用验证 | 可用 `-Xverify:none` 关闭（生产不建议，提速有限） |
| 准备 | 为**静态变量**分配内存并赋**零值** | `static int a = 1` 此阶段 a=0，赋值在 `<clinit>`；但 `static final int A = 1`（编译期常量）此阶段直接为 1 |
| 解析 | 符号引用 → 直接引用（内存地址） | 可在初始化之后进行（动态绑定/反射） |
| 初始化 | 执行 `<clinit>`，即静态赋值 + 静态块 | **JVM 保证 `<clinit>` 线程安全**，这就是静态内部类单例线程安全的原因 |

**六种"主动引用"会触发初始化**（其余（如通过子类引用父类静态字段）都不触发）：

1. `new` / `getstatic` / `putstatic` / `invokestatic` 四条指令对应的场景；
2. 反射调用 `Class.forName(...)`（注意 `ClassLoader.loadClass` 不触发初始化）；
3. 初始化子类时先初始化父类；
4. 虚拟机启动时的主类（含 `main`）；
5. `MethodHandle` / `VarHandle` 解析到的目标类；
6. 接口中定义了 `default` 方法，其实现类初始化时先初始化该接口。

**双亲委派**：`loadClass` 先查已加载 → 委派父加载器 → 父加载不到才自己 `findClass`。

```
Bootstrap(rt/jrt:/java.base) ← Ext/Platform ← App(System) ← 自定义
```

价值有两条，都要能说出来：① **避免重复加载**（同一个类由同一加载器加载才"相等"）；② **防止核心类被篡改**（你写个 `java.lang.String` 永远加载不到自己的）。

### 2.6 破坏双亲委派的三次历史与四类现实场景

| 场景 | 为什么必须破坏 | 怎么做 |
| :--- | :--- | :--- |
| **SPI**（JDBC、Logback） | 接口在 `java.base`（Bootstrap 加载），实现类在 classpath（App 加载），父加载器看不见子类 | `Thread.currentThread().getContextClassLoader()`，即**线程上下文类加载器** |
| **Tomcat** | 一个容器跑多个 WebApp，需要应用隔离 + 共享公共库 | 每个 webapp 一个 `WebappClassLoader`，**先自己找、再委派父** |
| **热部署 / OSGi** | 类需要能卸载、能替换 | 自定义加载器 + 模块化依赖图 |
| **字节码增强**（Arthas、SkyWalking） | 需要替换已加载类的字节码 | `Instrumentation.retransformClasses` + 自定义 Transformer |

### 2.7 四种引用与回收时机

| 引用 | 回收时机 | 典型用途 |
| :--- | :--- | :--- |
| 强引用 | 永不可达才回收 | 普通对象；**集合里长期持有就是泄漏源头** |
| 软引用 `SoftReference` | 内存不足时回收（`-XX:SoftRefLRUPolicyMSPerMB` 控制存活时长） | 本地缓存（慎用，行为不可控） |
| 弱引用 `WeakReference` | **下次 GC 必回收** | `ThreadLocalMap.Entry` 的 key、`WeakHashMap`、缓存 key |
| 虚引用 `PhantomReference` | 随时可能被回收，仅用于回收通知 | `DirectByteBuffer` 的 Cleaner 释放堆外内存 |

> **`WeakReference` 与 `ReferenceQueue` 的配合**是理解 NIO 堆外内存释放的关键：`DirectByteBuffer` 被回收时，其 `Cleaner`（虚引用子类）被入队，由 `ReferenceHandler` 线程调用 `unsafe.freeMemory` 真正释放。所以"堆外内存泄漏"往往是 `DirectByteBuffer` 对象本身还被引用着。

### 2.8 GC 基础：四种算法与分代假设

| 算法 | 做法 | 优点 | 缺点 | 使用者 |
| :--- | :--- | :--- | :--- | :--- |
| 标记-清除 | 标记存活 → 清除垃圾 | 实现简单、不移动对象 | **碎片化**、标记/清除效率随存活对象增长 | CMS |
| 标记-复制 | 存活对象复制到另一半 | 无碎片、分配快（指针碰撞） | 浪费一半空间、复制成本随存活率上升 | 新生代（Eden:S0:S1 = 8:1:1） |
| 标记-整理 | 标记 → 存活对象向一端移动 | 无碎片、不浪费空间 | 移动成本高、需 STW | 老年代（Serial Old、Parallel Old） |
| 分代收集 | 按存活周期分区，各区用最合适的算法 | 综合成本最低 | 需要处理跨代引用 | 几乎所有现代收集器 |

**分代假设（Weak Generational Hypothesis）**：绝大多数对象朝生夕灭，熬过越多次 GC 的对象越难死。这就是"新生代用复制、老年代用标记整理/清除"的全部理由。

**跨代引用问题**：新生代 GC（Minor GC）要判断老年代是否引用新生代。若全量扫描老年代，Minor GC 就没有意义了。解法是**卡表 + 写屏障**——老年代对象引用新生代对象时，把对应的卡标记为 dirty，Minor GC 只扫脏卡（详见 3.6）。

### 2.9 对象分配策略：什么时候进老年代

| 策略 | 规则 | 参数 |
| :--- | :--- | :--- |
| **优先 Eden** | 新对象在 Eden 分配（TLAB） | — |
| **大对象直接进老年代** | 超过阈值直接进老年代，避免 Eden/Survivor 间反复复制 | `-XX:PretenureSizeThreshold`（只对 Serial/ParNew 有效） |
| **长期存活晋升** | 每熬过一次 Minor GC 年龄 +1，达到阈值晋升 | `-XX:MaxTenuringThreshold`（默认 15，因 Mark Word 只有 4 bit） |
| **动态年龄判定** | Survivor 中同龄对象总和 > 一半，则 ≥ 该年龄的直接晋升 | `-XX:TargetSurvivorRatio`（默认 50） |
| **空间分配担保** | Minor GC 前检查老年代最大连续空间是否 > 新生代总大小/历次晋升平均大小 | `-XX:-HandlePromotionFailure`（JDK6u24 后逻辑已简化） |

> **生产含义**："明明没多少长生命周期对象，老年代却涨得快" —— 先查是否 Survivor 太小导致**动态年龄提前晋升**，再看是否有大对象/大数组。这两个原因占晋升异常的大半。

### 2.10 七种收集器一览与组合关系

| 收集器 | 分代 | 算法 | 停顿 | 适用 |
| :--- | :--- | :--- | :--- | :--- |
| Serial / Serial Old | 新生代/老年代 | 复制 / 标记整理 | 长 STW | 客户端、小堆（< 100M） |
| ParNew | 新生代 | 复制（多线程） | STW | 曾与 CMS 搭配（JDK 14 后随 CMS 移除） |
| Parallel Scavenge / Parallel Old | 新生代/老年代 | 复制 / 标记整理 | STW，**吞吐优先** | 批处理、离线计算（JDK 8 默认） |
| CMS | 老年代 | 标记清除（并发） | 低停顿、有碎片 | JDK 9 废弃、14 移除 |
| **G1** | 整堆分区 | 复制 + 标记整理（Region） | 可预测停顿（默认 200ms 目标） | **JDK 9+ 默认**，大堆（4G~32G）通用 |
| **ZGC** | 整堆（JDK21 分代） | 并发标记-复制（染色指针） | **< 1ms（亚毫秒）** | 超大堆（TB 级）、极低延迟 |
| Shenandoah | 整堆 | 并发标记-整理（Brooks 转发指针） | 低停顿 | 低延迟，非 Oracle JDK 主线 |

**组合规则**：G1/ZGC/Shenandoah 是**整堆收集器**，不能再和新生代/老年代收集器自由搭配；CMS 只能配 Serial Old 或 ParNew。

### 2.11 必须会的诊断命令

| 命令 | 用途 | 关键用法 |
| :--- | :--- | :--- |
| `jps -lvm` | 找 Java 进程与启动参数 | 排查前先确认启动参数（有没有设 `-Xmx`） |
| `jstat -gcutil <pid> 1000` | 每秒看各区使用率与 GC 次数/耗时 | `O`（老年代）持续上涨 = 有对象晋升后不释放 |
| `jstat -gc <pid> 1000` | 看各区容量（KB）与 GC 时间 | 算晋升速率、算 GC 频率是否合理 |
| `jmap -histo:live <pid>` | 类直方图（**会触发 Full GC**） | 快速看谁占了内存；生产慎用，优先 `-histo` 不加 live |
| `jmap -dump:live,format=b,file=x.hprof <pid>` | 堆转储 | 生产用 `-XX:+HeapDumpOnOutOfMemoryError` 自动留现场 |
| `jstack <pid>` | 线程栈 | CPU 高、死锁、线程池满的第一手证据 |
| `jcmd <pid> VM.flags` | 查看最终生效参数 | 确认参数是否被覆盖 |
| `jcmd <pid> GC.heap_info` | 堆概况（比 jmap 轻） | 生产首选 |
| `jinfo -flag <name> <pid>` | 查看/动态修改可写参数 | `PrintGC`、`HeapDumpOnOutOfMemoryError` 可动态开 |

### 2.12 十二条入门误区（每条都对应一个真实故障）

1. "堆 = 新生代 + 老年代"——漏了**字符串常量池与 TLAB 都在 Eden 内**，且 G1 下没有物理连续的新生代。
2. "方法区 = 永久代"——JDK 8 起是**元空间（本地内存）**，永久代只是 JDK 7 及以前的 HotSpot 实现。
3. "`-Xmx` 就是进程最大内存"——还要加元空间、直接内存、CodeCache、线程栈、GC 自身结构（Remembered Set 可达堆的 10%~20%）。
4. "`System.gc()` 能立即回收"——只是建议，`-XX:+DisableExplicitGC` 可让它变成 `System.gc()` 空调用。
5. "`finalize()` 能兜底释放资源"——JDK 9 起废弃（`@Deprecated`），执行时机不确定，可能永不执行。
6. "调大 `-Xmx` 就能解决 OOM"——内存泄漏时只是把故障时间推后。
7. "`OutOfMemoryError` 一定是堆满"——至少 8 种，见 4.1。
8. "GC 停顿只发生在 GC 线程"——**安全点停顿（TTSP）** 可能比 GC 本身还长，尤其线程处于 `counted_loop` 之外的长循环。
9. "`-Xss` 越大越好"——每线程栈内存变大，同样内存下可创建线程数变少。
10. "`jmap -histo:live` 很轻"——它会触发一次 Full GC，大堆上可能停几秒。
11. "堆外内存不受 JVM 管所以不会 OOM"——受物理内存与 `MaxDirectMemorySize` 双重约束，且泄漏后排查更难。
12. "G1 一定比 CMS 快"——小堆（< 4G）下 G1 的 RSet 维护成本可能反超，只是 CMS 已被移除，没得选。

---

## 三、技术原理（怎么工作的）

### 3.1 类加载的底层链路

#### 3.1.1 ClassFile：一张表驱动的结构

class 文件不是"文本"，是严格按表排列的二进制。前 10 个字节就能读出关键信息：

```
CAFEBABE | minor(2B) | major(2B) | constant_pool_count(2B) | constant_pool[] | ...
  魔数      次版本        主版本        常量池计数（从1开始，索引0空出）
```

| 版本号 | JDK | 备注 |
| :--- | :--- | :--- |
| 52 | 8 | 目前存量系统最多 |
| 61 | 17 | LTS |
| 65 | 21 | LTS，虚拟线程 |

**主版本号高于 JVM 支持版本 → `UnsupportedClassVersionError`**。这条错误在生产上 90% 是"编译用 JDK 17，运行用 JDK 8"。

常量池结束后依次是：访问标志 → 类索引/父类索引/接口索引 → 字段表 → 方法表 → 属性表。所谓"元空间存类元信息"，指的就是这些表被解析后放进 `InstanceKlass` 及其嵌套结构（`ConstantPool`、`Method`、`ConstMethod`、`LocalVariableTable` 等）。

#### 3.1.2 loadClass 的模板方法

`ClassLoader#loadClass(String, boolean)`（JDK 8/17 结构一致）：

```java
protected Class<?> loadClass(String name, boolean resolve) {
    synchronized (getClassLoadingLock(name)) {      // ① 按类名分段的锁，不是锁整个加载器
        Class<?> c = findLoadedClass(name);          // ② 已加载直接返回
        if (c == null) {
            try {
                if (parent != null) c = parent.loadClass(name, false);  // ③ 委派
                else                c = findBootstrapClassOrNull(name); // ④ 到顶了
            } catch (ClassNotFoundException ignored) { }
            if (c == null) c = findClass(name);      // ⑤ 父加载不到才自己找
        }
        if (resolve) resolveClass(c);                // ⑥ 链接（通常懒解析）
        return c;
    }
}
```

四个必须理解的细节：

- **锁粒度是 `getClassLoadingLock(name)`**，JDK 7 起按类名分段（`parallelLockMap`），所以并发加载不同的类不会互相阻塞。这也是"自定义加载器要重写 `getClassLoadingLock`"的原因。
- **`findLoadedClass` 是 native**，查的是 JVM 的 `SystemDictionary`，键是 `(类名, 加载器)`。所以**同一个 class 文件被两个加载器加载，得到的是两个不同的类**（`instanceof` 为 false）——Tomcat 应用隔离、热部署 `ClassCastException` 全源于此。
- **`Class.forName` 与 `loadClass` 的区别**：前者默认 `initialize=true`（执行 `<clinit>`），后者不初始化。JDBC 老代码 `Class.forName("com.mysql.jdbc.Driver")` 之所以能注册驱动，正是靠静态块里的 `DriverManager.registerDriver`。
- **`resolve` 参数通常传 false**，解析延后到首次使用（惰性解析），这是启动速度优化。

#### 3.1.3 `<clinit>` 的线程安全与死锁

JVM 规范要求：**同一个类初始化时加锁（`InstanceKlass::_init_state` + `init_lock`），其他线程阻塞等待**，且必须保证只有一个线程执行 `<clinit>`。这带来两个后果：

- **好处**：静态内部类单例（Holder 模式）天然线程安全，不需要任何同步代码。
- **风险**：**类初始化死锁**。两个类的静态块互相 `new` 对方，两个线程分别触发初始化，就是一个经典的死锁，且 `jstack` 里显示的是 `in class loading` 而不是普通锁等待——很多人会误判。

#### 3.1.4 元空间为什么会满

元空间在本地内存，理论上只受物理内存限制，但：

| 泄漏源 | 机制 |
| :--- | :--- |
| 动态生成类 | CGLIB / JDK Proxy / JSON 序列化 / Groovy 脚本，每生成一个新类就多一份元数据 |
| 热部署 / 反复 reload | 旧加载器未被回收（有强引用），其加载的所有类元数据都无法卸载 |
| `-XX:MaxMetaspaceSize` 设得过小 | 默认无限，但容器里常被误设 |

**判定方法**：`jstat -gcutil` 看 `M` 列持续上涨且 Full GC 不降 → `jmap -clstats <pid>` 看类加载器数量是否持续增长 → 基本锁定动态类生成或加载器泄漏。

### 3.2 对象分配的源码链路与 TLAB

#### 3.2.1 从字节码到分配

`new Object()` 编译为：

```
0: new           #2      // class java/lang/Object
3: dup
4: invokespecial #1      // <init>
7: astore_1
```

`new` 指令在 HotSpot 里走 `InterpreterRuntime::_new`：

```
InterpreterRuntime::_new(thread, method, index)
  └─ constantPool->klass_at(index)            // 解析类符号引用（可能触发类加载）
  └─ klass->check_valid_for_instantiation()
  └─ InstanceKlass::allocate_instance(thread)
       └─ MemAllocator::allocate()            // 进入分配器
            ├─ 快速路径：TLAB 内指针碰撞（bump the top）
            └─ 慢路径：allocate_outside_tlab → Eden 加锁/CAS → 可能触发 Minor GC
```

**为什么先 TLAB？** 指针碰撞在无锁下只有一次指针加法（`top += size`），比 CAS 快一个数量级。TLAB 把"并发分配"降级为"私有分配"，只在 refill 时才需要同步——这是 JVM 分配性能的核心设计。

#### 3.2.2 TLAB 的三个指针与浪费

```
TLAB: [ start ........ top ........ end ]
       已分配区        当前分配点      边界
```

- 新对象：`top + size <= end` 则 `top += size`，返回旧 top；
- 放不下且 `size < TLAB 剩余空间的阈值(refill_waste_limit)`：**直接在 Eden 共享区分配（waste 掉剩余空间）**，避免频繁 refill；
- 否则：**retire 当前 TLAB，申请新的**（通常在 GC 时集中处理）。

**这解释了一个现象**：`jstat` 里 Eden 的使用量总比"对象总大小"略大，差额就是 TLAB 尾部浪费。参数 `-XX:TLABSize`（默认 0 表示自动，按 Eden 的 1% 与 `-XX:TLABWasteTargetPercent` 动态调整）。

#### 3.2.3 分配失败 → Minor GC → 晋升

```
Eden 不足
  → 触发 Minor GC（Young GC）
  → 存活对象复制到 Survivor（age+1）
  → age >= MaxTenuringThreshold 或 动态年龄达标 或 Survivor 放不下
  → 晋升老年代（晋升用 PLAB，同样是线程私有缓冲）
  → 老年代空间不足 → 触发 Full GC（G1 下可能是 Mixed GC 或 Full GC）
```

#### 3.2.4 逃逸分析：对象可能根本不进堆

JIT 在 C2 阶段做逃逸分析，三种优化：

| 优化 | 条件 | 效果 |
| :--- | :--- | :--- |
| **栈上分配** | 对象不逃出方法 | 随栈帧销毁，无需 GC（HotSpot 实际很少做真正的栈分配，多用标量替换达到等效效果） |
| **标量替换** | 对象可拆散且不被外部引用 | 字段变成局部变量，对象**根本不创建**（`-XX:+EliminateAllocations`） |
| **锁消除** | 锁对象不逃逸 | 去掉 `synchronized`（`-XX:+EliminateLocks`） |

实测手段：`-XX:+PrintEliminateAllocations`（debug 版）或对比 `-XX:-DoEscapeAnalysis` 前后的分配速率。**一句话：`new` 出来的对象不一定在堆上，"所有对象都在堆分配"是规范允许但不必然的实现。**

### 3.3 安全点与 OopMap：STW 到底停在哪

#### 3.3.1 为什么需要 OopMap

GC 要找到所有引用，就得知道**栈上和寄存器里哪些位置是引用、哪些是 int**。逐个扫描栈帧的解释器帧极其昂贵，所以 HotSpot 在**特定位置**（安全点）记录一张表：某偏移处是 Oop 还是非 Oop。

- **安全点位置**：方法返回前、循环回边（`counted_loop`）、异常抛出点、native 调用返回后。
- **主动式中断**：GC 设一个标志位，线程在安全点轮询（`test` 指令读内存页）发现标志就挂起——比挂起线程（`SuspendThread`）温和得多。

#### 3.3.2 两个真实的生产现象

1. **长循环导致 STW 变长**：`for (int i = 0; i < Integer.MAX_VALUE; i++)` 这种**非 counted loop**（int 边界循环在 C2 里是 counted，但复杂条件可能不是）内部没有安全点轮询，线程迟迟到不了安全点，GC 只能干等。这就是 `-XX:+UseCountedLoopSafepoints`（JDK 10+ 默认开）存在的原因。
2. **`TTSP`（Time To Safepoint）比 GC 时间长**：GC 日志里 `Total time for which application threads were stopped` 远大于各阶段耗时之和，差额就是等所有线程到达安全点的时间。

#### 3.3.3 安全区域（Safe Region）

线程处于 `sleep`/`blocked` 状态时无法走到安全点，此时用**安全区域**：线程进入区域时标记自己，GC 不管它；线程要离开时必须检查 GC 是否完成，未完成就等待。这就是"为什么 STW 时阻塞线程不会被算作逃逸"。

### 3.4 卡表与写屏障：跨代引用怎么解

#### 3.4.1 问题定义

Minor GC 只扫新生代，但老年代对象可能持有新生代对象的引用：

```
老年代: objA.field = objB   （objB 在 Eden）
若只扫 Eden 的 GC Roots，objB 会被误判为垃圾 → 灾难
```

#### 3.4.2 解法：把堆切成卡，脏卡记录"可能有跨代引用"

```
堆地址空间:  ... [card n-1] [card n] [card n+1] ...
每张卡 512B（2^9），卡表是 byte[]，1 字节对应 1 张卡
card_table[addr >> 9] == dirty(0x01) 表示该卡需要扫描
```

**写屏障**：所有引用赋值（`a.field = b`）在字节码执行时由 JIT 插入一段代码：

```c
// 伪代码（oop_store_at）
if (is_in_young(new_value) && !is_in_young(store_addr)) {
    card_table[card_of(store_addr)] = dirty;   // 标脏，不立即扫描
}
```

- **开销**：每次引用赋值多几条指令。这是"GC 不是免费的"的直接体现——**G1 的写屏障比 CMS 更重**（还要维护 SATB 队列与 RSet）。
- **成本控制**：脏卡不会重复标（幂等），且只在跨代赋值时标；同一张卡被反复写只付出一次成本。

#### 3.4.3 卡表在 G1 里的升级：RSet

G1 的 Region 之间没有"新生代/老年代"的物理边界，任何 Region 都可能引用任何 Region。G1 用 **RSet（Remembered Set）** 记录"**谁引用了我**"：

- 每个 Region 一个 RSet，粒度是"其他 Region 的卡"（points-into）；
- Minor GC 时，只要扫 CSet 中各 Region 的 RSet 指向的卡，就能找到外部引用；
- **RSet 占内存可达堆的 10%~20%**——这是 G1 不适合超小堆、也不适合"堆几乎全满"场景的原因之一。

> **一句话对比**：卡表解决"老→新"单向跨代；RSet 解决"任意 Region → 我"的多对一跨区。SATB 则解决并发标记期间"对象引用被修改导致漏标"。

### 3.5 G1：分区化 + 停顿预测

#### 3.5.1 Region 布局

堆被切成约 2048 个等大 Region（1MB~32MB，2 的幂，`-XX:G1HeapRegionSize`），每个 Region 动态扮演角色：

| 角色 | 含义 |
| :--- | :--- |
| Eden | 新对象分配 |
| Survivor | Minor GC 存活对象 |
| Old | 晋升对象 |
| Humongous | **≥ Region 一半**的大对象，连续多个 Region 存放（`-XX:G1HeapRegionSize` 太小时大对象会浪费大量 Region） |

**"逻辑分代、物理分区"**：新生代不再是一块连续内存，而是一组 Eden Region 的集合。

#### 3.5.2 一次 Young GC 的流程

```
1. 初始标记（STW，短暂）      ：标记 GC Roots 直接可达对象
2. 并发标记（与用户线程并行）  ：SATB 遍历对象图，统计每个 Region 的存活率
3. 最终标记（STW，短暂）      ：处理 SATB 剩余队列
4. 筛选回收（STW）           ：按"垃圾最多、收益最高"排序 Region，在 MaxGCPauseMillis 预算内选 CSet
                               复制存活对象到新 Region（ evacuation ），清空旧 Region
```

**`-XX:MaxGCPauseMillis=200` 是"目标"不是"保证"**：G1 用历史数据建立"停顿预测模型"（衰减均值 + 标准差），估算本次能回收多少个 Region。堆压力大时，200ms 目标经常被突破——日志里会出现 `Evacuation Failure`（晋升失败）。

#### 3.5.3 Mixed GC 与 Full GC

- **Mixed GC**：年轻代 + **部分**老年代 Region（并发标记算出的高收益 Region）一起回收；
- **触发**：老年代占用超过 `-XX:InitiatingHeapOccupancyPercent`（默认 45%）启动并发标记周期；
- **Full GC**：当 Mixed GC 也追不上分配速度（`to-space exhausted`）时退化为**单线程/多线程的整堆回收**，停顿可达数秒——**G1 下出现 Full GC 就是事故信号**。

#### 3.5.4 三个关键日志信号

| 日志 | 含义 | 处置 |
| :--- | :--- | :--- |
| `Pause Young (Normal)` | 正常 Young GC | 看 `Evacuation` 耗时 |
| `Evacuation Failure` | 晋升时 Survivor/Old 放不下，退化为 Full GC | 加 `-XX:G1ReservePercent`（默认 10）或增大堆 |
| `to-space exhausted` | 复制目标空间耗尽 | 同上，且要查是否有大量短命大对象 |

### 3.6 CMS：低停顿的代价（历史包袱，但要能讲）

虽然 JDK 14 已移除，CMS 是理解"并发标记"的最佳教材：

| 阶段 | 停顿 | 关键动作 |
| :--- | :--- | :--- |
| 初始标记 | STW | 只标 GC Roots 直接关联对象（快） |
| 并发标记 | 并发 | 从初始标记集合遍历对象图 |
| 重新标记 | STW | **增量更新**：修正并发期间被修改的引用（记录在 Dirty Card + 增量更新队列） |
| 并发清除 | 并发 | 清除垃圾，**不压缩** |

三个致命问题：

1. **浮动垃圾**：并发清除期间新产生的垃圾本轮收不掉，需预留空间（`-XX:CMSInitiatingOccupancyFraction`，默认 68%），预留不足 → `Concurrent Mode Failure` → 退化为 Serial Old 单线程 Full GC（秒级停顿）。
2. **碎片**：标记清除不整理，大对象分配失败 → `promotion failed` → 提前触发 Full GC。
3. **对 CPU 敏感**：并发阶段占用线程，核心数少时吞吐下降明显（`-XX:ParallelGCThreads` / `ConcGCThreads`）。

**增量更新 vs SATB**：CMS 用增量更新（记录**新值**，重新标记时以新值为根重扫）；G1 用 SATB（记录**旧值**，保证并发标记开始时活着的对象都被认为活着）。前者漏标风险靠"重扫新增引用"补，后者靠"多标不漏标"。

### 3.7 ZGC：染色指针 + 读屏障

#### 3.7.1 核心思路

ZGC 把 64 位指针切成：

```
┌─ 16 bit 保留 ─┬─ 42 bit 对象地址 ─┬─ 4 bit 元数据 ─┬─ 1 bit 标记 ─┐
│               │  (4TB 地址空间)   │ M0 M1 Remapped │  Finalizable│
```

- **Marked0 / Marked1**：两个标记位交替使用，避免每次 GC 都要清标记（"视图切换"）；
- **Remapped**：对象未被移动时的状态；
- **Finalizable**：仅用于 `finalize` 可终结对象。

因为标记信息在**指针里**而不是对象头里，所以对象移动后**不需要修改对象头**，只需要改指针——这是并发转移能成立的前提。

#### 3.7.2 并发转移与读屏障

```
应用线程读取引用 o.field
  → 触发读屏障（Load Barrier）
  → 检查指针的标记位：
      - 已 Remapped：直接返回
      - 未重映射：转发到新地址，并"自愈"（把旧指针修好）
```

**读屏障的代价**：每次引用读取多几条指令。ZGC 因此牺牲了一点吞吐换极低延迟（JDK 21 实测 P99 < 1ms，与堆大小无关）。

#### 3.7.3 为什么能做到亚毫秒

| 传统 GC | ZGC |
| :--- | :--- |
| STW 复制/整理 | 并发标记 + 并发转移，STW 只做根扫描（`-XX:ZCollectionInterval` 等） |
| 停顿随堆增大线性增长 | 停顿与堆大小、存活对象数量**基本无关** |
| 需要 RSet / 卡表 | 用**染色指针 + 读屏障**替代，不需要写屏障维护跨代记录（分代 ZGC 另说） |

**代价**：内存占用更高（染色指针需要 4TB 虚拟地址空间、多重映射）、CPU 开销更大（读屏障）、不支持压缩指针（`-XX:-UseCompressedOops` 强制）。

### 3.8 JIT：解释器、C1、C2 的分工

#### 3.8.1 分层编译五级

| 层级 | 编译器 | 特点 |
| :--- | :--- | :--- |
| 0 | 解释器 | 启动即执行，带 profiling |
| 1 | C1（无 profiling） | 快速编译，简单优化 |
| 2 | C1（有限 profiling） | 小幅优化 |
| 3 | C1（完整 profiling） | 收集分支/类型信息，供 C2 使用 |
| 4 | C2 | 激进优化（内联、逃逸分析、循环展开、向量化） |

**路径**：`0 → 3 → 4` 是常态；方法很简单时走 `0 → 1`（不收集 profiling，避免浪费）。

#### 3.8.2 热点探测

- **方法计数器**：调用次数超过 `-XX:CompileThreshold`（C2 默认 10000，分层编译下不同）；
- **回边计数器**：循环体执行次数（OSR，On-Stack Replacement），让长循环在运行中切换到编译版本。

#### 3.8.3 去优化（Deoptimization）

C2 的优化基于"假设"，假设被打破就要退回解释执行：

| 场景 | 触发 |
| :--- | :--- |
| 类型假设失败 | 某处只见过 `ArrayList`，突然来了 `LinkedList` → `uncommon trap` |
| 类加载改变 | 新加载的子类使"单态调用"变"多态" |
| 数组长度假设 | 编译时假设非负 |

日志：`-XX:+PrintCompilation` 出现 `made not entrant`，`-XX:+TraceDeoptimization` 看原因。

### 3.9 JMM 在 JVM 的落地：内存屏障

JMM 是**规范**，HotSpot 在 x86 上的落地方式：

| 屏障 | x86 实现 | 作用 |
| :--- | :--- | :--- |
| LoadLoad | 无需指令（x86 不重排读读） | 禁止读读重排 |
| StoreStore | 无需指令（x86 不重排写写） | 禁止写写重排 |
| LoadStore | 无需指令 | 禁止读写重排 |
| **StoreLoad** | `lock addl $0, (%rsp)` 或 `mfence` | **唯一必须显式指令**，代价最高（约 20~100 周期） |

这解释了一个常被追问的点：**x86 是 TSO（强内存模型），只有 StoreLoad 需要屏障**，所以 `volatile` 写的成本主要在 `lock` 前缀（它还顺带把写缓冲刷回主存，实现可见性）。在 ARM（弱内存模型）上四种屏障都可能有实际指令，所以同一段并发代码在 ARM 服务器上更容易暴露 bug。

**`final` 字段的特殊语义**：构造函数内对 final 字段的写入，与"把对象引用赋给其他变量"之间插入 **StoreStore 屏障**，保证"别人看到引用时，final 字段一定已初始化"。这是"安全发布"的底层支撑。

---

## 四、深入与高级延伸（生产怎么用）

### 4.1 八类 OOM，别再只答"堆满了"

| 错误信息 | 真正耗尽的是什么 | 典型根因 | 排查入口 |
| :--- | :--- | :--- | :--- |
| `Java heap space` | 堆 | 内存泄漏（集合无界增长、缓存无淘汰）、一次查询拉全表 | `jmap -histo` / hprof + MAT |
| `GC overhead limit exceeded` | 堆（GC 无收益） | 98% 时间在 GC 却只回收 <2%，通常是泄漏末期 | 同上，看 GC 日志 |
| `Metaspace` | 元空间 | 动态生成类（CGLIB/脚本）、热部署、加载器泄漏 | `jmap -clstats` |
| `Compressed class space` | 压缩类指针区（默认 1G） | 类数量超限（CompressedClassSpaceSize 太小） | 加大 `-XX:CompressedClassSpaceSize` |
| `Direct buffer memory` | 堆外直接内存 | Netty 未释放 `ByteBuf`、`DirectByteBuffer` 未回收 | `jcmd VM.native_memory`、Netty `ResourceLeakDetector` |
| `unable to create native thread` | 线程数 / 栈内存 | 线程池无界、每线程 1M 栈、`ulimit -u` 限制 | `jstack` 数线程、`pstree`、`ulimit` |
| `Requested array size exceeds VM limit` | 单数组长度 | `new byte[Integer.MAX_VALUE]` | 代码问题 |
| `Out of swap space / native memory` | 物理内存 / swap | 堆外泄漏、JNI、容器 limit 太小 | `top -p`、`pmap`、NMT |

**判断标准**：能区分这八类，说明你**真的看过 GC 日志和错误栈**，而不是背了"堆栈方法区"。

### 4.2 四类线上问题：从现象到结论的固定路径

#### 4.2.1 CPU 100%

```bash
top -Hp <pid>                     # 找最耗 CPU 的线程 id（十进制）
printf '%x\n' <tid>               # 转十六进制
jstack <pid> | grep -A 30 "nid=0x<hex>"
```

| 栈特征 | 结论 |
| :--- | :--- |
| `RUNNABLE` 且栈里有业务方法 | 死循环 / 算法复杂度爆炸 / 正则回溯 |
| `RUNNABLE` 且栈在 `GC` 相关（`GCTaskThread`） | GC 线程在疯狂回收 → 看 `jstat -gcutil` |
| 大量 `C2 CompilerThread` | JIT 编译风暴（大量动态生成方法、反射调用） |
| `BLOCKED` 多但 CPU 不高 | 不是 CPU 问题，是锁竞争（见并发文档） |

#### 4.2.2 频繁 Full GC

固定四步，不要跳：

```bash
jstat -gcutil <pid> 1000 20        # ① 看 O 列与 FGC 次数增长
jstat -gc <pid> 1000 10            # ② 算晋升速率：OU 增长 / 时间
jmap -histo:live <pid> | head -30  # ③ 看谁占大头（注意会触发 FGC）
jcmd <pid> GC.heap_info            # ④ G1 下看 Region 分布
```

判读表：

| 现象 | 结论 | 处置 |
| :--- | :--- | :--- |
| `O` 每次 FGC 后能降下来，但涨得很快 | 晋升速率过高（新生代太小 / 大对象多 / 缓存大） | 加大新生代或查大对象 |
| `O` 每次 FGC 后几乎不降 | **真泄漏**，对象被强引用 | dump + MAT 找 GC Root 路径 |
| `M` 持续上涨 | 元空间泄漏 | 查动态类生成 |
| `FGC` 次数少但单次极长 | 堆太大 / 有 `System.gc()` | 查代码、加 `-XX:+DisableExplicitGC` |

#### 4.2.3 内存缓慢上涨（最考验功力）

"缓慢上涨"往往是**低频泄漏**：

- 长连接 `ThreadLocal` 未清理（线程复用 → value 累积）；
- 本地缓存（`HashMap` 当缓存）只 put 不 remove；
- `static` 集合装 Listener / Callback，注册了不注销；
- `ClassLoader` 泄漏（每次热部署残留一个加载器及其全部类）。

**MAT 的三个必看视图**：

1. **Histogram**：哪个类实例数最多（先看 `char[]`、`byte[]`、业务 DTO）；
2. **Dominator Tree**：谁"独占"了最多内存（比 Histogram 更能找到真凶）；
3. **Path to GC Roots（排除弱/软引用）**：从大对象往上找"谁还引用着我"。

> **常见误判**：Histogram 里 `char[]` 排第一不代表字符串泄漏，`String` 的 `value` 都是 `char[]`/`byte[]`，要顺着看"这些 char[] 被哪个业务对象持有"。

#### 4.2.4 OOM 后现场保留（必须提前配）

```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/dump
-XX:+ExitOnOutOfMemoryError          # 让容器尽快重启，避免僵尸进程占着流量
-Xlog:gc*:file=/data/logs/gc.log:time,uptime,level,tags:filecount=10,filesize=50M
```

`-XX:+ExitOnOutOfMemoryError` 在容器里尤其重要：OOM 后 JVM 往往处于"半死"状态，健康检查可能还返回 200。

### 4.3 GC 日志怎么读（G1 为例）

```
[2.341s][info][gc] GC(12) Pause Young (Normal) (G1 Evacuation Pause) 512M->96M(1024M) 12.345ms
```

| 字段 | 含义 |
| :--- | :--- |
| `GC(12)` | 第 12 次 GC |
| `Pause Young (Normal)` | 普通 Young GC（`Concurrent Start` 表示并发标记周期开始） |
| `512M->96M(1024M)` | 回收前 → 回收后（堆总容量） |
| `12.345ms` | 本次 STW 时长 |

**三个判断基线**：

- Young GC 后存活对象 ≈ 96M，若这个数**逐次上升**，说明有对象持续晋升；
- 单次 Young GC > 50ms 且堆 < 8G，需要看是否 Region 太多或复制量大；
- `Pause Full (G1 Compaction Pause)` 出现即事故，必须分析是 `Evacuation Failure` 还是 `to-space exhausted`。

### 4.4 三个调优案例（讲得出来才算会）

#### 案例一：堆从 8G 加到 16G，Full GC 反而更频繁

- **现象**：8G 时每天 2 次 Full GC；16G 后每天 6 次，单次从 1.2s 涨到 3s。
- **分析**：`MaxGCPauseMillis=200` 未变，堆变大后 G1 每个 Region 变大（`G1HeapRegionSize` 自动取 8M/16M），单 Region 复制成本上升；同时 `IHOP=45%` 意味着要堆到 7.2G 才开始并发标记，标记周期被拉长，期间晋升的对象更多，最终 Mixed GC 追不上。
- **处置**：`-XX:InitiatingHeapOccupancyPercent=35` + `-XX:G1ReservePercent=15` + 限制 `-XX:MaxGCPauseMillis=150`，Full GC 归零。
- **结论**：**堆大小不是越大越好，G1 的调优是"让并发标记更早开始"**。

#### 案例二：接口超时，GC 日志显示 Young GC 只有 20ms

- **现象**：P99 从 80ms 涨到 800ms，但 GC 日志正常，CPU 也不高。
- **分析**：`jstack` 多次采样发现大量线程卡在 `Object.wait` → 定位到连接池 `maxWait`；再看 `jstat -gc` 的 `GCT`（总 GC 时间）与 `FGCT`，发现 **TTSP** 很长：日志里 `Total time for which application threads were stopped` 比 `Pause Young` 大 200ms+。
- **根因**：某段批量对账代码里有超长循环（非 counted loop），线程长时间不到安全点，每次 GC 都要等它。
- **处置**：循环体内加安全点友好写法（避免 `int` 溢出边界的写法、拆分批次），或升级到默认开启 `UseCountedLoopSafepoints` 的版本。
- **结论**：**GC 停顿 ≠ STW 总时长**，差额是"等线程到安全点"。

#### 案例三：容器 OOMKilled，JVM 却没抛 OOM

- **现象**：Pod 被 kill，退出码 137，JVM 日志无异常。
- **分析**：`-Xmx4g`，容器 limit 4Gi，但堆外还有：元空间 256M + 直接内存（Netty，约等于 `-Xmx`）+ CodeCache 240M + 线程栈（500 线程 × 1M）+ GC 结构。
- **处置**：
  ```bash
  -XX:MaxRAMPercentage=70          # 堆占容器 limit 的 70%
  -XX:MaxMetaspaceSize=256m
  -XX:MaxDirectMemorySize=512m
  -XX:ReservedCodeCacheSize=128m
  -Xss512k
  -XX:+UseContainerSupport         # JDK 10+ 默认开
  ```
- **结论**：**容器里配 JVM 参数的第一原则是"所有内存加起来 < limit"**，而不是只写 `-Xmx`。

### 4.5 参数速查与"不要乱动"清单

**建议显式设置的 8 个**：

| 参数 | 建议 | 理由 |
| :--- | :--- | :--- |
| `-Xms` = `-Xmx` | 相等 | 避免堆动态扩缩带来的抖动与额外 GC |
| `-XX:MaxRAMPercentage` | 60~75 | 容器场景替代写死 `-Xmx` |
| `-XX:MaxMetaspaceSize` | 256~512m | 防止元空间无限吃物理内存 |
| `-XX:MaxDirectMemorySize` | 按 Netty 用量 | 默认≈`-Xmx` 太激进 |
| `-Xss` | 512k~1m | 线程多时降栈大小 |
| `-XX:+HeapDumpOnOutOfMemoryError` | 必开 | 现场只有一次 |
| `-Xlog:gc*` | 必开 | 没日志的 GC 问题只能靠猜 |
| `-XX:+ExitOnOutOfMemoryError` | 容器建议开 | 避免半死进程 |

**不要乱动的 6 个**：

1. `-XX:NewRatio` / `-Xmn`：G1 下无效或有害，让 G1 自适应；
2. `-XX:MaxTenuringThreshold`：除非确认晋升异常，否则默认 15 合理；
3. `-XX:+UseCompressedOops` 关闭：除非堆 > 32G 被迫，否则不要关；
4. `-XX:ParallelGCThreads`：默认按 CPU 数推导，容器里 `UseContainerSupport` 已处理；
5. `-XX:+DisableExplicitGC`：会连带禁用 NIO 的 `DirectByteBuffer` 回收依赖的 `System.gc()`（部分框架靠它兜底），需评估；
6. `-XX:+AlwaysPreTouch`：启动时预触碰全部堆，启动变慢但运行更稳，**只在延迟敏感场景用**。

### 4.6 版本演进：每个版本改了什么

| 版本 | 关键变化 | 影响 |
| :--- | :--- | :--- |
| JDK 7 | 字符串常量池移到堆；`-XX:+UseG1GC` 实验 | `intern()` 不再受永久代限制 |
| **JDK 8** | **永久代 → 元空间**；G1 商用 | `PermGen space` OOM 消失，改为 `Metaspace` |
| JDK 9 | **G1 成为默认**；CMS 标记废弃；模块化（jrt:/） | `rt.jar` 消失，`-Xbootclasspath` 失效 |
| JDK 11 | ZGC 实验（`-XX:+UnlockExperimentalVMOptions`） | 亚毫秒 GC 登场 |
| JDK 14 | **CMS 移除**；`jpackage` | 只能选 G1/ZGC/Shenandoah |
| JDK 15 | ZGC 转正；Shenandoah 转正 | 生产可用 |
| JDK 17 | 密封类、强封装 JDK 内部 API | 反射 `sun.misc.Unsafe` 受限，Netty 等需升级 |
| **JDK 21** | **虚拟线程（JEP 444）**、**分代 ZGC**、`SequencedCollection` | 高并发 IO 场景线程模型改变；ZGC 支持分代后吞吐提升 |
| JDK 22+ | 虚拟线程无 pin（部分场景）、Foreign Function & Memory API | JNI 逐步被替代 |

### 4.7 容器化：三个必知差异

1. **内存**：JDK 8u191+ 支持 `UseContainerSupport`，按 cgroup limit 算默认堆（1/4）；**JDK 8 早期版本会按宿主机内存算**，这是"容器里堆开到 64G"的经典事故。
2. **CPU**：`Runtime.availableProcessors()` 在容器里可能返回宿主机核数（老版本），导致 GC 线程、ForkJoinPool、线程池默认大小全部失真。JDK 10+ 修复；必要时显式 `-XX:ActiveProcessorCount=N`。
3. **时区/字符集**：容器基础镜像常是 UTC + POSIX，`-Duser.timezone=Asia/Shanghai -Dfile.encoding=UTF-8` 要显式加（JDK 18 起默认 UTF-8）。

### 4.8 怎么把 JVM 讲成"我解决过的问题"

工作多年之后再看 JVM，不该停在"我知道分代"，而要给出一条**完整的问题链**：

> "2023 年一笔代扣批量任务，每天凌晨跑完后老年代不释放，第二天上午就开始频繁 Full GC，接口 P99 从 100ms 涨到 2s。
> 我先用 `jstat -gcutil` 确认 FGC 后老年代只降 5%，判断是泄漏不是容量问题；然后用 `jmap -dump` 拿快照，MAT 的 Dominator Tree 定位到一个 `static Map<String, List<Detail>>`，key 是商户号但 value 是当天的全量明细，跑批只 put 不 clear。
> 改成 Guava Cache 带 `expireAfterWrite` + 跑批结束显式清理，同时把 `-Xmx` 从 4G 降到 3G（因为原来是为了扛泄漏才加大），Full GC 从每天 6 次降到 0。
> 之后补了两个动作：一是加 `-XX:+HeapDumpOnOutOfMemoryError` 和 GC 日志采集，二是把这类 `static` 集合写进代码规约检查。"

这段话包含了**现象量化 → 定位手段 → 根因 → 修复 → 防复发**，比背十个名词有效得多。

---

## 五、总结整理

### 5.1 全景回顾

```
                     ┌── 类加载：加载→验证→准备→解析→初始化（<clinit> 线程安全）
   .class  ──────────┤
                     └── 双亲委派（可被 SPI/Tomcat/热部署破坏）

   运行时数据区
   ├ 私有：PC（不 OOM）/ 虚拟机栈（-Xss，SOF）/ 本地方法栈
   ├ 共享：堆（-Xmx，新生代+老年代）/ 元空间（-XX:MaxMetaspaceSize）
   └ 堆外：直接内存（-XX:MaxDirectMemorySize）

   对象生命周期
   new → 类加载检查 → TLAB 指针碰撞 → 零值 → 对象头 → <init>
       → Eden → Minor GC → Survivor（age++）→ 晋升 Old
       → 不可达 → 标记 → 清除/复制/整理 → 内存归还

   回收判定
   GC Roots（栈/静态/JNI/锁/常量）+ 可达性分析
   └ OopMap + 安全点（STW 停在哪）

   跨代引用
   卡表（512B/卡）+ 写屏障（脏卡）  →  G1 用 RSet + SATB

   收集器
   Serial / Parallel（吞吐） / CMS（低停顿，已移除） / G1（分区+停顿预测） / ZGC（染色指针+读屏障）

   执行引擎
   解释器 → C1（快速）→ C2（激进优化：内联/逃逸分析/标量替换）
   └ 去优化：假设被打破 → uncommon trap → 退回解释
```

### 5.2 速查清单（30 条，每条一句话）

| # | 问题 | 一句话答案 |
| :--- | :--- | :--- |
| 1 | JVM 内存区域 | PC、虚拟机栈、本地方法栈私有；堆、元空间共享；另有直接内存 |
| 2 | 哪个区域不会 OOM | 程序计数器 |
| 3 | 栈溢出和栈内存溢出 | 前者递归过深 `StackOverflowError`，后者线程太多 `unable to create native thread` |
| 4 | 对象创建五步 | 类加载检查 → 分配 → 零值 → 对象头 → `<init>` |
| 5 | 分配方式 | 指针碰撞（规整）/ 空闲列表（不规整） |
| 6 | 并发分配怎么保证 | CAS 或 TLAB |
| 7 | TLAB 是什么 | 线程私有 Eden 缓冲，分配无锁 |
| 8 | 对象头多大 | 64 位下 12B（Mark Word 8 + Klass 4，压缩指针） |
| 9 | 为什么堆不超过 32G | 超过则压缩指针失效，引用变 8B |
| 10 | 访问定位两种 | 句柄、直接指针（HotSpot 用后者） |
| 11 | 类加载五阶段 | 加载、验证、准备、解析、初始化 |
| 12 | 准备阶段赋什么值 | 静态变量零值；`static final` 常量直接赋初值 |
| 13 | 什么时候触发初始化 | 六种主动引用（new/静态字段/反射/父类/主类/MethodHandle） |
| 14 | 双亲委派好处 | 避免重复加载、防止核心类被篡改 |
| 15 | 怎么破坏双亲委派 | SPI 用线程上下文加载器；Tomcat 先自己找 |
| 16 | 四种引用 | 强、软（内存不足）、弱（下次 GC）、虚（回收通知） |
| 17 | GC 怎么判断死亡 | 可达性分析（GC Roots），不是引用计数 |
| 18 | GC Roots 有哪些 | 栈引用、静态变量、常量、JNI、锁、活跃线程 |
| 19 | 三种基本算法 | 标记清除（碎片）、复制（浪费一半）、标记整理（移动成本） |
| 20 | 为什么分代 | 弱分代假设：绝大多数对象朝生夕灭 |
| 21 | 什么时候晋升老年代 | 年龄达标、动态年龄、大对象、Survivor 放不下 |
| 22 | 跨代引用怎么处理 | 卡表 + 写屏障，只扫脏卡 |
| 23 | 安全点是什么 | 可安全暂停的代码位置（方法返回、循环回边） |
| 24 | 为什么 STW 比 GC 久 | 等所有线程到达安全点（TTSP） |
| 25 | G1 的 Region | 逻辑分代物理分区，Eden/Survivor/Old/Humongous |
| 26 | G1 怎么做到可预测停顿 | 按回收收益排序 Region，在预算内选 CSet |
| 27 | CMS 的两个致命问题 | 浮动垃圾（CMF）、内存碎片 |
| 28 | 增量更新 vs SATB | CMS 记新值重扫；G1 记旧值保证不漏标 |
| 29 | ZGC 为什么快 | 染色指针 + 读屏障，标记信息在指针里，停顿与堆大小无关 |
| 30 | JIT 分层编译 | 解释器 → C1（profiling）→ C2（激进优化），假设破裂则去优化 |

### 5.3 三条追问链

**链一：从 OOM 追到 GC 实现**

> "堆 OOM 怎么查？" → jmap + MAT → "如果是元空间呢？" → `jmap -clstats`、动态类生成 → "为什么动态类会导致元空间泄漏？" → 类卸载条件（加载器可回收 + 无实例 + 无 Class 引用）→ "G1 下类卸载什么时候发生？" → 并发标记周期中的 `ClassUnloading`，Full GC 也会做。

**链二：从 synchronized 追到对象头**

> "synchronized 原理？" → monitor + 对象头 → "对象头里都存什么？" → Mark Word 复用（hashCode/age/锁标志）→ "为什么对象调用 hashCode 后偏向锁失效？" → Mark Word 空间冲突 → "JDK 15 为什么废弃偏向锁？" → 维护成本高、收益低（JEP 374）。

**链三：从 G1 停顿追到写屏障**

> "G1 怎么保证低停顿？" → Region + CSet + 停顿预测 → "怎么知道哪些 Region 该回收？" → 并发标记 + SATB → "SATB 怎么实现？" → 写前屏障把旧值入队 → "写屏障开销大吗？" → 每次引用写多几条指令，比 CMS 的增量更新更重，这是 G1 在小堆上不占优的原因之一。

### 5.4 决策与排查速查表

| 场景 | 首选动作 | 关键判据 |
| :--- | :--- | :--- |
| 堆 < 4G，追求吞吐 | Parallel GC | 无低延迟要求 |
| 堆 4~32G，通用业务 | G1（默认） | `MaxGCPauseMillis` 100~200 |
| 堆 > 32G 或 P99 要求 < 10ms | ZGC（JDK 21 分代） | 接受吞吐下降与内存开销 |
| 老年代只涨不降 | dump + MAT | Dominator Tree + Path to GC Roots |
| 元空间持续上涨 | `jmap -clstats` | 类加载器数量是否只增不减 |
| CPU 100% | `top -Hp` + `jstack` | 线程状态是 RUNNABLE 还是 BLOCKED |
| 容器被 kill（137） | 算总内存账 | 堆 + 元空间 + 直接内存 + 栈 + CodeCache |
| GC 日志正常但接口慢 | 看 TTSP、线程栈、连接池 | STW 总时长 vs 各阶段之和 |

---

> 本文基线：HotSpot / OpenJDK 17 源码，兼容 JDK 8 / 21 差异。
