---
title: "ThreadLocal：哈希算法、内存泄漏与跨线程传递"
date: 2026-10-16T09:00:00+08:00
summary: "数据到底存在哪、0x61c88647 这个魔数从哪来、Entry 为什么是「key 弱 value 强」、内存泄漏的引用链需要同时满足什么条件，以及线程池串号与 TTL 的正确用法。"
categories: ["Java 并发"]
tags: ["ThreadLocal", "并发", "内存泄漏"]
author: ZLLiang
ShowToc: true
TocOpen: false
draft: false
---

## 一、基本概念

### 1.1 是什么

`ThreadLocal` 提供**线程封闭**能力：同一个 `ThreadLocal` 实例，在不同线程中读写到的是各自独立的副本，线程之间互不可见，因此**不需要加锁就能保证线程安全**。

一句话记法：

> **变量存在线程里，ThreadLocal 只是查表的钥匙。**

这句是理解全部源码的起点——很多错误认知（比如「ThreadLocal 存数据」「ThreadLocal 会导致内存泄漏」）都源于把「钥匙」当成了「仓库」。

### 1.2 解决什么问题

| 问题 | 传统做法 | ThreadLocal 做法 |
| :--- | :--- | :--- |
| 上下文层层透传 | 每个方法都加 `User user` 参数 | 入口 `set`，任意层 `get` |
| 非线程安全对象复用 | 每次 new 或加锁 | 每线程一份实例 |
| 事务内连接一致 | 手动传 Connection | 绑定到当前线程，DAO 直接取 |
| 日志链路追踪 | 每行日志手动拼 TraceId | MDC 自动携带 |

**它不解决的问题**：

- 不能替代锁——它保证的是「各线程数据独立」，不是「共享数据安全」
- 不能自动跨线程传递（需要 `InheritableThreadLocal` 或 TTL）
- 不能解决线程池复用带来的上下文残留（必须手动 `remove`）

### 1.3 与 synchronized 的对比

| 维度 | synchronized | ThreadLocal |
| :--- | :--- | :--- |
| 思路 | **共享**同一份数据，用锁串行化 | **隔离**数据，每线程一份 |
| 代价 | 阻塞、上下文切换 | 内存占用（每线程一份） |
| 适用 | 多线程协作修改同一状态 | 上下文传递、非线程安全对象复用 |
| 风险 | 死锁、锁粒度 | 内存泄漏、上下文串号 |

### 1.4 核心术语

| 术语 | 含义 |
| :--- | :--- |
| 线程封闭 | 数据只被一个线程访问，天然无竞争 |
| `ThreadLocalMap` | `Thread` 内部的静态内部类，实际存储容器 |
| `Entry` | Map 的条目，**key 弱引用、value 强引用** |
| 陈旧条目（stale entry） | key 已被 GC 回收（变为 null），但 value 仍被引用 |
| 开放地址法 | 哈希冲突时向后线性探测空槽，而非拉链表 |
| 启发式清理 | `get`/`set`/`remove` 时顺带清理部分陈旧条目 |

---

## 二、基础知识点

### 2.1 核心 API

| 方法 | 作用 | 注意 |
| :--- | :--- | :--- |
| `set(T)` | 为当前线程设置副本 | 首次调用会创建 `ThreadLocalMap` |
| `get()` | 取当前线程副本 | 未 set 过则调用 `initialValue()` |
| `remove()` | 移除当前线程的条目 | **防止泄漏与串号的唯一可靠手段** |
| `initialValue()` | 初始值，默认 null | 重写或用 `withInitial` |
| `withInitial(Supplier)` | Java 8+ 工厂方法 | Lambda 写法更简洁 |

### 2.2 标准用法

```java
// 推荐：private static final，全类共用一把"钥匙"
private static final ThreadLocal<UserContext> USER =
        ThreadLocal.withInitial(UserContext::new);
```

```java
// 铁律：set 与 remove 成对，remove 放 finally
try {
    USER.set(currentUser);
    doBusiness();
} finally {
    USER.remove();
}
```

为什么必须是 `private static final`：

- `static`：避免每个实例创建一把新钥匙，浪费且易泄漏
- `final`：防止被重新赋值导致旧 key 的 value 无法通过新引用清理
- `private`：避免外部代码误改

### 2.3 典型场景

