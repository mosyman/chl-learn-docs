
这是 **MyBatis 的核心执行器接口 `Executor`**。

它是 MyBatis SQL 执行引擎的**总指挥**。所有的 SQL 操作（增删改查）、事务控制（提交/回滚）、缓存管理（一级/二级缓存）以及延迟加载，最终都通过这个接口的方法来完成。

在之前的代码分析中我们提到，`Configuration.newExecutor` 会创建 `SimpleExecutor`、`BatchExecutor` 或 `ReuseExecutor`，并可能用 `CachingExecutor` 进行装饰。这些类全部实现了这个 `Executor` 接口。

---

### 一、接口方法详细解释

我们将这些方法分为四大类：**核心 CRUD**、**事务控制**、**缓存管理**、**生命周期与辅助**。

#### 1. 核心 CRUD 操作
*   **`int update(MappedStatement ms, Object parameter)`**
    *   **功能**：执行 `INSERT`, `UPDATE`, `DELETE` 语句。
    *   **返回**：受影响的行数。
    *   **原理**：获取数据库连接 -> 创建/复用 Statement -> 设置参数 -> 执行 `executeUpdate()` -> 处理生成的主键（如果有） -> 返回结果。
*   **`<E> List<E> query(...)` (两个重载)**
    *   **功能**：执行 `SELECT` 语句，将结果集映射为 Java 对象列表。
    *   **参数详解**：
        *   `MappedStatement ms`: 包含 SQL 语句、结果映射规则、缓存配置等元数据。
        *   `Object parameter`: 传入的参数对象（如 User 对象或 Map）。
        *   `RowBounds rowBounds`: 内存分页限制（offset, limit）。*注意：这不是数据库层面的分页，除非配合插件改写 SQL。*
        *   `ResultHandler resultHandler`: 自定义结果处理器。如果提供，MyBatis 不会返回 List，而是逐行回调此处理器（适用于海量数据处理）。
        *   `CacheKey cacheKey`: 用于二级缓存的键。如果为 null 或缓存未命中，则查库。
        *   `BoundSql boundSql`: 已经解析好、替换了 `?` 占位符的实际 SQL 对象。
    *   **重载区别**：第二个重载方法（不带 `CacheKey` 和 `BoundSql`）通常由上层调用，它会在内部调用 `createCacheKey` 和 `ms.getBoundSql` 生成这两个参数，然后委托给第一个重载方法。
*   **`<E> Cursor<E> queryCursor(...)`**
    *   **功能**：执行流式查询。
    *   **特点**：不一次性加载所有数据到内存，而是返回一个 `Cursor`（迭代器）。每次调用 `next()` 才从数据库拉取一行。
    *   **场景**：处理百万级大表导出，防止 OOM（内存溢出）。需要数据库驱动支持（如 MySQL 设置 `fetchSize = Integer.MIN_VALUE`）。

#### 2. 批量与事务控制
*   **`List<BatchResult> flushStatements()`**
    *   **功能**：强制执行批处理语句。
    *   **场景**：仅在使用 `BatchExecutor` 时有效。它会将暂存在内存中的所有 SQL 一次性发送给数据库执行。
    *   **返回**：每个批次执行的详细结果（包括更新行数和生成的主键）。
*   **`void commit(boolean required)`** & **`void rollback(boolean required)`**
    *   **功能**：提交或回滚当前事务。
    *   **参数 `required`**：
        *   `true`: 只有当当前 Executor 自己开启了事务（即不是加入外部事务）时，才执行 commit/rollback。
        *   `false`: 无论事务来源如何，强制执行。
    *   **原理**：委托给内部的 `Transaction` 对象执行 `connection.commit()` 或 `connection.rollback()`。
*   **`Transaction getTransaction()`**
    *   **功能**：获取当前持有的事务对象，进而获取物理数据库连接 (`Connection`)。

#### 3. 缓存管理 (一级/二级缓存)
*   **`CacheKey createCacheKey(...)`**
    *   **功能**：根据 SQL ID、参数值、分页信息、环境 ID 等生成唯一的缓存键。
    *   **原理**：使用特定的算法（通常是累加校验和）确保相同的查询条件生成相同的 Key。
