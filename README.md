# ZLLiang 的技术笔记

个人技术博客，基于 [Hugo](https://gohugo.io/) + [PaperMod](https://github.com/adityatelange/hugo-PaperMod)，通过 GitHub Actions 自动部署到 GitHub Pages。

线上地址：<https://zlliang-zz.github.io/>

## 内容方向

- **Java 运行时** —— 内存布局、类加载、GC、JIT
- **并发** —— JMM、锁升级、AQS、线程池、并发容器、虚拟线程
- **数据库** —— InnoDB 内核、索引与优化器、事务与锁
- **分布式** —— 注册中心、RPC、消息队列、缓存一致性

写作原则：讲机制而不是背结论，给出可执行的排查路径，不写没验证过的内容。

## 文章索引

### 全量深度（系统完整覆盖一个主题）

| 主题 | 文章 | 日期 |
|:---|:---|:---|
| Spring | [Spring 全量深度解析：从 IoC/AOP 入门到 Bean 生命周期、三级缓存与事务失效](https://zlliang-zz.github.io/posts/spring-full-deep-dive/) | 2026-11-12 |
| MyBatis | [MyBatis 全量深度解析：从 Mapper 代理到缓存与插件链](https://zlliang-zz.github.io/posts/mybatis-full-deep-dive/) | 2026-11-12 |
| JVM | [JVM 全量深度解析：内存分区、对象生命周期、GC 与线上调优](https://zlliang-zz.github.io/posts/jvm-full-deep-dive/) | 2026-11-10 |
| Java 并发 | [Java 并发编程全量深度解析：从 JMM、锁升级、AQS 到线程池与 ThreadLocal 工程](https://zlliang-zz.github.io/posts/java-concurrency-full-deep-dive/) | 2026-11-09 |
| Redis | [Redis 全量深度解析：线程模型、数据结构、持久化、高可用与分布式工程](https://zlliang-zz.github.io/posts/redis-full-deep-dive/) | 2026-11-09 |
| RocketMQ | [RocketMQ 全量深度解析：从消息的诞生到高可用、性能与并发工程](https://zlliang-zz.github.io/posts/rocketmq-full-deep-dive/) | 2026-10-27 |
| MySQL | [MySQL 索引与事务：B+ 树、MVCC 与锁的真实行为](https://zlliang-zz.github.io/posts/mysql-index-transaction/) | 2026-10-20 |

### 专题切片（单点深挖）

| 方向 | 文章 | 日期 |
|:---|:---|:---|
| 消息队列 | [RocketMQ 为什么把消息混写进一个文件：CommitLog 的顺序写与读放大](https://zlliang-zz.github.io/posts/rocketmq-commitlog-design/) | 2026-10-27 |
| 消息队列 | [RocketMQ 延时消息不是「等一会再投」：寄存一条、到期重写一条](https://zlliang-zz.github.io/posts/rocketmq-delay-message-truth/) | 2026-10-27 |
| 消息队列 | [Kafka 的高吞吐是怎么来的：零拷贝、稀疏索引，以及它们什么时候失效](https://zlliang-zz.github.io/posts/kafka-zero-copy-throughput/) | 2026-10-27 |
| 微服务 | [一次 RPC 调用，Dubbo 在背后做了多少件事：Invoker 与 SPI 微内核](https://zlliang-zz.github.io/posts/dubbo-rpc-invoker-spi/) | 2026-10-27 |
| 微服务 | [一个注册中心，为什么 AP 和 CP 要同时存在：Nacos 的双协议模型](https://zlliang-zz.github.io/posts/nacos-ap-cp-dual-protocol/) | 2026-10-27 |
| 分布式协调 | [ZooKeeper：ZAB 协议、Watcher 与会话机制](https://zlliang-zz.github.io/posts/zookeeper-zab-watcher/) | 2026-10-14 |
| Java 运行时 | [SPI 与 TCCL：父加载器如何加载子加载器的实现](https://zlliang-zz.github.io/posts/spi-tccl-classloader/) | 2026-10-25 |
| Java 并发 | [Java 线程池：ctl、Worker 与生产参数治理](https://zlliang-zz.github.io/posts/java-thread-pool/) | 2026-10-18 |
| 架构 | [拆库拆表之后，一条 SQL 变成了三大难题：分库分表的真实代价](https://zlliang-zz.github.io/posts/shardingsphere-sharding-tradeoffs/) | 2026-10-27 |

## 本地运行

需要 Hugo Extended ≥ 0.146。

```bash
hugo server -D
```

打开 <http://localhost:1313/>。改动会热重载。

构建静态文件：

```bash
hugo --gc --minify
```

产物在 `public/`。

## 写一篇新文章

```bash
hugo new content posts/my-new-post.md
```

或者在 `content/posts/` 下直接建文件，front matter 示例：

```yaml
---
title: "文章标题"
date: 2026-10-24T09:00:00+08:00
summary: "出现在列表页的一句话摘要"
categories: ["Java 并发"]
tags: ["AQS", "并发"]
ShowToc: true
draft: false
---
```

`draft: true` 的文章只在 `hugo server -D` 时可见，不会被部署。

## 部署

推送到 `main` 分支即触发 `.github/workflows/hugo.yaml`：

1. 安装指定版本 Hugo
2. `hugo --gc --minify` 构建
3. 上传产物并发布到 GitHub Pages

首次部署后需在仓库 **Settings → Pages → Build and deployment** 中把 Source 设为 **GitHub Actions**。

## 目录结构

```
.
├── assets/css/extended/    # 自定义样式（自动合并进主题 CSS）
├── content/
│   ├── posts/              # 文章
│   ├── about.md            # 关于页
│   ├── archives.md         # 归档页
│   └── search.md           # 搜索页
├── static/                 # 静态资源（favicon 等）
├── themes/PaperMod/        # 主题（随仓库提交，避免子模块）
├── hugo.toml               # 站点配置
└── .github/workflows/      # 自动部署
```

## License

文章内容版权归作者所有，转载请注明出处。主题与配置代码遵循 MIT。
