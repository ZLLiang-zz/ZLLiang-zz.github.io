---
title: "Redis 全量深度解析：线程模型、数据结构、持久化、高可用与分布式工程"
date: 2026-10-27T22:40:00+08:00
summary: "一篇把 Redis 讲透的长文：为什么快（四层归因而非一句'内存+单线程'）、SDS/listpack/skiplist/dict 的底层实现、过期与淘汰、RDB/AOF 的 COW 与恢复陷阱、主从/哨兵/Cluster 的丢数据边界、缓存一致性、分布式锁的看门狗与 Redlock 争议、事务与 Lua、大 Key/热 Key、秒杀实战、事故复盘，以及 Java 客户端与序列化工程。每个结论带成立条件和代价。"
categories: ["Redis"]
tags: ["Redis", "持久化", "高可用", "分布式锁", "缓存一致性", "skiplist", "listpack", "Redisson", "大Key"]
author: ZLLiang
ShowToc: true
TocOpen: false
draft: false
---

"Redis 为什么快"大概是后端开发里被讨论得最多、也答得最浅的一个问题。多数人会脱口而出"内存存储 + 单线程 + IO 多路复用"，把三件事并列成三条原因。但如果你再往下问一句"那为什么 `KEYS *` 一个命令就能把整个库卡住"，很多人就接不上了——因为这三件事解决的是**不同瓶颈**，而它们**都不解决单条命令的 O(N) 问题**。

这篇文章把 Redis 从线程模型到底层数据结构、从持久化到高可用、从缓存一致性到分布式锁全量展开。默认以 6.x/7.x 为主线。每个结论都落到"为什么只能这么做"和"代价是什么"。

## 一、线程模型：为什么快，快在哪一层

Redis 为什么快，要按**四层归因**说清每一层解决什么：

| 层次 | 机制 | 量级/边界 |
| :--- | :--- | :--- |
| 存储层 | 纯内存 + jemalloc 分配器 | 读 100ns 级；jemalloc 按 size class 分桶减少碎片与锁竞争 |
| 执行层 | 单线程事件循环，命令串行 | 省掉锁与上下文切换；单核可跑满 10w QPS |
| 网络层 | epoll 多路复用 + 6.0 io-threads | 单线程管数万连接；io-threads 只并行读写与解析 |
| 结构层 | SDS / listpack / skiplist / dict 按场景特化 | 同一语义多种编码，阈值由 `xxx-max-listpack-entries` 控制 |

**"单线程"到底指什么**：Redis 6.x/7.x 实际有 6 类线程（主线程、io-threads、bio_close_file、bio_aof_fsync、bio_lazy_free、replica IO 线程），**只有"命令执行"是单线程**。AOF fsync 必须另开线程，因为 `fsync()` 是阻塞系统调用，放主线程会让所有命令卡住。

**6.0 的 io-threads 为什么不能连命令执行一起并行**：瓶颈在网络不在计算——单核跑 10w QPS 时 `read/write` 系统调用与协议解析占 60%+ CPU，真正的命令执行占比很低。io 线程只负责"把字节读进缓冲区、解析成 argc/argv"，`processCommand()` 始终在主线程串行执行。三个不能并行命令的理由：① 数据结构不是线程安全的（dict 渐进式 rehash、listpack 原地扩容都没加锁）；② 语义会被破坏（INCR/SETNX 的原子性来自单线程串行）；③ 收益不对等（命令本身是纳秒级内存操作）。参数建议 `io-threads = CPU 核数 - 1 且 ≤ 8`，**4 核以下、QPS < 5w 不要开**。

**命令的原子性边界**：单命令原子是因为命令在事件循环里从头执行到尾不切走，但 `GET 后 SET`（读-改-写竞态）、多条命令组合（中间可插入别的客户端）、单命令 O(N)（阻塞所有人）都不原子。最典型的误用是"GET + 判断 + SET 实现分布式锁"，必须用 `SET key val NX PX 30000` 一条命令同时完成"不存在才设置"和"设置过期"。注意 Pipeline 只是合并 RTT，**不保证原子**。

