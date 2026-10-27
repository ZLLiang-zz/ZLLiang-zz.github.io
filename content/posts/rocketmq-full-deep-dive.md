---
title: "RocketMQ 全量深度解析：从消息的诞生到高可用、性能与并发工程"
date: 2026-10-27T22:30:00+08:00
summary: "一篇把 RocketMQ 讲透的长文：核心概念、集群高可用、CommitLog 混写存储、四种消息类型的落盘原理、消费与 Rebalance、可靠性三层、性能与容量、线上排查、源码级链路、通信层、定时/顺序/Pop 消费，以及以 RocketMQ 为样本的 JUC 并发与内存工程。每个结论都带成立条件和代价，而不是背参数。"
categories: ["消息队列"]
tags: ["RocketMQ", "CommitLog", "ConsumeQueue", "事务消息", "延时消息", "顺序消息", "DLedger", "NameServer", "JUC"]
author: ZLLiang
ShowToc: true
TocOpen: false
draft: false
---

一条消息从 `producer.send(msg)` 到消费者的 `onMessage`，中间到底发生了什么？大多数人能说出"经过 NameServer 拿路由、写到 Broker、消费者拉取"这条主线，但往下再问一层就卡住了：路由为什么是最终一致而不是强一致？所有 Topic 的消息为什么要混写进同一个文件？事务消息和延时消息在 Broker 里到底"变成了什么"？"同步刷盘 + 同步复制"为什么还不是绝对不丢？

这篇文章把 RocketMQ 从概念到源码链路、从可靠性到并发工程全量展开。默认以 4.x 为主线，5.x 差异单独标注。目标不是让你"会背参数"，而是能说清每个设计背后的**为什么**，以及它付出的**代价**。

## 一、核心概念：五个角色和一个关键约束

| 概念 | 层级 | 物理存在 | 作用 |
| :--- | :--- | :--- | :--- |
| NameServer | 路由层 | 是（进程） | 管理 Broker 注册与 Topic 路由，多节点无状态互不通信 |
| Broker | 物理层 | 是（进程） | 存储、投递消息，主从同步 |
| Topic | 逻辑层 | 否 | 消息一级分类，可跨多个 Broker |
| Queue | 存储层 | 是（文件） | 消息物理分区，并行消费最小单元 |
| Producer / Consumer | 客户端 | 是（进程） | 发送 / 消费 |

**Topic 是逻辑分类，Queue 是物理分区**。一个 Topic 的 Queue 尽量均匀分布在多个 Broker Master 上。每个 Queue 对应一个 `ConsumeQueue` 索引文件，消息实体存储在 `CommitLog`（读写分离设计）。

**为什么 Queue 是并行度的最小单元，而不是消息本身？** 因为消费位点（Offset）以 Queue 为单位管理。如果同一 Queue 被两个实例消费，位点提交会互相覆盖，导致消息丢失或重复。这是"分区独占"模型的根本约束，Kafka 的 Partition 同理。由此推出一条铁律：**消费者实例数不应超过 Queue 总数**，否则多出的实例空转。

Queue 数在创建 Topic 时显式指定（`mqadmin updateTopic -t X -r 8 -w 8`），读写队列数可以不同——`rw` 分离主要用于缩容时"先减写队列、待存量消费完再减读队列"的平滑下线，**正常情况下两者必须相等**。生产建议关闭自动建 Topic（`autoCreateTopicEnable=false`），避免路由漂移和垃圾 Topic；普通消息 Queue 数取消费者实例数的 2~3 倍，顺序消息按业务键基数预留 32 或 64。

5.x 引入了三个概念变化：**Proxy 层**（LOCAL/CLUSTER 两种模式，客户端走 gRPC，协议与存储解耦）、**Pop 消费**（允许组内多消费者"共享"同一 Queue，从根上解决"消费者数 ≤ Queue 数"）、**Topic 类型**（显式区分 NORMAL / FIFO / DELAY / TRANSACTION）。

## 二、集群架构与高可用

### 2.1 NameServer 为什么无状态、互不通信

这是 RocketMQ 与 Kafka 早期（依赖 ZooKeeper）最根本的架构分野，答案是 **AP 换 CP**：

1. ZooKeeper 是 CP 系统，选主抖动会拖垮整个集群的元数据可用性；RocketMQ 用 AP 思路，**路由信息短暂不一致可容忍**。
2. 路由最终一致：Broker 每 30s 心跳注册，NameServer 每 10s 扫描、剔除 120s 未心跳的 Broker；客户端每 30s 拉取路由刷新本地缓存。
3. **代价**：路由删除有最长约 2 分钟的延迟窗口，期间生产者可能向已宕机的 Broker 发消息——由客户端的**故障规避（MQFaultStrategy）**兜底。这正是"无状态路由"和"客户端故障规避"两个设计的呼应点。

### 2.2 Broker 部署模式对比

