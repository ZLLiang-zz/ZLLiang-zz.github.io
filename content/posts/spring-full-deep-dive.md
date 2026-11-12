---
title: "Spring 全量深度解析：从 IoC/AOP 入门到 Bean 生命周期、三级缓存与事务失效"
date: 2026-11-12T21:00:00+08:00
summary: "一篇覆盖初级到高级的 Spring 全量长文。初级：IoC/DI/AOP 是什么、为什么用、Spring 生态地图；中级：Bean 四种定义方式、依赖注入三种方式与取舍、常用注解、作用域与线程安全、SpringMVC 请求流程、声明式事务用法、AOP 完整用法（五类通知与切点表达式）；高级（最完善）：Bean 生命周期完整源码、循环依赖为什么要三级缓存、AOP 代理体系全景（JDK/CGLIB 拓扑、重复代理、@RefreshScope 双层代理、TargetSource）、四类扩展点边界、@Transactional 八大失效场景、事件机制与事务绑定、SpringBoot 自动配置原理、设计模式与生产踩坑。每个结论带成立条件和代价。"
categories: ["Spring"]
tags: ["Spring", "IoC", "AOP", "依赖注入", "Bean生命周期", "BeanPostProcessor", "循环依赖", "三级缓存", "事务", "SpringMVC", "SpringBoot", "自动配置", "JDK动态代理", "CGLIB", "事件机制"]
author: ZLLiang
ShowToc: true
TocOpen: false
draft: false
---


很多人写 Spring 写了好几年，依然卡在两个地方：一是**概念层**讲不清 IoC 和 DI 到底谁是谁、AOP 为什么"横切"能生效；二是**机制层**一被追问"`@Transactional` 为什么失效""循环依赖为什么要三级缓存"就哑火。原因通常是学的时候直接从源码和"高级八股"切入，跳过了中间的用法和原理层。

这篇按**初级 → 中级 → 高级**三层铺开：初级把"是什么、为什么"讲透，中级把"怎么用、怎么工作"讲全，高级把最难啃的硬骨头（生命周期源码、三级缓存、AOP 代理体系、扩展点、事务失效、自动配置）讲到源码级。**高级部分最完整**，因为那才是区分"会用 Spring"和"懂 Spring"的分水岭。读完你能：面对初级问题条理清晰、面对中级问题说到机制、面对高级追问落到源码。

---

## 一、初级：Spring 是什么、解决了什么问题

### 1.1 Spring 要解决的两个核心痛点

Spring 诞生于 EJB 时代，要解决两个痛点：

1. **对象创建与依赖关系的复杂度爆炸**。一个 `Service` 依赖几十上百个底层类，手动 `new` 意味着你要搞清每一个的构造顺序、生命周期。Spring 把"创建对象、装配依赖"这件事接管过来，你只声明"我需要一个 `OrderMapper`"，装配的苦差事交给容器——这就是 **IoC**。
2. **横切逻辑到处复制粘贴**。日志、事务、权限这些逻辑散落在几百个方法里，改一处要改几百处。Spring 用 **AOP** 把这些横切逻辑集中到"切面"里，织入目标方法，业务代码里一行都不用写。

### 1.2 IoC 与 DI：谁是谁

这是最容易被问倒的第一题。记住一句话：**IoC 是思想，DI 是手段。**

| 术语 | 全称 | 本质 | 通俗理解 |
| :--- | :--- | :--- | :--- |
| IoC | Inversion of Control | **设计思想** | 把"手动 new 对象"的控制权，交给容器 |
| DI | Dependency Injection | **实现方式** | 容器在创建对象时，主动把它的依赖"塞"进来 |

```java
// 没有 IoC：自己 new，自己管生命周期，依赖写死
public class OrderService {
    private OrderMapper mapper = new OrderMapperImpl();  // 耦合到具体实现
}

// 有 IoC/DI：只声明依赖，容器注入
@Service
public class OrderService {
    private final OrderMapper mapper;           // 只依赖接口
    public OrderService(OrderMapper mapper) {  // 构造器注入，容器负责塞进来
        this.mapper = mapper;
    }
}
```

**代价要认**：控制权交出去后，对象创建不再由你直接掌控，启动变慢、报错链路变长——这正是"Spring 启动慢、异常难定位"的根源，也是 AOT（提前编译）要解决的痛点。

### 1.3 AOP：面向切面编程是什么

**AOP（Aspect-Oriented Programming）**：把散布在各处的**横切关注点**（日志、事务、安全、缓存）抽出来，集中写进一个"切面"，在不改业务代码的前提下织入目标方法。

