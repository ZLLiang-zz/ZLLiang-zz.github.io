---
title: "ZooKeeper：ZAB 协议、Watcher 与会话机制"
date: 2026-10-14T09:00:00+08:00
summary: "ZAB 的两种模式与投票顺序、一次写请求经历的四个阶段、用临时顺序节点做分布式锁的边界在哪、Watcher 一次性触发会带来什么坑，以及会话超时之后到底发生了什么。"
categories: ["分布式"]
tags: ["ZooKeeper", "ZAB", "分布式锁", "一致性"]
author: ZLLiang
ShowToc: true
TocOpen: false
draft: false
---

## 一、基本概念

### 1.1 是什么

ZooKeeper 是一个**分布式协调服务**：它对外提供一棵类文件系统的树形命名空间（ZNode），并保证**写操作的全局顺序与强一致**，同时支持**一次性触发的监听（Watcher）**和**会话（Session）生命周期**。

一句话概括它的本质：

> **一个容量很小、写入强一致、带通知能力、能感知客户端存活的树形 KV 存储。**

理解这句话的四个限定词，就抓住了 ZK 的边界：

| 限定词 | 含义 | 直接后果 |
| :--- | :--- | :--- |
| 容量很小 | 单节点数据默认上限 1 MB | 只存元数据、地址、配置，不存业务数据 |
| 写入强一致 | 所有写请求走 Leader，过半落盘才提交 | 写性能受集群规模与磁盘 IO 限制 |
| 带通知能力 | Watcher 一次性触发 | 客户端必须重新注册，代码要处理「通知丢失窗口」 |
| 能感知存活 | 临时节点随会话消失而删除 | 服务上下线可自动感知，无需额外心跳表 |

### 1.2 解决什么问题

分布式系统缺少「共识」和「协调」的基础设施，ZK 把常见的协调原语沉淀成四种能力：

| 能力 | 依赖的 ZK 特性 | 典型场景 |
| :--- | :--- | :--- |
| 服务注册与发现 | 临时节点 + 子节点 Watcher | Dubbo Provider 上下线 |
| 配置管理 | 持久节点 + 数据 Watcher | 开关、路由规则、限流阈值 |
| 分布式锁 | 临时顺序节点 + 前驱监听 | 秒杀扣库存、定时任务互斥 |
| 选主 / 成员管理 | 临时顺序节点 + Watcher | Master 选举、集群存活列表 |

**它不解决的问题**（常被用来划边界）：

- 不做大容量存储（1 MB 限制、全量数据放内存）
- 不做高吞吐写（写要过半落盘，集群越大越慢）
- 不做复杂查询（只有按路径取，没有条件检索）
- 不做最终一致的消息队列（虽然能模拟队列，但没有消费位点、重试、堆积能力）

### 1.3 在技术体系中的定位

ZK 属于**协调层**，位于「业务服务」与「基础设施」之间：

```text
业务服务（Dubbo Consumer / Provider、定时任务、网关）
        │  注册、订阅、抢锁、选主
        ▼
协调层  ZooKeeper（CP，强一致，小容量）
        │
        ▼
存储层  本地磁盘（事务日志 + 快照）
```

按 CAP 归类，ZK 是 **CP** 系统：网络分区时优先保证一致性，少数派分区无法完成写操作、也无法选出新 Leader。

### 1.4 核心术语表

| 术语 | 含义 | 备注 |
| :--- | :--- | :--- |
| ZNode | 树上的一个节点 | 由 path + data + stat + children 组成 |
| zxid | 64 位事务 ID | 高 32 位 epoch，低 32 位自增计数 |
| epoch | Leader 任期号 | 换 Leader 递增，写入 zxid 高位 |
| Session | 客户端与服务端的会话 | 有超时时间，过期后临时节点被删除 |
| Watcher | 一次性事件监听 | 触发即失效，需重新注册 |
| Quorum | 过半节点集合 | `n/2 + 1`，写提交与选主都依赖它 |
| tick | 心跳基本时间单位 | `tickTime` 毫秒，`sessionTimeout` 以 tick 计 |
| myid | 集群内节点编号 | 存在于 `dataDir/myid`，参与投票比较 |
| Observer | 只同步不投票的节点 | 扩容读能力，不增加写 quorum 成本 |
| ACL | 节点访问控制 | scheme:id:permission，如 `world:anyone:cdrwa` |

