---
title: "Spring 框架核心原理全量深度解析：Bean 生命周期、扩展点、AOP 代理与循环依赖"
date: 2026-11-11T21:40:00+08:00
summary: "一篇把 Spring 内核讲透的长文：IoC/DI 与 BeanFactory、ApplicationContext 的内核与门面关系，Bean 生命周期主线（doCreateBean/initializeBean 完整源码与回调顺序），Aware、BeanPostProcessor、BeanFactoryPostProcessor 四类扩展点的精确边界，注解在 refresh() 第 5 步被解析的真相，装饰器/代理/观察者三类设计模式的落地，循环依赖为什么要三级缓存（getSingleton/getEarlyBeanReference 源码 + 为什么两级不行），AOP 代理体系全景（JDK vs CGLIB 拓扑、重复代理、@RefreshScope 双层代理、TargetSource 五种），以及 @Transactional 八大失效场景。每个结论带成立条件和代价。"
categories: ["Spring"]
tags: ["Spring", "IoC", "AOP", "Bean生命周期", "BeanPostProcessor", "循环依赖", "三级缓存", "事务", "JDK动态代理", "CGLIB", "事件机制"]
author: ZLLiang
ShowToc: true
TocOpen: false
draft: false
---


写 Spring 的人每天都在写 `@Autowired`、`@PostConstruct`、`@Transactional`，却很少有人能说清：这些注解为什么"自己就会生效"？`@PostConstruct` 是谁、在什么时候调的？AOP 的代理对象是在哪一步替换掉原始 Bean 的？`@Transactional` 为什么会"莫名失效"？答案都指向同一条主线——**Bean 生命周期**。`BeanPostProcessor`、Aware 回调、AOP 代理、事件监听，本质都是插在这条主线上不同位置的钩子。

这篇把主线从头到尾读透，再往里钉三个最硬的钉子：**循环依赖为什么要三级缓存**、**AOP 代理体系怎么分层**、**事务为什么失效**。读完你能画出一条完整的 Bean 生命周期时序，说清"为什么我的注解不生效"的排查路径，以及 `BeanFactoryPostProcessor` 和 `BeanPostProcessor` 的本质区别——这是区分"会用 Spring"与"懂 Spring"的分水岭。

---

## 一、基本概念：Spring 的魔法都挂在同一条线上

### 1.1 IoC 与 DI：控制权为什么值得交出去

**IoC（Inversion of Control，控制反转）** 是一种设计思想：把"手动 `new` 对象"的控制权交给容器。**DI（Dependency Injection，依赖注入）** 是 IoC 的一种实现方式——容器在创建对象时，主动把它依赖的对象"塞"进来。

为什么值得？想象一个 `Service` 依赖几百上千个底层类，手动装配意味着你要搞清楚每一个的构造顺序；交给容器后，你只声明"我需要一个 `OrderMapper`"，装配的苦差事由容器承担。**代价**：对象创建不再由你直接掌控，出问题时排查链路变长——这就是"Spring 启动慢、报错难定位"的根源之一，也是 AOT（提前编译）想解决的痛点。

两个高频细节：

- **`@Component` vs `@Bean`**：`@Component` 作用于**类**（靠 `@ComponentScan` 扫描），`@Bean` 作用于**方法**（在 `@Configuration` 类里显式声明）。引入第三方库的类、需要按条件返回不同实现时，只能用 `@Bean`——它比 `@Component` 灵活，但会把"装配逻辑"写进代码。
- **Bean 作用域**：`singleton`（默认，容器唯一实例）、`prototype`（每次获取新建）、`request`/`session`（Web 环境）。单例 Bean 里若持有**可变成员变量**，多线程写它就有线程安全问题——官方推荐用 `ThreadLocal` 隔离可变状态，而不是把单例改成 prototype。

### 1.2 BeanFactory 与 ApplicationContext：内核与门面

这是 Spring 最基础、也最容易答浅的一问。标准答法不是"ApplicationContext 比 BeanFactory 多了几个功能"，而是**两者是内核与门面的组合关系**。

| 层次 | 代表类型 | 定位 |
| :--- | :--- | :--- |
| 基础 IoC 容器 | `BeanFactory` | 最底层契约：BeanDefinition 注册、Bean 创建、依赖注入、生命周期 |
| 高级 IoC 容器 | `ApplicationContext` | 继承 `ListableBeanFactory`、`HierarchicalBeanFactory`，**本身就是一个 BeanFactory** |

关键在于：`ApplicationContext` 不自己实现 Bean 管理，而是**组合**一个真正的 `BeanFactory`（通常是 `DefaultListableBeanFactory`）来干活：

