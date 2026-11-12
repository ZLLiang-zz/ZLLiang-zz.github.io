---
title: "MyBatis 全量深度解析：从 Mapper 代理到缓存与插件链"
date: 2026-11-12T21:00:00+08:00
summary: "一篇把 MyBatis 讲透的长文：Mapper 接口没有实现类却能执行 SQL 的代理真相（MapperProxy 三条分支）、启动时一条 SQL 如何被解析进内存（SqlSource 三种形态与 StrictMap）、SqlSession 四个反直觉语义、三种 Executor 与 BaseExecutor/CachingExecutor 两级缓存源码、CacheKey 算法、插件四大扩展点与分页插件原理、Spring 集成的事务对齐、批量写入与主键回填、十个生产坑。每个结论带成立条件与代价。"
categories: ["MyBatis"]
tags: ["MyBatis", "MapperProxy", "动态代理", "SqlSession", "Executor", "一级缓存", "二级缓存", "插件", "N+1", "SQL注入"]
author: ZLLiang
ShowToc: true
TocOpen: false
draft: false
---

写过 MyBatis 的人都会碰到一个"灵异"现象：`UserMapper` 明明只是个接口、没有实现类，`userMapper.selectById(1)` 却能正常执行，SQL 还自动带上了参数。这背后是 JDK 动态代理加一个 `MapperProxy`，在运行时把接口方法翻译成 SQL。而 `#{}` 与 `${}` 的区别、一级缓存为什么会"读到旧值"、插件为什么能拦截四大对象、二级缓存为什么"事务提交才生效"——这些问题的答案，都藏在从 `SqlSession` 到 `Executor` 到 `StatementHandler` 的这条执行链里。本文以 MyBatis 3.5.x 源码为主线，按「初级 → 中级 → 高级」三层递进，高级部分覆盖最全的源码链路。读完你能：讲清 Mapper 接口"无实现"的原理，说清 `#{}` 预编译与 `${}` 拼接的安全边界，以及生产上怎么避开一二级缓存的坑。

---

## 一、初级：先建立心智模型

### 1.1 MyBatis 是什么：半自动 ORM 的三层定位

一句话：**MyBatis 是一个「SQL 映射框架」——把「手写的 SQL」和「Java 对象」之间的双向转换自动化，但不替你生成 SQL。**

这个定义里有两个关键词，决定了它的全部设计：

- **手写 SQL**：SQL 的可控权在你手上。执行计划怎么写、加什么索引、用不用 `union`，框架不插手。这是它和 Hibernate/JPA 最大的分野。
- **映射自动化**：`ResultSet` 的列 → 对象属性、对象属性 → `PreparedStatement` 参数，这两段最机械、最容易写错的代码由框架接管。

按「在技术栈中的位置」看，它有三层身份：

| 层次 | 身份 | 具体做什么 |
| :--- | :--- | :--- |
| 对下 | **JDBC 的封装层** | 管理连接获取与释放、`PreparedStatement` 创建与参数绑定、`ResultSet` 遍历与关闭、异常转换（`SQLException` → `PersistenceException`） |
| 对上 | **DAO 的实现层** | 你只写接口方法，框架用动态代理生成实现；方法名 + 命名空间定位到一条 `MappedStatement` |
| 横向 | **可插拔的拦截框架** | 在 `Executor`、`StatementHandler`、`ParameterHandler`、`ResultSetHandler` 四个点开放拦截 |

「半自动」这个词的准确含义：它帮你做**映射**（自动），但不帮你做 **SQL 生成**（手写）。Hibernate 是「全自动」——连 SQL 都替你生成。

### 1.2 它解决什么、不解决什么

**解决的四个痛点**：样板代码（原生 JDBC 一个查询 20 行，MyBatis 一行调用）、映射手工化（`rs.getString("user_name")` 这类代码量大且易错）、SQL 散落（写在 Java 字符串里无法格式化复用）、资源泄漏（`Connection`/`Statement`/`ResultSet` 三层关闭）。

**不解决的（也是它的边界）**：

| 不解决 | 说明 | 你需要另找的方案 |
| :--- | :--- | :--- |
| 跨数据库方言 | 把 SQL 原样发下去，`limit`/`rownum`/`top` 差异要自己处理 | 自己分页插件、或选 ORM |
| 对象状态管理 | 没有「持久化上下文」，对象改了不会自动 flush | 手动 `update` |
| 关联关系自动维护 | 一对多要写 `resultMap` 或二次查询，没有级联保存 | 手动写或引入 JPA |
| 分库分表路由 | 单库单表视角，不知道分片键 | ShardingSphere 等中间件 |
| 多租户隔离 | 不会自动加 `tenant_id` | 拦截器或中间件 |

### 1.3 与 JPA / JdbcTemplate 的取舍

| 维度 | MyBatis | Hibernate / JPA | JdbcTemplate |
| :--- | :--- | :--- | :--- |
| SQL 控制权 | **完全在手** | 框架生成，可插手但别扭 | 完全在手 |
| 映射 | XML/注解手写映射 | 注解/XML 自动映射 | 手写 `RowMapper` |
| 复杂查询 | **强**（动态 SQL 表达力好） | 弱（Criteria 复杂、原生 SQL 逃逸） | 中（拼字符串） |
| 简单 CRUD | 中（要写 SQL） | **强**（`save`/`findById` 直接用） | 弱 |
| 缓存 | 一级/二级，语义简单可控 | 一级/二级 + 查询缓存，语义复杂 | 无 |
| 性能调优 | **直接**（SQL 就是最终 SQL） | 间接（要看生成的 SQL） | 直接 |

**一句话选型**：业务查询复杂、DBA 要审 SQL、需要精细控制执行计划 → MyBatis；领域模型复杂、以增删改查为主、团队接受学习成本 → JPA。

### 1.4 核心术语表（读源码前必须建立的心智模型）