| 模式 | 可用性 | 消息不丢 | 适用 |
| :--- | :--- | :--- | :--- |
| 单 Master | 宕机不可用 | 风险大 | 开发测试 |
| 多 Master | 单机宕机该机 Queue 不可写 | 有风险 | 日志类、可容忍丢失 |
| 多 Master 多 Slave（异步复制） | Master 宕机可读不可写 | 异步复制有少量丢失窗口 | 通用业务（最常见） |
| 多 Master 多 Slave（同步双写） | 高 | 基本不丢 | 金融、交易类 |
| DLedger（Raft） | 自动切换 | 不丢 | 4.5+ 无人值守高可用 |
| 5.x Controller 模式 | 自动切换 | 可配同步复制 | 5.x 推荐 |

两个高级细节：

- **同步刷盘 + 同步复制**才是金融级"不丢"组合，任一环节缺失都存在丢失窗口。
- 4.x 的 Master 宕机后，Slave 只能提供读服务（`slaveReadEnable=true` 时），**不能接管写入**；DLedger / 5.x Controller 才能自动选主切换写入。这是 4.x 与 5.x 高可用的本质区别。DLedger 基于 Raft，写入需过半节点确认，代价是吞吐下降约 30%~40%。

## 三、存储机制：为什么"混写 + 索引"

### 3.1 四个核心文件

| 文件 | 作用 |
| :--- | :--- |
| CommitLog | 消息物理存储，所有 Topic 混写，顺序追加，单文件默认 1GB |
| ConsumeQueue | 逻辑队列索引，每条 20 字节（Offset 8B + Size 4B + TagHash 8B），定长条目，定位 O(1) |
| IndexFile | 消息索引，支持 Key/时间区间查询，哈希冲突用链表法 |
| checkpoint | 刷盘与恢复进度 |

**为什么 CommitLog 混写而不是像 Kafka 按 Partition 分文件？**

1. **写路径极致简化**：所有 Topic 的消息顺序追加到同一个 CommitLog，磁盘始终是纯顺序写，Topic/队列数增多不会产生随机写。Kafka 每个 Partition 各自一个日志文件，分区数过大时写盘变散、页缓存竞争加剧。
2. **代价是读路径多一次间接寻址**：消费者先查 ConsumeQueue 拿 CommitLog 偏移，再读消息体。RocketMQ 用 20B 定长条目把这一步压到极低开销。
3. 结论：**适合"多业务共享一个集群"的公司内部场景**；Kafka 更适合"少数超大吞吐 Pipeline"。

### 3.2 刷盘与 mmap

**同步刷盘（SYNC_FLUSH）不是逐条 fsync**，而是 `GroupCommitService` 把同组请求挂到写队列，每次刷盘批量唤醒等待线程。异步刷盘（默认）写入页缓存即返回，由后台线程批量刷。

**mmap 与零拷贝的真相**：RocketMQ 存储读写都围绕 `MappedByteBuffer`（mmap）展开，但**不是严格意义的 sendfile 零拷贝**。写路径用 mmap 映射页缓存由 OS 异步刷盘；读路径消费者直接读 mmap 页缓存（热数据命中 Page Cache），但**从 Broker 到网卡仍有内核态到用户态的拷贝**——这与 Kafka 的 sendfile（数据不进用户态）相比，大流量场景 CPU 开销略高，这是 Kafka 极限吞吐普遍高于 RocketMQ 的原因之一。开启 `transientStorePoolEnable=true` 时先写堆外 DirectBuffer 再 commit 到 FileChannel，实现读写缓冲隔离。

**文件预热与锁页**（`warmMapedFileEnable=true`）：新建 MappedFile 时预写入 0 遍历整个文件，提前触发缺页中断；再 `mlock` 锁页 + `madvise(MADV_WILLNEED)` 提示内核预读。一句话："RocketMQ 通过预热+锁页，把 mmap 首次访问的缺页抖动从读路径消灭在写路径。"

## 四、消息发送：故障规避与重试的隐患

三种发送方式：**同步**（可重试、RT 最高）、**异步**（回调重试、吞吐高）、**One-way**（无保证、日志埋点）。生产端路由的核心是 `MQFaultStrategy.selectOneMessageQueue`：

```java
public MessageQueue selectOneMessageQueue(TopicPublishInfo tpInfo, String lastBrokerName) {
    if (sendLatencyFaultEnable) {
        int index = tpInfo.getSendWhichQueue().incrementAndGet();
        for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
            int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
            MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
            if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {
                if (lastBrokerName == null || !mq.getBrokerName().equals(lastBrokerName)) {
                    return mq;
                }
            }
        }
        return tpInfo.selectOneMessageQueue();   // 全部不可用强制选一个，宁可失败也不空转
    }
    return tpInfo.selectOneMessageQueue(lastBrokerName);
}
```

