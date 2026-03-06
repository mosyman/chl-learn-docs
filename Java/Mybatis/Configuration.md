
## 目录
- [Configuration](#configuration)
- [newExecutor](#newexecutor)



## Configuration

### 1. 核心需求理解
你希望我详细解释 MyBatis 核心的 `Configuration` 类——这个类是 MyBatis 整个框架的**全局配置容器**，所有核心配置（如环境、缓存、映射器、类型处理器、插件等）都集中存储在这里，是 MyBatis 初始化和运行的核心中枢。

### 2. 详细代码解释
#### 2.1 类的整体定位
```java
public class Configuration { ... }
```
- **核心角色**：MyBatis 的“配置中心”，相当于框架的“大脑”——
    1. 存储所有全局配置项（如 `mapUnderscoreToCamelCase`、`cacheEnabled` 等）；
    2. 管理核心组件注册表（如映射器、类型处理器、别名、插件等）；
    3. 提供默认配置（构造方法中注册常用别名、默认组件）；
    4. 作为 MyBatis 各组件（Executor、StatementHandler 等）的配置来源。
- **设计特点**：采用“大而全”的设计，聚合所有配置，便于各组件统一访问；大部分属性为 `protected`，支持子类扩展（如自定义 Configuration）。

#### 2.2 基础配置属性（全局开关/默认值）
这部分属性对应 `mybatis-config.xml` 中的 `<settings>` 节点，是 MyBatis 最基础的行为控制开关：
```java
// 环境配置（包含数据源、事务管理器）
protected Environment environment;

// 安全开关：是否允许使用 RowBounds 分页（防止内存溢出）
protected boolean safeRowBoundsEnabled;
// 安全开关：是否允许使用 ResultHandler（防止内存溢出），默认开启
protected boolean safeResultHandlerEnabled = true;
// 是否开启下划线转驼峰命名（如 user_name → userName）
protected boolean mapUnderscoreToCamelCase;
// 是否开启激进的懒加载（已废弃，3.4.1 后用 lazyLoadingEnabled + aggressiveLazyLoading 替代）
protected boolean aggressiveLazyLoading;
// 是否允许使用自增主键（insert 时自动回填主键）
protected boolean useGeneratedKeys;
// 是否使用列标签（Column Label）而非列名（Column Name），默认开启
protected boolean useColumnLabel = true;
// 是否开启全局缓存，默认开启
protected boolean cacheEnabled = true;
// 是否允许对 null 值调用 setter 方法（如结果集中的 null 字段是否赋值给实体）
protected boolean callSettersOnNulls;
// 是否使用参数的实际名称（如方法参数名），默认开启
protected boolean useActualParamName = true;
// 空结果集时是否返回空实例（而非 null）
protected boolean returnInstanceForEmptyRow;
// 是否压缩 SQL 中的空白字符（如换行、空格）
protected boolean shrinkWhitespacesInSql;
// foreach 标签中是否允许 null 值
protected boolean nullableOnForEach;
// 是否基于构造方法参数名自动映射
protected boolean argNameBasedConstructorAutoMapping;

// 日志前缀（如 MyBatis 日志输出时的前缀）
protected String logPrefix;
// 日志实现类（如 Slf4jImpl、Log4j2Impl）
protected Class<? extends Log> logImpl;
// VFS（虚拟文件系统）实现类（用于加载资源）
protected Class<? extends VFS> vfsImpl;
// 默认 SQL 提供者类型（注解式 SQL 用）
protected Class<?> defaultSqlProviderType;
// 本地缓存作用域：SESSION（会话级，默认）/STATEMENT（语句级）
protected LocalCacheScope localCacheScope = LocalCacheScope.SESSION;
// null 值对应的 JDBC 类型，默认 OTHER
protected JdbcType jdbcTypeForNull = JdbcType.OTHER;
// 触发懒加载的方法（调用这些方法会触发懒加载），默认 equals/clone/hashCode/toString
protected Set<String> lazyLoadTriggerMethods = new HashSet<>(
    Arrays.asList("equals", "clone", "hashCode", "toString"));
// 默认语句超时时间（JDBC Statement 超时）
protected Integer defaultStatementTimeout;
// 默认 fetchSize（JDBC 读取结果集的批次大小）
protected Integer defaultFetchSize;
// 默认结果集类型（FORWARD_ONLY/SCROLL_SENSITIVE/SCROLL_INSENSITIVE）
protected ResultSetType defaultResultSetType;
// 默认执行器类型：SIMPLE（简单执行器，默认）/REUSE（复用预处理语句）/BATCH（批处理）
protected ExecutorType defaultExecutorType = ExecutorType.SIMPLE;
// 自动映射行为：PARTIAL（默认，只映射非嵌套的属性）/FULL（全量映射）/NONE（关闭）
protected AutoMappingBehavior autoMappingBehavior = AutoMappingBehavior.PARTIAL;
// 自动映射未知列的行为：NONE（默认，忽略）/WARNING（警告）/FAILING（失败）
protected AutoMappingUnknownColumnBehavior autoMappingUnknownColumnBehavior = AutoMappingUnknownColumnBehavior.NONE;

// 全局变量（如 properties 配置的参数，用于替换 SQL 中的 ${} 占位符）
protected Properties variables = new Properties();
// 反射工厂（默认 DefaultReflectorFactory，封装类的反射信息）
protected ReflectorFactory reflectorFactory = new DefaultReflectorFactory();
// 对象工厂（创建结果集映射的实体对象，默认 DefaultObjectFactory）
protected ObjectFactory objectFactory = new DefaultObjectFactory();
// 对象包装工厂（封装对象属性访问，默认 DefaultObjectWrapperFactory）
protected ObjectWrapperFactory objectWrapperFactory = new DefaultObjectWrapperFactory();

// 是否开启懒加载
protected boolean lazyLoadingEnabled;
// 代理工厂（创建懒加载代理对象，默认 Javassist 实现）
protected ProxyFactory proxyFactory = new JavassistProxyFactory();

// 数据库标识（如 mysql/oracle，用于多数据库适配）
protected String databaseId;
// 配置工厂类（用于反序列化时加载未读属性）
protected Class<?> configurationFactory;
```
- **关键属性说明**：
    - `mapUnderscoreToCamelCase`：MyBatis 最常用的配置之一，解决数据库下划线命名与 Java 驼峰命名的自动映射；
    - `cacheEnabled`：全局二级缓存开关，关闭后所有 Mapper 的二级缓存都失效；
    - `localCacheScope`：一级缓存作用域，SESSION 级缓存会跨 SQL 语句生效，STATEMENT 级仅当前语句生效；
    - `lazyLoadingEnabled`：懒加载核心开关，开启后关联对象（如订单的用户）会延迟加载，直到调用其方法。

#### 2.3 核心注册表（管理框架组件）
这部分是 `Configuration` 的核心，存储 MyBatis 所有核心组件的注册表，对应 `mybatis-config.xml` 中的 `<mappers>`、`<typeHandlers>`、`<plugins>` 等节点：
```java
// Mapper 注册表：管理所有 Mapper 接口（key=接口全类名，value=MapperProxyFactory）
protected final MapperRegistry mapperRegistry = new MapperRegistry(this);

// 拦截器链：管理所有插件（Interceptor），执行 SQL 时按顺序拦截
protected final InterceptorChain interceptorChain = new InterceptorChain();

// 类型处理器注册表：管理 Java 类型 ↔ JDBC 类型的转换处理器
protected final TypeHandlerRegistry typeHandlerRegistry = new TypeHandlerRegistry(this);

// 类型别名注册表：管理自定义别名（如 User → com.example.User）
protected final TypeAliasRegistry typeAliasRegistry = new TypeAliasRegistry();

// 语言驱动注册表：管理 SQL 解析驱动（默认 XML，支持 RAW 原生 SQL）
protected final LanguageDriverRegistry languageRegistry = new LanguageDriverRegistry();
```
- **注册表作用**：
    - `MapperRegistry`：MyBatis 实现接口式 Mapper 的核心，通过动态代理生成 Mapper 实例；
    - `TypeHandlerRegistry`：默认注册了所有基础类型的处理器，也支持自定义（如 LocalDateTime ↔ JDBC Timestamp）；
    - `InterceptorChain`：插件的执行入口，MyBatis 分页插件、通用 Mapper 等都通过这里注册。

#### 2.4 运行时缓存（存储解析后的 SQL/映射配置）
这部分是 MyBatis 解析 XML/注解后生成的运行时对象，是框架执行 SQL 的直接依据：
```java
// MappedStatement 缓存：key=MapperId（如 com.example.UserMapper.selectById），value=SQL 执行配置
protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>(
    "Mapped Statements collection")
        .conflictMessageProducer((savedValue, targetValue) -> ". please check " + savedValue.getResource() + " and "
            + targetValue.getResource());
// 缓存缓存：key=命名空间，value=Cache 实例（二级缓存）
protected final Map<String, Cache> caches = new StrictMap<>("Caches collection");
// 结果集映射缓存：key=resultMapId，value=ResultMap（结果集→实体的映射规则）
protected final Map<String, ResultMap> resultMaps = new StrictMap<>("Result Maps collection");
// 参数映射缓存：key=parameterMapId，value=ParameterMap（参数→SQL 的映射规则）
protected final Map<String, ParameterMap> parameterMaps = new StrictMap<>("Parameter Maps collection");
// 主键生成器缓存：key=语句 ID，value=KeyGenerator（如自增主键生成器）
protected final Map<String, KeyGenerator> keyGenerators = new StrictMap<>("Key Generators collection");

// 已加载的资源：记录已解析的 Mapper XML/配置文件，防止重复加载
protected final Set<String> loadedResources = new HashSet<>();
// SQL 片段缓存：存储 Mapper XML 中的 <sql> 片段（如 <sql id="baseColumns">id,name</sql>）
protected final Map<String, XNode> sqlFragments = new StrictMap<>("XML fragments parsed from previous mappers");

// 未完成的语句：解析过程中暂存的未完成 MappedStatement（解决循环依赖）
protected final Collection<XMLStatementBuilder> incompleteStatements = new LinkedList<>();
// 未完成的缓存引用：暂存未解析的 <cache-ref> 节点
protected final Collection<CacheRefResolver> incompleteCacheRefs = new LinkedList<>();
// 未完成的结果集映射：暂存未解析的 <resultMap> 节点
protected final Collection<ResultMapResolver> incompleteResultMaps = new LinkedList<>();
// 未完成的方法：暂存未解析的 Mapper 方法
protected final Collection<MethodResolver> incompleteMethods = new LinkedList<>();

// 并发锁：保护未完成资源的线程安全（解析时可能多线程）
private final ReentrantLock incompleteResultMapsLock = new ReentrantLock();
private final ReentrantLock incompleteCacheRefsLock = new ReentrantLock();
private final ReentrantLock incompleteStatementsLock = new ReentrantLock();
private final ReentrantLock incompleteMethodsLock = new ReentrantLock();

// 缓存引用映射：key=引用缓存的命名空间，value=实际缓存的命名空间（<cache-ref namespace="xxx"/>）
protected final Map<String, String> cacheRefMap = new HashMap<>();
```
- **关键设计**：
    - `StrictMap`：MyBatis 自定义的 Map，当 key 重复时抛出明确的异常（如两个 Mapper 定义了相同的 `selectById`），便于定位冲突；
    - 未完成资源集合（`incompleteXXX`）：解决 Mapper 解析中的循环依赖（如 A ResultMap 引用 B，B 又引用 A），先暂存未完成的资源，解析完所有资源后再统一处理；
    - 并发锁：解析配置时可能多线程（如 Spring 集成时），通过 ReentrantLock 保证未完成资源的线程安全。

#### 2.5 构造方法（初始化默认配置）
```java
// 带环境的构造：创建 Configuration 并设置环境（数据源+事务）
public Configuration(Environment environment) {
  this();
  this.environment = environment;
}

// 无参构造：注册默认别名和默认组件
public Configuration() {
  // 事务工厂别名
  typeAliasRegistry.registerAlias("JDBC", JdbcTransactionFactory.class);
  typeAliasRegistry.registerAlias("MANAGED", ManagedTransactionFactory.class);

  // 数据源工厂别名
  typeAliasRegistry.registerAlias("JNDI", JndiDataSourceFactory.class);
  typeAliasRegistry.registerAlias("POOLED", PooledDataSourceFactory.class);
  typeAliasRegistry.registerAlias("UNPOOLED", UnpooledDataSourceFactory.class);

  // 缓存实现别名
  typeAliasRegistry.registerAlias("PERPETUAL", PerpetualCache.class);
  typeAliasRegistry.registerAlias("FIFO", FifoCache.class);
  typeAliasRegistry.registerAlias("LRU", LruCache.class);
  typeAliasRegistry.registerAlias("SOFT", SoftCache.class);
  typeAliasRegistry.registerAlias("WEAK", WeakCache.class);

  // 数据库标识提供者别名
  typeAliasRegistry.registerAlias("DB_VENDOR", VendorDatabaseIdProvider.class);

  // 语言驱动别名
  typeAliasRegistry.registerAlias("XML", XMLLanguageDriver.class);
  typeAliasRegistry.registerAlias("RAW", RawLanguageDriver.class);

  // 日志实现别名
  typeAliasRegistry.registerAlias("SLF4J", Slf4jImpl.class);
  typeAliasRegistry.registerAlias("COMMONS_LOGGING", JakartaCommonsLoggingImpl.class);
  typeAliasRegistry.registerAlias("LOG4J", Log4jImpl.class);
  typeAliasRegistry.registerAlias("LOG4J2", Log4j2Impl.class);
  typeAliasRegistry.registerAlias("JDK_LOGGING", Jdk14LoggingImpl.class);
  typeAliasRegistry.registerAlias("STDOUT_LOGGING", StdOutImpl.class);
  typeAliasRegistry.registerAlias("NO_LOGGING", NoLoggingImpl.class);

  // 代理工厂别名
  typeAliasRegistry.registerAlias("CGLIB", CglibProxyFactory.class);
  typeAliasRegistry.registerAlias("JAVASSIST", JavassistProxyFactory.class);

  // 设置默认语言驱动为 XML（解析 XML 中的 SQL）
  languageRegistry.setDefaultDriverClass(XMLLanguageDriver.class);
  // 注册 RAW 语言驱动（支持注解式原生 SQL）
  languageRegistry.register(RawLanguageDriver.class);
}
```
- **核心作用**：
    1. 注册默认别名：让用户在配置文件中使用简写（如 `<transactionManager type="JDBC"/>` 而非全类名）；
    2. 设置默认组件：如默认语言驱动为 XML，默认代理工厂为 Javassist，无需用户手动配置即可使用核心功能；
    3. 简化配置：用户只需写别名，MyBatis 会通过 `TypeAliasRegistry` 找到对应的实现类。

### 3. 总结
#### 核心关键点
1. **Configuration 的核心定位**：
    - 是 MyBatis 的“配置中枢”，聚合了所有全局配置、组件注册表和运行时缓存；
    - 是连接配置文件（XML/注解）和运行时组件（Executor、StatementHandler）的桥梁，所有组件初始化时都会从这里获取配置。

2. **核心设计特点**：
    - **默认配置优先**：构造方法中注册了大量默认别名和组件，用户无需配置即可使用基础功能；
    - **严格的冲突检测**：使用 `StrictMap` 存储核心对象，重复定义时抛出明确异常，便于排查问题；
    - **解决循环依赖**：通过 `incompleteXXX` 集合暂存未完成的资源，解析完所有配置后再统一处理；
    - **线程安全保障**：通过 ReentrantLock 保护未完成资源的并发访问。

3. **关键组件关联**：
    - `MapperRegistry` → 动态代理生成 Mapper 实例；
    - `TypeHandlerRegistry` → 处理 Java 与 JDBC 类型转换；
    - `InterceptorChain` → 执行插件的拦截逻辑；
    - `MappedStatement` → 存储 SQL 执行的所有信息（SQL 语句、参数、结果映射等）。

#### 应用场景
理解 `Configuration` 是掌握 MyBatis 核心原理的关键：
- 自定义 MyBatis 配置：如扩展 `Configuration` 子类，添加自定义属性；
- 排查配置问题：如 SQL 映射冲突、类型处理器失效等，可通过 `Configuration` 的注册表定位；
- 扩展 MyBatis 功能：如自定义插件、类型处理器、缓存实现，都需要注册到 `Configuration` 的对应注册表中。



[目录](#目录)


## newExecutor

这段代码是 **MyBatis 核心配置类 `Configuration` 中创建 SQL 执行器（Executor）的工厂方法**。

它是 MyBatis **执行引擎的装配线**，决定了 SQL 将以何种策略被执行（简单执行、重用预处理语句、批量执行），是否**启用二级缓存**，以及是否应用了用户自定义的插件（Plugin）。

这段代码完美体现了 **工厂模式 (Factory Pattern)**、**策略模式 (Strategy Pattern)** 和 **装饰器模式 (Decorator Pattern)** 的组合运用。

---

### 一、代码流程详细解析

#### 1. 方法重载与默认值处理
```java
public Executor newExecutor(Transaction transaction) {
    return newExecutor(transaction, defaultExecutorType);
}

public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    // 如果传入 null，则使用全局默认配置
    executorType = executorType == null ? defaultExecutorType : executorType;
    Executor executor;
    // ...
}
```
*   **逻辑**：提供了便捷的重载方法。如果调用者不指定执行器类型，则使用 `Configuration` 中配置的 `defaultExecutorType`（默认为 `SIMPLE`）。
*   **意义**：允许全局统一配置，同时也支持在特定会话（Session）级别覆盖默认行为。

#### 2. 策略选择：核心执行器实例化
```java
if (ExecutorType.BATCH == executorType) {
    executor = new BatchExecutor(this, transaction);
} else if (ExecutorType.REUSE == executorType) {
    executor = new ReuseExecutor(this, transaction);
} else {
    executor = new SimpleExecutor(this, transaction);
}
```
这里根据 `executorType` 枚举值，实例化三种不同的底层执行器实现类：
*   **`BatchExecutor`**: 用于批量更新操作。它会将 SQL 暂存，直到 `flushStatements()` 被调用或事务提交时才统一发送给数据库。
*   **`ReuseExecutor`**: 专注于复用 `PreparedStatement`。它会缓存预编译好的 Statement 对象，避免重复解析 SQL 和预编译的开销。
*   **`SimpleExecutor`**: 默认实现。每次执行 SQL 都创建一个新的 `PreparedStatement`，用完即关。最简单，但性能通常不如前两者（在高重复 SQL 场景下）。
*   **依赖注入**：所有执行器构造函数都传入了 `this` (Configuration) 和 `transaction`。
    *   `Configuration`: 用于获取映射信息（MappedStatement）、类型处理器等。
    *   `Transaction`: 用于获取物理数据库连接 (`Connection`)。

#### 3. 装饰器模式：二级缓存包装
```java
if (cacheEnabled) {
    executor = new CachingExecutor(executor);
}
```
*   **逻辑**：如果全局配置 `cacheEnabled` 为 `true`（默认开启），则在刚才创建的核心执行器之外，再包裹一层 `CachingExecutor`。
*   **原理**：
    *   `CachingExecutor` 实现了 `Executor` 接口。
    *   在执行查询 (`query`) 时，它先查二级缓存（Region）。
    *   如果命中，直接返回结果，**不调用**底层的 `Simple/Batch/ReuseExecutor`。
    *   如果未命中，才委托给底层执行器去查数据库，并将结果存入缓存。
    *   在执行更新 (`update`) 时，它会先清空相关缓存，再委托底层执行器。
*   **设计精髓**：缓存逻辑与核心 SQL 执行逻辑完全解耦。核心执行器根本不知道缓存的存在，它只负责和数据库打交道。

#### 4. 插件链：AOP 动态代理
```java
return (Executor) interceptorChain.pluginAll(executor);
```
*   **逻辑**：将组装好的执行器（可能是 `CachingExecutor` 包裹的 `SimpleExecutor` 等）传入 `interceptorChain.pluginAll()`。
*   **作用**：这是 MyBatis **插件机制 (Plugin Interceptor)** 的核心入口。
*   **底层原理**：
    *   MyBatis 允许用户编写实现 `Interceptor` 接口的类，通过 `@Intercepts` 注解声明要拦截的方法（如 `Executor.query`, `Executor.update` 等）。
    *   `pluginAll` 方法会遍历所有注册的插件。
    *   利用 **JDK 动态代理 (Dynamic Proxy)** 生成一个代理对象。
    *   这个代理对象包裹了原始的 `executor`。当调用 `executor.query()` 时，实际上先执行插件的 `intercept()` 方法，插件可以选择：
        *   修改 SQL 参数。
        *   修改 SQL 语句本身（如分页插件 PageHelper 就是在这里改写 SQL 添加 `LIMIT`）。
        *   直接返回结果（阻断后续执行）。
        *   调用 `invocation.proceed()` 继续执行原始逻辑。
*   **返回结果**：最终返回给用户的是一个**被多层代理包裹的对象**。

---

### 二、背后/底层原理深度剖析

#### 1. 策略模式 (Strategy Pattern) 的实现
MyBatis 通过 `ExecutorType` 枚举灵活切换 SQL 执行策略，而无需修改上层业务代码。
*   **SimpleExecutor**:
    *   **生命周期**：`PreparedStatement` -> 创建 -> 执行 -> 关闭。
    *   **适用**：SQL 变化多端，几乎不重复的场景。
*   **ReuseExecutor**:
    *   **缓存结构**：内部维护一个 `Map<String, PreparedStatement>`。Key 是 SQL 字符串，Value 是预编译对象。
    *   **原理**：数据库预编译 SQL 是有成本的（解析、优化、生成执行计划）。对于高频重复的 SQL（如 `SELECT * FROM user WHERE id = ?`），复用 Statement 能显著降低 CPU 消耗。
    *   **注意**：它不复用 `ResultSet`，也不复用连接，只复用 Statement 对象。
*   **BatchExecutor**:
    *   **原理**：利用 JDBC 的 `addBatch()` 和 `executeBatch()` 机制。
    *   **流程**：
        1.  接收 SQL 和参数 -> 存入内部列表 (`List<Statement>` 和 `List<List<Object>>`)。
        2.  不立即发送网络请求。
        3.  当调用 `flushStatements()` 或 `commit()` 时，遍历列表，一次性将所有操作发送给数据库。
    *   **优势**：极大减少网络 RTT (Round-Trip Time)。对于 1000 条 Insert，Simple 需要 1000 次网络往返，Batch 可能只需要 1 次（取决于驱动实现）。
    *   **限制**：通常只适用于 `INSERT`, `UPDATE`, `DELETE`。`SELECT` 通常不支持批处理（因为需要立即返回结果集）。

#### 2. 装饰器模式 (Decorator Pattern) 的层级结构
最终的 `Executor` 对象是一个典型的“洋葱模型”：
```java
// 假设开启了缓存且有 1 个插件
Executor finalExecutor = 
    PluginProxy (
        CachingExecutor (
            SimpleExecutor
        )
    );
```
*   **调用链**：
    1.  用户调用 `sqlSession.selectList()`。
    2.  进入 `PluginProxy.invoke()` -> 执行插件逻辑。
    3.  插件调用 `proceed()` -> 进入 `CachingExecutor.query()`。
    4.  `CachingExecutor` 查缓存 -> 未命中 -> 调用 `delegate.query()`。
    5.  进入 `SimpleExecutor.doQuery()` -> 获取 Connection -> 创建 Statement -> 执行 SQL。
*   **优势**：
    *   **开闭原则**：增加缓存功能或新的插件，不需要修改 `SimpleExecutor` 的代码。
    *   **灵活组合**：可以任意组合“缓存 + 批量 + 插件 A + 插件 B”。

#### 3. JDK 动态代理与插件原理 (`interceptorChain.pluginAll`)
这是 MyBatis 最强大的扩展点。
*   **实现细节**：
    *   `InterceptorChain` 持有所有 `Interceptor` 实例。
    *   `pluginAll(Object target)` 方法循环调用每个 interceptors 的 `plugin(target)` 方法。
    *   用户的 `Interceptor` 通常继承 `Plugin` 类，该类实现了 `InvocationHandler` 接口。
    *   `Plugin.wrap()` 方法判断目标对象的方法是否在 `@Intercepts` 注解的签名列表中。
        *   **是**：创建 JDK Proxy 包装目标对象。
        *   **否**：直接返回原对象（不代理，减少开销）。
*   **经典案例：分页插件 (PageHelper)**
    *   它拦截 `Executor.query()` 方法。
    *   解析参数中的分页信息。
    *   **改写 SQL**：将原始 SQL `SELECT * FROM table` 改写为 `SELECT * FROM table LIMIT ?, ?`。
    *   同时自动生成一条 `SELECT COUNT(*) ...` 查询总数。
    *   这一切都在 `Executor` 真正执行 SQL 之前完成，对业务代码完全透明。

#### 4. 事务绑定
注意构造函数 `new XxxExecutor(this, transaction)`。
*   **原理**：`Executor` 内部持有了 `Transaction` 引用。
*   **执行时**：当 `Executor` 需要执行 SQL 时，它调用 `transaction.getConnection()`。
*   **一致性保证**：由于 `SqlSession` 持有一个 `Executor` 实例，而 `Executor` 持有一个 `Transaction` 实例（进而持有一个 `Connection`），这确保了**同一个 SqlSession 内的所有操作都使用同一个数据库连接**，从而保证了事务的原子性。

---

### 三、总结

`Configuration.newExecutor` 方法是 MyBatis **运行时核心的组装车间**。

1.  **策略路由**：根据配置决定是用“简单模式”、“复用模式”还是“批量模式”来执行 SQL。
2.  **功能增强**：通过 `CachingExecutor` 无侵入地添加了二级缓存能力。
3.  **生态扩展**：通过 `interceptorChain.pluginAll` 利用动态代理技术，为 SQL 执行插入了“钩子”，使得分页、性能监控、多租户隔离等高级功能得以实现，而无需修改 MyBatis 源码。
4.  **架构美学**：短短几行代码，串联起了策略、装饰器、代理三大设计模式，构建了一个既高效又极具扩展性的执行引擎。

理解这段代码，就理解了 MyBatis 是如何在保持核心轻量的同时，支撑起庞大的插件生态和复杂的企业级需求的。
