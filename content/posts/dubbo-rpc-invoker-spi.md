---
title: "一次 RPC 调用，Dubbo 在背后做了多少件事：Invoker 与 SPI 微内核"
date: 2026-10-27T01:05:00+08:00
summary: "调用一个远程服务，代码里只写了一行 `demoService.sayHello()`，跟本地方法一样。但这行代码背后，Dubbo 穿过了十层架构。这篇顺着一次调用的链路，讲清 Dubbo 最核心的两个抽象——Invoker 如何统一本地与远程调用、SPI 微内核如何用三级装配实现可插拔——以及重试机制在什么情况下会把「高可用」变成「重复执行」。"
categories: ["微服务"]
tags: ["Dubbo", "RPC", "Invoker", "SPI", "ExtensionLoader", "Failover"]
author: ZLLiang
ShowToc: true
TocOpen: false
draft: false
---

调用一个远程服务，你的代码里只有一行：

```java
String result = demoService.sayHello("world");
```

跟调本地方法一模一样。没有连接管理、没有序列化、没有超时重试——这些脏活累活都被 Dubbo 藏起来了。但「藏起来」不等于「不存在」：这行代码背后，Dubbo 穿过了整整十层架构。理解这十层，你才能理解 Dubbo 为什么能「可插拔」，也才能理解为什么「看起来很简单」的 RPC，会在某个凌晨变成一次重复扣款的事故。

## 一次调用穿过十层

Dubbo 官方把架构抽象成十层，自上而下是：

```text
Service（业务接口）→ Config（配置解析）→ Proxy（代理生成）
→ Registry（注册发现）→ Cluster（集群容错）→ Monitor（监控）
→ Protocol（协议）→ Exchange（信息交换）→ Transport（网络传输）→ Serialize（序列化）
```

Consumer 侧的一次调用，走的是这条链：`Proxy` 代理对象 → `Cluster`（容错 + 负载均衡）→ `Protocol`（refer 得到的 Invoker）→ `Exchange`（封装 Request）→ `Transport`（Netty 发送）→ 序列化。反方向在 Provider 侧逐层解开，最终调用到业务实现。

这十层不是「堆在一起的功能列表」，而是「每一层都能被替换」。能做到这一点，靠的是 Dubbo 最核心的一个抽象——Invoker。

## Invoker：把两件本质不同的事统一成一个接口

Dubbo 的 `Invoker` 是理解一切的钥匙。它用同一个接口统一了两件本质不同的事：

- **Provider 侧的 Invoker**：包装业务实现，`invoke()` 就是反射调用本地方法。
- **Consumer 侧的 Invoker**：包装远程调用，`invoke()` 就是发网络请求。

于是 Cluster 层根本不需要关心「本地还是远程」，它只面对一组 Invoker。**这是 Dubbo 能用同一套容错、路由、负载均衡逻辑，处理多种协议和本地调用的根本原因。**

集群层的「四件套」就是围绕 Invoker 展开的：`Directory`（有哪些 Invoker）→ `Router`（筛掉不符合规则的）→ `LoadBalance`（选一个）→ `Cluster`（调用失败怎么办）。Consumer 拿到的代理，本质是「把 Directory 伪装成一个 Invoker」，所以调用方看到的是「一个」，背后其实是「一群」。

## SPI 微内核：可插拔是怎么实现的

Dubbo 的「可插拔」靠自研的 SPI，它针对 JDK SPI 的四个局限（全量实例化、只能遍历、线程不安全、无 IOC/AOP）做了强化。`ExtensionLoader` 在加载一个扩展时，完成四件事：

1. **按名称加载**：`getExtension("dubbo")` 用双重检查锁 + 缓存保证单例与线程安全。
2. **Adaptive 自适应**：启动时不知道用哪个实现，就由代码生成器**动态生成 `$Adaptive` 类**，运行时从 URL 取扩展名再委托。比如 `Protocol$Adaptive` 会根据 `url.getProtocol()` 决定用 `dubbo` 还是 `tri` 协议。
3. **Activate 自动激活**：按 `group`/`value`/`order` 批量激活，用于 Filter 链、Listener 链。
4. **Wrapper 包装**：构造器只有一个 SPI 接口参数的类，自动被识别为 Wrapper 层层包装——`ProtocolFilterWrapper` 就是这么给协议加上过滤能力的。

