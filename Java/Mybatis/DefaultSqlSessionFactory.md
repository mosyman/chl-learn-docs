






## DefaultSqlSessionFactory

### 1. 核心需求理解
MyBatis 核心类 `DefaultSqlSessionFactory`——这个类是 `SqlSessionFactory` 接口的默认实现，
是 MyBatis 创建 `SqlSession`（会话）的核心工厂类，负责整合配置（`Configuration`）、事务（`Transaction`）、执行器（`Executor`）等核心组件，
提供多重载的 `openSession` 方法适配不同的会话创建场景，是 MyBatis 会话管理的入口。

### 2. 详细代码解释
#### 2.1 类的整体定位与核心设计
```java
/**
 * @author Clinton Begin
 */
public class DefaultSqlSessionFactory implements SqlSessionFactory {
  private final Configuration configuration;
  // ... 构造方法 + 核心方法
}
```
- **核心定位**：
    - 实现 `SqlSessionFactory` 接口，是 MyBatis 中**唯一的默认会话工厂实现**（开发者也可自定义，但极少用）；
    - 持有全局唯一的 `Configuration` 实例（MyBatis 所有配置的容器，包含环境、映射器、插件、设置等）；
    - 核心职责：根据不同参数（执行器类型、事务隔离级别、自动提交、外部连接等）创建 `SqlSession` 实例，封装会话创建的复杂逻辑。
- **设计模式**：
    - 工厂模式：通过 `openSession` 系列方法创建 `SqlSession`，屏蔽 `SqlSession` 实例化的细节；
    - 单一职责：仅负责创建 `SqlSession`，不参与会话的具体操作（CRUD 由 `SqlSession` 完成）。

#### 2.2 核心成员与构造方法
```java
private final Configuration configuration;

public DefaultSqlSessionFactory(Configuration configuration) {
  this.configuration = configuration;
}
```
- `configuration`：MyBatis 的全局配置中心，是创建 `SqlSession` 的核心依赖——包含环境（`Environment`，数据源、事务工厂）、执行器类型、插件、映射器等所有配置；
- 构造方法：通过传入 `Configuration` 初始化工厂，保证所有创建的 `SqlSession` 都基于同一套配置，是 MyBatis 全局配置的“入口传递”。

#### 2.3 重载的 `openSession` 方法（对外API）
`DefaultSqlSessionFactory` 提供了 8 个重载的 `openSession` 方法，核心目的是适配不同的会话创建场景，
所有方法最终都会转发到两个核心私有方法：`openSessionFromDataSource`（从数据源创建）、`openSessionFromConnection`（从外部连接创建）。

| 重载方法 | 参数 | 核心逻辑 | 适用场景 |
|----------|------|----------|----------|
| `openSession()` | 无参 | 用默认执行器类型、无隔离级别、自动提交=false | 最常用，默认创建事务性会话 |
| `openSession(boolean autoCommit)` | 自动提交 | 用默认执行器类型、无隔离级别、指定自动提交 | 控制会话是否自动提交事务 |
| `openSession(ExecutorType execType)` | 执行器类型 | 指定执行器类型、无隔离级别、自动提交=false | 自定义执行器（如批量执行 `BATCH`） |
| `openSession(TransactionIsolationLevel level)` | 隔离级别 | 默认执行器、指定隔离级别、自动提交=false | 自定义事务隔离级别（如 `READ_COMMITTED`） |
| `openSession(ExecutorType execType, TransactionIsolationLevel level)` | 执行器+隔离级别 | 指定执行器、隔离级别、自动提交=false | 精细化控制执行器和事务隔离 |
| `openSession(ExecutorType execType, boolean autoCommit)` | 执行器+自动提交 | 指定执行器、无隔离级别、指定自动提交 | 自定义执行器+事务自动提交 |
| `openSession(Connection connection)` | 外部连接 | 默认执行器、使用外部连接 | 复用已有数据库连接（如分布式事务） |
| `openSession(ExecutorType execType, Connection connection)` | 执行器+外部连接 | 指定执行器、使用外部连接 | 复用连接+自定义执行器 |