---

## 二、基础知识点

### 2.1 部署与核心配置

单机最小配置 `conf/zoo.cfg`：

```properties
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/var/zookeeper/data
dataLogDir=/var/zookeeper/logs
clientPort=2181
maxClientCnxns=200
autopurge.snapRetainCount=5
autopurge.purgeInterval=24
```

集群在三台机器的 `zoo.cfg` 中追加：

```properties
server.1=10.0.0.1:2888:3888
server.2=10.0.0.2:2888:3888
server.3=10.0.0.3:2888:3888
```

并在各自 `dataDir/myid` 写入 `1` / `2` / `3`。

| 参数 | 作用 | 调优要点 |
| :--- | :--- | :--- |
| `tickTime` | 心跳与超时的基本单位 | 默认 2000 ms，通常不动 |
| `initLimit` | Follower 首次同步允许的 tick 数 | 数据量大时调大（如 10~20） |
| `syncLimit` | Follower 与 Leader 心跳超时 tick 数 | 网络抖动大时调大 |
| `dataDir` | 快照目录 | 与日志目录分开，避免 IO 争抢 |
| `dataLogDir` | 事务日志目录 | **强烈建议独立磁盘**，写性能瓶颈在这里 |
| `maxClientCnxns` | 单客户端 IP 最大连接数 | 默认 60，连接池场景常需调大 |
| `autopurge.*` | 自动清理旧快照与日志 | 不配会撑爆磁盘，生产必配 |

### 2.2 常用命令

```bash
# 会话内
ls /path                    # 列出子节点
stat /path                  # 查看节点状态（含版本、zxid）
get /path                   # 取数据 + stat
create /node "data"         # 持久节点
create -e /temp "data"      # 临时节点
create -s /seq "data"       # 持久顺序节点
create -e -s /lock/lock- "" # 临时顺序节点
set /path "new"             # 更新（可带版本号做 CAS）
delete /path                # 删除（有子节点会失败）
deleteall /path             # 递归删除
```

四字命令（`echo stat | nc 127.0.0.1 2181`）：

| 命令 | 用途 |
| :--- | :--- |
| `stat` | 版本、连接数、znode 数、模式（Leader/Follower） |
| `srvr` | 服务器角色与延迟统计 |
| `mntr` | 关键指标（`zk_avg_latency`、`zk_outstanding_requests`、`zk_znode_count`） |
| `cons` | 客户端连接明细 |
| `ruok` | 健康检查，返回 `imok` |
| `conf` | 当前生效配置 |

> 3.5+ 起四字命令需要通过 `4lw.commands.whitelist` 白名单开启，默认只允许 `srvr`。

### 2.3 Java API 与 Curator

原生 API：

```java
ZooKeeper zk = new ZooKeeper("127.0.0.1:2181", 30_000, event -> {
    // 全局 Watcher：只处理会话级事件（连接、过期）
});

zk.create("/config", "v=1".getBytes(StandardCharsets.UTF_8),
        ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);

byte[] data = zk.getData("/config", event -> {
    // 节点数据变化回调；回调后该 Watcher 失效
}, null);

zk.exists("/config", event -> { /* 节点创建/删除/数据变化 */ });

zk.getChildren("/dubbo/com.example.DemoService/providers", event -> {
    // 子节点列表变化（NodeChildrenChanged）
});
```

`create` 四参数：**路径、数据字节数组、ACL、CreateMode**；返回实际路径（顺序节点带 10 位序号）。

Curator（生产推荐）：