故障延迟机制用两张表：`latencyMax = {50,100,550,1000,2000,3000,15000}`（毫秒档位）和 `notAvailableDuration = {0,0,30000,60000,120000,180000,600000}`（隔离时长）。一次发送耗时落在哪档，对应 Broker 就被隔离相应时长（如耗时 >10s 隔离 10 分钟）。**生产建议开启 `sendLatencyFaultEnable=true`**。

**发送重试的三个隐患（高级必辨析）**：

1. **重试导致重复**：Broker 写入成功但响应超时，客户端重试导致同一消息写两次（不同 Offset）。幂等要从发送端（业务 Key）+ 消费端双保险。
2. **异步回调吞异常**：`onException` 不处理是线上消息"静默丢失"的常见根因。
3. **`retryAnotherBrokerWhenNotStoreOK=false`（默认）时**，Broker 返回 `FLUSH_DISK_TIMEOUT` / `SLAVE_NOT_AVAILABLE` 不会换 Broker 重试。大消息默认上限 4MB，超大报文应外置存储。

## 五、消息类型全景：CommitLog 里没有"类型"字段

这是全篇最容易答错、也最能拉开差距的部分。**第一个必须扭转的认知：CommitLog 里没有"类型"字段。** 定长头里只有 `QUEUEID`、`FLAG`、`SYSFLAG`、时间戳这些通用字段，没有任何一位表示"这是顺序消息"。

所谓"消息类型"，是下面四件事叠加出来的**行为结果**，而不是消息身上的标签：

| 手段 | 普通消息 | 顺序消息 | 延时消息 | 事务消息 |
| :--- | :--- | :--- | :--- | :--- |
| 物理落点 | 业务 Topic | 业务 Topic | 改写为 `SCHEDULE_TOPIC_XXXX`，到期改回 | 改写为 `RMQ_SYS_TRANS_HALF_TOPIC`，commit 后改回 |
| sysFlag 事务位 | 0 | 0 | 0 | `0x4` 半消息 / `0x8` commit / `0xC` rollback |
| 消息属性 | 业务自定义 | 无（靠发送时选队列） | `REAL_TOPIC`、`REAL_QID`、`DELAY` | `REAL_TOPIC`、`REAL_QID`、`TRAN_MSG` |
| 谁消费 | 并发监听器 | 顺序监听器 + 消费组 `order=true` | 并发 | 并发 |

由这个底座直接推出三条结论：

1. **为什么 4.x 能把普通和顺序消息发进同一个 Topic**——顺序的保证点在消费端（监听器 + 队列锁），不在消息上。
2. **为什么 5.x 一升级就报"类型不匹配"**——类型进了 Topic 元数据后就是强约束：`FIFO` Topic 收到不带 `MessageGroup` 的普通消息，服务端直接拒绝。5.x 把类型收敛为 Topic 的一等属性 `message.type`（NORMAL/FIFO/DELAY/TRANSACTION），且"类型必须一致、类型必须单一"。
3. **为什么延时/事务在 5.x 必须独立 Topic**——它们的落点是系统 Topic 要来回改写，与业务消息混在一个 Topic 会搅乱位点语义。

一句话记牢：**类型 = 写到哪 + 打了什么标记 + 谁按什么顺序消费**。而"重试消息 / 死信消息 / 消息过滤"都不是生产端主动选择的类型，是投递与订阅机制的产物，别把它们和四类 Topic 并列归类。

### 5.1 普通消息：两个 offset 和"写入成功 ≠ 可见"

普通消息的生命周期是"初始化 → 待消费 → 消费中 → 消费提交 → 消息删除"。物理形态里有两个 offset 必须分清，这是排查问题的分水岭：

| 字段 | 是什么 | 用在哪 |
| :--- | :--- | :--- |
| `PHYSICALOFFSET` | 消息在 CommitLog 的物理位置，全局唯一 | `queryMsgById`、事务回查、`offsetMsgId` 解析 |
| `QUEUEOFFSET` | 消息在 `Topic@QueueId` 逻辑队列里的第几条 | ConsumeQueue 下标、**消费位点的单位**、堆积计算 |

消费端的位点、堆积、重置，动的都是 `QUEUEOFFSET`；只有查消息体才用 `PHYSICALOFFSET`。

**写入成功 ≠ 对消费者可见**：`putMessage` 写 CommitLog 返回 `SEND_OK` 后，`ReputMessageService` 要异步追平 CommitLog 尾部、构建 ConsumeQueue 条目，消费者才能按 queueOffset 拉到。存在一个毫秒级窗口。反过来，异步刷盘下消息已对消费者可见（在 Page Cache）但尚未落盘，此时进程崩溃不丢（页缓存归 OS 管）、机器掉电会丢。两个窗口一个"先成功后可见"、一个"先可见后持久"，别混为一谈。

