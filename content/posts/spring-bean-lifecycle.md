---
title: "Spring 的「魔法」都挂在一条主线上：Bean 生命周期"
date: 2026-10-27T00:55:00+08:00
summary: "IoC、AOP、@Transactional、@Value、事件监听……Spring 的「魔法」看起来很多，其实都挂在同一条主线上：Bean 生命周期。把 doCreateBean 的四步和 initializeBean 的内部顺序走通，其余机制都能自己推导。这篇顺着这条主线，讲清 AOP 代理在哪一步创建、为什么同类内部调用不走事务、以及 @Order 为什么对 BeanPostProcessor 无效。"
categories: ["框架"]
tags: ["Spring", "Bean 生命周期", "BeanPostProcessor", "AOP", "循环依赖", "三级缓存"]
author: ZLLiang
ShowToc: true
TocOpen: false
draft: false
---

很多人学 Spring，是把 IoC、AOP、事务、事件、注解一个一个拆开记的。记到后面就乱了：`@Value` 什么时候生效？`@PostConstruct` 和 `afterPropertiesSet` 谁先谁后？为什么同类里自己调自己的方法，`@Transactional` 就不生效了？

其实这些问题有一个共同的答案：**它们都挂在同一条主线上——Bean 生命周期**。Spring 里几乎所有的「魔法」，本质都是插在这条主线不同位置上的钩子。把这条主线走通一遍，上面那些问题不用背，自己就能推导出来。

## 一条主线：doCreateBean 的四步

一个 Bean 从无到有，主流程在 `AbstractAutowireCapableBeanFactory#doCreateBean` 里，只有四步：

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, Object[] args) {
    BeanWrapper instanceWrapper = createBeanInstance(beanName, mbd, args); // ① 实例化
    // ② 提前暴露：解决循环依赖的三级缓存
    populateBean(beanName, mbd, instanceWrapper);   // ③ 属性填充：依赖注入
    exposedObject = initializeBean(beanName, exposedObject, mbd); // ④ 初始化
    return exposedObject;
}
```

四步里最值得抠的是第 ④ 步 `initializeBean`，它的内部顺序是最高频的考点，也是理解一切机制的关键：

```java
invokeAwareMethods(beanName, bean);                       // ① 第一批 Aware
applyBeanPostProcessorsBeforeInitialization(...);         // ② 所有 BPP 的「初始化前」回调
invokeInitMethods(beanName, wrappedBean, mbd);            // ③ 初始化方法
applyBeanPostProcessorsAfterInitialization(...);          // ④ BPP 的「初始化后」回调：AOP 在这里创建
```

从这个顺序能推出三个必须记牢的结论：

1. **Aware 回调都在「属性填充之后、初始化方法之前」**，只是分两批——`BeanNameAware` 等由容器直接调用，`ApplicationContextAware` 等由 `ApplicationContextAwareProcessor` 在 BPP 链里执行。所以「实现 `ApplicationContextAware` 能拿到 `@Value` 注入的值吗」的答案是**能**，因为属性填充已经完成。

2. **初始化方法三级顺序固定**：`@PostConstruct` → `InitializingBean#afterPropertiesSet()` → 自定义 `init-method`。

3. **AOP 代理在「初始化后」（第 ④ 步）创建**。这一条是理解事务的钥匙——见下文。

## 从主线推导：为什么同类内部调用不走事务

理解了「AOP 代理在初始化后创建」，就能解释 Spring 里最经典的一个坑：**同一个类里，方法 A 调用带 `@Transactional` 的方法 B，事务不生效**。

因为 `@Transactional` 是通过 AOP 代理实现的：Spring 给你注入的，是一个**代理对象**。外部调用 `service.methodA()` 时，经过代理，代理在方法前后织入事务。但一旦进了方法内部，`this.methodB()` 调的是**原对象**，不经过代理，事务拦截器自然拦不到。

这跟「AOP 代理在初始化后创建」是同一件事的两个面：代理是容器初始化 Bean 时包上去的一层壳，方法内部的自调用绕开了这层壳。所以「同类内部调用不走事务」不是 Spring 的 bug，是代理机制的必然推论——理解了主线，这个坑就从「玄学」变成了「本该如此」。

## 代价一：@Order 对 BeanPostProcessor 无效