```java
CuratorFramework client = CuratorFrameworkFactory.builder()
        .connectString("10.0.0.1:2181,10.0.0.2:2181,10.0.0.3:2181")
        .sessionTimeoutMs(30_000)
        .connectionTimeoutMs(15_000)
        .retryPolicy(new ExponentialBackoffRetry(1000, 3))
        .build();
client.start();

// 自动创建父节点，解决原生 API 的 NoNodeException
client.create().creatingParentsIfNeeded()
      .withMode(CreateMode.PERSISTENT)
      .forPath("/config", "v=1".getBytes());

// 分布式锁
InterProcessMutex lock = new InterProcessMutex(client, "/lock/order-1");
if (lock.acquire(3, TimeUnit.SECONDS)) {
    try { /* 临界区 */ } finally { lock.release(); }
}
```

Curator 相对原生 API 的四个增强：**连接重试、Watcher 自动重注册、递归创建、封装好的锁/选主/屏障**。

### 2.4 节点类型

节点由「生命周期 × 命名方式」两个维度组合：

| 类型 | CreateMode | 生命周期 | 命名 | 典型用途 |
| :--- | :--- | :--- | :--- | :--- |
| 持久节点 | `PERSISTENT` | 永久，需显式删除 | 指定 | 配置、元数据、注册中心父路径 |
| 持久顺序节点 | `PERSISTENT_SEQUENTIAL` | 永久 | 父节点维护自增序号 | 分布式队列、全局 ID |
| 临时节点 | `EPHEMERAL` | 会话结束自动删除 | 指定 | 服务注册、存活感知 |
| 临时顺序节点 | `EPHEMERAL_SEQUENTIAL` | 会话结束自动删除 | 自增序号 | 分布式锁、选主 |
| 容器节点（3.5+） | `CONTAINER` | 永久，子节点为空时自动清理 | 指定 | 大量子节点的父容器 |
| TTL 节点（3.5+） | `PERSISTENT_WITH_TTL` | 到期且无子节点时删除 | 指定/顺序 | 带过期语义的临时数据 |

三条硬性语义：

1. **临时节点不能有子节点**——服务端 `PrepRequestProcessor` 直接拒绝（`EphemeralNodeWithChildren`），根源是会话过期时的级联删除会带来复杂的同步问题。
2. **顺序号由父节点的 `cversion` 生成**，同一父节点下单调递增且唯一；不同父节点各自独立。
3. **节点数据默认上限 1 MB**（`jute.maxbuffer` 可调，但不建议）。

### 2.5 四种典型使用模式

**① 服务注册（临时节点）**

```text
/dubbo/com.example.DemoService/providers
    ├── dubbo://10.0.0.1:20880/...   ← 临时节点，data 是 URL
    └── dubbo://10.0.0.2:20880/...   ← 临时节点
```

Provider 启动时创建临时子节点；会话断开后 ZK 删除该节点；Consumer 监听父节点子列表变化感知上下线。

**② 配置管理（持久节点 + 数据 Watcher）**

配置的生命周期不应与会话绑定，因此用持久节点；客户端 `getData` 时注册 Watcher，配置变更后重新拉取。

**③ 分布式锁（临时顺序 + 前驱监听）**

```java
String myPath = client.create().withMode(CreateMode.EPHEMERAL_SEQUENTIAL)
        .forPath("/lock/lock-");           // 返回 /lock/lock-0000000012

List<String> children = client.getChildren().forPath("/lock");
Collections.sort(children);
int index = children.indexOf(myPath.substring("/lock/".length()));

if (index == 0) {
    // 最小序号，获得锁
} else {
    // 只监听紧邻前驱，避免羊群效应
    String prev = "/lock/" + children.get(index - 1);
    client.getData().usingWatcher(prevWatcher).forPath(prev);
}
```

三个设计点缺一不可：**临时**防死锁、**顺序**保公平、**只听前驱**防惊群。

**④ 选主（临时顺序节点）**

所有候选者创建临时顺序节点，序号最小者为 Master；其余监听前驱，Master 挂掉后下一位接任。

### 2.6 入门常见误区