| 术语 | 是什么 | 生命周期 | 一句话作用 |
| :--- | :--- | :--- | :--- |
| `Configuration` | 全局配置容器 | 应用级，单例 | 持有所有 `MappedStatement`、`ResultMap`、`TypeHandler`、`Interceptor` |
| `SqlSessionFactory` | 会话工厂 | 应用级，单例 | 由 `Configuration` 构建，负责 `openSession()` |
| `SqlSession` | 一次数据库会话 | 请求级/事务级，**非线程安全** | 门面：`selectOne`/`insert`/`commit`/`rollback` |
| `Executor` | 执行器 | 每个 `SqlSession` 一个 | 调度 StatementHandler、管一级缓存、管事务 |
| `MappedStatement` | 一条 SQL 的完整描述 | 启动时构建，不可变 | 持有 `SqlSource`、结果映射、缓存与超时配置 |
| `BoundSql` | 一次执行的 SQL 与参数 | 每次执行 | `sql` 字符串 + `ParameterMapping` 列表 + 附加参数 |
| `StatementHandler` | JDBC Statement 的封装 | 每次执行 | 创建 PS、绑定参数、执行、交给结果处理器 |
| `TypeHandler` | Java 类型 ↔ JDBC 类型的转换器 | 应用级 | 处理 `String`/`Date`/`Enum`/JSON |
| `MapperProxy` | Mapper 接口的代理 | 每个接口 | 把方法调用翻译成 `SqlSession` 调用 |
| `Interceptor` | 插件 | 应用级 | 在四个扩展点包装目标对象 |

### 1.5 最小可用示例 + Spring Boot 接入

不用 Spring 的最小示例：

```java
String resource = "mybatis-config.xml";
InputStream in = Resources.getResourceAsStream(resource);
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(in);

try (SqlSession session = factory.openSession(true)) {   // true = 自动提交
    UserMapper mapper = session.getMapper(UserMapper.class);
    User user = mapper.selectById(1L);
}
```

**关键点**：`openSession(true)` 的 `true` 是 `autoCommit`。默认 `false`，增删改必须手动 `commit()`，否则连接关闭时回滚——这是新手最常见的「插入了但查不到」。

Spring Boot 下接入：

```yaml
mybatis:
  mapper-locations: classpath*:mapper/**/*.xml
  type-aliases-package: com.example.domain
  configuration:
    map-underscore-to-camel-case: true
```

```java
@MapperScan("com.example.mapper")   // 或每个接口加 @Mapper
@SpringBootApplication
public class App { }
```

三条容易踩的规则：① `mapper-locations` 用 `classpath*:`（带星）才能扫到依赖 jar 里的 XML；② `@MapperScan` 与 `@Mapper` 二选一即可；③ XML 放在 `src/main/java` 下时，Maven 默认不打包 `.xml`，需在 `<resources>` 里显式包含。

### 1.6 `#{}` 与 `${}`：唯一必须背熟的语法差异

| 维度 | `#{}` | `${}` |
| :--- | :--- | :--- |
| 编译结果 | `?` 占位符 | 直接拼进 SQL 文本 |
| 执行方式 | `PreparedStatement` 预编译 | `Statement` 字符串拼接 |
| SQL 注入 | **不会**（参数被当值处理） | **会**（参数被当 SQL 片段） |
| 类型处理 | 走 `TypeHandler`，自动加引号 | 不加引号，需自己拼 |
| 预编译复用 | 可复用执行计划 | 每次都是新 SQL，无法复用 |
| 适用 | 所有**值** | 表名、列名、`ORDER BY`、`LIMIT` 等**标识符** |

```xml
<!-- 正确：值用 #{} -->
<select id="findByName" resultType="User">
  SELECT * FROM t_user WHERE name = #{name}
</select>

<!-- 危险：排序字段用 ${}，必须白名单校验 -->
<select id="list" resultType="User">
  SELECT * FROM t_user ORDER BY ${sortColumn} ${sortOrder}
</select>
```

`${}` 的唯一安全用法是**白名单枚举**：

```java
private static final Set<String> ALLOWED = Set.of("create_time", "amount");
if (!ALLOWED.contains(sortColumn)) throw new IllegalArgumentException("非法排序字段");
```

> **面试口径**：不要只说「`#{}` 防注入」。要补一句「`${}` 不是不能用，而是只能用于标识符，且必须在 Java 层做白名单，因为预编译占位符不能替换表名」。

### 1.7 入门最常错的八个点

1. **以为一级缓存是线程共享的**。它是 `SqlSession` 级，而 `SqlSession` 非线程安全，多线程共用会串数据。
2. **以为加了 `@Transactional` 就有一级缓存**。恰好相反：有事务时同一个 `SqlSession` 被复用，一级缓存**才生效**；无事务时每次 Mapper 调用新建会话，缓存等于没有。
3. **以为 `resultType` 能处理下划线转驼峰**。需要 `mapUnderscoreToCamelCase=true`，且只对**自动映射**生效；手写 `resultMap` 时不生效。
4. **以为 `parameterType` 必填**。可以省略，MyBatis 会从参数推断；写错反而报错。
5. **在 `#{}` 里写表达式**。`#{age > 18}` 不合法，`#{}` 里只能是属性路径，条件判断放 `<if test>`。
6. **`foreach` 的 `collection` 写 `list` 却传了 `@Param`**。加了 `@Param("ids")` 后必须写 `collection="ids"`。
7. **`useGeneratedKeys` 不写 `keyProperty`**。主键取不回来，`insert` 后对象 id 仍为 null。
8. **以为 XML 改了会热生效**。`Configuration` 在启动时构建完成，改 XML 必须重启。

---

## 二、中级：一条 SQL 怎么跑起来

### 2.1 一次调用的全景链路