**为什么叫"切面"**：业务代码是一条条的"纵向"调用链（Controller → Service → Mapper），而日志/事务是"横向"横切所有方法的一层——把它抽出来就是"切面"。

```java
@Aspect
@Component
public class LogAspect {
    @Around("execution(* com.example.service.*.*(..))")   // 切点：所有 service 方法
    public Object log(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("before " + pjp.getSignature().getName());
        Object result = pjp.proceed();                      // 放行，执行真实方法
        System.out.println("after " + pjp.getSignature().getName());
        return result;
    }
}
```

上面这段，就把"每个 service 方法前后打日志"这件事，从几百个方法里抽到了 6 行。**优点**：业务代码干净、横切逻辑集中可改。**代价**：执行流不再直观（代理帮你插了层），排查和调试时要注意"这个方法其实被代理了"。

### 1.4 Spring 的生态地图

先分清三个名字，很多人混着说：

| 名称 | 是什么 | 关系 |
| :--- | :--- | :--- |
| **Spring Framework** | 核心框架（IoC 容器、AOP、数据访问、Web MVC） | 地基 |
| **Spring Boot** | 在 Framework 之上做"自动配置 + 起步依赖 + 内嵌容器"，让项目开箱即用 | 加速器 |
| **Spring Cloud** | 基于 Boot 的微服务生态（注册中心、配置中心、网关、熔断） | 微服务全家桶 |

Spring Framework 自身的模块：

| 模块 | 作用 |
| :--- | :--- |
| Core Container（Core/Beans/Context/SpEL） | IoC 容器本体，最核心 |
| AOP / Aspects | 面向切面编程 |
| Data Access（JDBC/ORM/Transactions） | 数据访问与事务抽象 |
| Web（Web MVC/WebFlux） | Web 层 |
| Test | 集成测试支持 |

### 1.5 术语表（先对齐口径）

| 术语 | 含义 |
| :--- | :--- |
| **Bean** | 由 Spring 容器创建并管理的对象 |
| **IoC 容器** | 负责创建 Bean、装配依赖、管理生命周期的对象（`BeanFactory`/`ApplicationContext`） |
| **BeanDefinition** | Bean 的"图纸/元数据"：类名、作用域、构造参数、属性值 |
| **依赖注入（DI）** | 容器把 Bean 的依赖注入给它（构造器/Setter/字段三种） |
| **自动装配（Autowiring）** | 容器根据类型/名称自动找依赖，无需手动指定 |

---

## 二、中级：怎么用（基础知识点）

### 2.1 Bean 的四种定义方式

| 方式 | 适用场景 | 示例 |
| :--- | :--- | :--- |
| 注解扫描 | 自己写的类 | `@Component`/`@Service`/`@Repository`/`@Controller` |
| `@Bean` 方法 | 第三方类、需要自定义创建逻辑 | 在 `@Configuration` 类里写方法返回对象 |
| `@Import` | 导入配置类/普通类/ImportSelector | `@Import(MyConfig.class)` |
| XML（历史遗留） | 老项目 | `<bean id="..." class="..."/>` |

`@Component` 与 `@Bean` 的区别是高频题：**前者作用于类（靠扫描发现），后者作用于方法（显式声明）**。引入第三方库、需要按条件返回不同实现时只能用 `@Bean`。

### 2.2 依赖注入的三种方式与取舍

| 方式 | 写法 | 优点 | 缺点 |
| :--- | :--- | :--- | :--- |
| **构造器注入** | 构造参数 | 依赖不可变（可 `final`）、必填、便于测试、**官方推荐** | 参数多时构造器臃肿 |
| Setter 注入 | `setXxx` | 可选依赖、可改 | 依赖可能漏注入（NPE 风险） |
| 字段注入 | 字段上 `@Autowired` | 简洁 | 隐藏依赖、难测试、易 NPE |

**结论（可直接背）**：首选**构造器注入**，依赖用 `final` 修饰，保证"对象创建好就能用、依赖不可能为空"；可选依赖才用 Setter；字段注入最简洁但**隐藏了依赖关系**，大项目里会让"这个类到底依赖谁"变得不可见。

### 2.3 常用注解速查

| 注解 | 作用 |
| :--- | :--- |
| `@Autowired` | 按**类型**自动装配（默认必填，可用 `required=false`） |
| `@Qualifier` | 配合 `@Autowired` 按**名称**消歧（多个同类型 Bean 时） |
| `@Resource` | JDK 注解，**默认按名称**，找不到再按类型（顺序和 @Autowired 相反） |
| `@Value` | 注入配置值/占位符 `${...}`/SpEL |
| `@Configuration` + `@Bean` | 配置类 + 声明 Bean |
| `@ComponentScan` | 指定扫描包路径 |
| `@Scope` | 指定作用域 |
| `@Lazy` | 懒加载（用到才创建） |
| `@Primary` | 同类型多个 Bean 时的默认首选 |