| 误区 | 事实 |
| :--- | :--- |
| Watcher 注册一次就能一直监听 | 一次性，触发即失效，必须重新注册 |
| 用持久节点做服务注册 | 实例宕机后节点残留，永远无法自动剔除 |
| 在临时节点下创建子节点 | 服务端直接拒绝 |
| 把 ZK 当配置数据库存大 JSON | 1 MB 限制 + 全量内存，容易 OOM |
| `ConnectionLoss` 就认为节点不存在 | 连接丢失是**不确定状态**，不能据此判定节点不存在 |
| 锁节点用持久节点 | 客户端崩溃后锁永不释放 |
| 集群部署偶数台 | 容错能力不增加，只多浪费一台 |

---

## 三、技术原理

### 3.1 数据模型与 stat 结构

每个 ZNode 由四部分组成：

```text
path      /dubbo/com.example.DemoService/providers/dubbo%3A%2F%2F...
data      byte[]（业务数据，≤1MB）
stat      元数据（版本、zxid、时间、子节点数）
children  子节点引用列表
```

`stat` 字段必须能背下来，因为版本控制与并发判断全靠它：

| 字段 | 含义 |
| :--- | :--- |
| `czxid` | 创建该节点的事务 zxid |
| `mzxid` | 最后一次修改数据的事务 zxid |
| `pzxid` | 最后一次修改**子节点列表**的事务 zxid |
| `ctime` / `mtime` | 创建 / 修改时间 |
| `version` | 数据版本号，每次 `setData` +1 |
| `cversion` | 子节点版本号，子节点增删 +1（**顺序号的来源**） |
| `aversion` | ACL 版本号 |
| `ephemeralOwner` | 临时节点所属 sessionId；持久节点为 0 |
| `dataLength` | 数据长度 |
| `numChildren` | 子节点数量 |

内存结构在 `DataTree` 中：

```java
// 简化示意
private final ConcurrentHashMap<String, DataNode> nodes;
// DataNode 内部持有 byte[] data、StatPersisted stat、Set<String> children
```

**关键点**：整棵树常驻内存，磁盘只负责「事务日志 + 快照」的持久化。这解释了为什么 ZK 读极快、容量却很小。

### 3.2 会话机制

会话是 ZK 感知客户端存活的唯一手段，也是临时节点生命周期的锚点。

**sessionId 的构成**（64 位）：

```text
高 8 位   ：会话超时时间对应的 tick 数（用于分桶）
中 40 位  ：Leader 的 epoch / 机器标识
低 16 位  ：自增计数器
```

服务端 `SessionTrackerImpl` 用**分桶**管理超时：把所有会话按「下次到期 tick」放进 `HashMap<Long, HashSet<Long>>`，一个后台线程按 tick 推进，只处理当前桶，避免每次扫描全量会话。

客户端侧 `ClientCnxn` 在 `sessionTimeout * 2/3` 时发送 `PING`；服务端收到 `PING` 会 `touchSession`，把会话移到更晚的桶。

会话过期的处理链路：

```text
SessionTracker 发现超时
  → 生成 EXPIRED 事件
  → DataTree.killSession(sessionId)
      → 遍历 ephemeralOwner == sessionId 的节点
      → 逐个删除，并触发这些路径上的 Watcher
  → 服务端向客户端发送 SessionExpired
```

**生产含义**：客户端发生长时间 Full GC（超过 sessionTimeout）时，服务端会认为会话已死并删除临时节点；GC 恢复后客户端收到 `SessionExpired`，必须重建会话并重新注册所有节点与 Watcher。这是「ZK 注册中心抖动导致服务列表瞬间清空」的常见根因。

### 3.3 请求处理链

**读请求**：由本机直接处理（`FinalRequestProcessor` → `DataTree.getData`），不经过 Leader。因此 **Follower 可能读到落后于 Leader 的数据**；需要强读时调用 `sync()` 把当前节点与 Leader 同步后再读。

**写请求**（集群模式）：

