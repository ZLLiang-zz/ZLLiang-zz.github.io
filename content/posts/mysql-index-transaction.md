---
title: "MySQL 索引与事务：B+ 树、MVCC 与锁的真实行为"
date: 2026-10-20T09:00:00+08:00
summary: "一条 SQL 从解析到落盘的完整路径：页结构、三层 B+ 树能存多少行、RC 与 RR 的差别为什么只在于 ReadView 的创建时机、RR 下的加锁规则，以及慢查询与死锁的排查顺序。"
categories: ["数据库"]
tags: ["MySQL", "InnoDB", "索引", "事务"]
author: ZLLiang
ShowToc: true
TocOpen: false
draft: false
---

## 一、基本概念

### 1.1 MySQL 的分层：一条 SQL 经过哪些手

```
┌────────────────────────────────────────────────────────────┐
│ 连接层：连接器（认证、权限、连接池）                          │
├────────────────────────────────────────────────────────────┤
│ Server 层（与引擎无关）                                      │
│   分析器（词法/语法）→ 优化器（成本模型、索引选择）→ 执行器    │
│   内置函数、视图、触发器、存储过程、binlog                    │
├────────────────────────────────────────────────────────────┤
│ 存储引擎层（插件式）                                          │
│   InnoDB（事务、行锁、MVCC、崩溃恢复）                        │
│   MyISAM（表锁、无事务）、Memory、Archive、CSV                │
└────────────────────────────────────────────────────────────┘
```

**为什么要分层**：Server 层只管"语义与执行计划"，引擎层只管"数据怎么存怎么取"。这决定了两个重要事实：

1. **绝大多数"SQL 优化"发生在优化器**（选哪个索引、join 顺序），而不是引擎；
2. **事务、锁、MVCC 由引擎实现**，所以 MyISAM 根本不支持事务（不是"没开"）。

### 1.2 存储引擎选型

| 维度 | InnoDB | MyISAM | Memory |
| :--- | :--- | :--- | :--- |
| 事务 | ✅ ACID | ❌ | ❌ |
| 锁粒度 | 行锁（基于索引） | 表锁 | 表锁 |
| 外键 | ✅ | ❌ | ❌ |
| 崩溃恢复 | ✅ redo log | ❌ | ❌ |
| 聚簇索引 | ✅（数据即索引） | ❌（索引与数据分离） | ❌ |
| 全文索引 | 5.6+ 支持 | 支持 | ❌ |
| 适用 | **默认，一切 OLTP** | 只读历史表（已淘汰） | 临时表、缓存小表 |

**结论**：8.0 下除极特殊场景，一律 InnoDB。答题时说"MyISAM 适合读多写少"已经过时了。

### 1.3 一次查询的完整生命周期

```
客户端
  → 连接器（TCP 握手、认证、读权限）
  → 查询缓存（8.0 已移除；5.7 及以前按 SQL 文本精确匹配，命中率低、失效频繁）
  → 分析器（词法分析 → 语法树；报错 "You have an error in your SQL syntax" 在这一步）
  → 优化器（改写、选索引、决定 join 顺序、算成本）
  → 执行器（调引擎接口，逐行取数，判断权限与 where）
  → 引擎（InnoDB：Buffer Pool → 索引 → 回表 → 返回）
  → 返回结果集（边查边发，不是全部查完再发）
```

**八个能体现深度的细节**：

1. **连接器之后权限就定了**（8.0 支持 `FLUSH PRIVILEGES` 后新连接生效，老连接不变）；
2. **`SELECT 1` 也要走完整个链路**（除了不碰引擎），所以"用 `SELECT 1` 做健康检查"也有成本；
3. **查询缓存在 8.0 被移除**，因为任何写操作都会让整表缓存失效，高并发下弊大于利；
4. **优化器不执行 SQL**，只生成执行计划；`EXPLAIN` 看到的就是它的产物；
5. **执行器是"取一行、判断一行"**（`WHERE` 中引擎不支持的条件下推到 Server 层过滤），8.0 的 ICP 把部分条件下推到引擎；
6. **结果集是流式返回**（`net_buffer_length` 分批），大结果集不会一次性占满内存，但客户端 `ResultSet` 全量加载会；
7. **`ORDER BY`/`GROUP BY` 可能建临时表**（`Using temporary`），临时表超 `tmp_table_size` 会落盘；
8. **`LIMIT` 深分页仍然要扫前面所有行**（`LIMIT 1000000, 20` 扫 100 万行），必须用游标/延迟关联优化。

### 1.4 核心术语对齐

| 术语 | 准确含义 | 常见误解 |
| :--- | :--- | :--- |
| **聚簇索引** | 索引与数据存在一起，叶子节点就是整行；InnoDB 的主键索引 | ❌ 以为"主键就是聚簇索引"（没主键时用唯一非空索引/隐藏 rowid） |
| **二级索引（辅助索引）** | 叶子节点存"索引列 + 主键值" | ❌ 以为存行指针（那是 MyISAM） |
| **回表** | 二级索引拿到主键 → 再查聚簇索引取整行 | ❌ 与"索引下推"混淆 |
| **覆盖索引** | 查询的列全在索引里，无需回表 | ❌ 以为 `SELECT *` 也能覆盖 |
| **最左前缀** | 联合索引按定义顺序匹配，遇到范围查询即停止后续列 | ❌ 以为 `(a,b,c)` 能用于 `WHERE b=1` |
| **MVCC** | 多版本并发控制，读不加锁靠 undo 版本链 + ReadView | ❌ 以为 MVCC 解决了所有幻读（当前读仍有幻读） |
| **快照读 / 当前读** | 普通 `SELECT` 是快照读；`SELECT ... FOR UPDATE`/`UPDATE`/`DELETE` 是当前读 | ❌ 以为 RR 下完全无幻读 |
| **间隙锁 Gap Lock** | 锁住索引记录之间的"空隙"，**只在 RR 生效** | ❌ 以为 RC 也有间隙锁（RC 下基本没有） |
| **临键锁 Next-Key** | 记录锁 + 间隙锁，左开右闭区间 | — |
| **两阶段提交** | redo prepare → binlog → redo commit | ❌ 与"分布式事务 2PC"混淆（虽然思想同源） |
| **LSN** | Log Sequence Number，redo 日志的全局递增序号 | — |
| **脏页 / 刷脏** | Buffer Pool 中已修改未落盘的页 | — |
| **change buffer** | 对非唯一二级索引的写缓存，减少随机 IO | ❌ 以为所有索引写都走它（唯一索引必须读页校验） |

### 1.5 体系定位：MySQL 在金融系统里的角色

对渠道类系统（授信/用款/还款），MySQL 的定位是**最终一致的状态机存储**：

```
渠道请求 → 应用（幂等 + 状态机）→ MySQL（事务边界内改状态）
                                  → MQ（异步通知/记账）
                                  → 对账（T+1 核对）
```

这决定了三条设计原则：

1. **事务要短**：事务里不做 RPC、不发 MQ（用本地消息表/事务消息）；
2. **热点行要拆**：账户余额、额度这类行必然成为瓶颈，要么乐观锁重试，要么串行化到单线程；
3. **对账兜底**：任何"扣减"都要有流水表 + 对账补偿，不能只依赖事务。

---

## 二、基础知识点（怎么用）

### 2.1 索引：B+ 树为什么胜出

| 结构 | 范围查询 | 单点查询 | 磁盘友好 | 结论 |
| :--- | :--- | :--- | :--- | :--- |
| 二叉搜索树 | 差（中序回溯） | O(log n) | 树高太高，IO 多 | ❌ |
| 平衡二叉树（AVL/红黑） | 差 | O(log n) | 每个节点一行，树高 20+ | ❌ |
| B 树 | 较好 | O(log n) | 节点存数据，扇出小 | 可用但不如 B+ |
| **B+ 树** | **优（叶子链表）** | O(log n) | **非叶只存键，扇出大，树高 3~4** | ✅ |
| Hash | **不支持** | O(1) | 内存结构 | 只适合等值（Memory 引擎） |

**B+ 树的三个关键优势**：

1. **非叶子节点不存数据**，只存键 + 指针 → 一页能放上千个键 → 树高 3 层可存 2000 万行（见 3.2）；
2. **叶子节点用双向链表连接** → 范围查询、`ORDER BY` 顺序扫描无需回溯；
3. **所有查询都走到叶子** → IO 次数稳定（树高）。

### 2.2 索引类型与语法