## 二、底层数据结构与编码

### 2.1 编码切换表

| 类型 | 底层编码 | 触发条件（默认） |
| :--- | :--- | :--- |
| String | `int` / `embstr` / `raw` | 整数 / ≤44B / >44B |
| List | `listpack`（7.0）/ `quicklist` | 超阈值转 quicklist |
| Hash | `listpack` / `hashtable` | 字段≤512 且值≤64B / 超出 |
| Set | `intset` / `listpack` / `hashtable` | 全整数≤512 / 小集合 / 超出 |
| ZSet | `listpack` / `skiplist` | ≤128 且值≤64B / 超出 |

版本演进：3.2 用 quicklist 取代 ziplist+linkedlist；**7.0 用 listpack 全面取代 ziplist**（因 ziplist 的 `prevlen` 连锁更新问题被彻底移除）。

### 2.2 SDS：二进制安全 + O(1) 长度 + 预分配

SDS 不以 `\0` 判断结尾（C 字符串存不了图片/序列化字节流），`len` 字段让 strlen 从 O(N) 降到 O(1)，预分配 + 惰性释放减少 realloc。5 种类型 `sdshdr5/8/16/32/64` 按长度选最小元数据。

**embstr 44 字节的推导**（常被深挖的细节）：`jemalloc 64B 块 = redisObject(16B) + sdshdr8头(3B) + buf(44B) + '\0'(1B)`，正好一个内存块零碎片；超过 44B 落不进 64B 块就转 raw（两次分配）。

### 2.3 listpack 如何从结构上消灭连锁更新

ziplist entry 是 `<prevlen><encoding><data>`，`prevlen` 记录前一节点长度，前节点变大触发 `prevlen` 扩容，可能**级联更新后续所有节点（最坏 O(N²)）**。listpack entry 是 `<encoding><content><backlen>`，只记录**自身**长度，删除了 prevlen 前向依赖。

### 2.4 dict 的渐进式 rehash

`dict` 有两个哈希表 `ht[0]/ht[1]` + `rehashidx`。渐进式 rehash 每次增删改查顺带迁移一个桶，配合 `serverCron` 定时批量迁移，**避免一次迁移百万 key 阻塞主线程**。rehash 期间**读查两表、写只进 ht[1]**。触发条件：负载因子 ≥1 扩容（BGSAVE 期间 ≥5 才扩，提高 COW 内存利用率），≤0.1 缩容。

### 2.5 skiplist 为什么不用红黑树

三个理由：**范围查询**（ZRANGE）跳表按层顺序遍历天然高效；**span 跨度**用于 `ZRANK` 快速算排名，红黑树难以直接支持；实现/调试更简单。ZSet 用"跳表 + 哈希表双结构"：跳表负责排序范围查询，哈希表负责 `ZSCORE` O(1) 点查——空间换时间的典型组合。

## 三、过期删除与内存淘汰

**惰性删除的源码坑在"从库"**：`expireIfNeeded` 里，从库（`server.masterhost != NULL`）**不主动删过期键**，只返回 nil（逻辑过期），物理内存要等主库的 `DEL` 同步过来才释放。原因：如果从库自己删，主库却认为键还在（时钟慢、键刚续期），主从就不一致了。**过期判定必须以主库为准**。后果是从库内存可能高于主库，这是正常现象，不是泄漏。

**定期删除是"随机采样 + 25% 阈值"**，不是全量扫描（全量是 O(N)，大实例一次几秒）。每次采样 20 个，过期比例 > 25% 就继续下一轮——用概率收敛代替确定性遍历，CPU 可控。两种模式分工：`FAST` 在 `beforeSleep`（每轮事件循环后，1ms 预算），`SLOW` 在 `serverCron`（100ms 一次，25% CPU 预算）。