**重试消息是一条新消息**：消费失败后 Broker 重新投递到 `%RETRY%{consumerGroup}`，有新的 msgId、新的 CommitLog offset、`reconsumeTimes+1`。所以：① 用原 msgId 追不到重试记录，必须用 `UNIQ_KEY` 或业务 Key；② 幂等键不能取 msgId/offset，只能取业务唯一键；③ 堆积监控要同时看 `%RETRY%`。

### 5.2 顺序消息：三段式里哪一段最容易断

RocketMQ 的"顺序"是**分区顺序**，不是全局顺序：同一业务键的消息路由到同一 Queue，队列内串行消费。全局顺序几乎不用（读写队列数 = 1，吞吐极低）。

顺序消息不是"一个开关"，而是三段各自负责一件事，任何一段断掉都没有顺序：

| 段 | 靠什么保证 | 真实断点 |
| :--- | :--- | :--- |
| 生产端 | 同一业务键 hash 到同一 Queue | 路由列表变化 → `hash % mqs.size()` 漂移 → 同 key 换队列 |
| Broker 端 | 队列内 `queueOffset` 严格递增 | 几乎不断 |
| 消费端 | 队列锁 + 单线程 + 位点连续提交 | Rebalance 锁交接窗口、锁过期（60s 未续期） |

**为什么是"锁队列"而不是"消息级锁"**：顺序的单位是队列。锁队列才能保证"同一时刻只有一方推进这个队列"；消息级锁挡不住两个实例各拿一半并行处理——各自内部有序，合起来仍乱序。

**为什么位点必须连续提交**：位点是单调标量，提交到 N 就永远不会再投 N 之前的消息。并发消费那种"失败跳过、位点继续走"在顺序场景等于**丢弃**。所以顺序消费只能"整批成功才提交"，又反过来要求"整批串行"——三条约束互相锁死。

顺序消息的代价（必答）：单 Queue 串行吞吐受限、**失败固定 1000ms 原地重试并阻塞后续消息**、不支持广播消费、消费者数 ≤ Queue 数、Rebalance 期间不保证绝对不乱序（业务侧用状态机 / version 兜底）。

### 5.3 延时消息：本质是"先寄存、到期重写"

4.x 只有 **18 个固定延时级别**（1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h）。完整链路多数资料只讲到"塞进 SCHEDULE_TOPIC"就停了，漏掉最后的重写：

```text
① 生产端 setDelayTimeLevel(16)
② Broker 写 CommitLog 之前改写：REAL_TOPIC=原Topic、REAL_QID=原queueId、DELAY=级别，
   Topic=SCHEDULE_TOPIC_XXXX，queueId = delayLevel - 1   ← 级别与队列一一对应
③ ScheduleMessageService 每级一个定时任务扫描对应队列，读到期消息，恢复真实 Topic/queueId
④ 以一条"新消息"重新 putMessage 回 CommitLog → 新 PHYSICALOFFSET、新 QUEUEOFFSET
```

三个细节决定全部优缺点：

1. **为什么按"级别"分队列**：同级别内到期时间随 `storeTimestamp` 单调递增，所以"顺序扫 ConsumeQueue + 判队首是否到期"就够了，不需要堆或跳表。代价是队列数 = 级别数，级别只能固定。
2. **为什么业务 Topic 位点没有空洞**：到期消息是**重写**成新消息重新分配 queueOffset，业务消费者从没见过"还没到期的那条"。
3. **精度只能是"大概"**：级别是近似延迟桶，误差 = 定时器轮询间隔 + 扫描量。

4.x 的四个硬伤（5.x 重写的动机）：级别固定、级别间不排序、每级单线程调度（某级堆积拖慢该级全部消息）、精度粗。

**5.x 用三层结构换任意延迟**：CommitLog（消息本体，Topic 改写为 `TIMER_TOPIC`）+ TimerLog（定长索引：到期时间、CommitLog 偏移、长度、`prevPos`）+ TimerWheel（槽位数组）。`prevPos` 是**倒链且只追加**：新消息入槽时把"槽当前指向的头"写进新记录的 `prevPos`，再把槽指向新记录，旧记录一个字节不改——写入恒为 O(1) 顺序写盘。`round` 解决超一圈：一圈覆盖 `槽数 × 精度`，超出按圈数记账，`round == 0` 才投递。

**任何版本的延时消息都不是"原地等待的一条消息"，而是两条消息**（寄存的一条 + 投递的一条），存储与 IO 成本约为普通消息的 2 倍。限制：延时/事务消息单条 ≤ 64KB、不支持批量、事务消息不能设延时、不适合周期性任务（那是调度系统的职责）。

### 5.4 事务消息：commit 是"复制"不是"移动"

二阶段流程：半消息（消费者不可见）→ 执行本地事务 → Commit/Rollback → 未确认由 Broker 定时回查。半消息不是特殊消息，是**换了 Topic 的普通消息**：写入时 Topic 改为 `RMQ_SYS_TRANS_HALF_TOPIC`、queueId 固定 0（因为回查要"从头扫一遍未决消息"，单队列只需维护一个扫描位点）、属性带 `REAL_TOPIC/REAL_QID/TRAN_MSG`、sysFlag 置 `0x4`。