这套机制的精髓在于：**扩展点不是「写死的 if-else」，而是「运行时从 URL/配置里动态决定」**。这解释了为什么 Dubbo 加一个协议、换一个负载均衡策略，只需要改配置、不需要改代码。

## 代价：Failover 重试什么时候变成灾难

理解了 Invoker 和 SPI，就能理解 Dubbo 最危险的一个默认行为。默认的 `FailoverClusterInvoker` 会在调用失败时重试（默认 2 次），核心逻辑里有一个关键判断：

```java
for (int i = 0; i <= retries; i++) {
    Invoker<T> invoker = select(loadbalance, invocation, invokers, invoked);
    try {
        return invoker.invoke(invocation);
    } catch (RpcException e) {
        if (e.isBiz()) throw e;   // 业务异常不重试
    }
}
```

三个点必须记牢：重试会尽量落到**不同** Provider（`invoked` 集合传给 `select`）；**业务异常直接抛出不重试**（参数校验失败重试没意义）；重试只在当前 Invoker 列表内选，不会重新拉地址。

而真正的灾难来自一个更隐蔽的机制：**`timeout` 是客户端等待响应的超时，Provider 端不会因为超时而中断执行**。这意味着「客户端已经超时、发起重试，Provider 还在吭哧吭哧处理第一次请求」——重试反而造成了**重复执行**。

于是有了那条写操作铁律：**写操作必须 `retries=0` + 业务幂等，两者缺一不可**。重复下单、重复扣款，几乎都源于「超时重试 + 非幂等」这个组合。这里有个反直觉的结论：**对写操作，「重试」不是高可用，是危险源**。

## 代价：注册中心挂了怎么办

另一个常被问的边界：注册中心（ZooKeeper）挂了，Dubbo 还能不能调用？

答案是能，靠的是三层容错：客户端（Curator）自动重注册监听器规避 Watcher 一次性；`AbstractRegistry` 用「内存 + 磁盘」双缓存，注册中心不可用时仍能读旧地址；`FailbackRegistry` 用时间轮对网络抖动导致的订阅失败做重试恢复。所以注册中心挂了，Consumer 还能用本地缓存的地址列表继续调用——**但只能「用旧的」，新上线的 Provider 地址就感知不到了**。这也是为什么注册中心可以短暂不可用、但不能长期不可用的原因。

这里还有一个容易答错的细节：Dubbo 在 ZooKeeper 里**不是读临时节点的子节点**（临时节点不能有子节点），而是读 `providers` 这个持久父节点的子节点列表，再逐个 `getData` 拿 URL。服务实例本身才是那个「断了连接就消失」的临时节点。

## 收口

一行 `demoService.sayHello()` 背后，是「十层架构 + Invoker 统一抽象 + SPI 可插拔 + 集群容错」这套完整机制。Dubbo 把复杂度藏起来，不是让它消失，而是把它转移到了「运行时动态装配」里——这正是 SPI 微内核的意义。

而理解这套机制的回报是实在的：你会在设计写操作时条件反射地想到 `retries=0` 和幂等，会在注册中心抖动时知道「还能调用、但地址是旧的」，会在线程池打满时意识到是慢调用占满了 200 个业务线程。这些不是经验，是机制的必然推论。

延伸一个问题：Dubbo 3 把默认协议从 `dubbo` 换成了 `tri`，还把服务发现从「接口级」换成了「应用级」。后者把 ZNode 数量降了一个数量级——你知道接口级发现在大规模集群下为什么会 ZNode 爆炸吗？这跟「一个接口写一条记录」的设计有什么直接关系？