*   **`boolean isCached(MappedStatement ms, CacheKey key)`**
    *   **功能**：判断某条查询是否已经在二级缓存中存在。
*   **`void clearLocalCache()`**
    *   **功能**：清空**一级缓存**（PerpetualCache）。
    *   **时机**：通常在执行完 `update`, `insert`, `delete` 或者手动调用 `sqlSession.clearCache()` 时触发。这保证了读写一致性（修改后下次查肯定是新数据）。
*   **`void deferLoad(...)`**
    *   **功能**：注册延迟加载任务。
    *   **场景**：当查询结果中包含关联对象（如 `User` 中有 `Order`），且配置了 `lazyLoadingEnabled=true` 时。此时 `Order` 对象是一个代理，当用户第一次访问 `user.getOrder()` 时，Executor 会通过这里注册的逻辑去数据库补查 `Order` 数据。

#### 4. 生命周期与辅助
*   **`void close(boolean forceRollback)`**
    *   **功能**：关闭 Executor。
    *   **逻辑**：如果 `forceRollback` 为 true 且事务未提交，则先回滚。然后关闭事务（归还连接），清空缓存。
*   **`boolean isClosed()`**: 检查是否已关闭。
*   **`void setExecutorWrapper(Executor executor)`**
    *   **功能**：设置包装器。
    *   **原理**：主要用于 `CachingExecutor`。`CachingExecutor` 内部持有一个 `delegate` (如 `SimpleExecutor`)。MyBatis 启动时，先创建 `SimpleExecutor`，再创建 `CachingExecutor`，最后调用 `cachingExecutor.setExecutorWrapper(simpleExecutor)` 建立委托关系。这是一种避免循环依赖的构造技巧。

---

### 二、背后/底层原理深度剖析

#### 1. 策略模式与执行流程的分发
`Executor` 接口定义了标准动作，但具体怎么做由实现类决定：
*   **SimpleExecutor**:
    *   **流程**：`getConnection()` -> `createStatement(sql)` -> `setParams()` -> `execute()` -> `closeStatement()`.
    *   **特点**：每次执行都创建新的 `PreparedStatement`。无状态，线程不安全（但 SqlSession 本身也不建议多线程共享）。
*   **ReuseExecutor**:
    *   **优化点**：内部维护 `Map<String, PreparedStatement>`。
    *   **流程**：执行前先看 Map 里有没有该 SQL 对应的 Statement。有则复用，无则创建并放入 Map。执行后**不关闭** Statement，只重置参数。
    *   **风险**：如果 SQL 动态变化极多（如大量拼接不同表名），Map 会无限膨胀，导致内存泄漏。
*   **BatchExecutor**:
    *   **优化点**：利用 JDBC Batch 机制。
    *   **流程**：`addBatch()` 存入列表。直到 `flushStatements()` 或 `commit()` 时才遍历列表执行 `executeBatch()`。
    *   **限制**：不支持 `SELECT`（因为 SELECT 需要立即返回 ResultSet，无法批量）。

#### 2. 缓存架构：一级 vs 二级
`Executor` 是两级缓存的分水岭。
*   **一级缓存 (Local Cache)**:
    *   **归属**：绑定在 `SqlSession` (即 `Executor` 实例) 上。
    *   **实现**：`PerpetualCache` (基于 HashMap)。
    *   **作用域**：同一个 SqlSession 内有效。
    *   **失效**：执行任何更新操作或手动清除。
    *   **代码体现**：`clearLocalCache()` 方法。所有 Executor 实现类都有这个本地缓存。