### 2.4 Bean 作用域与线程安全

| 作用域 | 含义 |
| :--- | :--- |
| `singleton` | 默认，容器内唯一实例 |
| `prototype` | 每次获取新建一个实例 |
| `request` / `session` | Web 环境，一个请求/会话一个 |

**线程安全**：单例 Bean 若持有**可变成员变量**，多线程写它就有安全问题。官方推荐的解法不是"改成 prototype"（治标不治本，还引入管理复杂度），而是**用 `ThreadLocal` 隔离可变状态**，或把状态外置到无状态服务。

### 2.5 生命周期回调三件套

一个 Bean 初始化/销毁时想执行自己的逻辑，有三种写法（执行顺序固定）：

```java
@Component
public class MyBean implements InitializingBean, DisposableBean {
    @PostConstruct
    public void postConstruct() { /* ① 最先执行 */ }

    @Override
    public void afterPropertiesSet() { /* ② 其次 */ }

    // init-method 通过 @Bean(initMethod = "init") 或 XML 指定
    public void init() { /* ③ init-method 最后 */ }
}
```

**初始化顺序**：`@PostConstruct` → `InitializingBean#afterPropertiesSet()` → 自定义 `init-method`。**销毁顺序**同理：`@PreDestroy` → `DisposableBean#destroy()` → `destroy-method`。

### 2.6 BeanFactory 与 ApplicationContext

**内核与门面的组合关系**——`ApplicationContext` 本身就是一个 `BeanFactory`，它内部**组合**一个 `DefaultListableBeanFactory` 真正干活：

```text
AnnotationConfigApplicationContext
  ├── beanFactory: DefaultListableBeanFactory   ← 真正干活的引擎
  │     ├── beanDefinitionMap                    ← 定义（图纸）
  │     └── singletonObjects                     ← 一级缓存（单例池）
  ├── environment / messageSource / resourceLoader
  └── applicationEventMulticaster
```

`ApplicationContext` 比裸 `BeanFactory` 多的是**企业级能力**：自动注册后处理器（BPP/BFPP）、事件机制、国际化、资源加载，以及**启动时预实例化所有非懒加载单例**（让配置错误尽早暴露）。一句话：**ApplicationContext 是门面，BeanFactory 是引擎**。

### 2.7 SpringMVC 请求流程（中级高频）

一个 HTTP 请求进来到返回，核心链路：

```text
请求 → DispatcherServlet（前端控制器，唯一入口）
  → HandlerMapping        找处理器（哪个 Controller 的哪个方法）
    → HandlerInterceptor  拦截器前置
  → HandlerAdapter        执行 Handler（适配各种类型的处理器）
    → Controller 方法返回 ModelAndView（或 @ResponseBody 直接写响应）
    → HandlerInterceptor  拦截器后置
  → ViewResolver          解析视图（前后端分离后常被 @RestController + JSON 替代）
  → 渲染响应
```

**关键组件职责**：`HandlerMapping` 负责"找"（把 URL 映射到处理器），`HandlerAdapter` 负责"调"（适配不同处理器类型），`ViewResolver` 负责"渲染"。**前后端分离后**，`@RestController` 直接写 JSON，`ViewResolver` 基本退出舞台——能说出这一点，比背"九大组件"更显理解。

### 2.8 声明式事务怎么用（用法层）

```java
@Service
public class OrderService {
    @Transactional(rollbackFor = Exception.class, timeout = 30)
    public void createOrder(Order order) {
        orderMapper.insert(order);
        stockService.decrease(order.getItemId(), 1);
    }
}
```

三个最常用的属性：`rollbackFor`（指定哪些异常回滚）、`timeout`（超时）、`readOnly`（只读优化）。**一个新手最容易踩的坑**：默认只回滚 `RuntimeException` 和 `Error`，**受检异常（checked exception）不回滚**——所以要么用 `rollbackFor = Exception.class`，要么捕获受检异常后手动回滚。

---

## 三、中级→高级：技术原理（怎么工作的）

### 3.1 Bean 生命周期主线（一切机制的地基）