```text
Client
  → 任意节点（若为 Follower，转发给 Leader）
Leader
  → PrepRequestProcessor      校验会话、ACL、节点状态，生成事务头与事务体
  → ProposalRequestProcessor  封装 Proposal
  → 广播 PROPOSE 给所有 Follower
Follower
  → 写入事务日志（FileTxnLog.append + force 落盘）
  → 返回 ACK
Leader
  → 收到过半 ACK（含自己）
  → CommitProcessor 按 zxid 顺序放行
  → FinalRequestProcessor 应用到 DataTree，触发 Watcher
  → 广播 COMMIT
```

三个必须能说清的点：

1. **提交条件是「过半 Follower 落盘 ACK」**，不是「过半收到」——落盘才保证崩溃恢复后不丢。
2. **`CommitProcessor` 保证按 zxid 顺序提交**，这是 ZAB 原子广播顺序性的落点。
3. **Follower 的读不参与 quorum**，所以 ZK 是「写强一致、读可能陈旧」。

### 3.4 ZAB 协议

ZAB（ZooKeeper Atomic Broadcast）包含两种模式，对应 `QuorumPeer` 状态机的不同状态：

```java
// QuorumPeer.run() 简化
while (running) {
    switch (getPeerState()) {
        case LOOKING:   // 崩溃恢复：选举 Leader
            setCurrentVote(makeLEStrategy().lookForLeader());
            break;
        case LEADING:   // 消息广播：本节点是 Leader
            setLeader(makeLeader());
            break;
        case FOLLOWING: // 消息广播：本节点是 Follower
            setFollower(makeFollower());
            break;
        case OBSERVING: // 只同步，不投票
            setObserver(makeObserver());
            break;
    }
}
```

**① 消息广播**：类似二阶段提交，但砍掉了 2PC 的「事务中断」逻辑，只要求**过半 Follower 落盘 ACK** 即可 Commit。因为写请求由 Leader 串行分配 zxid，天然有序，不需要协调者处理冲突。

**② 崩溃恢复**：Leader 失联或启动时进入 `LOOKING`，完成两件事——**选新 Leader**、**数据同步**。目标是：已提交事务不丢，未提交事务丢弃。

**zxid 结构**：

```text
64 bit = [ 高 32 位 epoch | 低 32 位 counter ]
epoch    ：Leader 任期号，换 Leader 递增
counter  ：该任期内的提议序号，单调递增
```

因此比较两个 zxid 就能判断「谁的数据更新」：先比 epoch，再比 counter。

**FastLeaderElection 投票比较顺序**：

```text
先比 epoch（任期大的优先）
  → epoch 相同比 zxid（数据新的优先）
    → 再相同比 serverId（myid 大的优先）
```

获得**过半票**的节点成为 Leader；少数派分区永远凑不够票，因此同一时刻至多一个有效 Leader，这是防脑裂的根本。

### 3.5 Watcher 机制

服务端 `WatchManager` 维护两个 Map：

```java
private final HashMap<String, Set<Watcher>> watchTable;   // path → watchers
private final HashMap<Watcher, Set<String>> watch2Paths;  // watcher → paths
```

`watch2Paths` 是反向索引，用于会话过期时按 Watcher 批量清理，避免遍历全表。

一次性的源码根源：

```java
public Set<Watcher> triggerWatch(String path, EventType type, Set<Watcher> suppress) {
    WatchedEvent e = new WatchedEvent(type, KeeperState.SyncConnected, path);
    HashSet<Watcher> watchers;
    synchronized (this) {
        watchers = watchTable.remove(path);   // 注意：remove，不是 get
        if (watchers == null || watchers.isEmpty()) return null;
        for (Watcher w : watchers) {
            Set<String> paths = watch2Paths.get(w);
            if (paths != null) paths.remove(path);
        }
    }
    for (Watcher w : watchers) w.process(e);
    return watchers;
}
```

`remove` 发生在通知**之前**，所以通知到达客户端时，服务端的注册关系已经不存在了。客户端如果想继续监听，必须在回调里重新 `exists` / `getData` / `getChildren`。

**事件矩阵**（注册方式决定能收到什么事件）：