```text
AnnotationConfigApplicationContext
  ├── beanFactory: DefaultListableBeanFactory   ← 真正干活的引擎
  │     ├── beanDefinitionMap                    ← 定义（图纸）
  │     └── singletonObjects                     ← 一级缓存（单例池）
  ├── environment / messageSource / resourceLoader
  └── applicationEventMulticaster
```

`getBean()` 最终转发给这个内部工厂。**相比裸 BeanFactory，ApplicationContext 多的是企业级能力**：自动注册后处理器（BPP/BFPP）、事件机制、国际化、资源加载、环境抽象，以及**启动时预实例化所有非懒加载单例**（让配置错误尽早暴露，而裸 BeanFactory 是懒加载）。

一句话记忆：**ApplicationContext 是门面，BeanFactory 是引擎**。AOP 代理、`@RefreshScope` 这些机制能生效，前提正是 ApplicationContext 在启动时自动注册了对应的 `BeanPostProcessor`。

### 1.3 术语对齐与核心结构

| 术语 | 含义 |
| :--- | :--- |
| **BeanDefinition** | Bean 的"图纸/元数据"：类名、作用域、构造参数、属性值。Bean 实例按它生产 |
| **容器扩展点** | 允许外部代码介入容器机制的接口总称：`BeanDefinitionRegistryPostProcessor`、`BeanFactoryPostProcessor`、`BeanPostProcessor`、`Aware` |
| **BeanFactoryPostProcessor（BFPP）** | **元数据级**钩子，在所有 Bean 实例化之前修改"图纸" |
| **BeanPostProcessor（BPP）** | **实例级**钩子，在每个 Bean 初始化前后处理实例 |

**主线只有一条**：`refresh()` → 解析配置类并注册 BeanDefinition → 实例化 Bean → 注入属性 → 初始化（含 BPP 前后钩子）→ 放入单例池 → 发布容器事件。本文讲的所有机制，都能在这条线上找到确切位置。

---

## 二、基础知识点：你会用的那些"钩子"

### 2.1 Aware 接口：让 Bean 反向感知容器

**是什么**：一组**标记接口**（只有 `setXxx` 方法），Bean 实现后，容器在初始化过程中**回调 setter**，把容器内部资源注入进来。九个核心 Aware 必背：

| 接口 | 拿到什么 | 典型用途 |
| :--- | :--- | :--- |
| `BeanNameAware` | Bean 名称 | 日志、动态注册 |
| `BeanFactoryAware` | 创建自己的 BeanFactory | 手动 getBean、判断作用域 |
| `ApplicationContextAware` | 完整容器引用（**最常用**） | 拿容器全部能力 |
| `EnvironmentAware` | 配置属性、Profile | 读环境变量 |
| `ApplicationEventPublisherAware` | 事件发布器 | 发事件 |
| `ResourceLoaderAware` | 资源加载器 | 读 classpath 文件 |
| `MessageSourceAware` | 国际化消息源 | 多语言 |
| `EmbeddedValueResolverAware` | 占位符解析器 | 手动解析 `${...}` |
| `BeanClassLoaderAware` | 类加载器 | 动态加载类、SPI |

**边界（官方立场）**：这些接口会把代码**耦合到 Spring API、违反 IoC 风格**，官方建议只用于"需要编程式访问容器"的基础设施 Bean。业务代码要拿容器，优先 `@Autowired` 注入 `ApplicationContext`；要发事件，优先注入 `ApplicationEventPublisher`（接口更小、更好 mock）。**代价**：过度使用 Aware 会让单测必须启动容器，显著降低可测试性。

### 2.2 BeanPostProcessor 与 BeanFactoryPostProcessor

**BeanPostProcessor** 是 Spring 的**工厂钩子**，在每个 Bean 的初始化前后介入：

| 方法 | 执行时机 | 典型用途 |
| :--- | :--- | :--- |
| `postProcessBeforeInitialization` | `@PostConstruct` / `afterPropertiesSet` / `init-method` **之前** | 检查标记接口、预处理 |
| `postProcessAfterInitialization` | 初始化回调**之后** | **创建 AOP 代理**、包装装饰器 |

两个方法都返回 `(Object bean, String beanName)`，**返回值被容器当作最终 Bean**——返回 `null` 等于放弃该 Bean。