`AbstractAutowireCapableBeanFactory#doCreateBean` 是 Bean 诞生的主流程：

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, Object[] args) {
    // ① 实例化：反射调用构造器/工厂方法（此刻属性全是默认值）
    BeanWrapper instanceWrapper = createBeanInstance(beanName, mbd, args);

    // ② 提前暴露（解决循环依赖的三级缓存，见 4.1）

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
    // ① 第一批 Aware：BeanName / BeanClassLoader / BeanFactory
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

**三个必须记牢的结论**：

1. **Aware 回调分两批**，都发生在"属性填充之后、初始化方法之前"：第一批 `invokeAwareMethods` 直调（BeanName/BeanClassLoader/BeanFactory），第二批由 `ApplicationContextAwareProcessor`（一个 BPP）在 BPP 链里执行。
2. **初始化方法三级顺序固定**：`@PostConstruct` → `afterPropertiesSet()` → `init-method`。
3. **AOP 代理产生于"初始化后（④）"**，所以 `@Transactional`、`@Async` 才拦得住——这直接解释了"同类内部调用不走事务"（内部调用不经过代理）。

### 3.2 AOP 原理：切面、切点、通知

AOP 的四个核心概念：

| 概念 | 是什么 | 对应注解 |
| :--- | :--- | :--- |
| **切面（Aspect）** | 横切逻辑的载体，= 切点 + 通知 | `@Aspect` |
| **切点（Pointcut）** | 匹配哪些方法要被织入 | `@Pointcut` / 表达式 |
| **通知（Advice）** | 织入的时机和逻辑 | 五种通知注解 |
| **连接点（JoinPoint）** | 具体被拦截的方法 | 方法参数 |

**五种通知类型**（按执行顺序记）：

| 注解 | 执行时机 |
| :--- | :--- |
| `@Before` | 方法执行**前** |
| `@After` | 方法执行**后**（类似 finally，无论成功失败都执行） |
| `@AfterReturning` | 方法**正常返回后** |
| `@AfterThrowing` | 方法**抛异常后** |
| `@Around` | **环绕**，可同时控制前后，最强大也最危险 |

**切点表达式**最常用的写法：

```text
execution(修饰符? 返回类型 包名.类名.方法名(参数) throws?)
// 例：execution(* com.example.service.*.*(..))
//     所有返回值、service 包下所有类的所有方法、任意参数
```

**JDK 动态代理 vs CGLIB 的选择规则**：

| 条件 | 用哪种 |
| :--- | :--- |
| 目标类**实现了接口** | 默认 JDK 动态代理（基于接口） |
| 目标类**没实现接口** | CGLIB（基于子类继承） |
| Spring Boot 2.x 起 | 默认**统一用 CGLIB**（`proxyTargetClass=true`） |

**优点**：横切逻辑集中、业务代码无侵入。**代价**：① 只有从**外部进入代理**的调用才被拦截，同类内部 `this.method()` 绕过了代理；② `final` 方法、`private` 方法无法被 CGLIB 代理（无法覆写）。

### 3.3 事务原理：@Transactional 靠 AOP 拦截

`@Transactional` 本身不干活，它靠 **AOP 代理 + `TransactionInterceptor`** 实现：

```text
调用者 → 事务代理（@Transactional 生成的代理对象）
  → TransactionInterceptor.invoke()
    → 事务管理器开启事务（或加入已有事务）
      → method.invoke(target)   执行业务方法
    → 提交 / 回滚
```

**两个基础概念（必被问，且常问串）**：

**隔离级别**（从低到高）：

| 级别 | 解决 | 引入问题 |
| :--- | :--- | :--- |
| `READ_UNCOMMITTED` | 无 | 脏读、不可重复读、幻读 |
| `READ_COMMITTED` | 脏读 | 不可重复读、幻读 |
| `REPEATABLE_READ`（MySQL 默认） | 脏读、不可重复读 | 幻读（InnoDB 靠 MVCC/间隙锁基本解决） |
| `SERIALIZABLE` | 全部 | 性能最差 |

**传播行为**（高频）：

| 行为 | 含义 |
| :--- | :--- |
| `REQUIRED`（默认） | 有则加入，无则新建 |
| `REQUIRES_NEW` | **总是新建独立事务**，挂起当前 |
| `NESTED` | 嵌套事务，用**保存点**实现，内层可独立回滚 |
| `SUPPORTS` / `MANDATORY` / `NOT_SUPPORTED` / `NEVER` | 组合条件 |

> `REQUIRES_NEW` vs `NESTED` 的区别是经典追问：前者是**完全独立的物理事务**（外层回滚不影响内层），后者是**外层事务的保存点**（内层回滚后外层可继续）。

### 3.4 自动装配原理：@Autowired 怎么找到依赖

`@Autowired` 的解析发生在**属性填充（populateBean）阶段**，由 `AutowiredAnnotationBeanPostProcessor` 处理：

```text
populateBean → AutowiredAnnotationBeanPostProcessor
  → 解析 @Autowired 字段/方法
  → 按类型（byType）从容器找候选 Bean
    ├─ 唯一匹配 → 直接注入
    ├─ 多个匹配 → 再按名称（byName）消歧；仍冲突 → @Primary 优先 → 仍冲突 → 抛 NoUniqueBeanDefinitionException
    └─ 无匹配 → required=false 跳过；否则抛 NoSuchBeanDefinitionException
```

**记忆点**：`@Autowired` **先按类型、再按名称**（`@Qualifier` 指定）；`@Resource` **先按名称、再按类型**——两者顺序相反，这是高频对比题。

### 3.5 SpringBoot 自动配置原理（高级高频）

`@SpringBootApplication` 是三个注解的合成：

```java
@SpringBootConfiguration   // = @Configuration
@EnableAutoConfiguration   // ← 自动配置的入口
@ComponentScan             // 扫描当前包及子包
```

自动配置的核心链路：

```text
@EnableAutoConfiguration
  → @Import(AutoConfigurationImportSelector)
    → 读取 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
      （Boot 2.7 之前是 spring.factories 里的 EnableAutoConfiguration 键）
    → 得到几百个候选自动配置类
    → 用 @ConditionalOnXxx 逐条判断是否生效
    → 排序（@AutoConfigureBefore/After/Order）→ 注册 BeanDefinition
```

**条件注解是自动配置的灵魂**（每个都对应"满足才生效"）：

| 条件注解 | 生效条件 |
| :--- | :--- |
| `@ConditionalOnClass` | classpath 存在某类 |
| `@ConditionalOnMissingBean` | 容器里没有某 Bean（**用户自定义优先的关键**） |
| `@ConditionalOnProperty` | 配置项满足条件 |
| `@ConditionalOnBean` | 容器里有某 Bean |

**经典案例**：为什么你引入 `spring-boot-starter-data-redis` 后 `RedisTemplate` 就有了？因为自动配置类 `RedisAutoConfiguration` 上有 `@ConditionalOnClass(RedisOperations.class)`（引入了 redis 依赖就满足）+ `@ConditionalOnMissingBean(RedisTemplate.class)`（你没自己定义才生效）。**`@ConditionalOnMissingBean` 是"用户自定义 Bean 覆盖默认配置"的机制保证**。

---

## 四、高级（最完善）：深入与高级延伸

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
| 三级 `singletonFactories` | `ObjectFactory` 工厂 | **延迟决定是否提前生成代理**——它存在的唯一理由 |

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

**为什么必须是三级，两级不行？** 这是核心论点。假设只有两级（一级成品 + 二级早期引用），当 A 需要 AOP 代理时：实例化 A 后若把 `A_raw` 直接放二级，B 从二级拿到的是**没有事务、没有切面逻辑的裸对象**，而 A 初始化后生成的 `A_proxy` 才进一级——**B 拿到裸对象，容器持有代理，两者不一致**。

修这个问题只有两条路：① 实例化后立刻生成代理放二级——但这样**所有 Bean 都会提前代理**，打乱 BPP 时序、让不需要代理的 Bean 白付代理开销；② **二级缓存放一个"能按需生成代理"的东西，而不是对象**——这就是三级缓存存的 `ObjectFactory` 工厂。

**三级缓存的本质价值：把"是否提前生成代理"的决定，延迟到"确实发生循环依赖、且有人来取"的那一刻。** 无循环依赖 → 工厂不被调用 → 代理正常生成；有循环依赖且需要代理 → 工厂被调用 → 代理提前生成。

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

### 4.2 AOP 代理体系全景

**两条代理路径**：

| 技术 | 原理 | 叠加结构 |
| :--- | :--- | :--- |
| JDK 动态代理 | 生成实现相同接口的**兄弟类**（继承 `Proxy`） | **扁平**，合并到接口代理的拦截器链 |
| CGLIB | 生成目标类的**子类** | **纵向嵌套**（子类的子类） |

**为什么 JDK 能缓解代理叠加**：CGLIB 是"子类继承"的纵向嵌套，事务代理再包作用域代理时形成"子类的子类"，带来属性穿透和生命周期耦合问题；JDK 是"接口兄弟类"的扁平结构，不继承真实类状态，AOP 可把增强逻辑合并到同一个接口代理的拦截器链。

**多注解会不会重复代理？** 一个 Bean 同时标注 `@Transactional`、`@Cacheable`、`@Aspect`，通常由同一个 `AutoProxyCreator` 收集所有匹配的 Advisor，**合并到一个代理对象的拦截器链**，不会因"注解多"就异常。真正的高风险是**容器存在多个 `AutoProxyCreator`**（如同时引入 `spring-boot-starter-aop` 和旧版安全框架的 `DefaultAdvisorAutoProxyCreator`），同一个 Bean 被先后代理两次。排查先查容器里有多少个创建 AOP 代理的 BPP，而不是怀疑"注解太多"。

**`@RefreshScope` + `@Transactional` 嵌套**：两者来自两套独立机制——`@RefreshScope` 本质是 `@Scope(value = "refresh", proxyMode = TARGET_CLASS)`，创建**作用域代理**管理真实 Bean 生命周期；`@Transactional` 通过 `InfrastructureAdvisorAutoProxyCreator` 创建**事务代理**。CGLIB 下可能纵向嵌套：

```text
调用者 → 事务代理（@Transactional）→ 作用域代理（@RefreshScope）→ 原始 OrderService
```

嵌套带来属性丢失、刷新后外层事务代理仍持旧引用、内部调用失效。**规避**：职责分离（热更新配置放独立 `@RefreshScope` Bean）、或用 `ScopedProxyMode.INTERFACES` 改 JDK 接口代理。

**TargetSource 五种类型**（决定"代理把调用转发给谁"）：

| 类型 | 行为 | 典型场景 |
| :--- | :--- | :--- |
| `SingletonTargetSource` | 静态，永远返回同一实例 | Spring AOP 默认 |
| `SimpleBeanTargetSource` | 动态，每次从 BeanFactory 取 | `@RefreshScope` |
| `PrototypeTargetSource` | 每次创建新实例 | 每次调用都要新对象 |
| `ThreadLocalTargetSource` | 每线程一个目标实例 | 线程隔离 |
| `CommonsPoolTargetSource` | 对象池，借出/归还 | 目标对象创建成本高 |

> 高频陷阱：`AopProxyUtils.getSingletonTarget(proxy)` 对普通 `@Transactional` Bean 返回真实对象，但对 `@RefreshScope` Bean 返回 **null**——因为它要求 `TargetSource instanceof SingletonTargetSource`，而 `@RefreshScope` 的目标源是 `SimpleBeanTargetSource`。

### 4.3 四类扩展点的边界（选错就等于白写）

| 扩展点 | 作用对象 | 执行时机 | 能做什么 | 不能做什么 |
| :--- | :--- | :--- | :--- | :--- |
| `BeanDefinitionRegistryPostProcessor` | BeanDefinition 注册表 | **最早**（BFPP 子接口，先执行） | **新增/删除/替换** BeanDefinition（`@MapperScan` 动态注册 Mapper） | 改 Bean 实例 |
| `BeanFactoryPostProcessor` | BeanDefinition 内容 | 所有 Bean 实例化前 | 修改属性值、占位符替换 | 新增 BeanDefinition |
| `BeanPostProcessor` | Bean 实例 | 每个 Bean 初始化前后 | 注入、代理、包装、注册回调 | 修改 BeanDefinition |
| `Aware` | 自身 | Bean 初始化前段 | 拿到容器资源 | 拿不到"依赖它的 Bean" |

**选择口诀**：要"凭空造 Bean" → `BeanDefinitionRegistryPostProcessor`（MyBatis 的 Mapper 接口就这么注册的）；要"改图纸" → `BeanFactoryPostProcessor`；要"改成品"/加代理 → `BeanPostProcessor`；只想"拿点容器东西" → `Aware` 或直接 `@Autowired`。

**BPP 的两个高频坑**：

1. **`@Order` 注解对 BPP 不生效**——BPP 排序发生在"注解读取排序器"就绪之前，Spring 只认 `PriorityOrdered`/`Ordered` **接口**。
2. **BPP 不能被 BPP 处理**——`registerBeanPostProcessors` 先把所有 BPP 实例化出来，若某 BPP 依赖业务 Bean，日志会打 `Bean 'xxx' is not eligible for getting processed by all BeanPostProcessors`，意味着该 Bean 跳过部分 BPP（AOP 代理可能没创建）。**结论：BPP 内部只允许依赖其他 BPP 或基础设施类。**

### 4.4 @Transactional 八大失效场景

`@Transactional` 靠 AOP 代理拦截，**只要调用不经过代理、或代理判定"不该管"，事务就不生效**。八大失效：

| # | 失效场景 | 根因 | 处置 |
| :--- | :--- | :--- | :--- |
| 1 | **同类内部调用**（`this.method()`） | 内部调用不经过代理 | 拆分 Bean / 自注入 / `AopContext.currentProxy()` |
| 2 | 方法非 `public` | Spring AOP 只拦截 public 方法 | 改成 public，或 AspectJ 织入 |
| 3 | 异常被 `catch` 吞掉 | 事务只在**抛出**异常时回滚 | catch 里手动 `setRollbackOnly()` |
| 4 | 异常类型不匹配 | 默认只回滚 `RuntimeException`/`Error`，受检异常不回滚 | `rollbackFor = Exception.class` |
| 5 | 数据库引擎不支持事务 | MyISAM 不支持 | 改用 InnoDB |
| 6 | 传播行为配错 | `NOT_SUPPORTED`/`NEVER` 等方法不在事务里跑 | 检查 `propagation` |
| 7 | 类没被 Spring 管理 | `new` 出来的、忘加 `@Service` | 确保由容器管理、通过注入获取 |
| 8 | 多线程 / `@Async` 中调用 | 事务绑定线程，新线程拿不到原事务 | 重新设计事务边界 |

其中 **1 号（同类内部调用）是最高频**，它和 3.1 结论 3 是同一件事：AOP 代理只拦"从外部进代理"的调用，`this.method()` 直接调原始对象，绕过 `TransactionInterceptor`。

### 4.5 事件机制：从发布到监听 + 事务绑定

**完整链路**：

```text
publisher.publishEvent(event)
  → ApplicationEventMulticaster.multicastEvent(event, type)
    → 解析事件类型 → 查找匹配监听器（带缓存）
    → for (listener) invokeListener(listener, event)
        ├─ 同步：当前线程直接调用（默认）
        └─ 异步：若配置 taskExecutor 则提交线程池
```

**角色映射**：发布者 `ApplicationEventPublisher`、监听者 `@EventListener`/`ApplicationListener`、广播器 `ApplicationEventMulticaster`、注解处理器 `EventListenerMethodProcessor`（把 `@EventListener` 方法包装成监听器）。

**事务绑定的 `@TransactionalEventListener`**：让监听器在**事务提交后/回滚后**等指定阶段触发，解决"事务没提交监听器就查到不数据"的问题：

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT, fallbackExecution = true)
public void afterCommit(OrderCreatedEvent event) { ... }
```

**生产高频坑**：

| 坑 | 现象 | 处理 |
| :--- | :--- | :--- |
| 默认同步 | 监听器抛异常 → 发布方事务回滚 | `@Async` 异步化，或 try/catch 兜住 |
| 异步丢上下文 | 拿不到事务/MDC/SecurityContext | 手动透传（`TransmittableThreadLocal`） |
| `@TransactionalEventListener` 不触发 | 发布时无活动事务，默认等不到 | 设 `fallbackExecution = true` |
| 事务未提交就消费 | 监听器查不到刚写的记录 | 用 `AFTER_COMMIT` 阶段 |

### 4.6 Spring 用到的设计模式（含易混辨析）

**装饰器 vs 代理**（结构相似、本质不同）：

| 维度 | 装饰器模式 | 代理模式 |
| :--- | :--- | :--- |
| 目的 | **增强功能**（叠加职责） | **控制访问**（权限、延迟、远程） |
| 谁组合 | 调用方主动组合 | 框架自动生成 |
| 典型例子 | `TransactionAwareCacheDecorator`、`ContentCachingRequestWrapper` | `@Transactional` 事务代理 |

> ⚠️ 面试问"Spring AOP 是装饰器模式吗"，答案是**不是**——Spring AOP 用的是**代理模式**。

除装饰器、代理、观察者外，还有：**工厂模式**（`BeanFactory` 创建 Bean）、**单例模式**（Bean 默认单例）、**适配器模式**（AOP 的 `AdvisorAdapter`、SpringMVC 的 `HandlerAdapter`）、**模板方法模式**（`JdbcTemplate`、`RestTemplate`、`AbstractApplicationContext#refresh`）。