```text
mapper.selectById(1L)
  └─ MapperProxy.invoke                 // 动态代理，算出 statementId = 接口全名.方法名
      └─ MapperMethod.execute            // 判断 SELECT，调 sqlSession.selectOne
          └─ DefaultSqlSession.selectOne
              └─ Executor.query          // CachingExecutor（二级）→ BaseExecutor（一级）
                  ├─ createCacheKey      // 一级缓存查找
                  ├─ BoundSql = ms.getBoundSql(param)   // 动态 SQL 在这里求值
                  └─ SimpleExecutor.doQuery
                      ├─ StatementHandler.prepare        // 取连接、建 PS
                      ├─ ParameterHandler.setParameters  // 绑参
                      ├─ Statement.execute               // 执行
                      └─ ResultSetHandler.handleResultSets // 映射成对象
```

这条链上有三个「可变点」，也是所有面试追问的落点：`BoundSql` 在 `Executor.query` 之前生成（所以插件改 SQL 要拦 `Executor` 或 `StatementHandler`）；一级缓存在 `BaseExecutor`、二级缓存在 `CachingExecutor`（两者失效时机不同）；参数绑定与结果映射分别由 `ParameterHandler`/`ResultSetHandler` 负责（自定义类型转换只需写 `TypeHandler`）。

### 2.2 Mapper 接口为什么不用实现类

核心是 JDK 动态代理。每个 Mapper 接口在启动时被注册成一个 `MapperProxyFactory`，`getMapper` 时用它生成代理对象：

```java
// MapperProxyFactory.newInstance
protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(
        mapperInterface.getClassLoader(),
        new Class[]{ mapperInterface },
        mapperProxy);
}
```

注意：**每个 Mapper 接口只有一个 `MapperProxyFactory`，但 `MapperProxy` 是每次 `getMapper` 时新建的**（因为它要绑定当前的 `SqlSession`）。代理方法调用时，按「接口全限定名 + 方法名」拼出 `statementId`，去 `Configuration` 里找到对应的 `MappedStatement` 执行。

### 2.3 映射文件五组成 + `resultType`/`resultMap` 边界

映射文件五部分：`namespace`（绑定 Mapper 接口全限定名）、`<resultMap>`（列与属性映射规则）、`<sql>`+`<include>`（SQL 片段复用）、`<select>/<insert>/<update>/<delete>`（语句定义）、`<cache>`/`<cache-ref>`（二级缓存）。

`resultType` 与 `resultMap` 怎么选：

| 场景 | 用什么 |
| :--- | :--- |
| 列名与属性名一致（或开了驼峰） | `resultType` |
| 列名与属性名不一致、需要类型转换 | `resultMap` |
| 一对一/一对多 | `resultMap` + 嵌套 |
| 返回 `Map` | `resultType="map"`（列名做 key，注意 Oracle 返回大写） |
| 返回聚合值 | `resultType="java.lang.Long"` 等 |

**不要混写**：同时指定 `resultType` 与 `resultMap` 时，`resultMap` 生效，`resultType` 被**静默忽略**。这个「静默」是排查映射问题的常见陷阱。

### 2.4 动态 SQL 用法（`<if>`/`<foreach>`/`<choose>`/`<bind>`）

MyBatis 提供 9 种动态标签：`trim`/`where`/`set`/`foreach`/`if`/`choose`/`when`/`otherwise`/`bind`。典型用法：

```xml
<select id="findByIds" resultType="User">
  SELECT * FROM t_user WHERE id IN
  <foreach collection="ids" item="id" open="(" separator="," close=")">
    #{id}
  </foreach>
</select>

<select id="search" resultType="User">
  <bind name="pattern" value="'%' + keyword + '%'"/>
  SELECT * FROM t_user
  <where>
    <if test="keyword != null and keyword != ''">
      name LIKE #{pattern}
    </if>
  </where>
</select>
```

三个使用约束：`collection` 的取值名——`List` 传 `list`、数组传 `array`、`Map` 传 key、加了 `@Param` 传注解名，名字错了报 `BindingException`；`IN` 元素个数受数据库限制（Oracle 上限 1000，`ORA-01795`）；`<foreach>` 里写 `${}` 会把值直接拼进 SQL，等于放弃预编译，禁止用于外部输入。

### 2.5 一级缓存 vs 二级缓存（用法层）

先记住两个默认值：`localCacheScope` 默认 `SESSION`（一级缓存作用域是整个 `SqlSession`）；`cacheEnabled` 默认 `true`（但二级缓存仍需 `<cache/>` 才真正开启）。

| 维度 | 一级缓存 | 二级缓存 |
| :--- | :--- | :--- |
| 作用域 | `SqlSession` 级 | namespace（Mapper）级，跨会话共享 |
| 开启方式 | 默认开启 | 需 `<cache/>` 显式声明 |
| 失效时机 | commit/rollback、同 namespace 写操作、`clearCache()` | 事务提交后写入；写操作 `flushCache` |
| 典型坑 | 同会话读到旧值 | 跨表更新无法感知（脏读） |

一级缓存的三条失效规则：执行 `commit`/`rollback`、执行同 namespace 的增删改、手动 `clearCache()`。

### 2.6 延迟加载与 N+1

一对多有两种写法：**嵌套结果**（一条 SQL join）与**嵌套查询**（主表 1 次 + 每个子对象 1 次 = N+1）。

```xml
<!-- 写法一：嵌套结果，一条 SQL -->
<resultMap id="orderWithItems" type="Order">
  <id column="order_id" property="id"/>
  <collection property="items" ofType="OrderItem">
    <id column="item_id" property="id"/>
    <result column="sku" property="sku"/>
  </collection>
</resultMap>

<!-- 写法二：嵌套查询，N+1 次 -->
<collection property="items" column="id"
            select="com.example.mapper.OrderItemMapper.selectByOrderId"/>
```

N+1 默认**立即加载**（`lazyLoadingEnabled` 默认 `false`），所以会立刻打满数据库。开启延迟加载需两个开关配合：

```yaml
lazyLoadingEnabled: true          # 全局开启延迟加载
aggressiveLazyLoading: false      # 3.5.x 默认 false：只有访问该属性才加载
```