| 注册方式 | 能监听的事件 |
| :--- | :--- |
| `exists` | 节点创建、删除、数据变更 |
| `getData` | 节点数据变更、删除 |
| `getChildren` | 子节点新增、删除（不包含子节点数据变更） |

客户端侧的 `ClientCnxn` 有两个队列：`outgoingQueue`（待发送请求）与 `pendingQueue`（已发送待响应）。Watcher 回调在**事件线程**执行，因此回调里不要做耗时操作，否则会阻塞后续事件。

### 3.6 节点类型的实现落点

| 类型 | 实现机制 | 源码落点 |
| :--- | :--- | :--- |
| 临时节点 | `stat.ephemeralOwner = sessionId`，会话过期时按 owner 批量删除 | `DataTree.killSession` |
| 顺序节点 | 父节点 `cversion` 自增，拼成 `%010d` 后缀 | `PrepRequestProcessor` 生成路径 |
| 容器节点 | 子节点为空时异步清理 | `ContainerManager` 定期扫描 |
| TTL 节点 | 到期且无子节点时删除 | `EphemeralType` + `ContainerManager` |

### 3.7 过半机制与脑裂

- quorum = `n/2 + 1`。3 台容 1 台，4 台仍只容 1 台，5 台容 2 台。
- 只有拿到过半票才能当 Leader；只有拿到过半 ACK 才能提交写。
- 网络分区时，少数派既选不出 Leader 也提交不了写，因此不会出现「双 Leader 同时接受写」。
- Observer 同步数据但不投票，扩容读能力而不抬高写 quorum 成本。

---

## 四、深入与高级延伸

### 4.1 写链路的存储实现

写请求在 Leader 上提交后，由 `FinalRequestProcessor` 调用 `DataTree.processTxn` 应用到内存，同时事务日志已由 `FileTxnLog` 落盘。

`FileTxnLog.append` 的三个细节：

1. **预分配**：日志文件按 `preAllocSize`（默认 64 MB）预分配，避免频繁扩容。
2. **`force` 落盘**：`commit` 时调用 `FileChannel.force(false)`，这是写延迟的主要来源。
3. **文件滚动**：单个日志文件写满后新建，文件名含起始 zxid，便于恢复时按序回放。

快照由 `FileSnap` 负责，采用**模糊快照**：序列化期间不暂停写，恢复时用「快照 + 之后的事务日志回放」得到最终状态。这避免了快照期间的长时间停顿。

### 4.2 崩溃恢复后的三种数据同步

新 Leader 选出后，按 Follower 与本机的差异选择同步方式：

| 模式 | 触发条件 | 行为 |
| :--- | :--- | :--- |
| DIFF | Follower 数据落后但仍在 Leader 日志范围内 | 增量补发缺失的提议 |
| SNAP | Follower 落后太多，日志已被清理 | 发送全量快照 |
| TRUNC | Follower 数据比 Leader 新（含未提交事务） | 截断多余部分再同步 |

这套「先对齐、再服务」的流程保证了「已提交不丢、未提交丢弃」。

### 4.3 性能与容量

| 维度 | 经验值 | 影响因素 |
| :--- | :--- | :--- |
| 读 QPS | 单机数万 | 纯内存读，受连接数与线程数限制 |
| 写 QPS | 单机数千，集群随节点数下降 | 过半落盘，磁盘 fsync 是瓶颈 |
| 单节点数据 | ≤1 MB（默认） | `jute.maxbuffer` |
| 总容量 | 受内存限制，通常 GB 级 | 全量数据常驻堆内 |
| 连接数 | 单机数千到一万 | `maxClientCnxns`、文件句柄 |

三个容易被忽略的成本：

- **写放大**：一次写要落到 quorum 个节点，集群越大写越慢。
- **Watcher 内存**：每个注册关系都占内存，百万级 Watcher 会显著推高 GC 压力。
- **快照开销**：数据量越大，SNAP 同步越慢，新节点加入时间越长。

### 4.4 生产高频踩坑