### 4.7 生产踩坑清单

**装饰器落地的三个坑**：① 忘写 `!(bean instanceof XxxDecorator)` 判断 → 被层层包裹直到 `StackOverflowError`；② 包装后容器内类型是装饰器，`@Autowired UserServiceImpl` 会报 `NoSuchBeanDefinitionException`（**装饰器必须面向接口**）；③ 装饰器 BPP 与 AOP BPP 的顺序决定"代理包装饰"还是"装饰包代理"，**直接决定事务语义**，必须显式实现 `Ordered` 并写测试固化。

**事件机制的坑**（见 4.5 表格）。

**循环依赖的坑**：Spring Boot 2.6 起**默认禁止循环依赖**（`spring.main.allow-circular-references=false`），启动即报错——很多老项目升级 Boot 后突然启动失败，根因就是这里的循环依赖，正确做法是重构而不是打开开关。

### 4.8 版本演进

| 版本 | 与本篇相关的变化 |
| :--- | :--- |
| Spring 4.2 | `@EventListener` 引入；事件对象不再强制继承 `ApplicationEvent` |
| Spring 4.3 | `@EventListener` 的 `condition`、`@TransactionalEventListener` |
| Spring 5.3 | `BeanPostProcessor` 两个方法有了 **default 实现** |
| Spring Boot 2.6 | 默认**禁止循环依赖** |
| Spring Boot 2.7 | 自动配置从 `spring.factories` 迁移到 `AutoConfiguration.imports` |
| Spring 6 / Boot 3 | Java 17 基线；AOT 与 GraalVM Native（反射与代理需提前注册） |
| 持续 | `@Configuration(proxyBeanMethods = false)` 在无 `@Bean` 互调场景推荐，避免 CGLIB 增强开销 |