| 场景 | 说明 | 框架落点 |
| :--- | :--- | :--- |
| 请求上下文 | 登录用户、租户 ID、TraceId | `RequestContextHolder`、`SecurityContextHolder` |
| 事务资源绑定 | 同一事务复用同一个 Connection | `TransactionSynchronizationManager` |
| 非线程安全对象 | `SimpleDateFormat`、旧版 `Random` | 各自封装 |
| 日志 MDC | 日志自动携带 TraceId | Logback `MDC` |
| 隐式参数 | 数据源标识、语言环境 | 多数据源路由、国际化 |

```java
// SimpleDateFormat 复用：每线程一份，既安全又不必每次 new
private static final ThreadLocal<SimpleDateFormat> FMT =
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

public String format(Date d) {
    return FMT.get().format(d);
}
```

### 2.4 入门误区

| 误区 | 事实 |
| :--- | :--- |
| ThreadLocal 里存着数据 | 数据存在 `Thread` 的 `threadLocals` 里，ThreadLocal 只是 key |
| 用完不 remove 也没事 | 线程池中线程长期存活，value 无法回收且会串号 |
| 子线程能拿到父线程的值 | 普通 `ThreadLocal` 不能，需 `InheritableThreadLocal` |
| 用了 InheritableThreadLocal 在线程池里也安全 | 线程池复用线程，继承只发生在创建时，值会错乱 |
| key 是强引用所以不会丢 | key 是**弱引用**，会被 GC 回收，导致 key=null 的僵尸条目 |
| remove 只是清空值 | remove 会清除 Entry 并触发 `expungeStaleEntry`，是完整清理 |

---

## 三、技术原理

### 3.1 数据到底存在哪里

```java
public class Thread implements Runnable {
    ThreadLocal.ThreadLocalMap threadLocals = null;
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
}
```

```text
Thread
  └── threadLocals: ThreadLocalMap
        └── table: Entry[]
              ├── Entry[key=ThreadLocal@1a2b(弱), value=UserContext@9f8e(强)]
              └── Entry[key=ThreadLocal@3c4d(弱), value=SimpleDateFormat@7e6f(强)]
```

由此得出三个直接结论：

1. **副本随线程生命周期存亡**——线程结束，Map 一起被回收。
2. **不同线程读写各自的 Map**——天然隔离，无需同步。
3. **ThreadLocal 实例是共享的 key**——所有线程用同一个 key 查各自的 Map。

### 3.2 ThreadLocalMap 的结构

```java
static class ThreadLocalMap {
    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;                 // 强引用
        Entry(ThreadLocal<?> k, Object v) {
            super(k);                 // key 交给 WeakReference
            value = v;
        }
    }

    private Entry[] table;
    private int size;
    private int threshold;            // 默认 2/3 容量
}
```

关键设计：

| 设计 | 内容 | 原因 |
| :--- | :--- | :--- |
| 数组 + 开放地址法 | 冲突时 `nextIndex` 向后探测 | 数据量小、无链表开销、局部性好 |
| 初始容量 16 | 2 的幂 | 可用位运算取模 |
| 负载因子 2/3 | `threshold = len * 2 / 3` | 比 HashMap 的 0.75 低，减少探测长度 |
| key 弱引用 | `WeakReference<ThreadLocal>` | 允许 ThreadLocal 实例被回收 |
| value 强引用 | 普通字段 | 业务对象需要正常可达 |
| 不实现 `Map` 接口 | 独立实现 | 只服务 ThreadLocal 场景，规避通用 Map 的复杂度 |

### 3.3 哈希计算：为什么是 0x61c88647

```java
private static final int HASH_INCREMENT = 0x61c88647;
private static AtomicInteger nextHashCode = new AtomicInteger();

private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

`0x61c88647` 是 **黄金分割数** 相关的常量（`(√5 - 1)/2 × 2^32` 的近似）。用它做等距递增，再与 2 的幂长度做与运算，能让 key 在数组上**分布得非常均匀**，显著减少线性探测的冲突长度。

这是 `ThreadLocal` 里最容易被追问的「冷门细节」。

### 3.4 set 源码

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);       // t.threadLocals
    if (map != null) {
        map.set(this, value);             // this 作为 key
    } else {
        createMap(t, value);              // 首次：创建 Map 并放入
    }
}
```