*   **二级缓存 (Global Cache)**:
    *   **归属**：绑定在 `Namespace` (Mapper) 上，跨 SqlSession 共享。
    *   **实现**：通过 `CachingExecutor` 装饰器实现。
    *   **流程**：
        1.  `CachingExecutor.query()` 被调用。
        2.  计算 `CacheKey`。
        3.  查二级缓存 (`tCache.getObject(key)`)。
        4.  **命中**：直接返回，**不调用** delegate (SimpleExecutor)。
        5.  **未命中**：调用 `delegate.query()` 查库 -> 结果存入二级缓存 -> 返回。
    *   **关键点**：`isCached` 和 `createCacheKey` 在二级缓存逻辑中至关重要。

#### 3. 延迟加载 (Lazy Loading) 的黑魔法
`deferLoad` 方法是 MyBatis 实现“按需加载”的关键。
*   **原理**：
    1.  查询主对象（如 User）时，发现关联属性（Order）配置了懒加载。
    2.  MyBatis 不查 Order 表，而是创建一个 Order 的**动态代理对象**（Javassist 或 ByteBuddy 生成）。
    3.  调用 `executor.deferLoad(...)`，将一个 `LoadContext` 注册到 Executor 的内部队列中。这个上下文包含了“如何去查 Order 的信息”（MappedStatement, 参数, Key 等）。
    4.  当用户代码调用 `user.getOrder().getAmount()` 时，代理对象的拦截器被触发。
    5.  拦截器找到之前注册的 `LoadContext`，调用 Executor 再次执行查询，填充真实数据。
*   **意义**：极大减少了不必要的联表查询（Join）或多余字段查询，提升性能。

#### 4. 事务的传递性与隔离
`commit(boolean required)` 中的 `required` 参数体现了对**事务上下文**的精细控制。
*   **场景**：
    *   **Spring 整合时**：Spring 管理事务（外部事务）。MyBatis 的 `SqlSession` 加入 Spring 的事务同步管理器。此时 `executor.getTransaction()` 拿到的是一个 `SpringManagedTransaction`。
    *   **提交时**：MyBatis 检测到事务是由 Spring 管理的（`!isTransactionRequired()` 或类似逻辑），如果传入 `required=true`，它知道不该由自己提交，于是跳过。如果传入 `false`（强制），它可能会尝试提交，导致异常或错误行为。
*   **设计意图**：让 MyBatis 既能独立运行（自己管事务），又能完美融入 Spring 等容器（交出事务控制权）。

#### 5. `BoundSql` 与 `CacheKey` 的解耦
注意 `query` 方法同时接收 `MappedStatement` 和 `BoundSql`。
*   **MappedStatement**: 静态配置，包含原始 SQL (`SELECT * FROM user WHERE id = #{id}`)。
*   **BoundSql**: 运行时对象，包含解析后的 SQL (`SELECT * FROM user WHERE id = ?`) 和参数映射。
*   **原理**：
    *   这种分离允许在 SQL 执行前进行多次修改。
    *   例如：分页插件拦截 `Executor.query`，它可以拿到 `BoundSql`，修改其中的 SQL 字符串（添加 LIMIT），修改参数映射，然后重新构建一个新的 `BoundSql` 传给后续的 `delegate.query`。
    *   `CacheKey` 的生成依赖于 `BoundSql` 的最终形态，确保“不同的 SQL 即使参数相同，缓存 Key 也不同”。

### 三、总结

`Executor` 接口是 MyBatis **运行时行为的中枢神经**。

1.  **统一入口**：屏蔽了底层 JDBC 的复杂性（连接获取、Statement 创建、参数设置、结果集映射）。
2.  **扩展基石**：通过策略模式支持多种执行方式（简单/重用/批量），通过装饰器模式支持二级缓存，通过 `deferLoad` 支持懒加载。
3.  **插件锚点**：它是 MyBatis 插件系统最主要拦截的接口（尤其是 `query` 和 `update` 方法），几乎所有高级功能（分页、多租户、审计）都是在这里切入的。
4.  **资源管家**：负责管理 Statement 的生命周期、事务的边界以及缓存的清理，确保资源不泄露、数据一致。

理解了 `Executor`，就理解了 MyBatis 是如何将一条 SQL 字符串最终转化为数据库操作并返回 Java 对象的完整闭环。