`aggressiveLazyLoading=true` 时，访问对象任意属性都会触发全部延迟属性加载——这是老版本默认值（3.4.x 之前），升级后行为变化常导致「懒加载不生效」的误判。延迟加载的实现是给结果对象生成代理（`JavassistProxyFactory` 默认），所以**被代理的类不能是 final，方法不能是 final/private**。

---

## 三、中级 → 高级：启动时一条 SQL 怎么被装进内存

### 3.1 解析总入口与严格顺序

```java
// SqlSessionFactoryBuilder
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
    return build(parser.parse());     // parse() 返回 Configuration
}
```

`XMLConfigBuilder.parseConfiguration()` 按固定顺序处理子节点，**这个顺序不能乱**：

```text
properties → settings → typeAliases → plugins → objectFactory
→ objectWrapperFactory → reflectorFactory → environments
→ databaseIdProvider → typeHandlers → mappers
```

两个后果：`typeAliases` 必须在 `mappers` 之前（否则 XML 里 `resultType="User"` 解析时别名还没注册）；`plugins` 在 `mappers` 之前（插件的 `@Signature` 校验依赖已注册的类型）。

### 3.2 Mapper XML 的解析步骤

```java
// XMLMapperBuilder.parse()
configurationElement(parser.evalNode("/mapper"));   // namespace、cache-ref
cacheElement(context.evalNode("cache"));            // <cache>
resultMapElements(...);                             // <resultMap>（先于 statement）
sqlElement(...);                                    // <sql> 片段
buildStatementFromContext(...);                     // <select>/<insert>/...
bindMapperForNamespace();                           // namespace → Mapper 接口注册
```

**顺序决定了两个经典现象**：① 同一 XML 内 `<resultMap>` 写在 `<select>` 后面仍能工作（`resultMapElements` 在 `buildStatementFromContext` 之前整体执行），但**跨文件**引用时依赖 `incompleteResultMaps` 队列 + 最后统一重试的补丁机制；② 为什么改 XML 必须重启——`MappedStatement` 启动时构建后视为不可变，运行期没有重新解析的入口。

### 3.3 `MappedStatement` 与 `StrictMap`（方法不能重载的根因）

`XMLStatementBuilder.parseStatementNode()` 把 id 拼成 `namespace.id`，最终由 `Configuration.addMappedStatement` 放进 `mappedStatements`。这个容器不是普通 `HashMap`，而是 `StrictMap`：

```java
// StrictMap.put：id 重复直接抛异常
if (map.containsKey(key)) {
    throw new IllegalArgumentException(
        "Mapped Statements collection already contains value for " + key);
}
```

**这就是「Mapper 接口不能方法重载」的根因**：`statementId = 接口全限定名 + "." + 方法名`，`selectById(Long)` 与 `selectById(String)` 会算出同一个 id，启动即失败。不是框架懒得支持，而是它的寻址模型就是「一个方法名对应一条语句」。

### 3.4 `SqlSource` 的三种形态：静态与动态的分界

| 实现 | 何时产生 | `getBoundSql()` 做什么 | 性能 |
| :--- | :--- | :--- | :--- |
| `DynamicSqlSource` | SQL 含 `${}` 或动态标签 | 每次执行都用 OGNL 求值、拼 SQL | 每次有解析开销 |
| `RawSqlSource` | 纯静态 SQL（只有 `#{}`） | 构造时已把 `#{}` 解析成 `?`，执行时直接返回缓存 | 无额外开销 |
| `StaticSqlSource` | 上面两者的最终产物 | 直接返回固定的 SQL 与参数映射 | 最快 |

```java
// XMLLanguageDriver.createSqlSource 的判定逻辑（简化）
if (scripting) {
    return new DynamicSqlSource(configuration, rootSqlNode);   // 含动态标签
}
if (hasPlaceholder) {
    return new RawSqlSource(configuration, sql, parameterType); // 只有 #{}
}
return new StaticSqlSource(configuration, sql, parameterType);
```

**工程含义**：`<if>` 用得多，SQL 就一定是 `DynamicSqlSource`，每次执行都要过一遍 OGNL + `SqlNode.apply`。这不是瓶颈（微秒级），但在「单条 SQL 执行本身只要 200μs」的高频场景里能占到可观比例——这就是有的团队把「固定条件」从 `<if>` 里拿出来拼成两条 statement 的原因。

### 3.5 `BoundSql` 里到底有什么

```java
public class BoundSql {
    private final String sql;                                  // 带 ? 的 SQL
    private final List<ParameterMapping> parameterMappings;    // 每个 ? 对应一个
    private final Object parameterObject;                       // 原始参数对象
    private final Map<String, Object> additionalParameters;    // foreach/bind 产生的临时变量
    private final MetaObject metaParameters;
}
```

三个字段的用途经常被搞混：`parameterMappings` 决定「第 n 个 `?` 从哪取值、用什么 TypeHandler」；`additionalParameters` 装的是 `_frch_item_0`、`_parameter`、`<bind>` 变量这类**运行时生成**的名字；`parameterObject` 在 `ParamMap` 场景下是 `MapperMethod.ParamMap`，不是你的原始 DTO。

---

## 四、高级（最完善）：执行链路的源码真相

### 4.1 `MapperProxy` 三条分支 + `MapperMethod` 分派 + 参数名来源