```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len - 1);

    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {                   // ① 命中：覆盖 value
            e.value = value;
            return;
        }
        if (k == null) {                  // ② 遇到陈旧条目：替换并顺带清理
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);       // ③ 空槽：新建
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)  // ④ 启发式清理，必要时扩容
        rehash();
}
```

三个分支对应三种情况：**命中覆盖、替换僵尸、探测到空槽插入**。

### 3.5 get 源码

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T) e.value;
            return result;
        }
    }
    return setInitialValue();             // 未命中：初始化
}

private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)      // 快速路径：一次命中
        return e;
    return getEntryAfterMiss(key, i, e);  // 慢路径：线性探测 + 清理
}
```

`getEntryAfterMiss` 在探测过程中遇到 `key == null` 会调用 `expungeStaleEntry` 清理——这是「读操作也会顺带清理」的原因。

### 3.6 remove 源码

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null) m.remove(this);
}

private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len - 1);
    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear();                    // 清除弱引用
            expungeStaleEntry(i);         // 清理槽位并重排后续元素
            return;
        }
    }
}
```

`remove` 不是简单置 null，而是**清引用 + 重排探测链**，否则开放地址法的探测链会断裂，导致后续 key 找不到。

### 3.7 清理机制三件套

| 方法 | 触发时机 | 作用范围 |
| :--- | :--- | :--- |
| `expungeStaleEntry(i)` | 发现 key=null 时 | 清理当前槽，并重排后续连续段 |
| `cleanSomeSlots(i, n)` | `set` 插入后 | 启发式扫描约 log₂(n) 个槽 |
| `rehash()` | size ≥ threshold | 全量清理，仍超阈值则扩容 |

**关键结论**：清理是「**顺带**」发生的，前提是**后续还有对该 Map 的操作**。如果一个线程 set 之后再无任何 get/set/remove，僵尸条目永远不会被清理——这正是线程池场景下泄漏的根源。

---

## 四、深入与高级延伸

### 4.1 内存泄漏的完整引用链

先明确「泄漏」的定义：对象已不再被业务使用，但仍存在**可达的强引用链**，GC 无法回收。

**强引用链（value 无法回收）**：

```text
Thread（存活）
  → Thread.threadLocals（ThreadLocalMap）
    → Entry[] table
      → Entry
        → value（强引用）
```

**弱引用（key 会被回收）**：

```text
Entry --弱引用--> ThreadLocal 实例
```

当业务代码把 `ThreadLocal` 实例的强引用置空（例如作为局部变量随方法结束），下一次 GC 会回收该实例，Entry 的 `key` 变为 `null`，形成：

```text
Entry[key=null, value=大对象]   ← 僵尸条目
```

此时 value 仍被存活线程的 Map 强引用，无法回收。

### 4.2 为什么 key 要用弱引用

反证法：如果 key 也是强引用。

```text
Thread → ThreadLocalMap → Entry → key（强）→ ThreadLocal 实例
```

那么只要线程存活，`ThreadLocal` 实例就永远无法回收，**泄漏的是 ThreadLocal 对象本身**（虽然它很小）。用弱引用至少让 key 可以被回收，把「必然泄漏」降级为「可能泄漏，且有清理机制兜底」。

所以弱引用不是「导致泄漏的原因」，而是「减轻泄漏的设计」；真正的问题是 **value 的强引用 + 线程长期存活**。

### 4.3 线程池场景：比泄漏更常见的是串号

线程池复用线程，`ThreadLocalMap` 不会随请求结束而清空：

```text
请求 A：set(userA) → 业务处理 → 未 remove
请求 B（复用同一线程）：get() → 拿到 userA  ← 越权/串号
```

这类问题的危险程度高于内存泄漏：

- 用户信息错乱 → 越权访问
- 租户 ID 串号 → 数据串库
- TraceId 残留 → 日志链路错乱

**防御三件套**：

1. `finally` 中 `remove()`
2. 入口处先 `remove()` 再 `set()`（覆盖残留）
3. 拦截器/过滤器统一清理，避免依赖业务代码自觉

```java
public class UserContextFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        try {
            UserContextHolder.set(parse(req));
            chain.doFilter(req, res);
        } finally {
            UserContextHolder.remove();     // 唯一可靠的清理点
        }
    }
}
```

### 4.4 InheritableThreadLocal 的原理与局限