内置核心 BPP（背下来能解释一切"魔法"）：注入靠 `AutowiredAnnotationBeanPostProcessor`，生命周期注解靠 `CommonAnnotationBeanPostProcessor`，Aware 靠 `ApplicationContextAwareProcessor`，AOP 靠 `AnnotationAwareAspectJAutoProxyCreator`。

**与 BFPP 的区别（高频混淆）**：

| 维度 | BeanFactoryPostProcessor | BeanPostProcessor |
| :--- | :--- | :--- |
| 作用对象 | **BeanDefinition（图纸）** | **Bean 实例（成品）** |
| 执行时点 | 所有 Bean 实例化**之前** | 每个 Bean 初始化**过程中** |
| 能否改 Bean 本身 | 不能（还没实例） | 能（可返回代理/装饰器） |
| 典型实现 | `ConfigurationClassPostProcessor`、`PropertySourcesPlaceholderConfigurer` | `AutowiredAnnotationBeanPostProcessor`、`AnnotationAwareAspectJAutoProxyCreator` |

### 2.3 设计模式视角：Spring 里最容易搞混的三种

**装饰器模式 vs 代理模式**——结构相似（都持目标引用），本质不同：

| 维度 | 装饰器模式 | 代理模式 |
| :--- | :--- | :--- |
| 目的 | **增强功能**（叠加职责） | **控制访问**（权限、延迟、远程） |
| 谁组合 | 调用方主动组合 | 框架自动生成 |
| 典型例子 | `TransactionAwareCacheDecorator`、`ContentCachingRequestWrapper` | `@Transactional` 事务代理 |

> ⚠️ 面试问"Spring AOP 是装饰器模式吗"，答案是**不是**——Spring AOP 用的是**代理模式**。

**观察者模式（发布-订阅）**：Spring 事件机制靠 `ApplicationEventMulticaster` 解耦发布者与监听者。发布者（`ApplicationEventPublisher`）、监听者（`@EventListener`）、广播器（`SimpleApplicationEventMulticaster`）三方互不感知。

除这三个，Spring 还用到了：**工厂模式**（`BeanFactory` 创建 Bean）、**单例模式**（Bean 默认单例）、**适配器模式**（AOP 的 `AdvisorAdapter`、SpringMVC 的 `HandlerAdapter`）、**模板方法模式**（`JdbcTemplate`、`RestTemplate`、`AbstractApplicationContext#refresh`）。

---

## 三、技术原理：Bean 生命周期主线与扩展点执行

### 3.1 Bean 生命周期主线（一切机制的地基）

`AbstractAutowireCapableBeanFactory#doCreateBean` 是 Bean 诞生的主流程：

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, Object[] args) {
    // ① 实例化：反射调用构造器/工厂方法（此刻属性全是默认值）
    BeanWrapper instanceWrapper = createBeanInstance(beanName, mbd, args);

    // ② 提前暴露（解决循环依赖的三级缓存，见 4.1）
    //    addSingletonFactory(...)

    // ③ 属性填充：依赖注入（@Autowired / setter / 构造参数）
    populateBean(beanName, mbd, instanceWrapper);

    // ④ 初始化：Aware 回调 + BPP 前后钩子 + 初始化方法
    exposedObject = initializeBean(beanName, exposedObject, mbd);

    return exposedObject;
}
```

`initializeBean` 的内部顺序（**笔试/面试最高频考点**）：

```java
protected Object initializeBean(String beanName, Object bean, RootBeanDefinition mbd) {
    // ① 第一批 Aware：BeanName / BeanClassLoader / BeanFactory（容器直接调用）
    invokeAwareMethods(beanName, bean);

    // ② 所有 BPP 的「初始化前」回调
    //    第二批 Aware（ApplicationContext / Environment ...）由 ApplicationContextAwareProcessor 在这里执行
    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);

    // ③ 初始化方法：@PostConstruct → afterPropertiesSet() → init-method
    invokeInitMethods(beanName, wrappedBean, mbd);

    // ④ 所有 BPP 的「初始化后」回调：AOP 代理在这里被创建
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);

    return wrappedBean;
}
```

**由此得到三个必须记牢的结论**：

1. **所有 Aware 回调都发生在"属性填充之后、初始化方法之前"**，分两批：第一批 `invokeAwareMethods` 直调（BeanName/BeanClassLoader/BeanFactory），第二批由 `ApplicationContextAwareProcessor`（一个 BPP）在 BPP 链里执行。
2. **初始化方法三级顺序固定**：`@PostConstruct` → `InitializingBean#afterPropertiesSet()` → 自定义 `init-method`（销毁顺序同理：`@PreDestroy` → `destroy()` → `destroy-method`）。
3. **AOP 代理产生于"初始化后（④）"**，所以 `@Transactional`、`@Async` 才拦得住——这也直接解释了"为什么同类内部调用不走事务"（内部调用不经过代理）。