| 类型 | 说明 | 示例 |
| :--- | :--- | :--- |
| 主键索引 | 聚簇索引，唯一且非空 | `PRIMARY KEY (id)` |
| 唯一索引 | 允许 NULL（多个 NULL 不冲突） | `UNIQUE KEY uk_no (order_no)` |
| 普通索引 | 最常用 | `KEY idx_user (user_id)` |
| 联合索引 | 多列，遵循最左前缀 | `KEY idx_a_b_c (a,b,c)` |
| 前缀索引 | 只索引前 N 个字符，省空间 | `KEY idx_name (name(10))` |
| 全文索引 | 倒排，中文需 ngram 插件 | `FULLTEXT KEY ft (content) WITH PARSER ngram` |
| 函数索引（8.0.13+） | 对表达式建索引 | `KEY idx_lower ((LOWER(email)))` |
| 降序索引（8.0+） | 真正支持 `DESC` | `KEY idx (a ASC, b DESC)` |
| 不可见索引（8.0+） | 优化器忽略，用于安全删索引 | `ALTER TABLE t ALTER INDEX idx INVISIBLE` |

**前缀索引的选择性计算**：

```sql
SELECT COUNT(DISTINCT LEFT(name, 10)) / COUNT(*) FROM t;   -- 越接近 1 越好
```

**注意**：前缀索引**无法用于覆盖索引**（因为不包含完整值），也无法用于 `ORDER BY`。

### 2.3 索引设计五条原则

| 原则 | 说明 | 反例 |
| :--- | :--- | :--- |
| **1. 区分度高的列在前** | 联合索引第一列应尽量唯一 | `(status, user_id)`：status 只有 3 个值，选择性极差 |
| **2. 最左前缀** | `(a,b,c)` 可用于 `a`、`a,b`、`a,b,c`，不能用于 `b`、`b,c` | 把 `user_id` 放第二位导致单独查 `user_id` 走全表 |
| **3. 范围查询放最后** | `WHERE a=1 AND b>2 AND c=3`，`c` 用不上索引 | `(a,b,c)` 中 `b` 是范围，`c` 失效 |
| **4. 覆盖索引优先** | 常用查询列都放进索引，避免回表 | 列表页查 5 个字段，索引只含 1 个 |
| **5. 控制索引数量** | 每个索引都增加写成本与空间 | 单表 10+ 索引，写入明显变慢 |

**联合索引的顺序经验**：

```
等值条件列（区分度降序） → 范围条件列 → 排序列（可选）
```

### 2.4 事务：ACID 与实现手段

| 特性 | 含义 | InnoDB 靠什么实现 |
| :--- | :--- | :--- |
| **原子性 A** | 要么全成功要么全失败 | **undo log**（回滚日志） |
| **一致性 C** | 约束不被破坏 | 由应用 + 数据库约束共同保证（ACID 里的"目的"，其余三个是手段） |
| **隔离性 I** | 并发事务互不干扰 | **锁 + MVCC** |
| **持久性 D** | 提交后不丢 | **redo log**（WAL）+ 双写缓冲 |

> **关键理解**：能说清"一致性是目的，AID 是手段"，而不是把四个字母并列背诵。

### 2.5 隔离级别与并发问题

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 实现 |
| :--- | :--- | :--- | :--- | :--- |
| READ UNCOMMITTED | ✅ 可能 | ✅ | ✅ | 直接读最新版本 |
| **READ COMMITTED** | ❌ | ✅ 可能 | ✅ 可能 | 每次快照读新建 ReadView |
| **REPEATABLE READ**（默认） | ❌ | ❌ | ⚠️ 快照读不会，当前读可能 | 首次快照读建 ReadView + 间隙锁 |
| SERIALIZABLE | ❌ | ❌ | ❌ | 所有读加共享锁 |

**三个易错点**：

1. **RR 下幻读并没有被完全解决**：快照读靠 MVCC 看不到新插入的行；但**当前读**（`UPDATE`/`SELECT FOR UPDATE`）会看到，InnoDB 用**间隙锁**阻止插入，从而在"当前读"场景也避免幻读。所以准确说法是"RR 通过 MVCC + 间隙锁在绝大多数场景避免了幻读"。
2. **RC 是很多互联网公司的实际选择**：binlog 用 ROW 格式后，RC 的"不可重复读"可接受，且**RC 没有间隙锁**，死锁概率大幅降低、并发更高。
3. **隔离级别是会话级**（`SET SESSION TRANSACTION ISOLATION LEVEL`），但 ORM/连接池可能重置，排查时要确认当前会话的值。

### 2.6 锁的全景

| 维度 | 类型 | 说明 |
| :--- | :--- | :--- |
| 粒度 | 表锁 / 行锁 / 页锁 | InnoDB 行锁；`ALTER TABLE` 等 DDL 会加元数据锁 |
| 模式 | 共享 S / 排他 X | 读读兼容，读写/写写互斥 |
| 意向 | IS / IX | **表级**，表示"事务打算给某些行加 S/X 锁"，用于快速判断表锁能否加 |
| 行锁算法 | Record / Gap / Next-Key | 记录锁、间隙锁、临键锁 |
| 特殊 | 插入意向锁、AUTO-INC 锁、MDL | 插入意向锁是间隙锁的一种，多个插入同一间隙**不互斥**（只要插入位置不同） |

**InnoDB 行锁的三个真相**：

1. **行锁加在索引上**，不是加在行上。**没走索引 → 锁全表所有记录 → 等效表锁**（这是"update 没加索引导致全表锁"的根因）。
2. **间隙锁只在 RR 存在**，RC 下几乎没有（除了外键检查、唯一性检查）。
3. **加锁顺序由索引扫描顺序决定**，所以"同样两条 SQL 不同执行计划"可能一个死锁一个不死锁。

### 2.7 日志体系：五种日志各司其职

| 日志 | 层级 | 作用 | 是否可关 |
| :--- | :--- | :--- | :--- |
| **redo log** | InnoDB | 崩溃恢复（持久性），WAL | 不能关（可调大小） |
| **undo log** | InnoDB | 回滚 + MVCC 版本链 | 不能关 |
| **binlog** | Server | 主从复制、数据恢复（归档） | 可关（生产必须开） |
| **relay log** | Server（从库） | 中转 binlog | — |
| **slow query log** | Server | 慢查询记录 | 可关（建议开） |

**redo log 与 binlog 的本质区别**（高频追问）：

| 维度 | redo log | binlog |
| :--- | :--- | :--- |
| 归属 | InnoDB 引擎层 | Server 层（所有引擎都能用） |
| 内容 | **物理逻辑**：某页某偏移改了什么 | **逻辑**：SQL 语句或行变更（ROW 格式） |
| 写入 | **循环写**（固定大小，覆盖旧数据） | **追加写**（文件切换，不覆盖） |
| 用途 | 崩溃恢复 | 主从复制、归档、闪回 |
| 事务性 | 与事务强绑定（prepare/commit） | 一次事务一个事务（`XID` 关联） |

### 2.8 EXPLAIN 的关键列

```sql
EXPLAIN SELECT o.order_no, o.amount FROM t_order o
WHERE o.user_id = 1001 AND o.status = 1 ORDER BY o.create_time DESC LIMIT 20;
```

| 列 | 关注点 |
| :--- | :--- |
| `type` | 访问类型，优先级：`system > const > eq_ref > ref > range > index > ALL`。**出现 `ALL` 或 `index` 就要警惕** |
| `possible_keys` | 候选索引（可能为 NULL 但 `key` 有值 → 用了覆盖索引） |
| `key` | 实际使用的索引 |
| `key_len` | 用了索引的多少字节 → **判断联合索引用了几列** |
| `rows` | 预估扫描行数（不是精确值） |
| `filtered` | 引擎返回的行中，满足剩余条件的比例 |
| `Extra` | 信息量最大，见下表 |

**`Extra` 的六种关键值**：

| 值 | 含义 | 好坏 |
| :--- | :--- | :--- |
| `Using index` | 覆盖索引，无回表 | ✅ 最好 |
| `Using index condition` | 索引下推（ICP），条件下推到引擎 | ✅ 好 |
| `Using where` | Server 层过滤（可能没用上索引） | ⚠️ 看情况 |
| `Using filesort` | 需要额外排序（内存或磁盘） | ⚠️ 需优化 |
| `Using temporary` | 用了临时表（GROUP BY / DISTINCT / UNION） | ⚠️ 需优化 |
| `Using join buffer (hash join)` | 8.0.18+ 的 hash join | ✅ 通常比 BNL 好 |

**`key_len` 怎么算**（判断联合索引用了几列）：

```
int          → 4 字节（可为 NULL +1）
bigint       → 8 字节（可为 NULL +1）
varchar(n)   → utf8mb4 下 4n + 2（长度前缀），可为 NULL +1
char(n)      → 4n，可为 NULL +1
datetime     → 5 字节（8.0）/ 8 字节（5.7）
```

