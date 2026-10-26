---
title: "MyBatis 的「魔法」：一个没有实现类的接口，是怎么执行 SQL 的"
date: 2026-10-27T00:50:00+08:00
summary: "你写了一个 Mapper 接口，只声明了方法，没有实现类，MyBatis 却能把它变成一条真正的 SQL 执行。这背后是 JDK 动态代理 + 一条严格的 statementId 约定。这篇顺着 MapperProxy 的 invoke 三条分支，讲清这个「魔法」的机制，以及它带来的几个硬边界：为什么接口能继承不能重载、参数名找不到时报的错是什么意思。"
categories: ["框架"]
tags: ["MyBatis", "动态代理", "MapperProxy", "SqlSource", "一级缓存", "N+1"]
author: ZLLiang
ShowToc: true
TocOpen: false
draft: false
---

刚接触 MyBatis 的人几乎都会被同一个「灵异现象」惊到：你写了一个 `UserMapper` 接口，里面只有 `User selectById(int id)` 这一行方法签名，没有实现类。然后 `userMapper.selectById(1)` 一调用，它居然真的去数据库查出了一条记录。一个没有实现类的方法，是怎么「跑起来」的？

这就是 MyBatis 最核心的机制，也是理解它一切行为的地基。这篇顺着一次方法调用的完整链路，讲清这个魔法是怎么实现的——以及它带来的几个你不理解就会踩的硬边界。

## 那个「实现类」是动态代理生成的

真相不神秘：MyBatis 在启动时，用 JDK 的动态代理给每个 Mapper 接口生成了一个代理对象。

`MapperRegistry.addMapper` 只做两件事：把接口存进 `knownMappers`，并为它建一个 `MapperProxyFactory`；然后解析接口上的 `@Select`、`@Insert` 等注解。真正生成代理是在 `getMapper` 时，`MapperProxyFactory.newInstance` 里那句 `Proxy.newProxyInstance`：

```java
return (T) Proxy.newProxyInstance(
    mapperInterface.getClassLoader(),
    new Class[]{ mapperInterface },
    mapperProxy);
```

注意一个细节：**每个 Mapper 接口只有一个 `MapperProxyFactory`，但 `MapperProxy` 每次 `getMapper` 都新建**——因为它要绑定当前的 `SqlSession`。同一个接口，从不同 SqlSession 拿到的代理，执行上下文是不同的。这也是「Mapper 实例能跨线程用吗」这类问题的答案起点。

## invoke 的三条分支：不是所有方法都查库