**八种淘汰策略**：`noeviction`（默认，满了报 OOM）、`allkeys-lru/lfu`（缓存首选）、`volatile-lru/lfu/ttl/random`、`allkeys-random`。近似 LRU 不是全局链表，而是随机采样 5 个（`maxmemory-samples`）淘汰最久未访问的；LFU 复用 24bit（16bit 分钟级衰减时间 + 8bit 对数计数器），解决"历史热点永生"。**淘汰是同步阻塞的**（`performEvictions` 在主线程跑），大 value 密集场景会有命令延迟尖刺——这是"内存打满后 Redis 变慢"的真实原因。

`noeviction` 下不是所有写都失败：`DEL/UNLINK/EXPIRE` 等"减少内存"的命令仍可执行。

## 四、持久化：RDB 的 COW 与 AOF 的三层缓冲

### 4.1 RDB 最大的不确定性是 fork

RDB 用 `fork + COW`（写时复制）。**fork 复制的是页表，不是内存数据**：父子共享同一批物理页，页表项标记只读，写入时触发缺页中断复制该页。所以"COW 复制的量"取决于快照期间被修改的页数，不是实例总内存。

三个真实代价：① **fork 阻塞**（复制页表，10GB 实例约 100~300ms，大实例 + 虚拟化可能秒级）；② **内存膨胀**（写入量大时接近 2 倍）；③ CPU 抖动。**THP（透明大页）必须关**：默认 4KB 页，THP 把页放大到 2MB，一次 COW 要复制 2MB，膨胀与卡顿都被放大数十倍——`echo never > /sys/kernel/mm/transparent_hugepage/enabled`。

### 4.2 AOF 的三层缓冲

```
命令 → server.aof_buf（用户态内存）
     → write() → 内核 page cache（进程崩溃不丢，机器断电会丢）
     → fsync() → 磁盘（真正持久化）
```

| 策略 | fsync 时机 | 宕机丢失 | 性能 |
| :--- | :--- | :--- | :--- |
| `always` | 每轮事件循环立即 | 最多 1 条命令 | 最差 |
| `everysec` | bio 线程每秒一次 | 1~2 秒 | 好（推荐） |
| `no` | 交给 OS（约 30s） | 可达 30 秒 | 最好 |

`everysec` 的精确语义是"最多丢 2 秒"：上次 fsync 距今超 2 秒时主线程会阻塞等待（`aof_delayed_fsync` 计数）。**AOF 重写期间双写**（AOF 缓冲区 + AOF 重写缓冲区），子进程完成后主线程将重写缓冲区增量原子追加并 rename——7.0 的 Multi-part AOF 分离 base/incr/manifest 三文件，消除了双写的内存峰值。

### 4.3 恢复流程的四个陷阱

启动加载优先级：**开启 AOF 且文件存在 → 只加载 AOF（忽略 RDB）**；否则加载 RDB。四个陷阱：① 以为 RDB+AOF 会合并加载（不会，AOF 优先）；② AOF 尾部截断（`aof-load-truncated` 默认截断继续，恢复生产建议临时设 `no`）；③ 无盘复制 + `rdb-del-sync-files yes` 会删掉本地 RDB（承担冷备职责时自断后路）；④ 恢复速度被误判（RDB 是解析二进制，AOF 是逐条重放，后者慢 2~5 倍，切主/扩容要算进不可用窗口）。

## 五、高可用：主从、哨兵、Cluster 的丢数据边界

### 5.1 主从复制与 backlog

2.8+ 用 `PSYNC`（replication id + offset）替代 `SYNC`：网络抖动重连后，若 offset 仍在 `repl_backlog` 环形缓冲区内则**增量复制**，否则退化全量。**backlog 大小计算：`≥ 平均写入速率 × 断线最大时长 × 2`**，默认 1MB 在高写入实例上远远不够——这是"偶发全量复制风暴"的常见根因。例：峰值 50MB/s、容忍 30 秒断线 → 至少 3GB。

### 5.2 哨兵的 quorum 与 majority 之别