### 3.2 BeanPostProcessor 的执行机制

**注册时机**：`AbstractApplicationContext#refresh()` 第 6 步 `registerBeanPostProcessors(beanFactory)`，**早于普通单例 Bean 的实例化**（第 11 步 `finishBeanFactoryInitialization`）。

**排序规则**：容器自动检测时按 `PriorityOrdered` → `Ordered` → 无接口分组，组内按 `getOrder()` 升序；编程式 `addBeanPostProcessor` 则严格按注册顺序。

> ⚠️ **高频坑：`@Order` 注解对 BPP 不生效**。因为 BPP 排序发生在"注解读取排序器"就绪之前，Spring 只认 `PriorityOrdered` / `Ordered` **接口**。要控制顺序，请实现这两个接口。

**为什么 BPP 不能被 BPP 处理？** `registerBeanPostProcessors` 会先把所有 BPP 实例化出来，此时容器处于"BPP 尚未就绪"状态。若某个 BPP 依赖了业务 Bean，日志会打出 `Bean 'xxx' is not eligible for getting processed by all BeanPostProcessors`——意味着该 Bean **跳过了部分 BPP**（AOP 代理可能没创建、`@Autowired` 可能没注入）。**结论：BPP 内部只允许依赖其他 BPP 或基础设施类，不要注入业务 Bean。**

### 3.3 注解到底在哪一步被解析

很多人以为"注解在创建 Bean 时解析"，其实**绝大部分注解在 Bean 实例化之前就解析完了**，发生在 `refresh()` 第 5 步：

| 阶段 | 时机 | 核心动作 |
| :--- | :--- | :--- |
| ① 注册处理器 | 容器构造时 | `AnnotationConfigUtils.registerAnnotationConfigProcessors()` 注册 `ConfigurationClassPostProcessor` 等 |
| ② 解析注解 | `refresh()` 第 5 步 `invokeBeanFactoryPostProcessors` | `ConfigurationClassPostProcessor`（`BeanDefinitionRegistryPostProcessor` + `PriorityOrdered`）**最早执行** |
| ③ 生成 BeanDefinition | 同上 | `ConfigurationClassParser` 处理 `@ComponentScan`、`@Import`、`@Bean` |
| ④ `@Configuration` 增强 | 紧随其后 | CGLIB 生成子类，保证 `@Bean` 方法互调返回**同一单例** |
| ⑤ Bean 实例化 | `refresh()` 第 11 步 | 此刻才有对象，BPP 与 Aware 开始工作 |

为什么这个顺序重要：`@Bean` 方法互调能拿到单例靠 ④ 的 CGLIB 增强（若 `@Configuration(proxyBeanMethods = false)` 则不做增强，方法互调会**各自 new 一个**）；`@Conditional` 能决定"要不要这个 Bean"必须在实例化前（③）。

### 3.4 Bean 的实例化顺序：继承 ≠ 依赖

**Java 层面**：`new ChildBean()` 时父类构造器先执行——但这是**同一个对象**的初始化，父类并未成为独立的 Spring Bean。

**Spring 层面**：Bean 创建顺序取决于**依赖关系**，与类继承**无关**：

| 情况 | 结果 |
| :--- | :--- |
| 父类也是 Bean，子类 `@Autowired` 了父类 | 父 Bean 先创建（因为是依赖） |
| 父类也是 Bean，子类不依赖它 | **无保证**，取决于扫描/注册顺序 |
| 父类不是 Bean | 只创建子 Bean；父类构造器被调用，但父类不成为 Bean |

**结论（可直接背）**：**类继承既不产生 Bean 依赖，也不保证创建顺序。** 要让父 Bean 先创建，必须显式建立依赖——构造器/字段注入父 Bean，或 `@DependsOn("parentBean")`。

### 3.5 父类 Bean 的属性注入会覆盖父类实例吗？

**根因**：Java 字段是**实例级**的。`ParentBean` 与 `ChildBean` 是两个独立对象，各自字段内存互不相干。

| 场景 | 是否覆盖父类 Bean 的值 |
| :--- | :--- |
| 父子都是独立 Bean，子类注入自己的字段（最常见） | **不会** |
| 子类**显式注入父 Bean 并调用其 setter** | **会**（你主动改了共享的那个父实例） |
| 子类隐藏（shadow）父类同名字段 | 两个字段并存，不会自动影响父 Bean 实例 |