两个反直觉但生产上重要的结论：

1. **commit 是"复制"而不是"移动"**：消息被重写成新记录，所以事务消息磁盘占用约为普通消息的 2 倍——这也是"单条 ≤ 64KB"限制的现实背景。
2. **rollback 之后原文仍在盘上**：只在 OP Topic 写一条记录表示"已处理"。如果消息体里有身份证、卡号这类敏感信息，不能指望"回滚即删除"，合规场景消息体应只放引用 ID。

**为什么需要 OP 消息（`RMQ_SYS_TRANS_OP_HALF_TOPIC`）**：半消息自己无法表达"已被处理"——commit 后它变成了业务 Topic 里的另一条消息，rollback 后原文还在。所以每处理完一条半消息就往 OP Topic 写一条记录。回查时先读 OP 队列推进位点，已记账的部分整段跳过。

回查链路三个最容易被忽略的点：① 回查 RPC 打到的是同一个 producer group 的**任意实例**，`checkLocalTransaction` 依赖的状态必须来自共享存储（DB/Redis），放 JVM 内存必然判错；② 次数用尽后默认动作是**丢弃（等价回滚），不是提交**；③ 事务消息只覆盖"本地事务 + 发送"这一段，消费端仍是 at-least-once。

### 5.5 其余类型速览

- **批量消息**：同 Topic、同 `waitStoreMsgOK`、不支持延时、单批 ≤ 1MiB。**批量 ≠ 顺序**，失败整批重试会放大重复。
- **单向消息**：不保证送达、失败无感知，只用于日志埋点。
- **过滤**：Tag 走 ConsumeQueue 的 TagHash 两段式（Broker 粗筛 + 消费端精筛）；SQL92 需 `enablePropertyFilter=true`，Broker 端逐条解析，表达式求值异常时**默认过滤掉该消息**（"消息莫名消失"的排查点）。
- **重试/死信**：并发消费阶梯重试默认 16 次后进 `%DLQ%`；顺序消费固定 1000ms 原地重试不进 RETRY。死信必须独立消费，**禁止直接转回原 Topic**（死循环）。

## 六、消费机制：Push 的本质是长轮询 Pull

`DefaultMQPushConsumer` 并没有真正的服务端推送，底层是：`PullMessageService` 周期性拉取（`pullBatchSize` 默认 32）→ Broker 无新消息时**挂起请求**（`PullRequestHoldService`），有新消息或超时（约 5s）再唤醒返回 → 拉到本地 `ProcessQueue` 再交线程池消费。**兼顾推的低延迟与拉的流控能力**。

集群模式下"每个 Queue 只有一个消费者"靠三层机制：① **Rebalance 分配**（每 20s，默认平均分配 `AllocateMessageQueueAveragely`）；② **Broker 端队列锁**（主要用于顺序消费，60s 过期定期续期）；③ **位点集中管理**（存 Broker 的 `consumerOffset.json`）。

**流控参数**：`pullThresholdForQueue=1000`（单 Queue 缓存消息数）、`pullThresholdSizeForQueue=100MB`、`consumeConcurrentlyMaxSpan=2000`。触发流控时日志出现 `the consumer's request ... is not compliant`——排查消费慢的入口。

## 七、可靠性与一致性

### 7.1 全链路"不丢"方案

先把丢的物理机制说清，再谈配置：

| 阶段 | 丢的机制 | 保障手段 | 代价 |
| :--- | :--- | :--- | :--- |
| 生产端 | 异步回调异常被吞 / oneway 不等结果 | 同步发送或异步 + 回调落库 | RT 上升；重试放大重复 |
| Broker 页缓存 | 写 PageCache 即返回，断电丢 | `flushDiskType=SYNC_FLUSH` | 吞吐降 30%~50% |
| Broker 主从 | 异步复制下 Master 先宕机 | `brokerRole=SYNC_MASTER` 或 DLedger | RT 增加网络往返 |
| 消费端 | 先提交位点后处理、异常被吞 | 成功后再返回 `CONSUME_SUCCESS` | 重复消费，需幂等 |
| 存储老化 | 保留期（72h）内没消费 | 监控堆积斜率 | 磁盘成本 |

三个反直觉结论：① **同步刷盘 + 同步复制 ≠ 绝对不丢**（磁盘坏、机房没了还是会丢）；② **可靠性是链路的乘积，不是最强环节**；③ **"不丢"与"不重"不可同时免费获得**——要"不丢"必须允许重试重投，所以必然 at-least-once，要"不重"就得消费端幂等。

### 7.2 同步刷盘的真相：GroupCommit