这是最常见的混淆点：**quorum 决定客观下线（ODOWN）**——认为 SDOWN 的 Sentinel 数 ≥ quorum 即标记；**majority 决定选举 Leader**——必须获超过半数 Sentinel 投票（> N/2）。5 个 Sentinel 配 `quorum=2` 时，2 个就能判 ODOWN，但选 Leader 仍要 3 票。**quorum 调小只会让 ODOWN 更容易，不会让选举更容易**——这是"配了 quorum=1 却发现切换很慢"的原因。

**脑裂防护**：`min-replicas-to-write 1` + `min-replicas-max-lag 10`，旧主被分区隔离后因"看不到健康从库"拒绝写入，避免双主分叉。

### 5.3 Cluster 的槽位与迁移

16384 个哈希槽，`CRC16(key) % 16384`，Hash Tag `{user1001}.order` 强制同槽。**为什么是 16384 而不是 65536**：心跳包携带槽位图，16384 = 2KB bitmap，65536 = 8KB 过大；且设计上限约 1000 节点，16384 足够。

**MOVED 与 ASK 的分界**：MOVED = 槽已永久迁移，更新客户端槽表；ASK = 迁移中，本次重定向到新节点但**不更新槽表**（因为同一槽里"一部分 key 在新节点、一部分还在旧节点"，客户端必须**先发 ASKING 再发命令**）。`MIGRATE` 是同步阻塞的，所以 reshard 都按小批量迁移。

### 5.4 故障转移到底会丢多少数据

**必须承认：Redis 主从是异步复制**，主库写完就回客户端，不等从库确认。丢失窗口由三段时间叠加：复制延迟（<1ms 到秒级）+ 故障发现（`down-after-milliseconds` 默认 30s）+ 选举晋升（秒级）。**真正决定丢失量的是第一段复制延迟**，所以监控 `master_repl_offset - slave_repl_offset` 比调小超时更有意义。四种缓解各有代价：`WAIT numreplicas timeout`（每次写多一个 RTT，且不保证切换后不丢）、`min-replicas-to-write`（分区期间主库不可写）、业务对账、换 Raft 系存储（吞吐降一个量级）。

**一句话结论：Redis 的主从切换是高可用方案，不是不丢数据方案。** 要强一致就别用 Redis 存核心账务。

## 六、缓存三大问题与一致性

穿透（查不存在数据直击 DB）→ 布隆过滤器 + 空值缓存；击穿（热点 key 过期并发重建）→ 互斥锁 + 逻辑过期 + 永不过期；雪崩（大量 key 同时过期/宕机）→ TTL 随机打散 + 高可用 + 限流降级。

**缓存一致性为什么是 Cache Aside（先更 DB 再删缓存）**：删缓存而非更新缓存（并发写时更新缓存存在乱序覆盖）；先 DB 后删而非先删后更（先删后更有"旧值回填"长窗口，先更后删窗口极小）。仍不彻底，两级兜底：**延迟双删** + **Binlog 订阅（Canal）异步删缓存**（以 DB 为唯一事实源）。一致性本质是**最终一致**，追求强一致就不该引入缓存。

## 七、分布式锁：从 SET NX 到 Redlock 的取舍

`SET NX PX` 三个经典缺陷：锁过期业务未完成、误删他人锁（校验 + 删除非原子，需 Lua）、主从切换锁丢失（客户端 A 在主节点加锁，主宕机锁未同步，新主上 B 再次加锁成功——双持锁）。

**Redisson 看门狗**：默认锁过期 30s（`lockWatchdogTimeout`），续期任务**每 1/3 过期时间（10s）**检查持锁状态续期回 30s。为什么是 1/3 不是 1/2——留两次重试机会。**一旦显式传了 leaseTime，看门狗失效**（代码评审高频问题）。存储结构用 **Hash**（`{clientId:threadId : 重入次数}`）而非 String，天然支持可重入。