```java
// 反例：主动修改共享的父 Bean 单例 —— 会污染所有使用方
@Component
public class ChildBean extends ParentBean {
    @Autowired private ParentBean parentBean;

    @PostConstruct
    public void init() {
        parentBean.setTimeout(9999);   // ★ 改的是容器里的父 Bean，全局生效
    }
}
```

**一句话**：继承不会让两个 Bean 共享字段；除非你显式拿到父 Bean 的引用去改它。

### 3.6 事件驱动：从 publishEvent 到监听器执行的完整链路

```text
publisher.publishEvent(event)
  → AbstractApplicationContext#publishEvent(event, eventType)
      → ApplicationEventMulticaster.multicastEvent(event, type)
          → 解析事件类型（泛型 ResolvableType）
          → getApplicationListeners(event, type) 找到所有匹配监听器
              ├─ 监听器缓存（按事件类型 + sourceType 缓存，避免每次全量匹配）
              └─ 缓存未命中 → 遍历所有监听器，用 GenericTypeResolver 判断是否匹配
          → for (listener) invokeListener(listener, event)
              ├─ 同步：直接在当前线程调用（默认）
              └─ 异步：若设置了 taskExecutor，则提交线程池
```

四个关键机制：**监听器缓存**（避免每次全量遍历）、**泛型匹配**（`ResolvableType` 精确投递）、**注解包装**（`EventListenerMethodProcessor` 把 `@EventListener` 方法包装成 `ApplicationListenerMethodAdapter`）、**事务阶段绑定**（`@TransactionalEventListener` 通过 `TransactionSynchronizationManager` 在指定阶段触发）。

> **面试追问**："监听器抛异常会怎样？" → 默认同步广播，**异常会向上抛给发布者**，导致发布方业务事务回滚——这是"事件处理失败拖垮主流程"的经典事故。要隔离，用 `@Async`（并配置独立线程池 + 异常处理器）。

---

## 四、深入与高级延伸：三个最难啃的硬骨头

### 4.1 循环依赖与三级缓存：为什么必须是三级

Spring 用**三级缓存**解决单例循环依赖：

```java
// DefaultSingletonBeanRegistry
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);   // 一级：成品
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16); // 二级：早期引用
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);    // 三级：ObjectFactory 工厂
```

| 缓存 | 存储内容 | 作用 |
| :--- | :--- | :--- |
| 一级 `singletonObjects` | 完整成品 Bean | 日常 `getBean()` 直接取 |
| 二级 `earlySingletonObjects` | 早期引用（原始对象或提前生成的代理） | 避免重复调用工厂、重复生成代理 |
| 三级 `singletonFactories` | `ObjectFactory` 工厂 | **延迟决定是否提前生成代理**——这是它存在的唯一理由 |

**查找顺序（一级 → 二级 → 三级）**：

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);        // 1. 一级
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        singletonObject = this.earlySingletonObjects.get(beanName);       // 2. 二级
        if (singletonObject == null && allowEarlyReference) {
            synchronized (this.singletonObjects) {
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName); // 3. 三级
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();      // 调用工厂生成早期引用
                    this.earlySingletonObjects.put(beanName, singletonObject); // 升级到二级
                    this.singletonFactories.remove(beanName);            // 移除三级
                }
            }
        }
    }
    return singletonObject;
}
```

**为什么必须是三级，两级不行？** 这是核心论点。假设只有两级（一级成品 + 二级早期引用），当 A 需要 AOP 代理时：实例化 A 后若把 `A_raw` 直接放二级，B 从二级拿到的是**没有事务、没有切面逻辑的裸对象**，而 A 初始化后生成的 `A_proxy` 才进一级——**B 拿到的是裸对象，容器持有的是代理，两者不一致**。

要修这个问题只有两条路：① 实例化后立刻生成代理放二级——但这样**所有 Bean 都会提前代理**，打乱 BPP 时序、让不需要代理的 Bean 白付代理开销；② **二级缓存放一个"能按需生成代理"的东西，而不是对象**——这就是三级缓存存的 `ObjectFactory` 工厂。

**三级缓存的本质价值：把"是否提前生成代理"的决定，延迟到"确实发生循环依赖、且有人来取"的那一刻。** 无循环依赖 → 工厂不被调用 → 代理在 `postProcessAfterInitialization` 正常生成；有循环依赖且需要代理 → 工厂被调用 → 代理提前生成。

**代理只生成一次的双重检查**：

```java
// AbstractAutoProxyCreator
public Object getEarlyBeanReference(Object bean, String beanName) {
    this.earlyProxyReferences.put(getCacheKey(bean.getClass(), beanName), bean); // 记录
    return wrapIfNecessary(bean, beanName, cacheKey);                            // 提前代理
}

