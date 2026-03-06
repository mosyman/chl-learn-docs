
这是 **MyBatis 的核心元数据类 `MappedStatement`**。

如果把 MyBatis 比作一个数据库操作工厂，那么 `SqlSession` 是车间主任，`Executor` 是流水线工人，而 **`MappedStatement` 就是那张详细的“生产图纸”**。

它封装了执行一条 SQL 语句所需的**所有配置信息**：SQL 本身、输入参数映射规则、输出结果映射规则、缓存策略、超时设置、主键生成策略等。在 MyBatis 启动时，XML 或注解中的 `<select>`, `<insert>` 等标签会被解析并转化为 `MappedStatement` 对象，存储在 `Configuration` 的 `StrictMap` 中。

---

### 一、核心字段详解（生产图纸的要素）

#### 1. 身份与来源
*   **`String id`**: 全局唯一标识符。通常是 `namespace + "." + statementId`（如 `com.example.UserMapper.selectById`）。它是获取该对象的 Key。
*   **`String resource`**: 定义该 Statement 的资源位置（如 XML 文件路径），主要用于报错时定位。
*   **`Configuration configuration`**: 持有全局配置对象，方便访问其他全局资源（如 TypeHandler, ObjectFactory）。
*   **`SqlCommandType sqlCommandType`**: 枚举值 (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `FLUSH`)。决定了 Executor 调用 `query()` 还是 `update()`。

#### 2. SQL 与语言驱动
*   **`SqlSource sqlSource`**: **核心中的核心**。它不是最终的 SQL 字符串，而是一个**工厂**，负责根据传入的参数动态生成 `BoundSql`（包含最终 SQL 和参数映射）。
    *   如果是静态 SQL，它是 `StaticSqlSource`。
    *   如果有 `${}` 或 `<if>` 标签，它是 `DynamicSqlSource`。
*   **`LanguageDriver lang`**: 脚本语言驱动器（默认是 `XMLLanguageDriver`），负责解析 XML 中的 SQL 标签。

#### 3. 输入/输出映射
*   **`ParameterMap parameterMap`**: 定义参数如何映射到 SQL 的 `?` 占位符。（注：现代 MyBatis 推荐使用 `#{}` 自动推导，此字段常为默认值）。
*   **`List<ResultMap> resultMaps`**: 定义数据库列如何映射到 Java 对象属性。支持复杂的嵌套映射（一对一、一对多）。
*   **`boolean hasNestedResultMaps`**: 标记是否存在嵌套结果映射，用于优化结果集处理逻辑（避免不必要的递归）。

#### 4. 执行控制与性能
*   **`Integer timeout`**: SQL 执行超时时间（秒），底层传递给 JDBC `Statement.setQueryTimeout()`。
*   **`Integer fetchSize`**: 提示 JDBC 驱动每次从数据库获取的行数。对于大结果集，设置此值可防止 OOM。
*   **`StatementType statementType`**: 指定使用哪种 JDBC Statement。
    *   `STATEMENT`: 普通 Statement（易 SQL 注入，少用）。
    *   `PREPARED`: `PreparedStatement`（默认，防注入，性能好）。
    *   `CALLABLE`: `CallableStatement`（用于存储过程）。
*   **`ResultSetType resultSetType`**: 控制结果集游标行为（如 `FORWARD_ONLY`, `SCROLL_SENSITIVE`）。

#### 5. 缓存策略
*   **`Cache cache`**: 指向二级缓存实例。如果为 null，则不使用该 Namespace 的二级缓存。
*   **`boolean useCache`**: 当前语句是否启用二级缓存（默认 select 为 true）。
*   **`boolean flushCacheRequired`**: 执行该语句后是否清空缓存（默认 insert/update/delete 为 true）。

#### 6. 主键生成
*   **`KeyGenerator keyGenerator`**: 主键生成策略。
    *   `Jdbc3KeyGenerator`: 利用 JDBC `getGeneratedKeys` 获取自增主键。
    *   `SelectKeyGenerator`: 执行额外的 SELECT 语句获取主键（如 Oracle 序列）。
    *   `NoKeyGenerator`: 无主键生成。
*   **`String[] keyProperties`**: 获取到的主键值回填到 Java 对象的哪个属性。
*   **`String[] keyColumns`**: 数据库中对应的主键列名。

#### 7. 其他高级特性
*   **`Log statementLog`**: 专门用于记录该语句执行日志的对象。
*   **`ParamNameResolver paramNameResolver`**: 用于解析方法参数名（当使用注解接口开发时，将参数名映射到 `#{paramName}`）。
*   **`boolean dirtySelect`**: (较新特性) 标记该查询是否为“脏读”或非纯净查询，可能影响缓存行为。

---

### 二、背后/底层原理深度剖析

#### 1. 不可变性与构建器模式 (Immutability & Builder Pattern)
*   **设计**：`MappedStatement` 构造函数是私有的 (`private MappedStatement()`)，只能通过内部静态类 `Builder` 创建。
*   **原理**：
    *   **线程安全**：`MappedStatement` 在 MyBatis 启动阶段构建完成后，在运行时是**只读**的。多个线程并发执行同一个 Mapper 方法时，共享同一个 `MappedStatement` 实例，无需加锁。
    *   **链式调用**：`Builder` 提供了流畅的 API (`builder.resource(...).timeout(...).build()`)，使得复杂的配置过程清晰易读。
    *   **防御性拷贝**：在 `build()` 方法中，`resultMaps` 列表被转换为 `Collections.unmodifiableList`，防止外部修改。