**Redlock 的争议**：Martin Kleppmann 批评其依赖时钟同步、缺 fencing token；antirez 回应可用单调时钟近似。**工程结论：绝大多数业务单实例/哨兵 + Redisson + 业务幂等已足够**；强正确性（资金、库存超卖）应使用支持 fencing 的存储（ZooKeeper/etcd）或数据库乐观锁。**Redis 锁是"效率锁"不是"正确性锁"**——判断标准很简单：丢锁会不会导致钱算错？会，就别只靠 Redis 锁。

## 八、事务、Lua 与 Pipeline：原子性的三个层次

三者解决的问题完全不同：**事务解决"多条命令一起执行"，Lua 解决"复杂逻辑原子执行"，Pipeline 只解决"减少网络往返"**。

**Redis 事务为什么不支持回滚**：`MULTI` 只是打上 `CLIENT_MULTI` 标记，命令进 `mstate.commands` 数组，`EXEC` 才逐条执行。两类错误处理不同：**入队期错误**（命令不存在、参数错）→ `EXEC` 整体拒绝返回 `EXECABORT`；**执行期错误**（类型不匹配）→ 该条报错，其余照常执行，不回滚。不回滚的真实原因：回滚需要 undo 日志，内存与性能开销翻倍，与"简单、快"的定位冲突；且这类错误是编程错误，开发期就该发现。所以 Redis 事务的实际语义是**批量执行 + 隔离**，不是"原子提交/回滚"。

`WATCH` 是 Redis 版乐观锁（CAS）：登记"这个 key 被谁监视"，任何修改该 key 的命令把相关客户端标记 `CLIENT_DIRTY_CAS`，`EXEC` 前检查脏则放弃。但**没有版本号**——只能发现"被改过"，不能发现"改成了什么"。高并发下 CAS 失败率很高（秒杀场景不用 WATCH，用 Lua 或 `DECR`）。

**Lua 是真正的"原子执行复杂逻辑"**：脚本被当作一条命令在主线程执行。六个细节：`redis.call` 遇错中断、`redis.pcall` 捕获继续；超时不是"自动杀"（超 `lua-time-limit` 后只对其他请求回 BUSY，脚本不会被终止，写过的脚本 `SCRIPT KILL` 也杀不掉）；集群下脚本所有 key 必须同一槽；不用随机/时间命令写数据；7.0 的 Functions 解决脚本不持久化问题。

**Pipeline 三个必须纠正的认知**：不原子（服务端逐条执行可插队）；不是越大越好（一次发 10 万条会撑爆输出缓冲触发 `client-output-buffer-limit`，建议每批 500~1000）；可与事务组合（`MULTI...EXEC` 放进 Pipeline 既减 RTT 又原子）。性能量级：1 万次 SET 逐条受 RTT 限制（同机房 0.2ms → 2 秒），Pipeline 批 1000 条只需 10 个 RTT；跨机房 RTT 10ms 时差 100 倍。

选型口诀：**能用单命令就用单命令；需要"读-判断-写"用 Lua；只是批量发用 Pipeline；WATCH 只在冲突极少时用。**

## 九、高级应用场景

- **排行榜**：ZSet 跳表 O(logN) 插入与 span 排名；大规模只保留 Top N（`ZREMRANGEBYRANK` 裁剪）。
- **限流**：固定窗口（简单，有临界突刺）→ 滑动窗口 ZSet（精确，内存与请求量线性）→ 令牌桶 Lua（平滑，支持突发，网关级首选）。
- **消息队列**：List 简易队列无 ACK（消费失败即丢，慎用核心链路）；**Stream + 消费者组**有 ACK + 回溯，但无分级存储、堆积能力受内存限制，核心链路仍选专业 MQ。
- **分布式 ID**：`INCR` 简单高性能，但依赖 Redis 可用性；进阶号段模式（一次 `INCRBY 1000` 本地分配）或 Snowflake。
- **购物车**：Hash 天然适合（field=商品，value=数量）。

## 十、性能调优：大 Key 与热 Key 是两个问题