public Object postProcessAfterInitialization(Object bean, String beanName) {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    if (this.earlyProxyReferences.remove(cacheKey) != bean) {   // 早期没代理过才代理
        return wrapIfNecessary(bean, beanName, cacheKey);
    }
    return bean;                                                // 已代理过，直接返回
}
```

**三级缓存不能解决的循环依赖（局限）**：

1. **构造器注入的循环依赖**：对象尚未实例化，无法提前暴露引用，直接抛 `BeanCurrentlyInCreationException`；
2. **prototype 作用域的循环依赖**：Spring 不缓存 prototype Bean，无法提前暴露；
3. **`@Async` 代理的循环依赖**：`@Async` 代理由 `AsyncAnnotationBeanPostProcessor` 在 `postProcessAfterInitialization` 生成，不走 `getEarlyBeanReference`，循环依赖时可能注入原始对象，需用 `@Lazy` 规避。

一句话：**三级缓存不是为了多存一层，而是为了让代理的生成时机可以按需延迟。**

### 4.2 AOP 代理体系：JDK vs CGLIB 的拓扑差异

**两条代理路径**：

| 技术 | 原理 | 叠加结构 |
| :--- | :--- | :--- |
| JDK 动态代理 | 生成实现相同接口的**兄弟类**（继承 `Proxy`） | **扁平**，合并到接口代理的拦截器链 |
| CGLIB | 生成目标类的**子类** | **纵向嵌套**（子类的子类） |

**为什么 JDK 能缓解代理叠加问题**：CGLIB 是"子类继承"的纵向嵌套，事务代理再包作用域代理时形成"子类的子类"，带来属性穿透和生命周期耦合问题；JDK 代理是"接口兄弟类"的扁平结构，不继承真实类的状态，Spring AOP 可以把增强逻辑合并到同一个接口代理的拦截器链里。

**多注解会不会重复代理？** 一个 Bean 同时标注 `@Transactional`、`@Cacheable`、自定义 `@Aspect`，通常由同一个 `AutoProxyCreator` 收集所有匹配的 Advisor，**合并到一个代理对象的拦截器链**，不会因为"注解多"就异常。真正的高风险是**容器中存在多个 `AutoProxyCreator`**（比如同时引入 `spring-boot-starter-aop` 和旧版安全框架的 `DefaultAdvisorAutoProxyCreator`），同一个 Bean 被先后代理两次。排查方向应先检查容器里有多少个创建 AOP 代理的 BPP，而不是怀疑"注解太多"。

**`@RefreshScope` + `@Transactional` 的嵌套问题**：两者来自两套独立机制——`@RefreshScope` 本质是 `@Scope(value = "refresh", proxyMode = TARGET_CLASS)`，创建**作用域代理**管理真实 Bean 生命周期；`@Transactional` 通过 `InfrastructureAdvisorAutoProxyCreator` 创建**事务代理**织入事务逻辑。CGLIB 模式下可能形成纵向嵌套：

```text
调用者 → 事务代理（处理 @Transactional）→ 作用域代理（处理 @RefreshScope 生命周期）→ 原始 OrderService
```

嵌套会带来属性丢失、刷新后外层事务代理仍持旧引用导致事务脱节、内部调用失效。**规避**：职责分离（热更新配置放独立 `@RefreshScope` Bean，事务业务放普通 `@Service`）、或用 `ScopedProxyMode.INTERFACES` 改 JDK 接口代理。

**TargetSource 五种类型**——决定"代理把调用转发给谁"：

| 类型 | 行为 | 典型场景 |
| :--- | :--- | :--- |
| `SingletonTargetSource` | 静态，永远返回同一实例 | Spring AOP 默认，`@Transactional` 普通单例代理 |
| `SimpleBeanTargetSource` | 动态，每次从 BeanFactory 取 | `@RefreshScope` 可替换目标 |
| `PrototypeTargetSource` | 每次创建新实例 | 每次调用都要新对象 |
| `ThreadLocalTargetSource` | 每线程一个目标实例 | 线程隔离 |
| `CommonsPoolTargetSource` | 对象池，借出/归还 | 目标对象创建成本高 |

一个高频陷阱：`AopProxyUtils.getSingletonTarget(proxy)` 对普通 `@Transactional` Bean 返回真实对象，但对 `@RefreshScope` Bean 返回 **null**——因为它内部要求 `TargetSource instanceof SingletonTargetSource`，而 `@RefreshScope` 的目标源是 `SimpleBeanTargetSource`。

### 4.3 @Transactional 为什么失效：八大场景

`@Transactional` 靠 AOP 代理拦截（`TransactionInterceptor`）实现，所以**只要调用不经过代理，或代理判定"不该管"，事务就不生效**。八大失效场景：

| # | 失效场景 | 根因 | 处置 |
| :--- | :--- | :--- | :--- |
| 1 | **同类内部调用**（`this.method()`） | 内部调用不经过代理 | 拆分 Bean / 自注入 / `AopContext.currentProxy()` |
| 2 | 方法非 `public` | Spring AOP 只拦截 public 方法（`@Transactional` 加在 private/protected 不生效） | 改成 public，或改用 AspectJ 织入 |
| 3 | 异常被 `catch` 吞掉 | 事务只在**抛出**异常时回滚，catch 后不抛 = 不回滚 | 在 catch 里手动 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` |
| 4 | 异常类型不匹配 | 默认只回滚 `RuntimeException` 和 `Error`，受检异常不回滚 | `@Transactional(rollbackFor = Exception.class)` |
| 5 | 数据库引擎不支持事务 | MyISAM 不支持事务 | 改用 InnoDB |
| 6 | 传播行为配错 | `NOT_SUPPORTED`/`NEVER` 等让方法不在事务里跑 | 检查 `propagation` 配置 |
| 7 | 类没被 Spring 管理 | `new` 出来的对象、忘加 `@Service` | 确保由容器管理、通过注入获取 |
| 8 | 多线程 / `@Async` 中调用 | 事务绑定线程，新线程拿不到原事务 | 重新设计事务边界 |