例：`KEY idx (a INT NULL, b VARCHAR(10) NULL)`，`key_len = 5 + 43` 表示两列都用上了。

### 2.9 分库分表与主键方案

| 方案 | 优点 | 缺点 |
| :--- | :--- | :--- |
| `AUTO_INCREMENT` | 简单、有序（B+ 树写入友好） | 分库后冲突、可预测（安全风险） |
| **雪花算法 Snowflake** | 全局唯一、趋势递增 | 依赖时钟（回拨问题）、workerId 分配 |
| **号段模式**（Leaf/美团） | 批量取号，DB 压力小 | 需独立服务、号段浪费 |
| UUID | 无中心化 | **无序导致页分裂**、占空间 |
| Redis `INCR` | 简单 | 依赖 Redis 可用性 |

**为什么推荐趋势递增**：InnoDB 聚簇索引按主键排序，**随机主键会导致频繁页分裂 + 随机 IO**；自增/雪花能保证新数据总在右侧插入（顺序写）。

**分片键选择**：

- 优先**业务主键**（`user_id`、`merchant_id`），保证单分片内能完成大多数查询；
- 避免**时间字段**做分片键（热点集中在新分片）；
- 必须接受"跨分片查询"的存在，用**冗余表/异构索引表**解决（如按 `order_no` 和 `user_id` 各存一份）。

### 2.10 字符集：一个真实的坑

| 场景 | 建议 |
| :--- | :--- |
| 建库建表 | `utf8mb4` + `utf8mb4_0900_ai_ci`（8.0）或 `utf8mb4_general_ci`（5.7） |
| 连接 | `character_set_client/connection/results` 三者一致，JDBC 加 `useUnicode=true&characterEncoding=utf8` |
| 排序规则 | `_ci` 不区分大小写；`_bin` 区分；`_0900_ai_ci` 支持 emoji 且性能更好 |

**为什么不用 `utf8`**：MySQL 的 `utf8` 是 **`utf8mb3`（最多 3 字节）**，存不了 emoji 与部分生僻字（4 字节），会报 `Incorrect string value`。**`utf8mb4` 才是真正的 UTF-8**。

**索引长度限制**：`utf8mb4` 下 `varchar(255)` 索引占 1020 字节，超过 InnoDB 单列索引 767 字节（`COMPACT` 行格式）会失败——8.0 默认 `DYNAMIC` 行格式 + `innodb_large_prefix` 已放宽到 3072 字节，但联合索引总长度仍受限。

### 2.11 十二条入门误区

1. "InnoDB 主键一定是聚簇索引"——是，但**没有主键时用第一个唯一非空索引，都没有则用 6 字节隐藏 rowid**。
2. "二级索引存的是行地址"——存的是**主键值**（MyISAM 才存行地址）。
3. "`SELECT *` 也能用覆盖索引"——不能，覆盖索引要求**查询列全在索引中**。
4. "索引越多查询越快"——写放大、优化器选择困难、空间膨胀。
5. "`LIKE '%x%'` 也能走索引"——不能（左模糊失效），`LIKE 'x%'` 可以。
6. "`OR` 两边都有索引就能用"——不一定，可能退化为全表；用 `UNION ALL` 改写。
7. "`NULL` 不占索引空间"——**占**，且 `IS NULL` 可用索引，但 `!=`、`NOT IN` 遇到 NULL 结果不符合直觉。
8. "`COUNT(*)` 比 `COUNT(1)` 慢"——8.0 下三者性能基本一致，`COUNT(列)` 还要判断非 NULL。
9. "`DELETE` 后表文件会变小"——不会，只是标记可复用，需要 `OPTIMIZE TABLE` 重建。
10. "`TRUNCATE` 可以回滚"——DDL，**隐式提交，不能回滚**。
11. "事务隔离级别越高越安全"——SERIALIZABLE 会严重降低并发，RR 也有代价。
12. "主从复制是同步的"——默认**异步**，半同步也只是"至少一个从库收到"，不是"应用完成"。

---

## 三、技术原理（怎么工作的）

### 3.1 InnoDB 的存储结构：从表空间到行

#### 3.1.1 五层结构

```
表空间 tablespace（.ibd 文件）
  └ 段 segment（叶子段 / 非叶子段 / 回滚段 …）
      └ 区 extent（默认 1MB = 64 个连续页）
          └ 页 page（默认 16KB，IO 的最小单位）
              └ 行 row（COMPACT / DYNAMIC 行格式）
```

**为什么用"区"而不是"页"连续分配**：B+ 树叶子节点在逻辑上相邻，物理上也应尽量相邻（减少随机 IO）。按区（64 页）分配能保证局部连续性，比逐页分配碎片少。

#### 3.1.2 页结构（16KB 的七段）

| 区段 | 大小 | 作用 |
| :--- | :--- | :--- |
| **File Header** | 38B | 页号、页类型、上一页/下一页指针、LSN、校验和 |
| **Page Header** | 56B | 记录数、空闲空间指针、**页目录槽数**、最大事务 ID、索引层级 |
| Infimum + Supremum | 26B | 两条虚拟记录，代表页内最小/最大边界 |
| **User Records** | 变长 | 实际行记录（按主键有序） |
| Free Space | 变长 | 未分配空间（链表） |
| **Page Directory** | 变长 | 每 4~8 条记录一个槽，存记录的**相对偏移**，用于页内二分查找 |
| File Trailer | 8B | 校验和 + LSN 低 32 位（防页损坏） |

**这张表能解释三个现象**：

1. **页内查询是二分**：`Page Directory` 先定位到槽，再在槽内链表遍历（不是全页扫）；
2. **主键有序插入很快**：新记录追加到 User Records 末尾，Free Space 相应减少；
3. **随机主键导致页分裂**：中间插入时要移动记录、可能新申请页并修改上下页指针。

#### 3.1.3 行格式

| 格式 | 特点 |
| :--- | :--- |
| `COMPACT` | 5.6 默认，变长字段长度列表 + NULL 位图 + 记录头 + 列数据 |
| `DYNAMIC` | **5.7+ 默认**，溢出列**完全存到外部页**（只留 20 字节指针），支持更长的变长字段 |
| `COMPRESSED` | 页级压缩（zlib），CPU 换空间 |
| `REDUNDANT` | 老格式，已淘汰 |

**行溢出的判定**：当一行超过页大小的一半（约 8KB），变长列（`TEXT`/`BLOB`/超长 `VARCHAR`）会存到"溢出页"，主记录只留指针。`DYNAMIC` 下即使能部分存下也全部溢出（避免行跨页）。

> **实践含义**：`SELECT *` 在有大字段时可能触发大量额外 IO（读溢出页），**列表查询务必只取需要的列**。

#### 3.1.4 Buffer Pool：真正的"数据库内存"

```
Buffer Pool（默认 128MB，生产建议物理内存的 50%~70%）
  ├ 数据页缓存（16KB × N）
  ├ 自适应哈希索引（AHI，热点等值查询加速）
  ├ change buffer（非唯一二级索引的写缓存）
  └ 改进版 LRU：young 区（5/8）+ old 区（3/8）
```

**改进版 LRU 解决什么问题**：全表扫描/大查询会把大量"一次性"页塞进 LRU 头部，挤掉真正的热数据（**缓冲池污染**）。InnoDB 的做法：

- 新页先插入 **old 区头部**；
- 只有在 old 区停留超过 `innodb_old_blocks_time`（默认 1000ms）后再次被访问，才晋升到 young 区；
- 这样"扫一遍就走"的页很快被淘汰，不污染热数据。

**三个关键参数**：

| 参数 | 作用 |
| :--- | :--- |
| `innodb_buffer_pool_size` | 最重要，命中率决定性能 |
| `innodb_buffer_pool_instances` | 多实例减少锁竞争（每个 ≥ 1G） |
| `innodb_old_blocks_time` | 防缓冲池污染的时间窗口 |

### 3.2 B+ 树容量：三层为什么能存 2000 万行

**计算过程**（假设主键 `bigint` 8B、页指针 6B、页 16KB）：

```
非叶子节点：每条索引项 = 8B(键) + 6B(指针) = 14B
            单页可存 = 16384 / 14 ≈ 1170 条

叶子节点：假设一行 1KB
          单页可存 = 16384 / 1024 = 16 行

三层 B+ 树：
  根（1 页）× 1170 × 1170 × 16 ≈ 2190 万行

两层：1170 × 16 ≈ 1.87 万行
四层：约 256 亿行（远超单表合理规模）
```