**先分清**：大 Key 是"单 key 体积/元素数过大"（靠**拆**），热 Key 是"单 key 访问频率过高"（靠**分散**）。大 Key 量化标准（经验值）：String > 10KB 关注、> 1MB 危险；集合元素 > 5000 关注、> 10 万或总量 > 100MB 危险。

**大 Key 的危害不是"慢"，是"连累所有人"**：`GET` 一个 10MB value 网络传输几十 ms，主线程期间不能处理其他命令；`DEL` 百万元素 Hash 是 O(N) 同步释放（**必须用 `UNLINK`**）；RDB/AOF 重写序列化大 key 拉长 COW 窗口；主从复制大 key 拉长从库 loading；Cluster 下大 key 导致分片倾斜。

**热 Key 治理阶梯**（成本从低到高）：本地缓存（Caffeine 扛 90% 读，短 TTL 或广播失效）→ 读写分离（读打多从库）→ key 打散（`hotkey:{0..9}` 存 10 份副本，读随机选，写写 10 份）→ 限流降级。慢查询高频元凶：`KEYS *`（生产禁用，用 SCAN）、大集合 `HGETALL/SMEMBERS`、大偏移量 `ZRANGE`、过长 Lua。内存碎片看 `mem_fragmentation_ratio > 1.5`（碎片多），**< 1 说明用 swap，更严重**。

## 十一、源码级核心链路

一条 `SET key value` 的旅程：客户端发 RESP → epoll 返回可读事件 → `readQueryFromClient`（6.0+ 解析可放 IO 线程）→ `processInputBuffer` 解析 argv → `lookupCommand` 从命令表 O(1) 找到 redisCommand → `call() → setCommand`（含 expire 判断）→ 传播 `feedAppendOnlyFile`（AOF）+ `replicationFeedSlaves`（主从）→ `addReply` 写输出缓冲 → `beforeSleep` 处理写回。**命令执行永远单线程，网络收发 6.0 起可并行**。

事件循环 `aeMain`：计算最近时间事件到期时间作为 `epoll_wait` 超时 → 处理文件事件 → 处理时间事件（serverCron）→ beforeSleep。**单线程能处理并发连接，靠的是"就绪通知"不是"线程数"**——epoll 返回的是就绪的 fd 列表，1 万连接里同一时刻可能只有几十个有数据。

## 十二、客户端选型与连接治理

| 维度 | Jedis | Lettuce | Redisson |
| :--- | :--- | :--- | :--- |
| IO 模型 | 阻塞 BIO | Netty NIO | Netty NIO |
| 线程安全 | 否（需连接池） | 是（单连接共享） | 是 |
| 定位 | 轻量直连 | Spring Boot 默认、通用首选 | 分布式锁/限流工具箱 |

**Lettuce 为什么单连接能扛高并发**：命令异步化 + 序号匹配（响应按序号对号入座，单连接上命令可以乱序返回）；线程安全来自"命令状态在业务线程创建、IO 读写全在 EventLoop 单线程"；Pipeline 天然支持。坑：阻塞命令（`BLPOP`）会阻塞该连接上的其他命令，必须用独立连接。**Jedis 为什么必须用连接池**：实例非线程安全，并发 N 线程需要 N 个实例，连接数上限 = 并发上限。三者可共存：Redis 命令走 Lettuce，分布式协调走 Redisson。

## 十三、多级缓存与扩展数据类型

多级缓存：本地 Caffeine（L1）→ Redis（L2）→ DB（事实源）。L1 不一致用短 TTL + 广播失效（pub/sub 通知各节点清 L1）；击穿用 `refreshAfterWrite` 异步刷新。一致性是"尽力而为"——L1 短 TTL 保证秒级自愈，要求强一致的字段（余额）不要进 L1。