其中 **1 号（同类内部调用）是最高频、也最容易被追问**。它和 3.1 节的结论 3 是同一件事：AOP 代理在 `postProcessAfterInitialization` 创建，代理只拦"从外部进代理"的调用，`this.method()` 是直接调原始对象，自然绕过了 `TransactionInterceptor`。

顺带厘清两个基础概念（容易被问串）：

- **隔离级别**：`ISOLATION_DEFAULT`（用数据库默认，MySQL 是 REPEATABLE_READ，Oracle 是 READ_COMMITTED）→ `READ_UNCOMMITTED` → `READ_COMMITTED` → `REPEATABLE_READ` → `SERIALIZABLE`，从低到高。
- **传播行为**：`REQUIRED`（默认，有则加入、无则新建）、`REQUIRES_NEW`（总是新建，挂起当前）、`NESTED`（嵌套事务，可独立回滚）、`SUPPORTS`/`MANDATORY`/`NOT_SUPPORTED`/`NEVER`。**`REQUIRES_NEW` 与 `NESTED` 的区别**是高频追问：前者是完全独立的物理事务（外层回滚不影响内层），后者是外层事务的保存点（内层回滚后外层可继续）。

### 4.4 四个扩展点的边界（选错就等于白写）

| 扩展点 | 作用对象 | 执行时机 | 能做什么 | 不能做什么 |
| :--- | :--- | :--- | :--- | :--- |
| `BeanDefinitionRegistryPostProcessor` | BeanDefinition 注册表 | **最早**（BFPP 子接口，先执行） | **新增/删除/替换** BeanDefinition（如 `@MapperScan` 动态注册 Mapper） | 改 Bean 实例 |
| `BeanFactoryPostProcessor` | BeanDefinition 内容 | 所有 Bean 实例化前 | 修改属性值、占位符替换 | 新增 BeanDefinition |
| `BeanPostProcessor` | Bean 实例 | 每个 Bean 初始化前后 | 注入、代理、包装、注册回调 | 修改 BeanDefinition |
| `Aware` | 自身 | Bean 初始化前段 | 拿到容器资源 | 拿不到"依赖它的 Bean"（可能尚未创建） |

**选择口诀**：要"凭空造 Bean" → `BeanDefinitionRegistryPostProcessor`（MyBatis 的 Mapper 接口就这么注册的）；要"改图纸" → `BeanFactoryPostProcessor`；要"改成品"/加代理 → `BeanPostProcessor`；只是"我自己想拿点容器东西" → `Aware` 或直接 `@Autowired`。

### 4.5 生产踩坑清单