**三个结论**：

1. **三层足够支撑 2000 万行**，每次查询最多 3 次页 IO（根页常驻内存，实际 1~2 次）；
2. **行越小，容量越大**——所以"大字段拆表"不只是规范，是真的影响树高；
3. **"单表 2000 万就要分表"是经验值，不是硬限制**——真正的判据是**索引大小是否能放进 Buffer Pool** 与 **B+ 树高度是否增加**。

### 3.3 聚簇索引、回表、覆盖索引、ICP

#### 3.3.1 两棵树的协作

```
聚簇索引（主键）叶子： [id=1, order_no, amount, status, ...整行]
二级索引（user_id）叶子：[user_id=1001, id=1] → [user_id=1001, id=5] ...

SELECT amount FROM t_order WHERE user_id = 1001;
  ① 扫二级索引找到 id 列表
  ② 用每个 id 回聚簇索引取 amount      ← 回表（随机 IO）
```

**回表的代价**：假设命中 1 万行，就是 1 万次随机 IO（虽然 Buffer Pool 会缓解）。这是"**索引区分度低时不如全表扫**"的根本原因——优化器算的是"回表成本 vs 全表顺序扫成本"。

#### 3.3.2 覆盖索引

```sql
-- 假设有 KEY idx_user_status (user_id, status)
SELECT user_id, status FROM t_order WHERE user_id = 1001;   -- Using index，无回表

-- 若查询 amount，则必须回表
SELECT user_id, status, amount FROM t_order WHERE user_id = 1001;
```

**设计技巧**：把列表页需要的少量字段加进联合索引尾部（`(user_id, status, create_time)`），用空间换掉回表。

#### 3.3.3 索引下推（ICP，5.6+）

```sql
-- KEY idx_name_age (name, age)
SELECT * FROM t_user WHERE name LIKE '张%' AND age = 25;
```

| 无 ICP | 有 ICP |
| :--- | :--- |
| 引擎按 `name` 找到所有"张%"，**每条都回表**，Server 层再过滤 `age` | 引擎在索引上直接判断 `age=25`，**只对满足条件的回表** |

`EXPLAIN` 显示 `Using index condition`。**注意**：ICP 只对**二级索引**有效（聚簇索引本身就有整行，无需下推）。

### 3.4 redo log：WAL 与崩溃恢复

#### 3.4.1 为什么需要 redo log

如果每次事务提交都把 16KB 的页随机写到磁盘：

- 一次事务可能改多个页 → 多次随机 IO；
- 页分散在磁盘各处 → 吞吐极低。

**WAL（Write-Ahead Logging）** 的做法：**先顺序写日志（几十字节），再延迟刷脏页**。

```
事务提交 → 写 redo log（顺序 IO，快）→ 返回成功
                ↓ 后台线程择机
          刷脏页到 .ibd（随机 IO，慢，可合并）
```

**关键**：只要 redo 落盘（`innodb_flush_log_at_trx_commit=1`），即使脏页没刷，崩溃后也能从 redo 恢复。

#### 3.4.2 环形结构与 LSN

```
ib_logfile0 / ib_logfile1（总大小 = innodb_log_file_size × innodb_log_file_n）
┌──────────────────────────────────────────────┐
│  ... 已 checkpoint ... │ 已写未刷 ... │ 空闲 │
└──────────────────────────────────────────────┘
        ↑                      ↑              ↑
   checkpoint LSN          write pos      （循环覆盖）
```

| 概念 | 含义 |
| :--- | :--- |
| `LSN` | 全局递增日志序号，标记日志位置 |
| `write pos` | 当前写到哪（顺序推进） |
| `checkpoint` | 哪些日志对应的脏页**已经刷盘**，之前的日志可被覆盖 |
| `innodb_log_file_size` | 单个日志文件大小（8.0 默认 48M×2，**生产建议 1~2G**） |

**两个生产问题**：

1. **redo 太小** → `write pos` 追上 `checkpoint` → 被迫**同步刷脏**（"checkpoint 抖动"），表现为周期性卡顿。日志里出现 `Waiting for checkpoint` 就是这个。
2. **redo 太大** → 崩溃恢复时间长（要重放更多日志）。

#### 3.4.3 组提交（Group Commit）

高并发下每个事务都 `fsync` 一次，磁盘扛不住。组提交把多个事务的 redo 合并成一次 `fsync`：

```
flush 阶段：多个事务的 redo 写入 OS 缓存
sync 阶段：一次 fsync 落盘（带上所有事务）
commit 阶段：按顺序返回
```

**代价**：`binlog` 也要组提交（`binlog_group_commit_sync_delay` / `sync_no_delay_count`），但**组提交会牺牲一点延迟换吞吐**。

#### 3.4.4 双写缓冲（Doublewrite）

**问题**：页 16KB，而磁盘扇区 512B/4KB，刷一个页时断电可能只写了半页（**页断裂/torn page**），redo 也无法修复（redo 是"页内偏移的物理修改"，前提是页完整）。

**解法**：

```
刷脏页 → 先顺序写入 doublewrite buffer（共享表空间，连续 2MB）
       → fsync
       → 再写到 .ibd 实际位置
```

崩溃恢复时：若 .ibd 的页校验失败，从 doublewrite 里找副本恢复。**代价**：写放大一倍（顺序写，影响可控）。可用 `innodb_doublewrite=0` 关闭，但**生产不建议**。

### 3.5 undo log 与 MVCC：读不加锁的秘密

#### 3.5.1 undo 的两类

| 类型 | 对应操作 | 是否参与 MVCC |
| :--- | :--- | :--- |
| **insert undo** | `INSERT` | ❌ 只在回滚时用，提交后可立即删除 |
| **update undo** | `UPDATE` / `DELETE` | ✅ 构成版本链，由 `purge` 线程延迟清理 |

#### 3.5.2 版本链

行记录有隐藏列：

| 隐藏列 | 含义 |
| :--- | :--- |
| `DB_TRX_ID` | 最后修改该行的事务 ID（6B） |
| `DB_ROLL_PTR` | 指向 undo log 的指针（7B），串起历史版本 |
| `DB_ROW_ID` | 无主键时的隐藏主键（6B） |

```
当前行 (trx=300, amount=100)
   ↓ roll_ptr
undo: (trx=200, amount=80)
   ↓ roll_ptr
undo: (trx=100, amount=50)
```

**读操作顺着 `roll_ptr` 往回找，直到找到"对自己可见"的版本**——这就是 MVCC 读不阻塞写的原理。

#### 3.5.3 ReadView：可见性判断的四个字段

```cpp
class ReadView {
    trx_id_t m_low_limit_id;    // 高水位：≥ 它的事务都不可见（创建时未提交的最大 trx_id + 1）
    trx_id_t m_up_limit_id;     // 低水位：< 它的事务都可见（活跃事务中最小 trx_id）
    trx_id_t m_creator_trx_id;  // 创建该 ReadView 的事务自己
    trx_ids_t m_ids;            // 创建时所有**活跃（未提交）**事务 ID 列表
};
```

**可见性判断流程**（对行的 `DB_TRX_ID = trx_id`）：

```
① trx_id == creator_trx_id          → 可见（自己改的）
② trx_id < up_limit_id              → 可见（创建前已提交）
③ trx_id >= low_limit_id            → 不可见（创建后才开始的事务）
④ trx_id 在 m_ids 中                 → 不可见（创建时还活跃）
⑤ 否则                               → 可见（创建时已提交）
不可见 → 顺着 roll_ptr 找上一个版本，重复判断
```

#### 3.5.4 RC 与 RR 的唯一区别

| 级别 | ReadView 创建时机 | 效果 |
| :--- | :--- | :--- |
| **READ COMMITTED** | **每次快照读**都新建 | 能看到别人新提交的数据 → 不可重复读 |
| **REPEATABLE READ** | **第一次快照读**时创建，整个事务复用 | 整个事务看到同一份快照 → 可重复读 |

> **一句话**：RC 与 RR 的实现差异**只有一个 ReadView 的创建时机**，不是"加锁策略不同"。这是最容易被追问倒的点。

#### 3.5.5 快照读与当前读

```sql
-- 快照读（走 MVCC，不加锁）
SELECT * FROM t_order WHERE id = 1;

-- 当前读（读最新已提交版本，加锁）
SELECT * FROM t_order WHERE id = 1 FOR UPDATE;      -- X 锁
SELECT * FROM t_order WHERE id = 1 LOCK IN SHARE MODE; -- S 锁
UPDATE t_order SET status = 2 WHERE id = 1;         -- X 锁
DELETE FROM t_order WHERE id = 1;                   -- X 锁
```