`SYNC_FLUSH` 不是"每条消息一次 fsync"——那样 IOPS 直接被打死。RocketMQ 用 **GroupCommit** 把并发写入合并成一次刷盘：多个线程的刷盘请求进队列，刷盘线程一次 `fsync` 把所有已写数据落盘，然后批量唤醒所有等待者。所以并发越高单条消息摊薄成本越低（吞吐不崩），但单条消息 RT 至少是一个刷盘周期（最坏 10ms 级）。**这也解释了"为什么同步刷盘 TPS 还能到几万"——不是每条一次 fsync，而是一批一次。**

### 7.3 DLedger：用 Raft 替代主从复制

`SYNC_MASTER` 能缓解，但它是"主从 + 半同步"，没有任期、没有投票、没有日志一致性检查——**本质仍不是共识协议**。DLedger 把 CommitLog 写入改成 Raft 日志复制：写入发给 Leader → 并行发 Follower → 过半数确认才标记 committed → 返回成功。Leader 挂掉后按 term + log index 选举，只有日志最新的节点能当选。收益是**自动选主 + 强一致**，代价是吞吐略降、需 3/5 节点奇数部署。金融级"不允许丢"用 DLedger（或 5.x Controller），一般业务 `SYNC_MASTER` + 3 副本足够。

### 7.4 幂等的三层落地

"消费端做幂等"是空话，要说清怎么做：**数据库唯一索引**（业务唯一键建唯一索引，重复插入报 `DuplicateKey`）、**状态机**（只允许 `待支付 → 已支付`）、**去重表/Redis SETNX**（处理前先占位）。去重表三个坑：先占位再处理（失败删占位）、去重键要用业务唯一键而非 msgId、保留期要大于消息最大重试周期。

## 八、性能、容量与调优

单机参考：发送 **5w~10w TPS**（异步刷盘 + 异步复制，小消息），堆积能力 TB 级。**性能的本质是"一次顺序写、两次异步建索引"**：全局单文件顺序写（磁盘永远只有一个写指针前进）、索引异步构建（代价是"写入成功 ≠ 立刻可消费"）、读放大被 20B 索引消掉。

RocketMQ 的性能高度依赖空闲内存——**给 Broker 留足 PageCache，比调 JVM 参数更有效**。代价要说清：所有 Topic 共享 CommitLog，**单 Topic 热点会影响同 Broker 上的其他 Topic**；大消息（>4MB）会退化为"消息体单独存、CommitLog 存引用"，性能下降明显。

容量估算公式：

```text
日增原始数据 = 峰值 TPS × 平均消息大小 × 86400 × 压缩比倒数
日增存储     = 日增原始数据 × 副本数（通常 3）
所需磁盘     = 日增存储 × 保留天数 × 1.3（ConsumeQueue/IndexFile/预留）
```

两个容易漏的项：ConsumeQueue 约占 CommitLog 的 1/100（每条 20B），IndexFile 固定 40MB/文件；**堆积不是免费的**，消费停滞时磁盘按"写入速率 × 堆积时长"增长。

**压测三个常见错误结论**：① 用默认 1KB 消息压出"百万 TPS"——报 TPS 不报消息大小等于没报，要同时报 **TPS + MB/s + P99**；② 只看平均值不看 P99，毛刺来自刷盘、主从同步、PageCache 回收、GC、CommitLog 跨文件；③ 压测端成为瓶颈（看发送 RT 分布，若 RT 随并发线性上升而 Broker CPU 不高，瓶颈在压测端）。磁盘选型上，**不要用网络存储（NAS/NFS）**——RocketMQ 依赖 mmap 与 PageCache。

## 九、线上问题排查手册

**发送超时先分清四类根因**：

| 现象 | 根因 | 判据 |
| :--- | :--- | :--- |
| `sendDefaultImpl call timeout` | 客户端等不到响应 | 看 Broker 是否收到（`queryMsgById`） |
| `system busy, start flow control` | **PageCache 繁忙**：写入速度超过刷盘速度 | Broker 日志 `[PC_SYNCHRONIZED]` |
| `remoting too much request` | Broker 处理线程池 + 队列满 | `sendMessageThreadPoolNums` 太小 |
| `SLOW` / 长尾 RT | 刷盘或主从同步抖动 | 刷盘耗时 P99 |

**PageCache 繁忙的机制**（RocketMQ 特有的坑）：写入是"写 PageCache 就返回"，如果刷盘线程跟不上，PageCache 里堆积的脏页触发内核回写阻塞，`putMessage` 就被卡住——所以"磁盘没满、CPU 不高"照样 busy。**处置顺序**：先看磁盘 IO 利用率与 `await` → 确认是否同步刷盘/同步复制 → 最后才考虑调线程池（**调线程池通常是错的**，只会让更多请求堵在更前面）。

**消费 TPS 上不去，按五层定位**：并行度上限（Queue 数）→ 消费线程 → 单条耗时（慢 SQL/外部超时）→ 拉取流控 → Broker 侧。一个反直觉判据：消费者 CPU 不高、线程都在 `WAIT`，**不是线程不够，是下游慢**，加线程只会把下游打得更惨。顺序消费的 TPS 上限 = 单队列串行处理速度，加线程无效。