**装饰器落地的三个坑**：① 忘写 `!(bean instanceof XxxDecorator)` 判断 → 被层层包裹，直到 `StackOverflowError`；② 包装后容器内类型是装饰器，`@Autowired UserServiceImpl` 会报 `NoSuchBeanDefinitionException`（**装饰器必须面向接口**）；③ 装饰器 BPP 与 AOP BPP 的顺序决定"代理包装饰"还是"装饰包代理"，**直接决定事务语义**，必须显式实现 `Ordered` 并写测试固化。

**事件机制的高频坑**：

| 坑 | 现象 | 处理 |
| :--- | :--- | :--- |
| 默认同步 | 监听器抛异常 → 发布方事务回滚 | `@Async` 异步化，或 try/catch 兜住 |
| 异步丢上下文 | 拿不到 `TransactionSynchronizationManager`/MDC | 手动透传（`TransmittableThreadLocal`） |
| `@TransactionalEventListener` 不触发 | 发布时无活动事务，默认 `AFTER_COMMIT` 等不到 | 设 `fallbackExecution = true` |
| 事务未提交就消费 | 监听器查不到刚写的记录 | 用 `AFTER_COMMIT` 阶段 |

### 4.6 版本演进

| 版本 | 与本篇相关的变化 |
| :--- | :--- |
| Spring 4.2 | `@EventListener` 引入；事件对象不再强制继承 `ApplicationEvent`（任意 POJO 可发布） |
| Spring 4.3 | `@EventListener` 的 `condition`、`@TransactionalEventListener` |
| Spring 5.3 | `BeanPostProcessor` 两个方法有了 **default 实现** |
| Spring Boot 2.6 | 默认**禁止循环依赖**（`spring.main.allow-circular-references=false`），启动即报错 |
| Spring 6 / Boot 3 | Java 17 基线；AOT 与 GraalVM Native（反射与代理需提前注册，BPP 在 AOT 下行为受限） |
| 持续 | `@Configuration(proxyBeanMethods = false)` 在不少场景推荐，避免 CGLIB 增强开销（但失去 `@Bean` 互调单例语义） |

---

## 五、总结与收口

### 5.1 全景回顾

| 主题 | 是什么 | 在哪一步生效 | 关键类 |
| :--- | :--- | :--- | :--- |
| Aware | Bean 感知容器资源 | 属性填充后、初始化前（分两批） | `invokeAwareMethods`、`ApplicationContextAwareProcessor` |
| BeanPostProcessor | Bean 实例级扩展点 | 每个 Bean 初始化前后 | `AutowiredAnnotationBeanPostProcessor`、`AnnotationAwareAspectJAutoProxyCreator` |
| BeanFactoryPostProcessor | BeanDefinition 级扩展点 | 实例化之前 | `ConfigurationClassPostProcessor` |
| 注解解析 | 大部分注解在此完成 | `refresh()` 第 5 步 | `ConfigurationClassParser` |
| 循环依赖 | 三级缓存提前暴露 | 实例化后 | `DefaultSingletonBeanRegistry#getSingleton` |
| AOP 代理 | 初始化后创建 | `postProcessAfterInitialization` | `AbstractAutoProxyCreator` |
| 事件 | 容器内解耦通信 | 运行期任意时刻 | `ApplicationEventMulticaster` |

### 5.2 边界与代价：五条最容易被误用的结论

1. **"三级缓存解决了所有循环依赖"是错的**——它只解决**单例 + setter/字段注入**；构造器注入、prototype、`@Async` 都救不了，别硬扛，优先重构。
2. **"`@Order` 能排 BPP 顺序"是错的**——必须实现 `PriorityOrdered`/`Ordered` 接口，`@Order` 注解无效。
3. **"`@Transactional` 加上就一定有事务"是错的**——八大失效场景里，同类内部调用、方法非 public、异常被吞掉、受检异常不回滚是最常见的四个，排查事务问题先从这四个查。
4. **"装饰器包装后不影响注入"是错的**——容器内类型变成了装饰器，按具体实现类注入会找不到 Bean，必须面向接口。
5. **"事件监听器报错不会影响主流程"是错的**——默认同步广播，异常会抛给发布者并回滚主事务；要隔离必须显式异步化。

**排查三板斧**：遇到"注解/事务不生效"→ 先确认调用是否经过代理（是否同类内部调用、方法是否 public）→ 再确认相关 BPP 是否被注册且没跳过该 Bean（日志里查 `not eligible`）→ 最后确认代理拓扑（装饰器/AOP/作用域代理的先后顺序）。

Spring 的"魔法"没有黑盒，只是一条 Bean 生命周期主线上的钩子按顺序执行。把这条线的顺序刻进脑子里，绝大多数"为什么"都能自己推导出来。