| 现象 | 根因 | 处理 |
| :--- | :--- | :--- |
| 服务列表瞬间清空 | 客户端 Full GC 超过 sessionTimeout，会话被判死 | 调大 sessionTimeout、优化 GC、减少单次停顿 |
| 选主反复抖动 | 磁盘 IO 打满，Follower 落盘 ACK 超时 | 日志独立磁盘、SSD、降低写入量 |
| 磁盘被写满 | 未配置 `autopurge` | 配置快照保留数与清理周期，监控磁盘 |
| `ConnectionLossException` 后误判节点不存在 | 连接丢失是**不确定状态** | 捕获后重试，不要当作「节点已删除」 |
| 惊群导致 ZK 被打爆 | 所有客户端监听同一个节点 | 改为监听前驱（临时顺序锁） |
| 会话过期后 Watcher 全部失效 | 会话级 Watcher 随会话销毁 | 在 `SyncConnected` 回调里重建注册 |
| 新节点加入同步极慢 | 快照过大、日志回放多 | 控制数据量、定期清理、必要时手工快照 |
| 升级后四字命令不可用 | 3.5+ 默认白名单限制 | 配置 `4lw.commands.whitelist` |

### 4.5 注册中心横向选型

| 维度 | ZooKeeper | Eureka | Nacos | etcd |
| :--- | :--- | :--- | :--- | :--- |
| CAP | CP | AP | AP/CP 可切换 | CP |
| 一致性协议 | ZAB | 异步复制 | Distro（AP）/ JRaft（CP） | Raft |
| 健康检查 | 会话 + 临时节点 | 客户端心跳 | 心跳 / 服务端探测 | Lease |
| 配置中心 | 弱（需自建 watch 封装） | 无 | 原生支持 | 需配合 |
| 大规模表现 | ZNode 与 Watcher 压力大 | 较好 | 较好 | 较好 |
| 典型使用者 | Dubbo 2.x | Spring Cloud Netflix | Spring Cloud Alibaba | Kubernetes |

**Dubbo 3 的演进**：从「接口级服务发现」升级为「应用级服务发现」（`register-mode: instance`），注册单位从「每个接口一个 ZNode」变成「每个应用实例一条记录」，大幅减少 ZNode 数量与 Watcher 压力。这是 ZK 在大规模集群下被诟病后的直接优化。

### 4.6 与 etcd 的关键差异

| 维度 | ZooKeeper | etcd |
| :--- | :--- | :--- |
| 协议 | ZAB（主备 + 原子广播） | Raft（任期 + 日志复制） |
| 数据模型 | 树形 ZNode | 扁平 key + 前缀范围 |
| 版本控制 | version / cversion | MVCC revision（全局单调） |
| 监听 | 一次性 Watcher | 长期 Watch（流式、可续订） |
| 典型场景 | Dubbo 注册中心、Hadoop 生态 | Kubernetes、服务发现、配置 |

etcd 的「长期 Watch」在设计上避免了 ZK「一次性 Watcher + 重新注册」的丢失窗口，这也是云原生体系更偏好 etcd 的原因之一。

---

## 五、总结整理

### 5.1 全景回顾

```text
                    ┌──────────── 客户端 ────────────┐
                    │ ClientCnxn：心跳/请求队列/Watcher │
                    └───────────────┬────────────────┘
                                    │ 会话
        ┌───────────────────────────▼───────────────────────────┐
        │                      ZK 集群（quorum）                 │
        │  Leader ──PROPOSE──► Follower ──ACK──► Leader Commit   │
        │     │                    │                             │
        │     └── 事务日志 + 快照（DataTree 常驻内存）             │
        └───────────────────────────────────────────────────────┘
                                    │
        四类协调原语：注册发现 / 配置 / 分布式锁 / 选主
```

### 5.2 速查清单