排查工具箱：`mqadmin consumerProgress -g 组`（看 `Diff` 堆积量，但要看每个 Queue 的 Diff 分布——单 Queue 堆积是倾斜，全 Queue 堆积是能力不足）、`queryMsgByKey`（依赖 IndexFile，发送时必须带 key）、`resetOffsetByTime`（重置前必须停消费者）。

## 十、源码级核心链路（4.x）

**文件地图**：生产者启动 `DefaultMQProducerImpl#start` → 发送 `sendDefaultImpl/sendKernelImpl` → Broker 接收 `SendMessageProcessor#processRequest` → 落盘 `CommitLog#putMessage` → 索引构建 `ReputMessageService#doReput` → 拉取 `PullMessageProcessor` → 位点 `RemoteBrokerOffsetStore#updateOffset` → 事务 `EndTransactionalProcessor` → Rebalance `RebalancePushImpl#doRebalance`。

**一个进程内复用 MQClientInstance**：`clientId = ip@instanceName`，同一 JVM 内多个 Producer/Consumer 共享 Netty 客户端、路由缓存、定时任务——生产上多数据源隔离靠 `instanceName` 区分。

**一条消息落盘**：`putMessage` 用 `putMessageLock`（默认自旋锁）写入 → `handleDiskFlush`（同步走 GroupCommitService）→ `handleHA`（同步复制走 GroupTransferService 等 Slave 上报 offset）。写入成功的消息此刻只存在于 CommitLog，消费者还看不到——索引由 Reput 线程异步补齐，这是存储与可见性的最终一致窗口。

**消费成功的位点提交是异步批量**（5s 一次），因此 Rebalance/宕机重启后最多重放 5s 内已消费消息——重复消费的又一个结构性来源。

## 十一、通信层：命令级线程池隔离

Broker 端线程模型：Netty Boss（只 Accept）→ Netty Worker（只编解码，不做业务）→ 按 RequestCode 隔离的业务线程池（`sendMessageThreadPool` / `pullMessageThreadPool` / `heartbeatThreadPool` / `adminBrokerThreadPool`...）。三大设计原则：**命令级隔离**（发送高峰不会饿死心跳/路由命令）、**EventLoop 不做业务**、**有界队列 + 快速失败**（打满后 `RejectedExecutionException` 转系统繁忙，而非无限排队）。

客户端所有"同步 API"在通信层都是"异步 + 阻塞等待"：`invokeSync` 用 `ResponseFuture` 里的 `CountDownLatch` 阻塞，这解释了为什么同步发送高并发下线程占用大——每个 in-flight 请求占用一个等待线程。

## 十二、定时消息全源码

4.x 的 `ScheduleMessageService` 有六个深挖点，其中最容易被漏的是 **`checkReputOffset`**：定时器读的是 `SCHEDULE_TOPIC` 的 ConsumeQueue，而它是 Reput 异步建出来的。如果不先确认"Reput 已追平到我要读的位置"就去读，会读到**还没建索引的空档**，误判为"该级别已投完"，把 `delayOffset.json` 进度写过头——结果就是一批消息永久滞留，直到重启才被发现。正确做法是读之前比对 `reputFromOffset` 与目标 offset，落后就等下一轮。

5.x 的 `TimerWheel + TimerLog` 用**两段式解耦**（写侧 `TimerEnqueueGetService` 追平 CommitLog 追加 TimerLog，读侧 `TimerDequeueGetService` 按精度推进指针），两侧用两个独立进度——入队只依赖 CommitLog 追平速度，投递慢不会反压生产。**写放大要算清楚**：延时消息 = 1 次 CommitLog 写 + 1 条 TimerLog 写（入队）+ 1 次 CommitLog 重写 + 1 条 ConsumeQueue 条目（到期投递），磁盘写入次数约 2 倍。把"所有超时关单"都塞延时消息是大促前最常见的容量误判。

## 十三、顺序消息与 Pop 消费

顺序消息消费端有三把锁：**Broker 队列锁**（防 Rebalance 窗口双实例）、**本地 messageQueueLock**（进程内消费线程与 Rebalance 互斥）、**ProcessQueue 状态锁**（队列被释放立即停止提交位点）。锁丢失时"这批消息消费完但不提交位点"，等重新锁定后从旧位点重投——所以顺序消息的重复消费窗口比并发消息更大，幂等更不可省。顺序消息扩容本质是**有损迁移**，只能把重复压到最小，不能消除。

**Pop 消费（5.x）**解决独占模式的三个问题：Queue 数 < 消费者数无法扩容、单 Queue 积压拖垮绑定实例、Rebalance 短暂暂停。机制是消费者**按需从任意 Queue "借取"消息**，通过 invisible 时间 + ACK 保证不丢不重；不丢的依据是每次 POP 在 revive topic 落一条检查点，invisible 超时后核对未 ACK 则重新投递——**用存储换语义**。代价是 revive topic 额外写放大、ACK 网络往返。