`MapperProxy.invoke` 有**三条分支**：

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
        if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, args);        // ① equals/hashCode/toString
        }
        return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
    } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
    }
}
```

第 ① 条常被忽略：`mapper.toString()` 不会去查库，直接反射调用代理自身。第 ② 条是 `DefaultMethodInvoker`（`method.isDefault()` 时反射调用接口默认实现，3.4.2+）。第 ③ 条 `PlainMethodInvoker` 才是真正的 `MapperMethod.execute` 入口。

`MapperMethod` 在构造时解析出 `SqlCommand`（`statementId` + `SqlCommandType`）与 `MethodSignature`，`execute` 按类型分派：

| 情况 | 走的方法 | 返回值处理 |
| :--- | :--- | :--- |
| INSERT / UPDATE / DELETE | `sqlSession.insert/update/delete` | `void`→null，`int`→影响行数，`boolean`→`>0`，`long`→行数 |
| SELECT，返回集合 | `selectList` | 直接返回 |
| SELECT，单值 | `selectOne` | 多行会抛 `TooManyResultsException` |
| SELECT，返回 `Map` | `selectMap` | 需要 `@MapKey` |
| SELECT，返回 `Cursor` | `selectCursor` | 流式，调用方负责关闭 |

**容易忽略的细节**：`selectOne` 的多行校验发生在 `DefaultSqlSession`——先按 `selectList` 查，再判断 `size == 1`。所以「`selectOne` 报 TooManyResults」不是 SQL 写错了，是数据真的有多条。

`ParamNameResolver` 的参数名有四种来源（优先级从高到低）：`@Param("name")` 注解 → 编译时保留的真实参数名（`-parameters`）→ 参数索引名 `param1`/`param2` → 单参数特例直接用。参数取不到时抛的异常非常具体：

```text
BindingException: Parameter 'userId' not found.
Available parameters are [arg0, arg1, param1, param2]
```

看到这个异常，先检查三件事：`@Param` 有没有漏、XML 里名字有没有拼错、`-parameters` 编译参数有没有开（老项目常常没开）。

### 4.2 `DefaultSqlSession` 的四个反直觉语义

`DefaultSqlSession` 只做三件事：按 `statementId` 找 `MappedStatement`、委托 `Executor`、维护事务状态。但它有四个语义决定了行为：

**语义一：`commit()` 不是无条件提交**

```java
private boolean isCommitOrRollbackRequired(boolean force) {
    return (!autoCommit && dirty) || force;
}
```

`autoCommit=true` 且没有写操作时，`commit()` 什么也不做。反过来，`autoCommit=false` 且有写操作时，**不 commit 就会在 `close()` 时回滚**——这就是「insert 成功但查不到」的源码答案。

**语义二：`close()` 会隐式回滚**。`Executor.close(boolean forceRollback)` 在 `BaseExecutor` 里判断 `forceRollback` 决定 `rollback` 还是 `commit`。所以「try-with-resources 里忘了 commit」不是丢弃语句，而是明确回滚。

**语义三：`update` 才置 `dirty`**。`insert`/`update`/`delete` 最终都调 `update(String, Object)`，成功返回行数并 `dirty = true`；查询不置脏——所以「只读会话」不会触发多余提交。

**语义四：`BATCH` 模式下 `commit` 会先 flush**。`BaseExecutor.commit` 是 `flushStatements()` 之后 `transaction.commit()`。BATCH 模式下如果没有这一步，`addBatch` 的语句永远不会发到数据库。

### 4.3 `openSessionFromDataSource` 与 `newExecutor` 源码

```java
private SqlSession openSessionFromDataSource(ExecutorType execType,
        TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = transactionFactory.newTransaction(
            environment.getDataSource(), level, autoCommit);   // 此时并不真正拿连接
    final Executor executor = configuration.newExecutor(tx, execType);
    return new DefaultSqlSession(configuration, executor, autoCommit);
}
```

**注意**：`transactionFactory.newTransaction()` 只是创建 `Transaction` 对象，真正获取数据库连接是在执行 SQL 时懒加载的——`JdbcTransaction.getConnection()` 里 `if (connection == null) openConnection()`。所以「`openSession` 慢」不会发生在连接池耗尽时；真正报 `Could not get JDBC Connection` 的是第一次查询。

```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    // 按类型选 Simple/Reuse/Batch
    Executor executor = ...;
    if (cacheEnabled) {
        executor = new CachingExecutor(executor);     // 二级缓存装饰
    }
    executor = (Executor) interceptorChain.pluginAll(executor);  // 插件链
    return executor;
}
```

两个要点：`cacheEnabled` 默认 `true`，所以默认所有 `Executor` 都被 `CachingExecutor` 包装——但它是否真正用二级缓存，取决于每个 `MappedStatement` 是否配置了 `<cache/>`；插件链在缓存**外层**（先包 `CachingExecutor` 再 `pluginAll`），所以插件（如分页）改写后的 SQL 会参与缓存 key。

### 4.4 三种执行器的行为差异

| 执行器 | Statement 复用 | 批量 | 典型场景 | 主要陷阱 |
| :--- | :--- | :--- | :--- | :--- |
| `SimpleExecutor` | 每次新建、用完关闭 | 不支持 | 默认，绝大多数查询 | 无 |
| `ReuseExecutor` | 按 SQL 文本缓存 `Statement`，同会话内复用 | 不支持 | 同会话重复执行相同 SQL | 忘了 `flushStatements()`；`Statement` 长期占用 |
| `BatchExecutor` | 复用同一个 `Statement` | 支持 | 大批量写入 | 必须在事务内；返回行数不准；与 `selectKey` 冲突 |

`BatchExecutor.doUpdate` 有个实际存在、文档里不显眼的常量：`BATCH_UPDATE_RETURN_VALUE = -2147482646`。所以 BATCH 模式下 `insert` 的返回值没有业务意义，不能用来判断成功。

### 4.5 `BaseExecutor` 模板方法与一级缓存源码

`BaseExecutor.query` 是模板方法，把一级缓存插在中间：

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds,
        ResultHandler resultHandler, CacheKey key, BoundSql boundSql) {
    if (closed) throw new ExecutorException("Executor was closed.");
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
        clearLocalCache();                     // flushCache=true 先清一级缓存
    }
    List<E> list;
    try {
        queryStack++;
        list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
        if (list != null) {
            handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
        } else {
            list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
        }
    } finally {
        queryStack--;
    }
    if (queryStack == 0) {
        for (DeferredLoad deferredLoad : deferredLoads) deferredLoad.load();
        deferredLoads.clear();
        if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
            clearLocalCache();                 // STATEMENT 级别：一次查询后立刻清
        }
    }
    return list;
}
```

