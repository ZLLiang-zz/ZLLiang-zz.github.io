---
title: "SPI 与 TCCL：父加载器如何加载子加载器的实现"
date: 2026-10-25T09:00:00+08:00
summary: "SPI 常被说成「打破双亲委派」，但它一行 loadClass 都没改。真正的机制是 Thread 上那个可变的 contextClassLoader：规则不动，另换一个加载器去找类。从 ServiceLoader 源码到 JDBC 注册驱动的完整链路，再辨清它与 Tomcat 重写 loadClass 的本质差别。"
categories: ["Java 运行时"]
tags: ["类加载", "SPI", "TCCL", "JDBC"]
author: ZLLiang
ShowToc: true
TocOpen: false
draft: false
---

## 一、基本概念

### 1.1 SPI 是什么

SPI（Service Provider Interface）是一种**运行时发现实现**的约定：接口由框架方定义，实现由第三方提供，双方不产生编译期依赖。

它由三个要素构成：

| 要素 | 载体 |
| :--- | :--- |
| 接口 | 框架 jar 里的 `public interface` |
| 实现 | 第三方 jar 里的实现类，必须是 `public` 且有无参构造 |
| 注册 | 实现方 jar 内 `META-INF/services/<接口全限定名>` 文本文件，一行一个实现类全名 |

JDK 自带的典型使用者是 JDBC（`java.sql.Driver`）、`Charset`、`FileSystemProvider`；生态里 SLF4J、Servlet 容器初始化、各类框架的扩展点也都基于同一套约定。Spring 的 `spring.factories` 是它的自研变体，思路一致但解析方式不同。

### 1.2 双亲委派的方向性

类加载器的层次是固定的四层：

```text
Bootstrap（java.base、rt.jar）
    ↑
Platform（java.sql、java.xml 等平台模块）
    ↑
Application（classpath / 应用类路径）
    ↑
自定义 ClassLoader（插件、WebApp、热部署）
```

箭头只朝上。`loadClass` 的逻辑是：查已加载 → 委派父加载器 → 父加载不到，才自己 `findClass`。**父加载器永远不会回头找子加载器帮忙。**

它带来两个价值，缺一不可：

1. **避免重复加载**——同一个类必须由同一个加载器加载才算"相等"，否则 `instanceof` 为 false、`ClassCastException` 随之而来；
2. **防止核心类被篡改**——你写一个 `java.lang.String`，永远轮不到它被加载。

### 1.3 关键术语

| 术语 | 含义 |
| :--- | :--- |
| TCCL | Thread Context ClassLoader，线程上下文类加载器，挂在 `Thread` 对象上的一个普通字段 |
| SPI 注册文件 | `META-INF/services/接口全限定名`，UTF-8 编码 |
| provider | 注册文件里声明的实现类 |
| 委派链 | 加载器之间"向上问"的关系，与 TCCL 无关 |

## 二、基础知识点

### 2.1 最小可用的 SPI

三步就能跑起来。

**第一步，接口（框架侧）**

```java
package com.example.api;

public interface Plugin {
    String name();
    void execute();
}
```

**第二步，实现（提供方）**

```java
package com.example.impl;

public class EchoPlugin implements Plugin {
    public String name() { return "echo"; }
    public void execute() { System.out.println("echo"); }
}
```

**第三步，注册文件**

```text
src/main/resources/META-INF/services/com.example.api.Plugin
内容：com.example.impl.EchoPlugin
```

消费侧只需要一行：

```java
ServiceLoader<Plugin> loader = ServiceLoader.load(Plugin.class);
for (Plugin p : loader) {
    p.execute();
}
```

### 2.2 使用时的五个事实

| 事实 | 含义 |
| :--- | :--- |
| **懒加载** | `load()` 只做构造，真正 `Class.forName` 发生在迭代 `next()` 时 |
| **每次都是新实例** | 每次 `load()` 重新实例化一遍，跨 `ServiceLoader` 实例不共享；想复用要自己缓存 |
| **非线程安全** | 同一个 `ServiceLoader` 实例不要多线程并发迭代 |
| **失败即中断** | 实现类没有 public 无参构造、类名写错 → `ServiceConfigurationError`，且它是 `Error` 不是 `Exception` |
| **顺序不保证** | 遍历顺序取决于 classpath 顺序与文件内声明顺序，不能当作优先级机制 |