## 十四、深水区疑难问答（延伸思考）

**Q：RocketMQ 写入为什么在"多 Topic 场景"下比 Kafka 稳？** Kafka 按 Partition 分文件，分区数大了顺序写退化成"多文件交错写"；RocketMQ 全部 Topic 混写单一 CommitLog，无论多少 Topic 都是纯顺序追加。所以"多业务共享集群"选 RocketMQ，"少数大管道"选 Kafka。

**Q：要 0 丢失，架构上怎么搭？** 三层全同步：发送端同步 + 重试 + 回调不吞异常；Broker 同步刷盘 + 同步复制（或 DLedger）；消费端成功后再返回。再辅以 T+1 对账兜底。但要说明：全同步组合会把单机 TPS 打到很低（约几千），**业务要按消息等级分区部署**，不能全集群一刀切。

**Q：百万级堆积要求 1 小时消化怎么设计？** 先定位是不是 Queue 数限制并发。是 → 临时扩 Queue + 搬运 Topic（新 Topic 3 倍 Queue，写搬运消费者分发，注意顺序被破坏）；不是 → 消费端扩容 + 线程拉满 + 批量拉取 + 下游批量化。治理上堆积告警要配"消费速率斜率"而不只是存量。

**Q：DLedger 和 5.x Controller 差在哪？** DLedger 让 Raft 组接管 CommitLog 本身，写入需过半确认——不丢但吞吐降 30%+；5.x Controller 复用普通主从复制的存储路径，只在故障时由 Controller 仲裁选主——性能无损、可灰度。

**Q：为什么 RocketMQ 不做 Exactly-Once？** 端到端 EOS 要把"投递、处理、位点提交"绑成一个原子事务，代价是吞吐数量级下降。RocketMQ 选择 at-least-once + 业务幂等，以"便宜的重复"替代"昂贵的精确一次"。

## 附录：RocketMQ 里的并发与内存工程

RocketMQ 本身就是一本 JUC 活教材，几乎每个并发原语都能在源码里找到教科书级应用：

| JUC 组件 | 应用场景 | 选择理由 |
| :--- | :--- | :--- |
| `CountDownLatch` | 同步发送 `invokeSync` 阻塞等响应 | 把异步 Netty 回调"栅栏"成同步返回 |
| `Semaphore` | 异步/单向发送 in-flight 限流 | 许可证耗尽即快速失败，不阻塞调用线程 |
| `ConcurrentHashMap` | 路由表 / 响应表 / 消费组表 | 读多写少，JDK8 CAS+锁桶头，读无锁 |
| `AtomicLong/AtomicInteger` | 消息 Offset / 轮询队列序号 | CAS 无锁递增 |
| `ReentrantLock` | CommitLog 写入锁（可配） | 竞争激烈时比自旋省 CPU |

**CommitLog 默认为什么用自旋锁**：临界区 `appendMessage` 只是一次微秒级内存拷贝，线程切换代价远大于自旋几圈。但竞争激烈时自旋会空耗 CPU，此时 `useReentrantLockWhenPutMessage=true` 切 AQS 挂起等待。锁的选型看的是**临界区长度 × 竞争强度**，不是功能。

**三层内存模型**：① JVM 堆（元数据、网络缓冲）② JVM 堆外（`transientStorePool` 的 DirectBuffer 池）③ OS 页缓存（mmap 的 CommitLog/ConsumeQueue）。**核心认知：RocketMQ 的消息数据绝大多数不在 JVM 堆里，而在 OS 页缓存里**——这是它"堆不大却存 TB 级消息"的根本原因。所以 Broker 堆建议只开 4~8G，其余内存让给页缓存；堆太大反而和页缓存抢内存，触发 swap 更糟。

## 收口：一页纸记住 RocketMQ

RocketMQ 的核心是**"一写多读"架构**：Producer 顺序写 CommitLog，Broker 异步构建 ConsumeQueue 和 IndexFile，Consumer 通过 ConsumeQueue 定位 CommitLog 用 mmap 读取。无状态 NameServer 用 AP 换运维简单，主从/DLedger 换高可用，Queue 分布换水平扩展。

三条边界要记住：**消息类型是"写到哪 + 打什么标记 + 谁怎么消费"的组合**，延时与事务都是"寄存 + 重写"，磁盘占用双份；**可靠性是链路乘积**，同步刷盘 + 同步复制也挡不住磁盘坏；**"不丢"必然"至少一次"**，消费端幂等是标准动作，不是可选优化。

选型一句话：业务语义（事务 / 延迟 / 轨迹 / 过滤）重、运维简单选 RocketMQ；极限吞吐的日志流 / 大数据管道选 Kafka。