扩展数据类型（内存换精度的极致）：**Bitmap**（1 亿用户 31 天全量标记 ≈ 372MB，1 bit/人/天）；**HyperLogLog**（16384 桶 × 6bit，固定 12KB 与元素量无关，标准误差 0.81%，原理是前导 0 的伯努利实验 + 调和平均压制离群桶）；**GEO**（底层是 ZSet，经纬度编码为 52bit geohash 整数作 score，范围查询 = score 区间查询，格子交界处要九宫格查询修正）。选型：ID 稠密去重用 Bitmap，稀疏用 Set，容忍 1% 误差用 HLL，存在性判断用布隆过滤器（有误判无漏判）。

## 十四、内存模型与容量估算

**一个 key-value 到底占多少内存**：看似 1B 的 value 实际花 60B+——key 的 SDS 头 + dictEntry（3 指针 24B）+ 哈希表槽位均摊 + jemalloc 对齐。**元数据开销远大于数据本身**（小对象场景 3~10 倍放大），容量规划必须按"每 KV 约 100B（含 key）"起步，而非按业务字段大小。

**1 亿 key 治理**：估算约 10GB+（含碎片 15GB 级）；结构优化用小 Hash 合并多个 String（把 N 个独立 key 的 dictEntry/SDS 头摊薄）；渐进迁移用 `SCAN` 游标分批双写。`INFO memory` 关键指标：`used_memory`（分配器视角）、`used_memory_rss`（OS 视角，差值即"未归还"）、`mem_fragmentation_ratio`（>1.5 碎片多，<1 危险）、`used_memory_dataset`（数据净占用）。

## 十五、RESP3 与客户端缓存、秒杀实战

RESP3（6.0+）新增 map/set/boolean/push 等原生类型，**push 是革命性能力**——服务端可在同一连接主动推送，这是客户端缓存失效通知的协议基础。`CLIENT TRACKING ON`（普通模式精确失效）/ `BCAST`（广播模式按前缀，省内存但粒度粗）。边界：失效推送不保证送达语义，本地缓存仍要设兜底 TTL。

**秒杀库存方案**：入口限流 → 前端静态化削峰 → Redis Lua 原子预扣 → MQ 异步落库 → DB 对账兜底。核心 Lua 脚本"查询-判断-扣减-登记"必须一个脚本原子完成，拆成多条命令并发下会"超卖 + 重复下单"双事故。**热点行打散**：`seckill:stock:1001` 拆 N 段，`hash(userId) % N` 路由到某段预扣，解决单 key 热点。未支付回补用延迟消息（15 分钟未支付回补库存），对账兜底定时比对"Redis 已售 + 余量"与"DB 成单数"。

## 十六、生产事故案例复盘

- **案例 1：DEL 大 Key 主线程阻塞**——每晚 3s 级慢查询尖刺，根因是 `DEL` 同步释放百万元素 Hash。修复：`UNLINK` + 全链路 lazyfree。
- **案例 2：缓存雪崩打穿 DB**——预热脚本统一 TTL 导致整点同时过期，DB QPS 从 2k 冲到 50k。修复：TTL 随机偏移 + 互斥回源 + 熔断。
- **案例 3：主从复制风暴循环**——THP 开启导致 fork 慢 + COW 膨胀 + backlog 1MB 太小，写入一冲就全量复制，恶性循环。修复：关 THP、backlog 调 512MB、从库错峰重连。
- **案例 4：Cluster 热点分片倾斜**——单热点 key 哈希固定落一个分片。修复：本地缓存 + 多副本后缀打散。
- **案例 5：List 当队列丢消息**——`BRPOP` "弹出即删除"，消费者崩溃后消息无记录。修复：改 Stream + 消费者组（PENDING + XAUTOCLAIM）或换 RocketMQ。

## 十七、疑难问答（延伸思考）

**Q：单机扛 10w QPS，涨到 100w 怎么扩？** 三步：读写分离（读多写少先加从库）→ Cluster 分片 → 读热点加本地缓存 + RESP3 client-side caching。但要先评估：如果 100w 里 90% 是同一个 key，扩容分片没用——先做热点治理。