### 2.3 TCCL 的读写

```java
ClassLoader tccl = Thread.currentThread().getContextClassLoader();
Thread.currentThread().setContextClassLoader(myLoader);
```

两条容易忽略的性质：

- 默认值是**系统类加载器**（Application），主线程在 JVM 启动时被设置；
- **子线程只在创建那一刻继承一次**，父线程之后再 `set`，已有子线程不受影响。

## 三、技术原理

### 3.1 矛盾：接口在父侧，实现在子侧

JDBC 的分布恰好踩在这个死角上：

| 角色 | 位置 | 由谁加载 |
| :--- | :--- | :--- |
| `java.sql.Driver`、`DriverManager` | JDK 8：`rt.jar`；JDK 9+：`java.sql` 模块 | JDK 8：Bootstrap；**JDK 9+：Platform** |
| `com.mysql.cj.jdbc.Driver` | 应用 classpath | Application |

`DriverManager` 在静态初始化时要拿到驱动的**实例**，但它自己的加载器看不见 classpath。这就是全部矛盾的来源：**框架代码在父侧，扩展实现在子侧，而委派只有单向。**

> 顺带纠一个流传很广的说法：JDK 9 之后 `java.sql` 并不由 Bootstrap 加载，而是定义给 Platform ClassLoader（`java.base` 才是 Bootstrap）。可以用 `DriverManager.class.getClassLoader()` 自行验证。结论不变——Platform 同样看不见 classpath——但版本差异要分清。

### 3.2 TCCL：Thread 上的一个可变字段

```java
// java.lang.Thread（简化）
private ClassLoader contextClassLoader;

Thread(ThreadGroup g, Runnable target, String name, long stackSize, AccessControlContext acc) {
    // ...
    this.contextClassLoader = parent.getContextClassLoader();   // 创建时继承一次
}
```

它**不是** `InheritableThreadLocal`，就是一个普通字段，在 `Thread` 构造时从父线程取一次值。这个细节决定了后面两个生产问题：线程池复用会让 TCCL 长期滞留，虚拟线程池化也会串号。

### 3.3 ServiceLoader 源码：TCCL 出现在哪一行

```java
// java.util.ServiceLoader（JDK 8 / 17 一致）
public static <S> ServiceLoader<S> load(Class<S> service) {
    ClassLoader cl = Thread.currentThread().getContextClassLoader();   // ← 取 TCCL
    return new ServiceLoader<>(service, cl);
}

public static <S> ServiceLoader<S> load(Class<S> service, ClassLoader loader) {
    return new ServiceLoader<>(service, loader);
}
```

真正加载实现类时，用的是构造时存下的 `loader`：

```java
private S nextService() {
    String cn = nextName;                              // 来自 META-INF/services 的一行
    nextName = null;
    Class<?> c = Class.forName(cn, false, loader);      // false：不触发 <clinit>
    if (!service.isAssignableFrom(c)) {
        throw new ServiceConfigurationError(service.getName() + " provider " + cn + " not a subtype");
    }
    S p = service.cast(c.getConstructor().newInstance());  // 这里才初始化
    // ...
}
```

`false` 这个参数值得注意：加载阶段不初始化，把 `<clinit>` 推到 `newInstance()` 那一刻。所以"注册文件里写了类名"不等于"类已经初始化"。

一个反证能说明 TCCL 的必要性：把这里换成 `service.getClassLoader()`，对 `java.sql.Driver` 拿到的是 `null`（Bootstrap）或 Platform loader，**永远读不到 classpath 下的实现**。

### 3.4 JDBC 的完整链路