BeanPostProcessor 是这条主线上最核心的扩展点，但它的排序规则藏着一个高频坑：**`@Order` 注解对 BPP 不生效**。

原因很实在：BPP 的排序发生在「注解读取排序器」就绪之前，Spring 只认 `PriorityOrdered` / `Ordered` 这两个**接口**。你要控制 BPP 顺序，得实现接口，而不是加 `@Order` 注解。

还有一条更要命的边界：**BPP 不能被 BPP 处理**。`registerBeanPostProcessors` 会先把所有 BPP 实例化出来，此时容器处于「BPP 尚未就绪」状态。如果你写的 BPP 依赖了业务 Bean，日志会打出 `Bean 'xxx' is not eligible for getting processed by all BeanPostProcessors`——意思是这个 Bean 跳过了部分 BPP，AOP 代理可能没创建、`@Autowired` 可能没注入。结论很硬：**BPP 内部只允许依赖其他 BPP 或基础设施类，不要注入业务 Bean**。

## 代价二：循环依赖与三级缓存

循环依赖是 Bean 生命周期里最容易出问题的场景，Spring 用**三级缓存**来解：

| 缓存 | 内容 | 作用 |
| :--- | :--- | :--- |
| `singletonObjects`（一级） | 完全初始化好的单例 | 最终成品 |
| `earlySingletonObjects`（二级） | 提前暴露的「半成品」 | 避免重复调用工厂 |
| `singletonFactories`（三级） | `ObjectFactory`，可生成早期引用 | **延迟决定**暴露原对象还是代理 |

为什么需要第三级？因为存在 AOP：A 被 B 依赖时，A 还没初始化完，但 B 需要的是**代理对象**。第三级缓存里的 `ObjectFactory` 在 `getEarlyBeanReference` 时调用 `getEarlyBeanReference`，**提前生成代理**放进二级缓存——保证 B 拿到的和最终放进一级缓存的是**同一个代理对象**，否则会出现「注入的是代理、单例池里是原对象」的诡异现象。

但这里有个生产必须知道的版本变化：**Spring Boot 2.6+ 默认禁止循环依赖**（`spring.main.allow-circular-references=false`），启动即报错。这是官方在推动「重构掉循环依赖」。所以正确姿势不是打开开关硬扛，而是用 `@Lazy`、抽取第三方 Bean、事件解耦把依赖理清——**循环依赖本质是设计问题，不是配置问题**。

## 选对扩展点，否则白写

理解了主线，就能理解 Spring 的四个扩展点各管什么、选错了会怎样：

| 扩展点 | 能做什么 | 口诀 |
| :--- | :--- | :--- |
| `BeanDefinitionRegistryPostProcessor` | 凭空造出 Bean | 要「新增/删除 Bean 定义」用它（MyBatis 的 Mapper 就是这么注册的） |
| `BeanFactoryPostProcessor` | 改图纸 | 改已有 BeanDefinition 的属性值、占位符替换 |
| `BeanPostProcessor` | 改成品、加代理 | 注入、代理、包装、注册回调 |
| `Aware` | 自己拿容器资源 | 只是「我想拿点容器东西」，别用它干别的 |

选错扩展点的典型后果：想「凭空造 Bean」却用了 `BeanFactoryPostProcessor`——它只能改已有的 BeanDefinition，造不出新的。所以 MyBatis 的 `@MapperScan` 用的是 `BeanDefinitionRegistryPostProcessor`，而不是别的。

## 收口

Spring 的「魔法」没有看起来那么多，它们都挂在「Bean 生命周期」这一条主线上：实例化 → 属性填充 → 初始化（Aware → BPP before → init → BPP after）→ 代理。AOP 代理、事务拦截、`@Value` 注入、事件监听器注册，本质都是插在这条主线不同位置上的钩子。

把这条主线在脑子里走通，比背十个零散知识点都管用：你会理解为什么同类内部调用不走事务、为什么 `@Order` 对 BPP 无效、为什么循环依赖要三级缓存、以及四个扩展点各自该在什么时候用。这些不再是「坑」，而是「机制的必然」。

延伸一个问题：Spring 的事件默认是同步的——监听器里抛异常，会把发布方的事务一起回滚。如果你用 `@Async` 把它异步化，又会在监听器里拿不到 `SecurityContext` 和 MDC 这些上下文。这个「同步丢事务、异步丢上下文」的两难，你会怎么解？