```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    protected T childValue(T parentValue) { return parentValue; }

    ThreadLocalMap getMap(Thread t) { return t.inheritableThreadLocals; }

    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

复制发生在 `Thread` 构造时：

```java
// Thread.init()
if (parent.inheritableThreadLocals != null) {
    this.inheritableThreadLocals =
        ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
}
```

**局限**：只在线程**创建时**复制一次。线程池中的线程是复用的，创建时继承的是「当时那个父线程」的值，之后不会更新，也不能反映提交任务的线程——所以在线程池里直接用它会拿到错误的值。

### 4.5 TransmittableThreadLocal（TTL）

阿里开源的 TTL 解决的正是「线程池场景跨线程传递」：

```java
private static final TransmittableThreadLocal<UserContext> CTX =
        new TransmittableThreadLocal<>();

// 提交任务时包装
ExecutorService pool = TtlExecutors.getTtlExecutorService(rawPool);
pool.submit(() -> {
    // 这里能拿到提交方线程的 CTX 值
});
```

原理三层：

1. `TransmittableThreadLocal` 继承 `InheritableThreadLocal`，保留父子继承能力
2. `Transmitter` 在任务提交时**捕获**当前线程的所有 TTL 值（快照）
3. 任务执行前**回放**快照，执行后**恢复**原值

```text
submit 时：capture()  → holder = 当前线程 TTL 快照
run 前：  replay(holder) → 把快照写入执行线程
run 后：  restore()      → 还原执行线程原值
```

`TtlExecutors` / `TtlRunnable` 就是对 `Runnable`、`ExecutorService` 做装饰，也可以用 Java Agent 无侵入增强。

三者对比：

| 类型 | 跨线程 | 线程池安全 | 原理 |
| :--- | :--- | :--- | :--- |
| `ThreadLocal` | 否 | — | 仅当前线程 |
| `InheritableThreadLocal` | 是（创建时） | **否** | 线程构造时复制 |
| `TransmittableThreadLocal` | 是（提交时） | **是** | 提交捕获 + 执行回放 |

### 4.6 框架中的实际落点

| 框架 | 用法 | 清理时机 |
| :--- | :--- | :--- |
| Spring `RequestContextHolder` | 绑定 `HttpServletRequest` | `FrameworkServlet` 请求结束后清理 |
| Spring `TransactionSynchronizationManager` | 绑定事务资源（Connection） | 事务提交/回滚后清理 |
| Spring Security `SecurityContextHolder` | 绑定认证信息 | 过滤器链结束时清理 |
| Logback `MDC` | 绑定日志上下文 | 需业务或过滤器手动 `clear()` |
| MyBatis `SqlSession` | 可绑定到线程 | 由 `SqlSessionTemplate` 管理 |

共同规律：**框架都在「请求/事务边界」统一清理**，而不是依赖业务代码。

### 4.7 虚拟线程（JDK 21）下的新问题

虚拟线程数量可达百万级，每个虚拟线程一份 `ThreadLocalMap` 会显著放大内存占用。JDK 21 引入 **`ScopedValue`（预览）** 作为更轻量的替代：

```java
static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

ScopedValue.where(CURRENT_USER, user).run(() -> {
    // 作用域内可读，退出自动失效
});
```

| 维度 | ThreadLocal | ScopedValue |
| :--- | :--- | :--- |
| 生命周期 | 手动 set/remove | 作用域结束自动失效 |
| 可变性 | 可 set 多次 | 作用域内不可变 |
| 跨线程 | 需 TTL | 支持 StructuredTaskScope 继承 |
| 内存 | 每线程一份 Map | 更轻量 |

### 4.8 生产高频坑

| 现象 | 根因 | 处理 |
| :--- | :--- | :--- |
| 用户信息错乱/越权 | 线程池复用 + 未 remove | `finally` 清理 + 入口先清理 |
| 内存缓慢上涨 | 大对象放 ThreadLocal 且未 remove | 只放轻量上下文；用堆 dump 查 Entry |
| 异步任务拿不到上下文 | 普通 ThreadLocal 不跨线程 | TTL 或显式传参 |
| 定时任务拿到上一次的值 | 调度线程复用 | 任务开始前 `remove` |
| MDC 日志串号 | 未 `MDC.clear()` | 过滤器 finally 清理 |
| 子线程改了值父线程看不到 | 副本隔离 | 用可共享容器（如 AtomicReference）而非 ThreadLocal |

---

## 五、总结整理

### 5.1 全景回顾

```text
Thread（仓库）
  └── threadLocals: ThreadLocalMap
        └── Entry[]（开放地址法）
              ├── key   → 弱引用 → ThreadLocal 实例（钥匙）
              └── value → 强引用 → 业务对象（货物）