```text
① DriverManager 的 <clinit> 执行 loadInitialDrivers()
        │
        ▼
② ServiceLoader.load(Driver.class) → TCCL = Application
        │  读 META-INF/services/java.sql.Driver
        │  Class.forName("com.mysql.cj.jdbc.Driver", false, AppLoader)
        │  反射 newInstance → 触发驱动静态块
        ▼
③ 驱动静态块 DriverManager.registerDriver(new Driver())
        ▼
   registeredDrivers 中有实例，getConnection() 时逐个 tryConnect
```

`DriverManager` 侧：

```java
static {
    loadInitialDrivers();
    println("JDBC DriverManager initialized");
}

private static void loadInitialDrivers() {
    String drivers = AccessController.doPrivileged(new GetPropertyAction("jdbc.drivers"));
    // 先处理 -Djdbc.drivers 显式指定的类 ...
    AccessController.doPrivileged((PrivilegedAction<Void>) () -> {
        ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);   // TCCL
        Iterator<Driver> it = loadedDrivers.iterator();
        try {
            while (it.hasNext()) it.next();     // 逐个实例化
        } catch (Throwable t) {
            // 单个驱动失败不影响其他驱动
        }
        return null;
    });
}
```

驱动侧：

```java
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    static {
        try {
            DriverManager.registerDriver(new Driver());
        } catch (SQLException e) {
            throw new RuntimeException("Can't register driver!");
        }
    }
}
```

这里跨了加载器，但没有跨"类"：`DriverManager` 是父侧加载的类，它的 public static 方法对子侧代码完全可见，调用不涉及额外加载。真正需要 TCCL 的只有"把实现类加载进来"这一步。

> JDBC 4.0 之后 `Class.forName("com.mysql.cj.jdbc.Driver")` 已非必需，SPI 会自动完成注册；这行老代码仍然有效，因为它触发静态块，走的还是同一个 `registerDriver`。

### 3.5 绕过，不是打破

这是最值得说清的一点。

| 维度 | 双亲委派所约束的 | TCCL 所改变的 |
| :--- | :--- | :--- |
| 对象 | **一个加载器怎么找类**（先父后子） | **用哪个加载器去找** |
| 机制 | `loadClass` 的模板方法 | 把一个 loader 引用传给 `Class.forName` |
| 是否被修改 | 没有，一行未改 | —— |

关键事实是：`Class.forName(name, false, loader)` 本来就接受任意 loader，委派链并不限制"你不能用别的加载器"。所以 SPI 没有动规则，它只是**换了一个执行者**。

规则管的是"怎么走"，TCCL 管的是"谁去走"。这是我理解的准确表述：**机制不变，通道另开。**

## 四、深入与高级延伸

### 4.1 JDK 9 模块化带来的变化

| 变化 | 影响 |
| :--- | :--- |
| `module-info.java` 支持 `provides Interface with Impl` | 模块路径下不再需要 `META-INF/services` |
| 消费方需 `uses Interface` 声明 | 缺少 `uses` 时 `ServiceLoader` 找不到任何 provider |
| 新增 `ServiceLoader.load(ModuleLayer, Class)` | 可按模块层隔离地查找 |
| `loadInstalled` 废弃 | 它只查已安装的扩展加载器，语义已过时 |
| classpath 模式仍然有效 | 非模块化 jar 继续读 `META-INF/services`，两种模式可共存 |

实践中最容易踩的坑：项目一半模块化、一半在 classpath，`uses` 漏写导致"本地能跑、容器里找不到实现"。

### 4.2 与 Tomcat 的重写 loadClass 对比

| 维度 | SPI（TCCL） | Tomcat `WebappClassLoader` |
| :--- | :--- | :--- |
| 手段 | 不重写 `loadClass`，显式传入子加载器 | **重写 `loadClass`**，先本地 `findClass`，失败才 `super.loadClass` |
| 委派顺序 | 不变 | 反转（子优先） |
| 目的 | 父侧框架加载子侧实现 | 多个 WebApp 依赖隔离、各自用不同版本 |
| 对委派链 | 绕过（另开通道） | 打破（重排顺序） |