**关键补充**：
- `ExecutorType`（执行器类型）：
    - `SIMPLE`：默认执行器，每次执行 SQL 都创建新的预处理语句（PreparedStatement）；
    - `REUSE`：复用预处理语句的执行器；
    - `BATCH`：批量执行 SQL 的执行器（适合批量插入/更新）；
- `TransactionIsolationLevel`（事务隔离级别）：对应数据库隔离级别（`NONE`/`READ_UNCOMMITTED`/`READ_COMMITTED`/`REPEATABLE_READ`/`SERIALIZABLE`）；
- 自动提交（`autoCommit`）：`true` 时执行 SQL 后自动提交事务，`false` 需手动调用 `commit()`。

#### 2.4 核心私有方法1：`openSessionFromDataSource`（从数据源创建会话）

- [2.4.1](#241)
- [2.4.2](#242)

#### 2.4.1
这是最核心的会话创建逻辑，用于从 `Configuration` 配置的数据源中获取连接、创建事务、初始化执行器，最终创建 `SqlSession`。
```java
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level,
    boolean autoCommit) {
  Transaction tx = null;
  try {
    // 步骤1：获取环境配置（数据源、事务工厂）
    final Environment environment = configuration.getEnvironment();
    // 步骤2：获取事务工厂（环境中配置的，无则用 ManagedTransactionFactory）
    final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
    // 步骤3：创建事务（关联数据源、隔离级别、自动提交）
    tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
    // 步骤4：创建执行器（关联事务、执行器类型，包含插件拦截）
    final Executor executor = configuration.newExecutor(tx, execType);
    // 步骤5：创建 DefaultSqlSession 实例
    return createSqlSession(configuration, executor, autoCommit);
  } catch (Exception e) {
    // 步骤6：异常处理——关闭事务（避免连接泄漏）
    closeTransaction(tx);
    throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
  } finally {
    // 步骤7：重置错误上下文（MyBatis 错误信息跟踪）
    ErrorContext.instance().reset();
  }
}
```

##### 步骤拆解：
1. **获取环境配置**：`environment` 包含数据源（`DataSource`）和事务工厂（`TransactionFactory`），是创建事务的核心依赖；
2. **获取事务工厂**：调用 `getTransactionFactoryFromEnvironment`，无配置则默认使用 `ManagedTransactionFactory`（托管事务工厂，通常用于容器管理事务）；
3. **创建事务**：通过事务工厂创建 `Transaction` 实例，关联数据源、隔离级别、自动提交规则——MyBatis 事务是对 JDBC 事务的封装，底层还是操作 `Connection.setAutoCommit()`/`setTransactionIsolation()`；
4. **创建执行器**：`configuration.newExecutor(tx, execType)` 是核心：
    - 初始化对应类型的执行器（`SimpleExecutor`/`ReuseExecutor`/`BatchExecutor`）；
    - 应用所有插件（`Interceptor`），通过动态代理包装执行器（插件的核心入口）；
5. **创建 SqlSession**：调用 `createSqlSession` 创建 `DefaultSqlSession`，传入配置、执行器、自动提交规则；
6. **异常处理**：创建过程中异常则关闭事务（释放连接），避免连接泄漏；
7. **重置错误上下文**：`ErrorContext` 是 MyBatis 用于跟踪错误信息的工具，重置避免错误信息污染后续会话。


[2.4 核心私有方法1：`openSessionFromDataSource`（从数据源创建会话）](#24-核心私有方法1opensessionfromdatasource从数据源创建会话)

#### 2.4.2
这段代码是 **MyBatis 核心入口 `SqlSessionFactory.openSession()` 的底层实现逻辑之一**（具体是从数据源创建会话的路径）。

它展示了 MyBatis 如何**组装**一个完整的数据库会话环境：从配置中获取环境信息 -> 创建事务对象 -> 创建执行器 -> 最终封装成 `SqlSession`。这是一个典型的**依赖注入与对象组装**过程。

### 一、代码流程逐行详解

#### 1. 获取环境配置 (`Environment`)
```java
final Environment environment = configuration.getEnvironment();
```
*   **含义**：`Configuration` 是 MyBatis 的全局配置中心。`Environment` 对象 encapsulates（封装）了当前运行环境的两个核心要素：
    1.  **`TransactionFactory`**：事务工厂（决定是用 JDBC 事务还是 JTA 事务）。
    2.  **`DataSource`**：数据源（决定从哪里获取物理数据库连接）。
*   **底层原理**：MyBatis 支持多环境配置（如 `development`, `production`），但在运行时，`Configuration` 只持有一个激活的 `Environment`。这行代码确保了后续操作基于当前激活的环境进行。

#### 2. 解析事务工厂
```java
final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
```
*   **逻辑**：从 `Environment` 中提取之前配置好的 `TransactionFactory` 实例。
*   **容错处理**：如果 `environment` 为 null（即没有配置数据源和事务管理器，通常用于测试或外部管理连接模式），该方法会返回一个默认的 `ManagedTransactionFactory`（这种工厂不管理连接的生命周期，完全由外部控制）。
*   **设计意图**：解耦。`openSessionFromDataSource` 方法不需要知道具体是 `JdbcTransactionFactory` 还是 `SpringManagedTransactionFactory`，只需要知道它实现了 `TransactionFactory` 接口。

#### 3. 创建事务对象 (`Transaction`) —— **关键步骤**
```java
tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
```
*   **动作**：调用工厂的 `newTransaction` 方法。
*   **底层行为**（以 `JdbcTransactionFactory` 为例）：
    1.  **获取连接**：内部调用 `dataSource.getConnection()`。这是物理连接真正从连接池（如 HikariCP, Druid）借出的时刻。
    2.  **配置连接**：
        *   调用 `conn.setTransactionIsolation(level)` 设置隔离级别。
        *   调用 `conn.setAutoCommit(autoCommit)` 设置自动提交模式。
            *   若 `autoCommit=false`，则开启手动事务，后续需显式 commit/rollback。
            *   若 `autoCommit=true`，每条 SQL 执行后立即提交。
    3.  **封装**：将配置好的 `Connection` 包装在 `JdbcTransaction` 对象中返回。
*   **异常风险**：如果数据库宕机、连接池满或网络不通，**异常通常在此处抛出**。

#### 4. 创建执行器 (`Executor`)
```java
final Executor executor = configuration.newExecutor(tx, execType);
```
*   **含义**：`Executor` 是 MyBatis 的**SQL 执行引擎**。它负责缓存管理、SQL 组装、参数映射、结果集映射以及调用 JDBC Statement。
*   **参数 `execType`**：决定执行器的类型，直接影响性能和行为：
    *   `SIMPLE`：简单执行器，不做特殊处理，每次执行都创建新的 Statement。
    *   `REUSE`：复用执行器，会缓存 `PreparedStatement`，避免重复预编译。
    *   `BATCH`：批量执行器，会将更新操作（INSERT/UPDATE/DELETE）暂存，直到 flush 或 commit 时统一发送给数据库。
*   **底层原理**：
    *   `configuration.newExecutor` 是一个工厂方法，根据 `execType` 实例化不同的 Executor 实现类（`SimpleExecutor`, `ReuseExecutor`, `BatchExecutor`）。
    *   **装饰器模式**：MyBatis 会在基础 Executor 之上包裹一层 `CachingExecutor`（如果开启了二级缓存）。所以实际得到的对象往往是 `CachingExecutor` -> `SimpleExecutor` 这样的链式结构。
    *   **绑定事务**：Executor 内部持有了 `tx` 对象。当 Executor 需要执行 SQL 时，它会通过 `tx.getConnection()` 获取连接，确保所有操作都在同一个事务上下文中。

#### 5. 创建并返回 SqlSession
```java
return createSqlSession(configuration, executor, autoCommit);
```
*   **动作**：创建最终的 `DefaultSqlSession` 实例。
*   **角色**：`SqlSession` 是面向开发者的**API 门面**。它内部持有 `Executor` 和 `Configuration`。
*   **用户视角**：开发者调用 `sqlSession.selectOne(...)`，实际上是该方法内部委托给 `executor.query(...)` 去执行。

#### 6. 异常处理与资源清理
```java
} catch (Exception e) {
  closeTransaction(tx); 
  throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
} finally {
  ErrorContext.instance().reset();
}
```
*   **资源防泄漏**：如果在创建 `Executor` 或 `SqlSession` 的过程中发生异常（例如内存不足、配置错误），此时 `tx` 可能已经获取了物理连接。如果不关闭，会导致**连接泄漏**（连接池耗尽）。
    *   `closeTransaction(tx)`：确保调用 `tx.close()`，将连接归还给连接池。
*   **异常包装**：使用 `ExceptionFactory` 将底层_checked exception_（如 `SQLException`）统一包装为 MyBatis 特有的 `PersistenceException`（RuntimeException），简化上层调用处理。
*   **线程局部变量清理**：`ErrorContext` 是基于 `ThreadLocal` 的，用于在复杂调用栈中记录错误上下文（如正在执行的 SQL ID）。`finally` 块确保无论成功失败，当前线程的错误上下文都被重置，防止污染下一次请求。

---

### 二、背后/底层核心原理

#### 1. 组件生命周期管理 (Lifecycle Management)
这段代码清晰地定义了 MyBatis 组件的**创建顺序**和**依赖关系**：
> **Configuration** (全局) -> **Environment** -> **TransactionFactory** -> **Transaction** (含 Connection) -> **Executor** -> **SqlSession**

*   **强依赖**：`Executor` 依赖 `Transaction`，因为执行 SQL 必须得有连接；`SqlSession` 依赖 `Executor`，因为它是执行的具体实施者。
*   **作用域**：
    *   `Configuration`, `Environment`, `TransactionFactory`：**单例**，应用启动时创建，全程共享。
    *   `Transaction`, `Executor`, `SqlSession`：**请求级/会话级**。每次 `openSession` 都会创建新的一组。
    *   **特别注意**：在默认配置下，`Executor` 也是每次会话新建的（除非使用了特定的缓存策略或 Spring 集成时的特殊配置），但 `CachingExecutor` 内部的缓存是全局共享的。

#### 2. 事务与连接的绑定 (Transaction-Connection Binding)
*   **原理**：在 `newTransaction` 阶段，物理连接 (`Connection`) 就已经从数据源借出，并绑定到了 `Transaction` 对象上。
*   **影响**：
    *   这意味着 `openSession()` 是一个**重量级操作**（涉及网络交互、连接池分配）。
    *   **最佳实践**：`SqlSession` 应该尽可能短命（Request Scope），用完后立即 `close()`。如果把 `SqlSession` 做成单例或长生命周期的，会一直占用数据库连接，导致系统并发能力急剧下降。

#### 3. 执行器策略模式 (Executor Strategy Pattern)
通过 `execType` 参数，MyBatis 在运行时动态切换 SQL 执行策略：
*   **SIMPLE**：适合读多写少、无批量的场景。
*   **REUSE**：适合大量重复执行相同 SQL 的场景（减少 PreparedStatement 编译开销）。
*   **BATCH**：适合大批量写入。底层原理是调用 `Statement.addBatch()` 和 `executeBatch()`，将多次网络往返合并为一次，极大提升 IO 效率。
*   **底层实现**：`configuration.newExecutor` 内部是一个简单的 `switch-case` 或 `if-else` 工厂逻辑，体现了策略模式的灵活性。

#### 4. 异常安全与资源担保 (Exception Safety)
代码结构遵循了 **"Acquire-Use-Release"** 的资源管理模式，但在 "Use" 阶段（创建 Executor/SqlSession）发生异常时，必须保证 "Release"。
*   **为什么要在 catch 中 close？**
    *   通常我们习惯在 `finally` 中 close。但这里 `tx` 是在 `try` 块中间创建的。如果 `newTransaction` 成功，但 `newExecutor` 失败，`tx` 持有连接。
    *   如果在 `finally` 中 close，可能会误关闭一个**成功创建并返回**给用户的 Session 中的事务（逻辑复杂化）。
    *   **当前写法**：只在**发生异常**时关闭 `tx`。如果成功，`tx` 的生命权移交给了 `SqlSession`，由用户调用 `sqlSession.close()` 时间接关闭 `tx`。这是一种清晰的**所有权转移**语义。

#### 5. ThreadLocal 错误上下文
`ErrorContext.instance().reset()` 揭示了 MyBatis 如何处理复杂的错误追踪。
*   **问题**：当深层调用栈（Mapper -> Proxy -> Executor -> StatementHandler -> JDBC）出错时，原始异常可能丢失了“正在执行哪个 Mapper 语句”的信息。
*   **解决**：MyBatis 在执行过程中，不断向当前线程的 `ErrorContext` 追加信息（SQL ID, 参数等）。一旦报错，异常消息会包含这些上下文。
*   **重置必要性**：由于 Java 线程池复用线程，如果不 reset，下一个使用该线程的请求可能会看到上一个请求残留的错误信息，导致排查误导。

---

### 三、总结

`openSessionFromDataSource` 方法是 MyBatis **初始化会话的标准流水线**。

1.  **它不仅是创建对象**，更是**资源申请**（从连接池借连接）和**环境配置**（设置隔离级别、自动提交）的关键点。
2.  **它体现了高度的模块化**：事务、执行器、会话各司其职，通过依赖注入组装。
3.  **它强调了健壮性**：严格的异常捕获和资源释放逻辑，防止了最常见的“数据库连接泄漏”问题。
4.  **它是性能调优的入口**：通过 `execType` 和 `autoCommit` 参数，开发者可以在此处决定是采用批量模式还是复用预处理语句模式，从而适应不同的业务场景。

理解这段代码，就理解了 MyBatis 是如何从一个配置项变成一个可执行数据库操作的活对象的。

[目录](#目录)

#### 2.5 核心私有方法2：`openSessionFromConnection`（从外部连接创建会话）
用于复用外部传入的 `Connection`（如应用层手动管理连接、分布式事务场景），逻辑与 `openSessionFromDataSource` 类似，但无需从数据源获取连接。
```java
private SqlSession openSessionFromConnection(ExecutorType execType, Connection connection) {
  try {
    // 步骤1：获取外部连接的自动提交状态（获取失败则默认 true）
    boolean autoCommit;
    try {
      autoCommit = connection.getAutoCommit();
    } catch (SQLException e) {
      autoCommit = true; // 兼容不支持事务的驱动/数据库
    }
    // 步骤2：获取环境和事务工厂
    final Environment environment = configuration.getEnvironment();
    final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
    // 步骤3：基于外部连接创建事务
    final Transaction tx = transactionFactory.newTransaction(connection);
    // 步骤4：创建执行器
    final Executor executor = configuration.newExecutor(tx, execType);
    // 步骤5：创建 SqlSession
    return createSqlSession(configuration, executor, autoCommit);
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```
- 核心差异：
    - 无需从数据源获取连接，直接使用外部传入的 `Connection`；
    - 自动提交状态从外部连接获取，失败则默认 `true`（兼容低版本驱动/数据库）；
    - 事务工厂创建事务时，关联的是外部连接而非数据源。

#### 2.6 辅助方法解析
##### （1）`createSqlSession`（创建 SqlSession 实例）
```java
protected SqlSession createSqlSession(Configuration configuration, Executor executor, boolean autoCommit) {
  return new DefaultSqlSession(configuration, executor, autoCommit);
}
```
- 工厂方法：封装 `DefaultSqlSession` 的实例化，子类可重写此方法自定义 `SqlSession` 实现；
- `DefaultSqlSession` 是 `SqlSession` 的默认实现，持有配置、执行器、自动提交规则，是 MyBatis 执行 CRUD 的核心类。

##### （2）`getTransactionFactoryFromEnvironment`（获取事务工厂）
```java
private TransactionFactory getTransactionFactoryFromEnvironment(Environment environment) {
  if (environment == null || environment.getTransactionFactory() == null) {
    return new ManagedTransactionFactory();
  }
  return environment.getTransactionFactory();
}
```
- 兜底逻辑：无环境配置或事务工厂时，默认返回 `ManagedTransactionFactory`（托管事务工厂，不主动管理事务，由容器如 Spring 管理）；
- 正常场景：返回环境中配置的事务工厂（如 `JdbcTransactionFactory`，基于 JDBC 事务）。

##### （3）`closeTransaction`（关闭事务）
```java
private void closeTransaction(Transaction tx) {
  if (tx != null) {
    try {
      tx.close();
    } catch (SQLException ignore) {
      // 忽略异常，优先抛出原错误
    }
  }
}
```
- 连接泄漏防护：创建会话异常时，关闭已创建的事务（释放连接）；
- 异常忽略：避免关闭事务的异常覆盖原创建异常，保证错误信息准确。

##### （4）`getConfiguration`（获取配置）
```java
@Override
public Configuration getConfiguration() {
  return configuration;
}
```
- 对外暴露全局配置，允许开发者在创建会话后修改/获取配置（如动态添加映射器）。

#### 2.7 核心调用链路（以默认 `openSession()` 为例）
```
DefaultSqlSessionFactory.openSession()
  → openSessionFromDataSource(默认执行器, null, false)
    → 获取 Environment（数据源+JdbcTransactionFactory）
    → 创建 JdbcTransaction（关联数据源，autoCommit=false）
    → 创建 Executor（SimpleExecutor，应用插件）
    → createSqlSession → new DefaultSqlSession()
  → 返回 DefaultSqlSession 实例
```

#### 2.8 典型使用场景示例
```java
// 1. 构建 Configuration（MyBatis 启动时完成）
Configuration configuration = new Configuration();
configuration.setEnvironment(new Environment(
    "dev",
    new JdbcTransactionFactory(),
    new PooledDataSource("com.mysql.cj.jdbc.Driver", "jdbc:mysql://localhost:3306/test", "root", "123456")
));
configuration.addMapper(UserMapper.class);

// 2. 创建 SqlSessionFactory
SqlSessionFactory sqlSessionFactory = new DefaultSqlSessionFactory(configuration);

// 3. 创建 SqlSession（默认：SIMPLE 执行器，autoCommit=false）
try (SqlSession session = sqlSessionFactory.openSession()) {
  UserMapper mapper = session.getMapper(UserMapper.class);
  User user = mapper.selectById(1);
  session.commit(); // 手动提交事务
}

// 4. 批量执行场景（BATCH 执行器）
try (SqlSession session = sqlSessionFactory.openSession(ExecutorType.BATCH)) {
  UserMapper mapper = session.getMapper(UserMapper.class);
  for (int i = 0; i < 1000; i++) {
    mapper.insert(new User(i, "user" + i));
  }
  session.commit(); // 批量提交
}

// 5. 复用外部连接场景
Connection conn = ...; // 外部获取的连接
try (SqlSession session = sqlSessionFactory.openSession(conn)) {
  // 使用外部连接执行操作
}
```

### 3. 总结
#### 核心关键点
1. **类的核心职责**：
   `DefaultSqlSessionFactory` 是 MyBatis 创建 `SqlSession` 的工厂，封装了事务、执行器、连接的创建逻辑，提供多重载方法适配不同场景，是会话管理的入口。

2. **核心设计逻辑**：
    - 所有 `openSession` 方法最终转发到两个核心方法（数据源/外部连接），保证逻辑统一；
    - 依赖 `Configuration` 全局配置，所有会话共享同一套配置；
    - 异常处理优先保证连接释放（避免泄漏），并保留原始错误信息。

3. **关键组件关联**：
    - `Environment`：提供数据源和事务工厂；
    - `Transaction`：封装 JDBC 连接，管理事务；
    - `Executor`：SQL 执行器（核心，包含插件扩展）；
    - `DefaultSqlSession`：最终返回的会话实例，执行 CRUD。

4. **典型使用场景**：
    - 默认会话：`openSession()`（事务性，手动提交）；
    - 批量操作：`openSession(ExecutorType.BATCH)`；
    - 复用连接：`openSession(Connection)`（分布式事务/容器管理连接）。

#### 应用价值
- **理解 MyBatis 会话创建流程**：掌握该类可明确 `SqlSession` 的创建细节，排查连接泄漏、事务异常等问题；
- **自定义扩展**：重写 `createSqlSession` 可自定义 `SqlSession` 实现，或通过插件扩展执行器；
- **整合 Spring**：Spring 整合 MyBatis 时，`SqlSessionFactoryBean` 最终创建的就是 `DefaultSqlSessionFactory`，理解该类可掌握整合底层逻辑。