#### 2. 动态 SQL 的延迟绑定 (`SqlSource` -> `BoundSql`)
注意 `MappedStatement` 中存储的是 `SqlSource` 而不是 `String sql`。
*   **原理**：
    *   XML 中的 SQL 可能包含动态标签 (`<if test="name != null"> AND name = #{name} </if>`)。
    *   在启动时，MyBatis 无法确定最终的 SQL 是什么，因为它依赖于运行时的参数。
    *   **运行时流程**：
        1.  用户调用 `sqlSession.selectList("id", param)`。
        2.  Executor 拿到 `MappedStatement`。
        3.  调用 `ms.getBoundSql(param)`。
        4.  `SqlSource.getBoundSql(param)` 被触发，解析动态标签，替换 `#{}` 为 `?`，生成最终的 `BoundSql` 对象。
*   **意义**：实现了**配置与运行的解耦**，支持强大的动态 SQL 能力。

#### 3. `getBoundSql` 方法中的“二次检查”
```java
public BoundSql getBoundSql(Object parameterObject) {
  BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
  List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
  
  // 【关键点 1】：如果 SqlSource 生成的映射为空，回退到 ParameterMap 的定义
  if (parameterMappings == null || parameterMappings.isEmpty()) {
    boundSql = new BoundSql(configuration, boundSql.getSql(), parameterMap.getParameterMappings(), parameterObject);
  }

  // 【关键点 2】：动态检测嵌套结果映射 (Issue #30)
  for (ParameterMapping pm : boundSql.getParameterMappings()) {
    String rmId = pm.getResultMapId();
    if (rmId != null) {
      ResultMap rm = configuration.getResultMap(rmId);
      if (rm != null) {
        hasNestedResultMaps |= rm.hasNestedResultMaps();
      }
    }
  }
  return boundSql;
}
```
*   **原理**：
    *   **兼容性**：早期 MyBatis 强依赖 `<parameterMap>` 标签，现代版本推荐 `#{}` 自动推断。这段代码兼容了两种模式。
    *   **动态标志位**：`hasNestedResultMaps` 本应在构建时确定，但如果参数映射中动态引用了某个 ResultMap，而这个 ResultMap 内部又有嵌套，那么必须在运行时动态更新这个标志位。这影响了 `ResultSetHandler` 的处理策略（是否需要复杂的结果映射逻辑）。

#### 4. 主键回填机制 (`KeyGenerator`)
*   **场景**：执行 `INSERT` 后，如何获取数据库生成的自增 ID 并塞回 Java 对象？
*   **流程**：
    1.  `Executor.update()` 执行 SQL。
    2.  执行完毕后，检查 `ms.getKeyGenerator()`。
    3.  如果不是 `NoKeyGenerator`，调用 `keyGenerator.processAfter()` (或 `processBefore`)。
    4.  `Jdbc3KeyGenerator` 会调用 `stmt.getGeneratedKeys()` 获取 ResultSet。
    5.  根据 `ms.getKeyProperties()` (如 "id")，利用反射将值设置回参数对象。
*   **灵活性**：通过策略模式，MyBatis 支持多种主键生成方式，甚至允许用户自定义 `KeyGenerator`。

#### 5. 缓存 Key 的生成依赖
虽然 `MappedStatement` 不直接生成 CacheKey，但它是生成 CacheKey 的基础数据源。
*   `CachingExecutor` 在计算 CacheKey 时，会用到：
    *   `ms.getId()` (区分不同语句)
    *   `ms.getSqlSource().getBoundSql().getSql()` (区分不同 SQL)
    *   `ms.getFetchSize()`, `ms.getTimeout()` 等配置（某些配置变化可能导致缓存失效）
    *   `ms.getDatabaseId()` (区分多数据源环境)
*   这意味着，**任何 `MappedStatement` 属性的变化理论上都应导致缓存策略的变化**（尽管 MyBatis 主要依赖 SQL 和参数）。

#### 6. 内存模型与性能
*   **重量级对象**：`MappedStatement` 包含大量对象引用（List, Map, Custom Objects）。
*   **单例复用**：每个 Mapper 语句在全局只有一个 `MappedStatement` 实例。
*   **轻量级上下文**：每次执行时，基于它生成的 `BoundSql` 和 `CacheKey` 是线程隔离的（存放在栈或 ThreadLocal 中），保证了高并发下的安全性。

### 三、总结

`MappedStatement` 是 MyBatis **配置信息的运行时载体**。

1.  **承上启下**：它将静态的 XML/注解配置转化为运行时可操作的元数据对象。
2.  **全能字典**：Executor、StatementHandler、ParameterHandler、ResultSetHandler 四大组件在执行过程中，都需要从 `MappedStatement` 中读取配置来指导自己的行为。
3.  **设计典范**：
    *   使用 **Builder 模式** 解决复杂构造问题。
    *   使用 **不可变设计** 保证多线程安全。
    *   使用 **延迟绑定 (SqlSource)** 支持动态 SQL。
    *   使用 **策略模式 (KeyGenerator, LanguageDriver)** 支持扩展。

如果说 `SqlSession` 是用户面对的门面，那么 `MappedStatement` 就是门面背后那个无所不知的“大脑”，它知道关于这条 SQL 的一切。