代理对象上的任何方法调用，都会进 `MapperProxy.invoke`，它有三个分支：

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);        // ① equals/hashCode/toString
    }
    return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
}
```

- **① Object 的方法**：`toString()`、`hashCode()` 直接反射调用代理自身，**不会查库**。这个分支最容易被忽略，但理解它你才不会被「调 mapper.toString() 居然没报错」搞懵。
- **② 接口的 default 方法**：3.4.2+ 支持，反射调用接口里的默认实现，让你能在接口里写带逻辑的默认方法。
- **③ 真正的 SQL 方法**：包装成 `MapperMethod` 执行，这才是查库的入口。

所以「没有实现类」的真相是：**实现类（代理对象）是有的，只是它的实现是「把方法调用转发给 MyBatis 去查 XML 或注解里的 SQL」**，而不是你手写的业务逻辑。

## statementId：方法名和 SQL 是怎么对上的

调用 `selectById` 时，MyBatis 怎么知道该执行哪条 SQL？靠一条严格的约定：**`statementId = 接口全限定名 + "." + 方法名`**。

`MapperMethod` 构造时解析出两个东西：`SqlCommand`（statementId + 语句类型）和 `MethodSignature`（返回类型特征 + 参数名解析结果）。`SqlCommand` 就是拿 `userMapper.selectById` 这个 statementId，去 `Configuration` 里找对应的 `MappedStatement`——它可能来自 XML 的 `<select id="selectById">`，也可能来自方法上的 `@Select` 注解。

找到后按语句类型分派：INSERT/UPDATE/DELETE 走 `sqlSession.insert/update/delete`；SELECT 根据返回类型分别走 `selectOne`、`selectList`、`selectMap`、`selectCursor`。这里有个容易忽略的细节：`selectOne` 查到多行会抛 `TooManyResultsException`，这个校验发生在 `DefaultSqlSession` 里——它先按 `selectList` 查、再判断 `size == 1`。所以看到这个异常，不是 SQL 写错了，是数据真的有多条。

## 这条约定带来的两个硬边界

因为 statementId 只由「接口名 + 方法名」决定，跟参数列表无关，于是有两个反直觉的边界：

**第一，接口可以继承，但不能重载。** 继承没问题——子接口继承父接口的方法时，statementId 按**当前调用的接口**算，所以 XML 的 namespace 必须写成子接口。但重载不行：两个方法参数不同、方法名相同，statementId 就撞了，`Configuration` 内部的 `StrictMap` 直接拒绝重复 key。

**第二，参数名找不到时的报错信息是排查的钥匙。** 参数名解析有四个来源，优先级从高到低：`@Param("name")` 注解 → 编译保留的真实参数名（`-parameters`）→ 参数索引名 `param1`/`param2` → 单参数特例直接用。如果都没对上，抛出的异常信息非常具体：

```text
BindingException: Parameter 'userId' not found.
Available parameters are [arg0, arg1, param1, param2]
```

看到这个错，检查三件事：`@Param` 漏没漏、XML 名字拼没拼错、`-parameters` 编译参数开没开（Spring Boot 的 maven 插件默认开，老项目常常没开）。这就是「魔法」的另一面：**它越省事，出问题时定位成本越高，你必须理解它内部的约定才能排查**。

## 代价：缓存脏数据与 N+1

理解了动态代理，再看两个最常见的坑，就都顺理成章。

**一级缓存的脏数据。** MyBatis 的一级缓存是 `SqlSession` 级别的：同一个 SqlSession 里，相同的 SQL + 相同的参数，第二次查直接返回缓存结果，不再打库。这带来一个经典问题——同一个 SqlSession 内，你先 `selectById` 缓存了结果，然后绕开 MyBatis 直接 `UPDATE` 改了数据库，再 `selectById`，拿到的还是旧值。根因是缓存没有感知到「外部修改」。生产环境里，因为 MyBatis 和 Spring 集成时每个请求通常一个 SqlSession，这个坑相对隐蔽，但在手动管理 SqlSession 的场景里很容易踩。

**N+1 查询。** 一对多关联查询时，如果先查主表（1 次），再对每一条主记录去查子表（N 次），就是 N+1 次查询。解决方案要么用 `JOIN` + 嵌套结果映射一次性查出来，要么接受延迟加载、并清楚它「用到才查」带来的额外开销。

这两个坑的共同点是：**MyBatis 的「省事」是有边界的，理解它的代理和缓存机制，才知道边界在哪、怎么绕**。

## 收口

回到开头的灵异现象：一个没有实现类的接口能执行 SQL，靠的是「JDK 动态代理 + statementId 约定」这套机制——代理对象把方法调用翻译成 statementId，再从 Configuration 里取回那条 SQL 去执行。它把「写实现类」这件事省掉了，代价是「接口名 + 方法名」成了唯一的寻址依据，于是有了不能重载、参数名要按优先级解析这些硬约束。

理解 MyBatis 的正确方式，不是背「它用了动态代理」，而是顺着 `invoke` 的三个分支、`MapperMethod` 的构造、`StrictMap` 的拒绝，把这条链路在脑子里走通。走通了，那些 BindingException、TooManyResults、缓存脏数据，就都从「灵异」变成了「本该如此」。

延伸一个问题：MyBatis 的动态 SQL（`<if>`、`<foreach>`）在运行时是一棵 `SqlNode` 树，那么 OGNL 表达式里取不到一个变量时，为什么它不报错、而是静默跳过？这个「静默失效」的设计，会在什么场景下让你查出错误的结果？