三个设计要点：

1. **`queryStack` 防嵌套污染**：嵌套查询（association 的 `select`）发生在父查询尚未完成时，此时若把子查询结果写入缓存，可能缓存到不完整的对象。`queryStack > 0` 时不做缓存清理。
2. **`localCacheScope=STATEMENT` 不是「关闭缓存」**：它仍在同一次查询内生效，只是查完立刻清。真正的「关闭」做不到。
3. **`flushCache=true` 只清当前会话的一级缓存**，不影响二级缓存（二级缓存由 `CachingExecutor` 的 `tcm.clear()` 处理）。

一级缓存的 Key 由 `BaseExecutor.createCacheKey()` 构造，包含**五个因素**：Mapper 的 Id（namespace + id）、`RowBounds`（偏移量与条数）、SQL 语句（最终执行的 SQL，含动态 SQL 拼接结果）、SQL 参数、Environment ID。任何一个变了，缓存就不命中。

### 4.6 `CachingExecutor` 与二级缓存源码 + 事务预缓存

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds,
        ResultHandler resultHandler, CacheKey key, BoundSql boundSql) {
    Cache cache = ms.getCache();
    if (cache != null) {
        flushCacheIfRequired(ms);
        if (ms.isUseCache() && resultHandler == null) {
            ensureNoOutParams(ms, boundSql);
            List<E> list = (List<E>) tcm.getObject(cache, key);
            if (list == null) {
                list = delegate.query(ms, parameter, rowBounds, resultHandler, key, boundSql);
                tcm.putObject(cache, key, list);  // 放进「待提交」缓冲区，不是直接写
            }
            return list;
        }
    }
    return delegate.query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```

四个必须记住的细节：`resultHandler != null` 时不走缓存（流式/逐行处理的结果无法复用）；`ensureNoOutParams`（有 `OUT` 参数的存储过程直接跳过二级缓存）；`tcm.putObject` 只是暂存，真正的写入发生在事务提交（`tcm.commit()`），回滚则丢弃——这是「二级缓存必须提交事务才生效」的源码依据；`<insert>/<update>/<delete>` 默认 `flushCache=true`，会清空该 namespace 的二级缓存。

二级缓存的关键设计是**事务预缓存**（`TransactionalCache`）：

```java
public class TransactionalCache implements Cache {
    private final Cache delegate;                       // 真正的二级缓存
    private boolean clearOnCommit;                      // 提交时是否清空
    private final Map<Object, Object> entriesToAddOnCommit; // 待提交的缓存条目
    private final Set<Object> entriesMissedInCache;     // 未命中的 key（防穿透）

    @Override
    public void putObject(Object key, Object value) {
        entriesToAddOnCommit.put(key, value);   // 不直接写二级缓存
    }

    public void commit() {
        if (clearOnCommit) delegate.clear();
        flushPendingEntries();                  // 提交时才真正写
        reset();
    }
}
```

这种设计的核心目的是**保证二级缓存的事务一致性**：事务提交前，其他事务看不到未提交的缓存数据；事务回滚，缓存数据也不被污染。

### 4.7 `CacheKey` 算法

`CacheKey` 是缓存的「身份证」，通过 `hashcode`、`checksum`、`count` 三个字段快速排除，最后逐元素比较防止哈希碰撞：

```java
public void update(Object object) {
    int baseHashCode = object == null ? 1 : ArrayUtil.hashCode(object);
    count++;
    checksum += baseHashCode;
    baseHashCode *= count;
    hashcode = multiplier * hashcode + baseHashCode;   // multiplier=37, 初始 17
    updateList.add(object);
}
```

### 4.8 `StatementHandler` 家族

`RoutingStatementHandler` 按 `ms.getStatementType()`（默认 `PREPARED`）选择：

| 类型 | 实现 | 用途 |
| :--- | :--- | :--- |
| `STATEMENT` | `SimpleStatementHandler` | 无参数、无预编译 |
| `PREPARED`（默认） | `PreparedStatementHandler` | 有参数、预编译、防注入 |
| `CALLABLE` | `CallableStatementHandler` | 存储过程，处理 `OUT` 参数 |

注意 `StatementHandler` 也会被插件包装一次——所以同一个插件如果同时声明了 `Executor` 与 `StatementHandler` 的签名，**同一次查询会被调用两次**。这是插件计数类逻辑出错的常见原因。

### 4.9 插件机制：四大扩展点 + 匹配规则 + 分页插件实战

**可拦截的四个接口**（只有这四个）：

| 接口 | 能做什么 | 典型用途 |
| :--- | :--- | :--- |
| `Executor` | 改查询流程、改 `BoundSql`、拦 `update`/`query`/`commit` | 分页、多租户、SQL 审计、读写分离 |
| `StatementHandler` | 改 SQL、改 `prepare` 行为、设置 `fetchSize` | 分页、强制走主库、超时控制 |
| `ParameterHandler` | 改参数绑定 | 敏感字段加密、参数脱敏 |
| `ResultSetHandler` | 改结果映射 | 字段解密、结果过滤 |

匹配机制的源码在 `Plugin.wrap` 与 `Plugin.invoke`：

```java
public static Object wrap(Object target, Interceptor interceptor) {
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    if (interfaces.length > 0) {
        return Proxy.newProxyInstance(type.getClassLoader(), interfaces,
                new Plugin(target, interceptor, signatureMap));
    }
    return target;   // 没有匹配的接口就不包装
}

public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Set<Method> methods = signatureMap.get(method.getDeclaringClass());
    if (methods != null && methods.contains(method)) {
        return interceptor.intercept(new Invocation(target, method, args));
    }
    return method.invoke(target, args);      // 不匹配的方法直接放行
}
```

**`@Signature` 的三个必填项一个都不能错**：`type` 必须是四个接口之一、`method` 必须是该接口声明的方法名、`args` 必须是**精确的参数类型数组**。写成父类（如 `Object.class` 代替 `RowBounds.class`）不会报错，但永远不匹配——插件静默失效的头号原因。

**执行顺序**：`pluginAll` 按注册顺序依次 wrap，**后注册的在外层、先被调用**：

```text
InterceptorB（后注册，最外层）
  └─ InterceptorA
      └─ CachingExecutor
          └─ SimpleExecutor