---

## 五、总结与收口

### 5.1 全景回顾

| 主题 | 是什么 | 在哪一步生效 | 关键类 |
| :--- | :--- | :--- | :--- |
| IoC/DI | 容器管理对象与依赖 | 全程 | `BeanFactory`、`ApplicationContext` |
| Aware | Bean 感知容器资源 | 属性填充后、初始化前（分两批） | `invokeAwareMethods`、`ApplicationContextAwareProcessor` |
| BeanPostProcessor | Bean 实例级扩展点 | 每个 Bean 初始化前后 | `AutowiredAnnotationBeanPostProcessor`、`AnnotationAwareAspectJAutoProxyCreator` |
| BeanFactoryPostProcessor | BeanDefinition 级扩展点 | 实例化之前 | `ConfigurationClassPostProcessor` |
| 自动装配 | 按类型/名称找依赖 | `populateBean` 阶段 | `AutowiredAnnotationBeanPostProcessor` |
| 循环依赖 | 三级缓存提前暴露 | 实例化后 | `DefaultSingletonBeanRegistry#getSingleton` |
| AOP 代理 | 初始化后创建 | `postProcessAfterInitialization` | `AbstractAutoProxyCreator` |
| 事务 | AOP 拦截 + 事务管理器 | 方法调用时 | `TransactionInterceptor` |
| 自动配置 | 条件装配 Bean | `refresh()` 第 5 步 | `AutoConfigurationImportSelector` |
| 事件 | 容器内解耦通信 | 运行期任意时刻 | `ApplicationEventMulticaster` |