Tomcat 的核心逻辑（简化）：

```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        Class<?> clazz = findLoadedClass(name);
        if (clazz == null) {
            clazz = findClass(name);                 // ① 先找 WEB-INF/classes、WEB-INF/lib
            if (clazz == null) {
                clazz = super.loadClass(name, resolve);   // ② 找不到才委派父
            }
        }
        return clazz;
    }
}
```

它必须重写，因为要解决的是"同一个库的两个版本"，而 SPI 解决的是"一个实现类在哪一层可见"。问题不同，手段自然不同——把两者都说成"打破双亲委派"，是把手段和目的混在一起了。

### 4.3 四个生产坑

**坑一：中间件覆盖 TCCL。** 一些 RPC 框架、任务调度、容器会把 TCCL 换成自己的加载器。此时回调里调 `ServiceLoader.load(...)` 会找不到实现。定位方式很直接：打印 `Thread.currentThread().getContextClassLoader()`。修法是指定加载器：

```java
ServiceLoader.load(Plugin.class, MyFramework.class.getClassLoader());
```

**坑二：线程池导致 TCCL 滞留。** 线程创建时继承 TCCL，之后不变。WebApp 卸载后，池中的线程仍持有该应用的加载器，整个应用的类无法卸载，元空间持续上涨。容器日志里那句 "appears to have started a thread but has failed to stop it" 多半就是这类问题。

**坑三：子线程不继承后续修改。** 父线程 `setContextClassLoader` 只影响自己，已经存在的线程（含池中线程）不跟随。异步任务里 TCCL 不符合预期，先查这条。

**坑四：`ServiceConfigurationError` 是 `Error`。** 写 `catch (Exception e)` 捕不到。实现类非 public、缺无参构造、类名拼错、`uses` 缺失，最终都会走到这里，且会让整个迭代中断。

### 4.4 排查路径

```text
现象：SPI 找不到实现 / 加载到错误版本
  ① 打印 TCCL，确认是不是预期加载器
  ② 确认注册文件在不在：jar 内 META-INF/services/<接口全限定名>
  ③ 确认实现类可见性：public + 无参构造 + 与接口同一侧可见
  ④ 看类到底由谁加载：-verbose:class 或 jcmd <pid> VM.classloader_stats
  ⑤ 类加载器数量持续增长 → 看是否有 TCCL 滞留导致的卸载失败
```

## 五、总结整理

### 5.1 全景回顾

```text
接口在父侧（Bootstrap / Platform）
        │  父加载器看不见 classpath
        ▼
Thread.contextClassLoader（默认指向 Application）
        │  ServiceLoader.load() 取它
        ▼
Class.forName(impl, false, TCCL) → newInstance
        │
        ▼
实现类被加载，双亲委派的规则一行未改
```

### 5.2 速查清单

| 现象 | 原因 | 动作 |
| :--- | :--- | :--- |
| `ServiceConfigurationError` | 类名错、非 public、无无参构造、`uses` 缺失 | 查注册文件与实现类签名 |
| 本地有、容器里没有 | TCCL 被容器换掉 | 显式 `ServiceLoader.load(Service, loader)` |
| 加载到旧版本实现 | 父加载器已加载同名类 | 看类由谁加载，必要时隔离加载器 |
| 元空间持续上涨 | 池化线程滞留 WebApp 的 TCCL | 卸载时还原 TCCL，或改用独立线程 |
| 子线程找不到实现 | TCCL 只在创建时继承 | 创建子线程前设置，或显式传 loader |

### 5.3 三句话结论

1. SPI 没有打破双亲委派，它绕过了"只能向上委派"的单向限制。
2. 手段是 `Thread` 上那个可变的 `contextClassLoader`，`ServiceLoader` 默认取它当加载器。
3. 真正重写 `loadClass`、重排委派顺序的是 Tomcat，两者的动机和机制都不同。

判断一句话是否准确，可以问自己：**它改的是"怎么找"，还是"谁来找"？** 前者是打破，后者只是绕过。