```

分页插件能改 SQL 的原理：拦截 `Executor.query`，取原 SQL 拼 `LIMIT`，用改写后的 `BoundSql` 重新构造 `MappedStatement` 再继续执行。**四个坑**：① 必须复制 `additionalParameters`（`new BoundSql(...)` 不会带上 `foreach`/`<bind>` 变量，否则 `#{__frch_...}` 绑定失败）；② count 语句要剥离 `ORDER BY`；③ 一级缓存 key 会包含改写后的 SQL，不同页各自缓存一份；④ 拦截 `Executor.query` 的**两个重载都要声明**（只声明 4 参数版本时，`CachingExecutor` 内部调用 6 参数版本不会经过插件，表现为「第一次查有分页、命中缓存后没分页」）。

### 4.10 Spring 集成：`SqlSessionTemplate` 线程安全 + 事务对齐

核心问题只有一个：**怎么让 MyBatis 的 `SqlSession` 与 Spring 的事务边界对齐**。答案是一条链：

```text
SqlSessionTemplate（线程安全门面）
  → SqlSessionInterceptor（JDK 动态代理）
    → SqlSessionUtils.getSqlSession()（从 ThreadLocal 取或新建）
      → SqlSessionHolder（引用计数）
        → SqlSessionSynchronization（事务回调）
```

`SqlSessionUtils.getSqlSession()` 的完整逻辑：先从 `TransactionSynchronizationManager.getResource(sessionFactory)`（一个 `ThreadLocal`）取 `SqlSessionHolder`，存在则复用；不存在则 `openSession` 新建，若当前有活跃事务则注册到事务同步器，无事务则用完即关。

`SqlSessionHolder` 的引用计数（`requested()`/`released()`）解决「事务中多个 Mapper 调用嵌套获取同一个 `SqlSession`」的问题：每次 `getSqlSession()` 时计数 +1，每次 `closeSqlSession()` 时 -1，只有归零才真正关闭。

有事务与无事务的行为对比：

| 场景 | SqlSession 来源 | 提交方式 | 关闭方式 | 一级缓存 |
|---|---|---|---|---|
| 无事务 | 每次 `getSqlSession()` 新建 | `sqlSession.commit(true)` | 真正关闭 | 不生效 |
| 有事务 | 事务内复用同一个 | Spring 事务管理器在 `afterCompletion` 中 commit | `SqlSessionSynchronization` 触发关闭 | 生效 |

**`SpringManagedTransactionFactory` 的必要性**：`registerSessionHolder()` 中会校验——如果不是 `SpringManagedTransactionFactory`，直接抛 `TransientDataAccessResourceException`。因为只有 `SpringManagedTransaction` 才能与 `DataSourceUtils.getConnection()` 协同，确保 `SqlSession` 获取的连接与 Spring 事务管理的连接是同一个；用 `JdbcTransactionFactory` 会导致每个 `SqlSession` 从连接池取不同连接，事务失效。

### 4.11 批量写入、主键回填、深分页

**三种批量写入方式的取舍**：

| 方式 | 往返次数 | 适用量级 | 主要限制 |
| :--- | :--- | :--- | :--- |
| `foreach` 拼多值 INSERT | 1 | 百~千行 | SQL 长度受 `max_allowed_packet`；Oracle 不支持多值 VALUES |
| `ExecutorType.BATCH` | 分批 | 万~十万行 | 必须在事务内；返回值不准；不能立即读回 |
| JDBC 原生 `addBatch` | 分批 | 极大批量 | 脱离 MyBatis 的映射能力 |

`ExecutorType.BATCH` 的正确用法（Spring 下）：配一个 `SqlSessionTemplate(factory, ExecutorType.BATCH)` 的 Bean，方法上加 `@Transactional`，循环 `insert`（只是 `addBatch`），事务提交时由 `BaseExecutor.commit → flushStatements` 统一发送。

**四个必须知道的约束**：① 必须在事务内（无事务时每次 Mapper 调用后 `SqlSession` 被关闭，`close` 触发 `flushStatements`，等于每行一次往返）；② `insert` 返回值是 `-2147482646`，不能判断成功；③ 与 `<selectKey order="AFTER">` 冲突（`AFTER` 依赖语句执行后立即取主键，而 BATCH 尚未执行）；④ BATCH 下不能「写了立刻查」。

MySQL 驱动的一个关键参数：

```yaml
url: jdbc:mysql://...?rewriteBatchedStatements=true
```

不开这个参数，`addBatch` 会被驱动逐条发送，BATCH 的性能优势基本消失——这是「明明用了 BATCH 还是慢」的最常见原因。

**主键回填的三种方式**：`useGeneratedKeys`（`useGeneratedKeys="true" keyProperty="id"`，MySQL/SQL Server 自增）、`<selectKey order="BEFORE">`（Oracle 序列）、`<selectKey order="AFTER">`（插入后查 `last_insert_id`，BATCH 下不可用）。批量回填有个隐蔽的生产问题：`Jdbc3KeyGenerator.processBatch` 依赖 `BatchResult.getParameterObjects()` 与 `getGeneratedKeys` 的顺序一致，部分旧版 Oracle 驱动不保证顺序时，批量回填会错位。

**深分页的两条路**：逻辑分页（`RowBounds`）不会改 SQL，`DefaultResultSetHandler` 在遍历结果集时跳过前 `offset` 行、取 `limit` 行——**数据库仍然返回全部结果**，一百万行的表查第 5 万页等于把一百万行全传回来再丢掉，生产基本不可用。物理分页（插件）改写 SQL 成 `LIMIT offset, size` 并额外执行 `COUNT`。深分页三种优化：