**这解释了一个经典困惑**："RR 下我 `SELECT` 看不到新插入的行，但 `UPDATE` 却能改到它"——因为前者是快照读，后者是当前读。

### 3.6 锁的实现：加在索引上的锁

#### 3.6.1 三种行锁算法

| 算法 | 锁范围 | 示例（索引值 5, 10, 15） |
| :--- | :--- | :--- |
| **Record Lock** | 单条索引记录 | `WHERE id = 10` → 锁 id=10 |
| **Gap Lock** | 记录之间的间隙（开区间） | `WHERE id BETWEEN 6 AND 9` → 锁 (5,10) |
| **Next-Key Lock** | 记录 + 前面的间隙（左开右闭） | `WHERE id = 10` → 锁 (5,10] |

**Next-Key 是 RR 的默认算法**，目的是防幻读（间隙被锁住，插不进来）。

#### 3.6.2 加锁规则（RR 下，可背）

以索引值 `5, 10, 15` 为例：

| SQL | 加锁范围 |
| :--- | :--- |
| `WHERE id = 10`（唯一索引，记录存在） | Next-Key 退化为 **Record Lock**：只锁 10 |
| `WHERE id = 10`（记录不存在） | **Gap Lock**：(5,15) |
| `WHERE id = 10`（普通索引） | Next-Key：(5,10] + Gap：(10,15) |
| `WHERE id >= 10` | Next-Key：(5,10] + (10,15] + (15,+∞) |
| `WHERE id > 10` | (10,15] + (15,+∞) |
| `WHERE id <= 10` | (-∞,5] + (5,10] |
| **无索引** | **锁全表所有记录 + 所有间隙**（等效表锁） |

**两个必须理解的机制**：

1. **唯一索引等值命中会退化**：因为唯一性已保证只有一行，不需要间隙锁防插入。
2. **插入意向锁（Insert Intention Lock）**：插入前在间隙上加的特殊间隙锁，**多个插入意向锁互不冲突**（只要插入位置不同），但会与 Gap Lock 冲突——这就是"间隙被锁后插不进去"的实现。

#### 3.6.3 死锁：为什么 InnoDB 死锁这么常见

**死锁的两个必要条件在 InnoDB 里天然存在**：行锁可持有并等待、加锁顺序由执行计划决定。

典型场景：

```sql
-- 事务 A                          -- 事务 B
BEGIN;                             BEGIN;
UPDATE t SET v=1 WHERE id=1;       UPDATE t SET v=1 WHERE id=2;
UPDATE t SET v=1 WHERE id=2;       UPDATE t SET v=1 WHERE id=1;   -- 死锁
```

**排查**：

```sql
SHOW ENGINE INNODB STATUS\G
-- 找 LATEST DETECTED DEADLOCK 段：
--   TRANSACTION 1 在等什么锁、持有什么锁
--   TRANSACTION 2 同理
--   WE ROLL BACK TRANSACTION (2)  ← InnoDB 自动回滚"代价小"的那个
```

**三个参数**：

| 参数 | 作用 |
| :--- | :--- |
| `innodb_deadlock_detect` | 默认 ON，**每次加锁都做死锁检测**（O(n) 遍历），高并发下 CPU 开销大 |
| `innodb_lock_wait_timeout` | 等锁超时（默认 50s），关闭检测后靠它兜底 |
| `innodb_print_all_deadlocks` | 把所有死锁打到 error log（**生产建议开**） |

**降低死锁的四个手段**：

1. **固定加锁顺序**（按主键排序后批量更新）；
2. **缩短事务**（不在事务里做 RPC）；
3. **用 RC 隔离级别**（几乎没有间隙锁，死锁大幅减少）；
4. **唯一索引 + `INSERT ... ON DUPLICATE KEY`** 替代"先查后插"。

### 3.7 两阶段提交：redo 与 binlog 的一致性

**问题**：一份事务要写两个日志（redo 在引擎、binlog 在 Server），如果只成功一个，主从/恢复就会不一致。

**流程**：

```
① InnoDB 写 redo log，标记 prepare     ← 此时事务未提交
② Server 写 binlog，fsync
③ InnoDB 写 redo log，标记 commit      ← 事务真正提交
```

**崩溃恢复的三种情况**：

| 崩溃点 | redo 状态 | binlog | 恢复动作 |
| :--- | :--- | :--- | :--- |
| ①之前 | 无 | 无 | 回滚 |
| ①之后 ②之前 | prepare | **无** | **回滚**（binlog 没写，从库不会有） |
| ②之后 ③之前 | prepare | **有** | **提交**（binlog 完整，必须与从库一致） |

**判定依据**：redo 里有事务的 `XID`，binlog 里也有 `XID`（`Xid_event`）。恢复时用 XID 去 binlog 里找：找到就提交，找不到就回滚。

> **判断标准**：能说清"为什么 prepare 之后要看 binlog 决定提交还是回滚"，比背"两阶段提交"五个字强十倍。

### 3.8 优化器：为什么它选错了索引

#### 3.8.1 成本模型

```
总成本 = IO 成本 + CPU 成本
IO 成本  = 页数 × innodb_page_cost（默认 1.0，SSD 可调低）
CPU 成本 = 行数 × 常数（比较、过滤的代价）
```

优化器会为每个候选索引估算"扫描行数 × 回表成本"，选总成本最低的。**注意：它不知道数据分布的真实情况，只依赖统计信息。**

#### 3.8.2 统计信息

```sql
SHOW INDEX FROM t_order;              -- Cardinality（区分度估计）
ANALYZE TABLE t_order;                -- 重新采样（默认采样 20 个页）
```

`Cardinality` 是**采样估算**，不是精确值。数据分布倾斜（如 `status` 99% 是 1）时，估算可能严重偏差。

#### 3.8.3 选错索引的五个原因

| 原因 | 表现 | 处置 |
| :--- | :--- | :--- |
| 统计信息过期 | 大批量导入/删除后计划突变 | `ANALYZE TABLE` |
| 数据倾斜 | 估算"走索引很便宜"，实际扫大量行 | 改写 SQL、加更合适的索引 |
| 回表成本低估 | 二级索引命中行多，回表随机 IO 被低估 | 覆盖索引、`FORCE INDEX` |
| 函数/隐式转换 | 索引列被函数包裹或类型不匹配 | 改写、函数索引 |
| 优化器 bug / 版本差异 | 5.7 与 8.0 计划不同 | 用 `optimizer_trace` 看决策过程 |

**`optimizer_trace` 用法**：

```sql
SET optimizer_trace='enabled=on';
SELECT ...;
SELECT * FROM information_schema.OPTIMIZER_TRACE\G
SET optimizer_trace='enabled=off';
```

能看到每个候选索引的 `rows_estimate`、`cost`、被淘汰的原因。

### 3.9 排序与 JOIN：两个最容易出性能问题的地方

#### 3.9.1 `ORDER BY` 的两种执行方式

| 方式 | 条件 | 过程 | 代价 |
| :--- | :--- | :--- | :--- |
| **索引排序** | 排序列与索引顺序一致（且满足最左前缀） | 直接按索引顺序读 | 最优，无 `filesort` |
| **filesort（全字段）** | 结果集字段总长 ≤ `max_length_for_sort_data` | 把**所有查询列**放进 sort buffer 排序 | 内存占用大 |
| **filesort（rowid）** | 字段总长超限 | 只排"排序键 + rowid"，再回表取字段 | 多一次回表（随机 IO） |

`EXPLAIN` 的 `Using filesort` 表示发生了额外排序。**优化方向**：

```sql
-- 优化前：filesort
SELECT * FROM t_order WHERE user_id=1 ORDER BY create_time DESC LIMIT 20;

-- 优化后：联合索引 (user_id, create_time) → 索引本身有序，无 filesort
```

**注意**：`ORDER BY a, b` 与索引 `(a, b)` 顺序一致才能用索引排序；`ORDER BY a DESC, b ASC` 需要 8.0 的降序索引。

#### 3.9.2 JOIN 的三种算法

| 算法 | 条件 | 过程 | 8.0 |
| :--- | :--- | :--- | :--- |
| **NLJ**（Nested-Loop Join） | 被驱动表 join 列**有索引** | 驱动表每行 → 被驱动表走索引 | ✅ 默认 |
| **BNL**（Block Nested-Loop） | 被驱动表**无索引** | 驱动表数据放 join buffer，批量扫被驱动表 | ✅ 兜底（`Using join buffer (Block Nested Loop)`） |
| **Hash Join** | 等值 join、无索引 | 建哈希表，O(1) 探测 | **8.0.18+**，`Using join buffer (hash join)` |