| # | 问题 | 一句话答案 |
| :--- | :--- | :--- |
| 1 | ZK 是什么 | 强一致、小容量、带一次性通知的树形协调服务 |
| 2 | ZAB 两种模式 | 消息广播（过半 ACK 提交）、崩溃恢复（选主 + 同步） |
| 3 | zxid 结构 | 高 32 位 epoch，低 32 位自增计数 |
| 4 | Watcher 为什么一次性 | `triggerWatch` 用 `remove` 删除注册关系，通知前已失效 |
| 5 | 临时节点为什么不能有子节点 | 会话过期的级联删除会带来复杂同步问题，服务端直接拒绝 |
| 6 | 分布式锁为什么用临时顺序节点 | 临时防死锁、顺序保公平、只听前驱防惊群 |
| 7 | 为什么部署奇数台 | 过半机制下容错能力相同，奇数台更省资源 |
| 8 | 如何防脑裂 | 过半投票 + 过半 ACK，少数派选不出 Leader 也提交不了写 |
| 9 | 读能保证最新吗 | 不能，Follower 可能落后；强读需 `sync()` |
| 10 | 会话过期后临时节点怎样 | `killSession` 按 `ephemeralOwner` 批量删除并触发 Watcher |
| 11 | 容器节点与持久节点区别 | 容器节点子节点为空时自动清理 |
| 12 | 顺序号会重复吗 | 同父节点下由 `cversion` 保证唯一，不同父节点独立 |
| 13 | ZK 能存多大 | 单节点默认 1 MB，总量受内存限制 |
| 14 | 注册中心挂了还能调用吗 | 能，Consumer 有本地缓存；但无法感知变更 |
| 15 | 和 etcd 最大区别 | Watcher 一次性 vs 长期 Watch；树形 vs 扁平 + MVCC |

### 5.3 三条追问链

**追问链 1：Watcher 一次性 → 怎么保证不丢事件**

```text
Watcher 为什么是一次性的？
  → 服务端 triggerWatch 里是 remove 而不是 get
  → 那客户端怎么持续监听？
    → Curator 的 ChildListener 在回调后自动重新注册
    → 重新注册期间发生变更怎么办？
      → 先重新注册再拉数据（注册与拉取顺序决定是否丢事件）
      → 更稳的做法：注册 + 全量拉取 + 版本号比对，容忍重复但不丢
```

**追问链 2：ZK 是 CP → 为什么 Dubbo 还能高可用**

```text
ZK 选主期间不可写，Dubbo 为什么还能调用？
  → Consumer 的 RegistryDirectory 持有本地 Invoker 列表
  → 注册中心不可用时 notify 不触发，但旧列表不清空
  → 路由到已挂节点怎么办？
    → FailoverClusterInvoker 在剩余 Invoker 中重试
  → 所以 CP 保证的是「注册数据一致」，不是「调用实时可用」
```

**追问链 3：写性能上不去 → 怎么定位**

```text
写延迟高先看什么？
  → mntr 的 zk_avg_latency / zk_outstanding_requests
  → 再看磁盘：事务日志是否与快照同盘、fsync 是否慢
  → 再看 quorum：节点数是否过多、是否有节点网络抖动拖慢 ACK
  → 最后看写入模式：是否存在大量临时节点创建/删除（会话抖动）
```

### 5.4 决策与排查速查

| 场景 | 判断 | 动作 |
| :--- | :--- | :--- |
| 选型：注册中心 | 已有 Dubbo 2.x + 强一致诉求 | ZK；新项目优先 Nacos |
| 选型：配置中心 | 需要灰度、推送、命名空间 | 不用 ZK，选 Nacos/Apollo |
| 选型：分布式锁 | 并发量中等、要求公平 | 临时顺序节点；高并发优先 Redis |
| 排查：连接抖动 | 看客户端 GC 日志与 sessionTimeout | 调大超时、优化 GC |
| 排查：写入变慢 | `mntr` + 磁盘 IO + quorum 节点状态 | 日志独立盘、减少写、剔除异常节点 |
| 排查：节点残留 | `ephemeralOwner` 是否为 0 | 改为临时节点；检查会话是否正常关闭 |
| 排查：磁盘增长 | `dataDir` / `dataLogDir` 大小 | 开 `autopurge`，定期清理 |

---

> 本文基线：Apache ZooKeeper 3.8.x（3.5+ 特性已标注）。