**Q：为什么 Cluster 用哈希槽而不是一致性哈希？** 哈希槽把映射**显式收敛为一张槽→节点表**：运维可控（槽可手动迁移、精细再平衡）、数据归属明确（迁移粒度 = 槽）、gossip 直接广播槽位图。本质是把"数据分布"从哈希函数的隐式行为变成集群的显式元数据。

**Q：跳表和 B+ 树有什么本质区别，为什么 Redis 不用 B+ 树？** B+ 树为磁盘设计（矮胖多叉降低树高，每层一次磁盘 IO）；跳表为内存设计（无平衡旋转，范围遍历沿底层链表顺序扫）。ZSet 数据全在内存，B+ 树"降低磁盘 IO"的优势不存在。**存储结构选型的第一性问题是：数据在内存还是磁盘。**

**Q：Redis 哪些操作会阻塞主线程（全集）？** ① O(N) 命令（KEYS/FLUSHALL/大集合全量）② 大 key 的 DEL ③ fork（RDB/AOF 重写）④ AOF fsync 抖动 ⑤ swap ⑥ 淘汰风暴 ⑦ 热 key 单核打满 ⑧ 大 value 传输。

**Q：Redis 做分布式锁，"误删"和"超时"都处理好了，还有什么坑？** ① 主从切换双持锁（定位是效率锁）② 读写锁语义在读写并发下可能死锁 ③ 看门狗随进程崩溃，锁自然过期后业务要能容忍"仍有一段执行" ④ 加锁失败要有降级路径（排队/拒绝），不能无限自旋。

## 十八、Java 客户端底层与序列化工程

**Redisson 加锁 Lua 的三个设计精妙点**：用 Hash 不是 String（field=UUID:threadId 支持可重入 + 归属识别）；用 Lua（`exists + hincrby + pexpire` 必须原子，否则"判断完被抢"或"设了值没设过期"）；解锁也要 Lua（先 `hexists` 校验归属再 `hincrby -1`，归零才 `del`，避免误删）。Redisson 分布式对象本质是"**把 Redis 当共享内存，用 Lua 保证原子，用 pub/sub 做事件通知**"。

**序列化是缓存的隐形杀手**：体积决定内存（JDK 序列化 500B vs Kryo 80B，内存差 6 倍）、速度决定延迟、兼容性决定能否平滑发布、安全性（反序列化是 RCE 重灾区）。选型结论：跨服务用 Protobuf；通用可读缓存用 Jackson JSON；内部高性能高频缓存用 Kryo（接受跨版本重建）；**绝对不要用 JDK 默认序列化**（Spring Data Redis 默认的 `JdkSerializationRedisSerializer` 要替换掉）。

**Spring Cache 的经典坑**：自调用失效（类内部 `this.getUser(id)` 不走 AOP 代理）；默认 key 是"方法名:参数"，对象参数要自定义；`@Cacheable` 默认不缓存 null。`sync=true` 用 SETNX 在缓存层直接防击穿，但只对单 key 生效。`RedisTemplate` 默认 JDK 序列化又大又慢且不安全，换 `GenericJackson2JsonRedisSerializer`（注意它会写 `@class` 类型信息，类路径/版本不一致会抛 ClassNotFoundException）。

## 收口：一页纸记住 Redis

Redis 的核心价值是**内存存储 + 单线程事件驱动模型**：命令执行单线程换无锁与原子性，6.0 的多线程只并行 IO 不并行计算；底层用 SDS、quicklist、listpack、dict、skiplist、intset 做空间与时间的权衡；持久化用 RDB + AOF 混合；高可用按容量在"主从 + 哨兵"与"Cluster"间选型。

三条边界记住：**Redis 主从切换是高可用不是不丢数据**；**Redis 锁是效率锁不是正确性锁**；**缓存一致性是最终一致，追求强一致就不该引入缓存**。调优三板斧：`maxmemory + allkeys-lru` 治容量，大 Key 拆分 + UNLINK 治阻塞，SCAN 替代 KEYS + 慢日志治性能毛刺。