**优化铁律**：**小表驱动大表**（驱动表行数少，被驱动表的索引查询次数就少）。`STRAIGHT_JOIN` 可强制顺序，但通常交给优化器。

**三个容易忽略的点**：

1. **`LEFT JOIN` 的 `WHERE` 条件写错会退化成 `INNER JOIN`**（对被驱动表的 `WHERE` 过滤会过滤掉 NULL 行）；
2. **join 字段字符集/类型不一致会导致索引失效**（隐式转换）；
3. **join 表超过 3 张时优化器可能选错顺序**，可用 `STRAIGHT_JOIN` 或拆查询。

### 3.10 主从复制：延迟从哪来

#### 3.10.1 三线程模型

```
主库：dump thread  →  binlog
从库：IO thread    →  写 relay log
      SQL thread   →  重放 relay log
```

| 复制模式 | 机制 | 数据安全 |
| :--- | :--- | :--- |
| 异步 | 主库提交即返回 | 主库宕机可能丢数据 |
| **半同步**（`rpl_semi_sync`） | 至少 N 个从库**收到** binlog 才返回 | 减少丢失，但仍有窗口（收到≠重放） |
| 全同步 / MGR | 多数派确认 | 最强，代价最高 |

#### 3.10.2 binlog 三种格式

| 格式 | 内容 | 优点 | 缺点 |
| :--- | :--- | :--- | :--- |
| `STATEMENT` | SQL 原文 | 日志小 | **`NOW()`/`UUID()`/`LIMIT` 不确定函数会导致主从不一致** |
| `ROW` | 行变更前后的值 | **绝对一致**、支持闪回 | 日志大（一条 UPDATE 改 10 万行就 10 万条） |
| `MIXED` | 自动选择 | 折中 | 仍有不确定性 |

**结论**：生产用 `ROW`（`binlog_format=ROW` + `binlog_row_image=MINIMAL` 减小体积）。

#### 3.10.3 主从延迟的四个原因

| 原因 | 说明 | 处置 |
| :--- | :--- | :--- |
| **大事务** | 一个事务改 100 万行，从库要重放很久 | 拆小批次（每批 1000 行） |
| **从库单线程重放**（5.6 前） | 主库并发写，从库串行 | 5.7+ 并行复制（按组提交/库并行），8.0 `WRITESET` 按行并行 |
| **从库配置差 / 有查询压力** | 从库还要扛读流量 | 提升配置、读写分离加从库 |
| **无主键表** | ROW 格式下全表扫描定位行 | **每张表必须有主键** |

**监控**：

```sql
SHOW SLAVE STATUS\G
-- Seconds_Behind_Master  （不总是准，大事务时会卡在 0 或跳变）
-- 更可靠：对比主从的 GTID / 心跳表（pt-heartbeat）
```

**读到旧数据的处理**：

1. **强制走主库**：写后立即读的场景（下单后查订单）；
2. **半同步 + `after_sync`**：至少保证 binlog 已到从库；
3. **业务容忍**：用"最终一致 + 前端轮询"。

---

## 四、深入与高级延伸（生产怎么用）

### 4.1 慢 SQL 排查：五步固定流程

```sql
-- ① 找出慢 SQL
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time';       -- 建议 1s，金融核心接口 200ms
-- 用 pt-query-digest 或 MySQL 8.0 的 sys.schema_* 视图聚合
SELECT * FROM sys.statement_analysis ORDER BY total_latency DESC LIMIT 10;

-- ② 看执行计划
EXPLAIN FORMAT=JSON SELECT ...;              -- JSON 能看到 cost 明细
EXPLAIN ANALYZE SELECT ...;                  -- 8.0.18+，真实执行并给实际耗时

-- ③ 看优化器怎么想的
SET optimizer_trace='enabled=on';
SELECT ...;
SELECT * FROM information_schema.OPTIMIZER_TRACE\G

-- ④ 看各阶段耗时（8.0.16+ 用 performance_schema）
SELECT * FROM performance_schema.events_statements_history_long
ORDER BY timer_end DESC LIMIT 5;

-- ⑤ 看锁等待
SELECT * FROM sys.innodb_lock_waits\G
```

**`EXPLAIN ANALYZE` 是 8.0 最值得用的新功能**：它真跑一遍，输出"预估行数 vs 实际行数"，一眼看出统计信息是否失真。

```
-> Nested loop inner join  (cost=... rows=1) (actual time=0.05..12.3 rows=1000 loops=1)
    -> Index lookup on o using idx_user (user_id=1001)  (rows=1000) (actual rows=50000)
```

**`actual rows` 比 `rows` 大 50 倍 → 统计信息严重失真**，先 `ANALYZE TABLE` 再谈优化。

### 4.2 索引失效的十二种场景

| # | 场景 | 示例 | 原因 |
| :--- | :--- | :--- | :--- |
| 1 | 左模糊 | `LIKE '%abc'` | 无法按前缀定位 |
| 2 | 函数包裹列 | `WHERE DATE(create_time)='2026-10-22'` | 索引存的是原值 |
| 3 | 表达式计算 | `WHERE id + 1 = 10` | 同上 |
| 4 | 隐式类型转换 | `WHERE phone = 13800138000`（phone 是 varchar） | 字符串转数字，索引失效 |
| 5 | 隐式字符集转换 | 表 A `utf8mb4` join 表 B `utf8` | 转换发生在被驱动表列上 |
| 6 | `OR` 连接非索引列 | `WHERE a=1 OR b=2`（b 无索引） | 退化为全表 |
| 7 | 联合索引跳过最左列 | `KEY(a,b,c)`，`WHERE b=1` | 最左前缀 |
| 8 | 范围查询后续列失效 | `WHERE a=1 AND b>2 AND c=3` | 范围后无法继续用索引 |
| 9 | `!=` / `NOT IN` | `WHERE status != 1` | 通常全表扫（区分度低时） |
| 10 | `IS NOT NULL` | `WHERE col IS NOT NULL` | 视区分度而定 |
| 11 | 前导 `%` 的 `LIKE` + 覆盖索引判断失误 | — | 优化器可能选全表 |
| 12 | 统计信息过期 / 数据倾斜 | `status` 99% 同值 | 优化器主动放弃索引 |

**验证方法**：`EXPLAIN` 对比 `key` 是否变化；或 `/*+ NO_INDEX(t idx) */`（8.0 优化器提示）做对照。

**修正示例**：

```sql
-- 失效
WHERE DATE(create_time) = '2026-10-22'
-- 改写成范围（可用索引）
WHERE create_time >= '2026-10-22 00:00:00' AND create_time < '2026-10-23 00:00:00'

-- 或建函数索引（8.0.13+）
ALTER TABLE t ADD KEY idx_date ((DATE(create_time)));
```

### 4.3 大表 DDL：怎么加字段不锁表

| 方式 | 原理 | 是否锁表 | 适用 |
| :--- | :--- | :--- | :--- |
| `ALGORITHM=COPY` | 建新表 → 拷数据 → 换名 | **全程锁写** | 老版本、改列类型 |
| `ALGORITHM=INPLACE` | 原地修改（部分操作） | 大部分时间可读写 | 加索引、加列（8.0 支持 instant） |
| `ALGORITHM=INSTANT`（8.0.12+） | 只改元数据 | **秒级，不锁** | 加列（末尾）、改默认值、重命名列 |
| **gh-ost** | 影子表 + binlog 同步 + 原子切换 | 不锁 | 大表、复杂变更 |
| **pt-osc** | 触发器同步影子表 | 不锁（但有触发器开销） | 老版本兼容 |

**8.0 的 `INSTANT ADD COLUMN` 限制**：

- 只能加在**最后一列**（8.0.29 起支持任意位置，但仍有限制）；
- 加列后**必须重建表**才能再用 INSTANT（有 `INSTANT` 列数量上限）；
- 改列类型、改字符集仍需 COPY/INPLACE。

**执行规范**：

```sql
ALTER TABLE t_order ADD COLUMN remark VARCHAR(200) NULL,
  ALGORITHM=INSTANT, LOCK=NONE;
```

**必须写 `ALGORITHM` 和 `LOCK`**——让 MySQL 在无法满足时报错，而不是"悄悄锁表"。这是避免 DDL 事故最简单的手段。

### 4.4 高可用架构的四代方案