| 方案 | SQL 形态 | 适用 |
| :--- | :--- | :--- |
| 延迟关联 | `SELECT * FROM t JOIN (SELECT id FROM t ORDER BY id LIMIT 100000,20) x USING(id)` | 有覆盖索引，减少回表 |
| 游标/书签 | `WHERE id > #{lastId} ORDER BY id LIMIT 20` | 顺序翻页、无随机跳页 |
| 预计算分页 | 把页码与边界 id 存到缓存/中间表 | 数据变化不频繁 |

---

## 五、生产实践与收口

### 5.1 十个生产坑（按踩中概率排序）

| # | 坑 | 现象 | 根因 | 处理 |
| :--- | :--- | :--- | :--- | :--- |
| 1 | 二级缓存脏读 | 多表关联查询后，另一张表更新了，缓存仍是旧值 | 二级缓存按 namespace 失效，**无法感知其他表的变更** | 默认关闭；只对单表、只读、变更少的表开；`<cache-ref>` 慎用 |
| 2 | 一级缓存读到旧值 | 同一事务内先查、再被别的服务改、再查还是旧值 | 同 `SqlSession` 内 `localCache` 命中 | 关键读加 `flushCache="true"` 或 `useCache="false"` |
| 3 | 无事务时一级缓存「不生效」 | 两次相同查询打了两次库 | 每次 Mapper 调用新建 `SqlSession` | 这是预期行为；要复用需同一事务 |
| 4 | 深分页拖垮库 | 第 N 页响应几十秒 | `RowBounds` 逻辑分页 / `LIMIT` 大偏移 | 游标或延迟关联；限制最大页 |
| 5 | 大结果集 OOM | 导出功能堆内存暴涨 | 一次性 `selectList` 全量加载 | `fetchSize` + `Cursor` 流式；或分片导出 |
| 6 | `${}` 注入 | 排序/表名参数被拼接 | 把用户输入直接放进 `${}` | 白名单枚举，禁止透传 |
| 7 | `IN` 超限 | Oracle `ORA-01795` | `IN` 列表上限 1000 | 分批 500/批，或临时表关联 |
| 8 | 插件静默失效 | 自定义插件不执行 | `@Signature` 的 `args` 类型不精确 | 对照接口签名逐个核对 |
| 9 | 批量插入慢 | 用了 BATCH 仍是逐条 | 无事务 / 未开 `rewriteBatchedStatements` | 加事务 + 开参数 |
| 10 | 多数据源事务不生效 | 跨库操作部分回滚 | 事务管理器绑定到了错误的 `DataSource` | 每套数据源配独立的 `SqlSessionFactory` + `DataSourceTransactionManager` |

### 5.2 排查手段

| 手段 | 配置 | 能看到什么 |
| :--- | :--- | :--- |
| 内置日志 | `logImpl: SLF4J` + mapper 包 debug | 最终 SQL、参数、返回行数、连接获取与释放 |
| `p6spy` | 替换驱动为 `P6SpyDriver` | 带真实参数的 SQL、执行耗时 |
| `ErrorContext` | 无需配置 | 异常信息中的 `### SQL`、`### Cause` |
| Dump `Configuration` | 启动时遍历 `getMappedStatements()` | 确认语句是否注册、id 是否与接口对得上 |
| 慢 SQL 日志 | MySQL `slow_query_log` / Oracle AWR | 定位到具体语句后再回到 Mapper |

一个实用技巧：启动时打印所有 `MappedStatement` 的 id 与 `SqlCommandType`，能一次性发现「XML 没被扫到」「id 写错」「重载冲突」三类问题，比运行时报错再回头找快得多。

### 5.3 全篇结论的边界与代价

**这些结论不是无条件成立的，钉死七条边界：**

1. **「MyBatis 防 SQL 注入」只在 `#{}` 成立**。`${}` 用于表名/排序字段时，白名单校验必须放在 Java 层，否则注入照样发生。
2. **「一级缓存会生效」只在有事务时成立**。无事务时每次 Mapper 调用新建 `SqlSession`，缓存等于没有——这个反直觉点是高频误判源。
3. **「二级缓存能加速」只在「单表、只读、单机、变更少」四条件同时满足时成立**。分布式部署或多表关联时，它按 namespace 失效的特性反而制造脏读，默认应关闭。
4. **「`localCacheScope=STATEMENT` 关闭缓存」是错的**。它只是「查完立刻清」，不是关闭，真正的「关闭一级缓存」做不到。
5. **「BATCH 能加速批量写入」依赖两个前提**：在事务内 + MySQL 开了 `rewriteBatchedStatements`。缺任一，都是逐条发送。
6. **「分页插件万能」不成立**。它的 `COUNT` 必须剥离 `ORDER BY`、处理 `GROUP BY`/`DISTINCT` 口径，一对多 join 场景总数天然偏大。
7. **「插件能拦截」只在 `@Signature` 三个字段精确匹配时成立**，且 `Executor.query` 的两个重载要同时声明，否则命中缓存后插件就"消失"了。

**代价视角收口**：MyBatis 的全部优势（SQL 可控、映射自动化、插件可扩展）都建立在「半自动」这个取舍上——你要为它付出的代价是：**手写 SQL 带来的重复劳动、动态 SQL 静默失效的排查成本、以及一二级缓存这个「默认开着却最容易踩坑」的设计**。理解这条执行链路，不是为了背源码，而是为了在踩到这些坑时，能立刻定位到是 `MapperProxy`、`Executor`、还是 `CacheKey` 的哪一环出了问题。

---

> 本文以 MyBatis 3.5.x 与 mybatis-spring 2.x/3.x 源码为校对来源。全量版覆盖映射解析、Mapper 代理、参数与结果映射、动态 SQL、SqlSession 与 Executor、插件链、一二级缓存、Spring 集成、批量/分页/主键回填、生产坑与排查。相关更深入的口答题卡（含追问链）见知识库「口答题卡」系列。