set：currentThread().threadLocals.set(this, value)
get：currentThread().threadLocals.getEntry(this)
remove：clear key + expungeStaleEntry（重排探测链）
```

泄漏条件（三者同时满足才会泄漏）：

```text
① 线程长期存活（线程池）
② value 是大对象/强引用
③ set 之后没有 remove，也没有后续 get/set 触发清理
```

### 5.2 速查清单

| # | 问题 | 一句话答案 |
| :--- | :--- | :--- |
| 1 | 数据存在哪 | 存在 `Thread.threadLocals`（ThreadLocalMap），ThreadLocal 只是 key |
| 2 | 为什么线程安全 | 每个线程一份副本，无共享写 |
| 3 | 冲突怎么解决 | 开放地址法，线性探测下一个槽 |
| 4 | hash 怎么算 | `nextHashCode` 每次加 `0x61c88647`，与长度-1 做与运算 |
| 5 | 为什么 key 用弱引用 | 允许 ThreadLocal 实例被回收，减轻泄漏 |
| 6 | 泄漏的根本原因 | value 强引用 + 线程长期存活 + 未 remove |
| 7 | 自动清理为什么靠不住 | 清理只在后续操作时顺带触发，无操作则永不执行 |
| 8 | remove 做了什么 | 清 key 引用 + `expungeStaleEntry` 重排探测链 |
| 9 | 为什么用 private static final | 全类共用一把钥匙，防止被重新赋值 |
| 10 | 子线程能继承吗 | 普通不能；`InheritableThreadLocal` 在创建时复制 |
| 11 | 线程池里为什么不能用 InheritableThreadLocal | 线程复用，继承只发生在创建时，值会错乱 |
| 12 | TTL 怎么解决 | 提交时 capture、执行前 replay、执行后 restore |
| 13 | 框架里谁在用 | RequestContextHolder、TransactionSynchronizationManager、MDC |
| 14 | 虚拟线程下有什么问题 | 每线程一份 Map 内存放大，JDK 21 提供 ScopedValue |
| 15 | 最重要的纪律 | 有 set 必有 remove，且放在 finally |

### 5.3 三条追问链

**追问链 1：内存泄漏**

```text
ThreadLocal 为什么会内存泄漏？
  → value 被 Entry 强引用，Entry 被线程的 Map 强引用
key 不是弱引用吗，为什么还会泄漏？
  → 弱引用只保证 key 可回收，回收后变成 key=null 的僵尸条目
  → value 仍然可达
不是有自动清理吗？
  → 清理是顺带触发，没有后续操作就永远不执行
最终怎么解决？
  → finally 中 remove，切断整条强引用链
```

**追问链 2：线程池串号**

```text
为什么请求 A 的用户信息会出现在请求 B？
  → 线程池复用线程，ThreadLocalMap 未清空
  → 请求 B 直接 get 到了 A 的值
怎么防？
  → 过滤器入口 set 之前先 remove
  → 出口 finally remove
  → 异步任务用 TTL，不用裸 ThreadLocal
```

**追问链 3：跨线程传递**

```text
主线程 set 的值，子线程为什么拿不到？
  → 副本存在各自线程的 Map 里，天然隔离
InheritableThreadLocal 为什么在线程池里失效？
  → 复制发生在 Thread 构造时，池中线程创建一次、复用多次
TTL 怎么做到正确传递？
  → 提交时捕获快照，执行前回放，执行后恢复
```

### 5.4 决策速查

| 场景 | 方案 |
| :--- | :--- |
| Web 请求上下文 | ThreadLocal + 过滤器统一清理 |
| 线程池内异步任务传递上下文 | TransmittableThreadLocal |
| 父子线程一次性传递 | InheritableThreadLocal（确认不是池化线程） |
| 非线程安全工具类复用 | `ThreadLocal.withInitial` |
| 虚拟线程高密度场景 | 优先 ScopedValue（JDK 21+）或显式传参 |
| 只是方法间传参 | 优先显式参数，不要滥用 ThreadLocal |

---

> 本文基线：JDK 8 / 17 / 21。