| 方案 | 原理 | RPO/RTO | 评价 |
| :--- | :--- | :--- | :--- |
| 主从 + MHA | 监控 + 选主 + VIP 切换 | 可能丢数据 / 30s+ | 老方案，仍在用 |
| **MGR**（组复制） | Paxos 多数派 | 强一致 / 秒级 | 官方方案，至少 3 节点，写性能有损 |
| Orchestrator | 拓扑发现 + 自动切换 | 依赖半同步 | 灵活，Uber 开源 |
| 云 RDS 高可用版 | 厂商托管（主备 + 代理） | 秒级 | **大多数公司的实际选择** |

**金融场景的现实**：核心账务多用"**同城双活 + 异地灾备**"，MySQL 层面靠半同步 + 定期全备 + binlog 归档，切换往往**人工确认**（自动切换的风险大于收益）。

### 4.5 参数与容量：一张必背表

| 参数 | 建议值 | 理由 |
| :--- | :--- | :--- |
| `innodb_buffer_pool_size` | 物理内存 50%~70% | 命中率决定一切 |
| `innodb_buffer_pool_instances` | 8（每实例 ≥ 1G） | 减少锁竞争 |
| `innodb_log_file_size` | 1~2G | 太小导致 checkpoint 抖动 |
| `innodb_flush_log_at_trx_commit` | **1**（金融必须） | 0/2 会丢事务 |
| `sync_binlog` | **1**（金融必须） | 每次提交刷 binlog |
| `innodb_io_capacity` | SSD 2000~4000 | 控制刷脏速率 |
| `innodb_flush_neighbors` | 0（SSD） | SSD 不需要邻页合并 |
| `max_connections` | 按应用连接池总和 × 1.2 | 不是越大越好（每连接耗内存） |
| `innodb_lock_wait_timeout` | 5~10s | 默认 50s 太长，故障时拖死应用 |
| `long_query_time` | 0.2~1s | 按业务 SLA |

**`innodb_flush_log_at_trx_commit` 的三档**：

| 值 | 行为 | 丢数据风险 |
| :--- | :--- | :--- |
| 1 | 每次提交 `fsync` redo | 不丢（除非磁盘坏） |
| 2 | 每次提交写 OS 缓存，每秒 `fsync` | 宕机丢 1 秒 |
| 0 | 每秒写 + `fsync` | 宕机丢 1 秒，MySQL 崩溃也可能丢 |

**"双 1"（`innodb_flush_log_at_trx_commit=1` + `sync_binlog=1`）是金融系统的底线**，任何"为了性能改成 2/0"的方案都要有业务方的书面确认。

### 4.6 五个生产事故复盘

#### 4.6.1 连接池耗尽

- **现象**：应用大量 `Communications link failure` / `Unable to acquire JDBC Connection`，DB 侧 `Threads_connected` 打满。
- **根因**：某慢 SQL 把连接占住 → 连接池（Hikari `maximumPoolSize=20`）耗尽 → 所有请求等连接。
- **排查**：`SHOW PROCESSLIST` 看 `Time` 最大的会话；`sys.innodb_lock_waits` 看锁等待。
- **处置**：`KILL` 慢查询、加 `maximumPoolSize` 前先确认 DB `max_connections`、给连接池设 `connectionTimeout`（默认 30s 太长）。
- **防复发**：慢 SQL 告警、连接池指标监控（`active/idle/pending`）、每个接口设 SQL 超时（`Statement.setQueryTimeout`）。

#### 4.6.2 长事务导致 undo 膨胀

- **现象**：磁盘暴涨、`SHOW ENGINE INNODB STATUS` 里 `History list length` 持续增长。
- **根因**：一个事务开了没提交（代码异常没 rollback，或事务里做了 RPC），导致 undo 无法 purge。
- **排查**：
  ```sql
  SELECT trx_id, trx_started, trx_mysql_thread_id, trx_query
  FROM information_schema.innodb_trx ORDER BY trx_started LIMIT 5;
  ```
- **处置**：`KILL` 长事务；`innodb_undo_log_truncate=ON`（8.0）自动截断。
- **防复发**：`transaction-isolation` 之外，加"事务内禁止 RPC"的代码规约 + 事务超时监控。

#### 4.6.3 死锁集中爆发

- **现象**：日志里大量 `Deadlock found when trying to get lock`。
- **根因**：批量更新顺序不一致（多个线程按不同顺序更新同一批账户）。
- **处置**：批量操作前**按主键排序**，保证所有事务加锁顺序一致。
- **防复发**：`innodb_print_all_deadlocks=ON` 收集样本，定期分析；高并发写入场景评估降为 RC。

#### 4.6.4 主从延迟导致"刚存就查不到"

- **现象**：用户下单成功，跳转详情页报"订单不存在"。
- **根因**：写主库、读从库，从库延迟 200ms。
- **处置**：写后读强制走主库（`Hint` 或事务内读主）。
- **防复发**：会话级"写后 N 秒走主库"的标记（ShardingSphere 支持）、前端轮询兼容。

#### 4.6.5 binlog 暴涨打满磁盘

- **现象**：磁盘告警，`du` 发现 binlog 几个小时涨了几十 G。
- **根因**：一条 `UPDATE` 批量改了 500 万行（ROW 格式每行一条记录），或有人误执行全表更新。
- **处置**：`PURGE BINARY LOGS BEFORE ...` 清理、`binlog_expire_logs_seconds` 设置保留期。
- **防复发**：`UPDATE/DELETE` 必须带 `WHERE` 且 `LIMIT`；SQL 审核平台拦截无 `WHERE` 语句。

### 4.7 分库分表：什么时候做、怎么做

#### 4.7.1 判断标准

| 信号 | 阈值 |
| :--- | :--- |
| 单表行数 | > 2000 万（经验值，非绝对） |
| 单表物理大小 | > 50GB |
| 索引大小 | 超过 Buffer Pool 可分配空间 |
| 写入 QPS | 单库写入瓶颈（通常 5000~10000 TPS） |
| 连接数 | 单实例连接打满 |

**先做的四件事**（在分表之前）：加索引、归档历史数据、读写分离、冷热分离。**很多"必须分表"的场景，归档后就够了。**

#### 4.7.2 分片策略

| 策略 | 优点 | 缺点 |
| :--- | :--- | :--- |
| `HASH(user_id)` | 数据均匀 | 扩容需迁移（一致性哈希可缓解） |
| `RANGE(时间)` | 扩容简单、易归档 | 热点集中在新分片 |
| `RANGE + HASH` | 兼顾 | 复杂度高 |

#### 4.7.3 五个必须解决的问题

| 问题 | 方案 |
| :--- | :--- |
| 跨分片查询 | 冗余表（按 `order_no` 再存一份）、ES 异构索引 |
| 跨分片 JOIN | 避免；小表广播、大表冗余 |
| 跨分片分页/排序 | 各分片取 N 条，内存归并（深分页仍有性能问题） |
| 分布式事务 | 本地消息表 + 对账（金融首选）、Seata AT/TCC |
| 扩容 | 成倍扩容（2→4→8）+ 双写迁移，或一致性哈希 |

### 4.8 版本演进：5.7 → 8.0 的六个关键变化

| 特性 | 5.7 | 8.0 | 影响 |
| :--- | :--- | :--- | :--- |
| 默认字符集 | `latin1` | **`utf8mb4`** | 新建库不再有 emoji 问题 |
| 查询缓存 | 有（默认关） | **移除** | 不用再纠结 `SQL_CACHE` |
| Hash Join | ❌ | ✅ 8.0.18+ | 无索引 join 性能大幅提升 |
| 窗口函数 / CTE | ❌ | ✅ | 复杂报表可少写临时表 |
| 降序索引 | 语法支持但实际忽略 | **真正支持** | `ORDER BY a DESC` 可走索引 |
| 原子 DDL | ❌ | ✅ | DDL 崩溃不再留下半成品 |
| 默认认证插件 | `mysql_native_password` | `caching_sha2_password` | **老客户端连不上**（需升级驱动或改插件） |
| 不可见索引 | ❌ | ✅ | 安全删索引（先 INVISIBLE 观察） |
| `EXPLAIN ANALYZE` | ❌ | ✅ 8.0.18+ | 真实执行耗时 |

**升级踩坑提醒**：`caching_sha2_password` 是 5.7 客户端连 8.0 报 `Unable to load authentication plugin` 的原因；要么升级 JDBC 驱动（8.x），要么 `ALTER USER ... IDENTIFIED WITH mysql_native_password`。

### 4.9 怎么把 MySQL 讲成"我解决过的问题"

**推荐准备"额度扣减"这个案例**（金融渠道系统最典型）：