### 5.2 边界与代价：六条最容易被误用的结论

1. **"三级缓存解决了所有循环依赖"是错的**——它只解决**单例 + setter/字段注入**；构造器注入、prototype、`@Async` 都救不了，别硬扛，优先重构。
2. **"`@Order` 能排 BPP 顺序"是错的**——必须实现 `PriorityOrdered`/`Ordered` 接口，`@Order` 注解无效。
3. **"`@Transactional` 加上就一定有事务"是错的**——八大失效场景里，同类内部调用、方法非 public、异常被吞掉、受检异常不回滚是最常见的四个。
4. **"装饰器包装后不影响注入"是错的**——容器内类型变成了装饰器，按具体实现类注入会找不到 Bean，必须面向接口。
5. **"事件监听器报错不会影响主流程"是错的**——默认同步广播，异常会抛给发布者并回滚主事务；要隔离必须显式异步化。
6. **"`@Autowired` 按名称注入"是错的**——它是**先按类型、再按名称**消歧；`@Resource` 才是先按名称。

**排查三板斧**：遇到"注解/事务不生效"→ 先确认调用是否经过代理（是否同类内部调用、方法是否 public）→ 再确认相关 BPP 是否被注册且没跳过该 Bean（日志查 `not eligible`）→ 最后确认代理拓扑（装饰器/AOP/作用域代理的先后顺序）。

Spring 的"魔法"没有黑盒，只是一条 Bean 生命周期主线上的钩子按顺序执行。初级记住"是什么"，中级记住"怎么用和怎么工作"，高级把这条线的顺序刻进脑子里——绝大多数"为什么"都能自己推导出来。