> "授信额度是热点行，早期用 `UPDATE credit SET available = available - #{amt} WHERE user_id=? AND available >= #{amt}` 靠行锁保证不超扣。压测发现同一用户并发 50 时 TPS 只有 200，因为所有请求串行在这把行锁上。
> 排查确认瓶颈是行锁等待（`sys.innodb_lock_waits`），不是 IO。
> 改造两步：① 把单行拆成 10 个子额度行，请求按 `hash(orderNo) % 10` 路由，把一把锁拆成十把；② 扣减失败时按剩余子额度重试，全部不足才判定额度不够。
> TPS 提升到 1800，且对账逻辑不变（子额度汇总等于总额度）。
> 另外补了两件事：一是 `available >= amt` 的条件放在 `UPDATE` 的 `WHERE` 里（原子判断，不能先查后改）；二是所有扣减都写流水表，T+1 对账兜底。"

这个案例覆盖了**热点行、行锁原理、拆分策略、原子性、对账**，是 9 年经验该有的深度。

---

## 五、总结整理

### 5.1 全景回顾

```
架构
  连接器 → 分析器 → 优化器（成本模型）→ 执行器 → 引擎
  查询缓存 8.0 移除；结果集流式返回

存储（InnoDB）
  表空间 → 段 → 区（1MB=64页）→ 页（16KB）→ 行（DYNAMIC）
  Buffer Pool：改进 LRU（young 5/8 + old 3/8，old_blocks_time 防污染）
  change buffer / AHI / doublewrite

索引
  B+ 树：非叶只存键（扇出 1170），叶子链表
  三层 ≈ 2000 万行（8B 主键 + 1KB 行）
  聚簇索引（整行） / 二级索引（主键） / 回表 / 覆盖索引 / ICP

日志
  redo：WAL、环形、LSN、checkpoint、组提交、doublewrite
  undo：insert/update，版本链，purge
  binlog：ROW 格式，主从复制与归档
  两阶段提交：prepare → binlog → commit；恢复看 XID 是否在 binlog

事务与并发
  ACID：A=undo，D=redo，I=锁+MVCC
  MVCC：DB_TRX_ID + DB_ROLL_PTR → undo 链 → ReadView 判断
  RC：每次快照读新建 ReadView；RR：首次创建并复用
  锁：行锁加在索引上；Record / Gap / Next-Key；RC 无间隙锁

优化
  EXPLAIN 的 type / key_len / Extra
  索引失效 12 种；深分页用延迟关联
  filesort（全字段/rowid）、NLJ/BNL/Hash Join
  优化器选错索引：统计信息、数据倾斜、回表成本、隐式转换

运维
  双 1 参数；Buffer Pool 50~70%；redo 1~2G
  大表 DDL：INSTANT / INPLACE / gh-ost
  主从延迟：大事务、并行复制、无主键
  分库分表：先归档、后拆分；冗余表解决跨片查询
```

### 5.2 速查清单（30 条）

| # | 问题 | 一句话答案 |
| :--- | :--- | :--- |
| 1 | InnoDB 与 MyISAM 区别 | 事务、行锁、外键、崩溃恢复、聚簇索引，InnoDB 全有 |
| 2 | 一条 SQL 的执行流程 | 连接器 → 分析器 → 优化器 → 执行器 → 引擎 |
| 3 | 为什么用 B+ 树 | 非叶存键扇出大、叶子链表支持范围、树高稳定 |
| 4 | 三层 B+ 树能存多少 | 约 2000 万（8B 主键、1KB 行） |
| 5 | 聚簇索引与非聚簇 | 聚簇叶子是整行；二级索引叶子是主键值 |
| 6 | 什么是回表 | 二级索引拿主键再查聚簇索引 |
| 7 | 覆盖索引 | 查询列全在索引中，无需回表（`Using index`） |
| 8 | 最左前缀 | 联合索引按定义顺序匹配，范围后失效 |
| 9 | 索引下推 | 条件下推到引擎在索引上过滤（`Using index condition`） |
| 10 | 索引失效场景 | 左模糊、函数、隐式转换、OR、最左前缀缺失等 12 种 |
| 11 | 为什么主键要自增 | 顺序插入避免页分裂与随机 IO |
| 12 | 为什么推荐雪花 | 全局唯一 + 趋势递增 |
| 13 | ACID 怎么实现 | A=undo，D=redo，I=锁+MVCC，C 是目的 |
| 14 | 四种隔离级别 | RU / RC / RR（默认）/ SERIALIZABLE |
| 15 | RR 解决幻读了吗 | 快照读靠 MVCC，当前读靠间隙锁，基本解决但非绝对 |
| 16 | RC 与 RR 的实现差异 | **只有 ReadView 创建时机不同** |
| 17 | MVCC 原理 | 隐藏列 + undo 版本链 + ReadView 可见性判断 |
| 18 | ReadView 四个字段 | `m_ids`、`up_limit_id`、`low_limit_id`、`creator_trx_id` |
| 19 | 快照读与当前读 | 普通 SELECT vs `FOR UPDATE`/UPDATE/DELETE |
| 20 | 行锁加在哪 | 索引上；无索引退化为锁全表 |
| 21 | 三种行锁 | Record / Gap / Next-Key |
| 22 | 间隙锁什么时候有 | RR 下；RC 基本没有 |
| 23 | 插入意向锁 | 间隙锁的一种，多个插入意向锁不互斥 |
| 24 | 死锁怎么排查 | `SHOW ENGINE INNODB STATUS` 的 `LATEST DETECTED DEADLOCK` |
| 25 | redo 与 binlog 区别 | 引擎层物理循环写 vs Server 层逻辑追加写 |
| 26 | 两阶段提交 | prepare → binlog → commit；恢复看 XID |
| 27 | 为什么需要 doublewrite | 防页断裂（redo 无法修复半页） |
| 28 | 优化器为什么选错索引 | 统计信息、数据倾斜、回表成本、隐式转换 |
| 29 | 深分页怎么优化 | 延迟关联（先取 id 再 join）或游标（`WHERE id > last`） |
| 30 | 主从延迟原因 | 大事务、单线程重放、从库压力、无主键表 |

### 5.3 三条追问链

**链一：从索引追到页结构**

> "为什么 B+ 树三层能存 2000 万？" → 非叶只存键 + 指针，一页 1170 条 → "一页为什么是 16KB？" → 与磁盘页/文件系统块对齐，`innodb_page_size` 可调 → "页里怎么找一条记录？" → Page Directory 槽位 + 槽内链表 → "页分裂什么时候发生？" → 随机主键中间插入，User Records 空间不足。

**链二：从 MVCC 追到 ReadView**

> "RR 怎么实现可重复读？" → 首次快照读建 ReadView 并复用 → "ReadView 怎么判断可见？" → 四个字段 + 五个判断分支 → "RC 呢？" → 每次快照读新建 → "那 RR 下 UPDATE 怎么能改到新插入的行？" → UPDATE 是当前读，读最新版本。

**链三：从慢 SQL 追到优化器**

> "这条 SQL 为什么没走索引？" → EXPLAIN 看 type/key → "key 是 NULL，但 possible_keys 有" → 优化器算完成本更高 → "为什么成本高？" → 回表行数估算大 / 数据倾斜 / 统计过期 → `EXPLAIN ANALYZE` 看预估 vs 实际 → `ANALYZE TABLE` 或改写。

### 5.4 决策与排查速查表

| 场景 | 首选动作 | 关键判据 |
| :--- | :--- | :--- |
| 单条 SQL 慢 | `EXPLAIN ANALYZE` | `type=ALL` / `rows` 与实际差几个数量级 |
| 索引没走 | 看 `possible_keys` 与 `key` | 有候选但未选中 → 成本问题 |
| 列表页慢 | 覆盖索引 + 延迟关联 | 是否回表、是否深分页 |
| 写入慢 | 看索引数量、redo、刷脏 | `innodb_io_capacity`、有无大事务 |
| 锁等待 | `sys.innodb_lock_waits` | 谁持有、持有多久 |
| 死锁 | `innodb_print_all_deadlocks` | 加锁顺序是否一致 |
| undo 膨胀 | `information_schema.innodb_trx` | 有无长事务 |
| 主从延迟 | `SHOW SLAVE STATUS` + 心跳表 | 大事务、并行复制配置 |
| 磁盘暴涨 | binlog 体积、undo、临时表 | 是否批量 UPDATE |
| 加字段 | `ALGORITHM=INSTANT, LOCK=NONE` | 8.0.12+ 且加在末尾 |
| 删索引 | `ALTER INDEX ... INVISIBLE` | 观察一周无影响再删 |
| 分表前 | 归档 + 索引 + 读写分离 | 单表是否真超 2000 万 |

---

> 本文基线：MySQL 5.7 / 8.0（InnoDB），差异处标注。
