



## Transaction接口



### 1. 核心需求理解
MyBatis 核心的 `Transaction` 接口——这个接口是 MyBatis 对数据库事务的**抽象封装**，定义了数据库连接的生命周期管理规范（获取连接、提交、回滚、关闭、超时设置），屏蔽了不同事务实现（如 JDBC 事务、Spring 托管事务）的细节，是 MyBatis 事务管理的顶层抽象。

### 2. 详细代码解释
#### 2.1 接口整体定位与设计初衷
```java
/**
 * Wraps a database connection. Handles the connection lifecycle that comprises: its creation, preparation,
 * commit/rollback and close.
 *
 * @author Clinton Begin
 */
public interface Transaction {
  // 核心方法定义
}
```
- **核心定位**：
    - MyBatis 事务管理的**顶层抽象接口**：定义了事务操作的最小行为集，所有具体事务实现（如 `JdbcTransaction`、`ManagedTransaction`）都需实现此接口；
    - 连接生命周期封装：专注于数据库连接的“获取-使用-提交/回滚-关闭”全生命周期，不涉及业务逻辑，仅做连接/事务的基础管理；
    - 解耦设计：屏蔽不同环境下的事务实现差异（如纯 JDBC 事务、Spring 容器托管事务），让 MyBatis 核心逻辑无需关心底层事务类型。
- **设计目的**：
  解决“不同环境下事务管理方式不同”的问题——比如：
    - 独立使用 MyBatis 时，用 `JdbcTransaction` 直接管理 JDBC 连接；
    - 整合 Spring 时，用 `SpringManagedTransaction` 交由 Spring 管理连接/事务；
      MyBatis 核心逻辑只需依赖 `Transaction` 接口，无需修改代码即可适配不同环境。

#### 2.2 核心方法逐一解析
##### （1）`getConnection()`：获取数据库连接
```java
/**
 * Retrieve inner database connection.
 *
 * @return DataBase connection
 * @throws SQLException the SQL exception
 */
Connection getConnection() throws SQLException;
```
- **核心语义**：获取当前事务关联的数据库连接（`java.sql.Connection`），是所有数据库操作的基础；
- **实现逻辑差异**：
    - `JdbcTransaction`：从数据源（`DataSource`）获取新连接，或返回已持有的连接；
    - `SpringManagedTransaction`：返回 Spring 容器管理的连接（由 `DataSourceTransactionManager` 分配）；
- **异常**：`SQLException`——获取连接失败时抛出（如数据源配置错误、数据库宕机）；
- **使用场景**：MyBatis 的 `Executor` 执行 SQL 前，会通过此方法获取连接，创建 `PreparedStatement`。

##### （2）`commit()`：提交事务
```java
/**
 * Commit inner database connection.
 *
 * @throws SQLException the SQL exception
 */
void commit() throws SQLException;
```
- **核心语义**：提交当前事务，将所有未提交的操作持久化到数据库；
- **实现逻辑差异**：
    - `JdbcTransaction`：调用 `Connection.commit()`，前提是连接的 `autoCommit` 为 `false`；
    - `SpringManagedTransaction`：不执行实际提交（由 Spring 统一管理），仅做日志/状态记录；
- **异常**：`SQLException`——提交失败时抛出（如数据库连接已关闭、事务隔离级别冲突）；
- **使用场景**：`SqlSession.commit()` 底层会调用此方法，完成事务提交。

##### （3）`rollback()`：回滚事务
```java
/**
 * Rollback inner database connection.
 *
 * @throws SQLException the SQL exception
 */
void rollback() throws SQLException;
```
- **核心语义**：回滚当前事务，撤销所有未提交的操作；
- **实现逻辑差异**：
    - `JdbcTransaction`：调用 `Connection.rollback()`；
    - `SpringManagedTransaction`：不执行实际回滚（由 Spring 处理）；
- **异常**：`SQLException`——回滚失败时抛出（如连接失效、无事务可回滚）；
- **使用场景**：`SqlSession.rollback()` 底层调用此方法，或执行 SQL 异常时自动回滚。

##### （4）`close()`：关闭连接/释放资源
```java
/**
 * Close inner database connection.
 *
 * @throws SQLException the SQL exception
 */
void close() throws SQLException;
```
- **核心语义**：关闭当前事务关联的数据库连接，释放资源；
- **实现逻辑差异**：
    - `JdbcTransaction`：调用 `Connection.close()`，真正关闭连接；
    - `SpringManagedTransaction`：不关闭连接（由 Spring 回收），仅清理状态；
- **异常**：`SQLException`——关闭连接失败时抛出（如连接已关闭）；
- **使用场景**：`SqlSession.close()` 底层会调用此方法，释放连接资源，避免连接泄漏。

##### （5）`getTimeout()`：获取事务超时时间
```java
/**
 * Get transaction timeout if set.
 *
 * @return the timeout
 * @throws SQLException the SQL exception
 */
Integer getTimeout() throws SQLException;
```
- **核心语义**：获取当前事务的超时时间（单位：秒），超时后事务会被自动回滚；
- **实现逻辑**：
    - 底层调用 `Connection.getNetworkTimeout()` 或数据源的超时配置；
    - 无超时设置时返回 `null`；
- **异常**：`SQLException`——获取超时配置失败时抛出；
- **使用场景**：MyBatis 执行器（`Executor`）会根据此超时时间设置 `PreparedStatement` 的超时，避免 SQL 执行阻塞。

#### 2.3 核心实现类（补充理解）
`Transaction` 接口的核心实现类有 3 个，覆盖不同使用场景：

| 实现类 | 适用场景 | 核心特点 |
|--------|----------|----------|
| `JdbcTransaction` | 独立使用 MyBatis（无 Spring） | 直接管理 JDBC 连接，`commit()`/`rollback()` 调用 `Connection` 原生方法，`close()` 真正关闭连接 |
| `ManagedTransaction` | 容器托管环境（如 Spring、JBoss） | 不主动管理事务/连接，`commit()`/`rollback()` 为空实现，`close()` 也不关闭连接（由容器管理） |
| `SpringManagedTransaction` | MyBatis + Spring 整合 | 适配 Spring 事务管理器，连接由 Spring 分配，事务由 Spring 提交/回滚 |

**示例：JdbcTransaction 核心实现（简化版）**
```java
public class JdbcTransaction implements Transaction {
  private final DataSource dataSource;
  private Connection connection;
  private boolean autoCommit;

  @Override
  public Connection getConnection() throws SQLException {
    if (connection == null) {
      openConnection(); // 从数据源获取连接
    }
    return connection;
  }

  @Override
  public void commit() throws SQLException {
    if (connection != null && !connection.getAutoCommit()) {
      connection.commit(); // 调用 JDBC 原生提交
    }
  }

  @Override
  public void rollback() throws SQLException {
    if (connection != null && !connection.getAutoCommit()) {
      connection.rollback(); // 调用 JDBC 原生回滚
    }
  }

  @Override
  public void close() throws SQLException {
    if (connection != null) {
      resetAutoCommit(); // 恢复自动提交状态
      connection.close(); // 真正关闭连接
    }
  }

  @Override
  public Integer getTimeout() throws SQLException {
    return connection != null ? connection.getNetworkTimeout() : null;
  }
}
```

#### 2.4 与 MyBatis 核心组件的关联
`Transaction` 是 MyBatis 会话（`SqlSession`）和执行器（`Executor`）的基础依赖，核心调用链路：
```
// 1. 创建 SqlSession 时，由 TransactionFactory 创建 Transaction
DefaultSqlSessionFactory.openSessionFromDataSource()
  → TransactionFactory.newTransaction() → 创建 JdbcTransaction/ManagedTransaction

// 2. Executor 持有 Transaction 实例，执行 SQL 时获取连接
SimpleExecutor.doUpdate()
  → Transaction.getConnection() → 获取连接
  → 创建 PreparedStatement 执行 SQL

// 3. SqlSession 提交/回滚时，调用 Transaction 方法
DefaultSqlSession.commit()
  → Executor.commit() → Transaction.commit()
```

#### 2.5 典型使用示例（独立使用 MyBatis）
```java
// 1. 配置数据源和事务工厂
DataSource dataSource = new PooledDataSource("com.mysql.cj.jdbc.Driver", 
    "jdbc:mysql://localhost:3306/test", "root", "123456");
TransactionFactory txFactory = new JdbcTransactionFactory();

// 2. 创建事务
Transaction tx = txFactory.newTransaction(dataSource, null, false);

// 3. 使用事务执行操作
try (Connection conn = tx.getConnection()) {
  // 执行 SQL...
  tx.commit(); // 提交事务
} catch (SQLException e) {
  tx.rollback(); // 回滚事务
} finally {
  tx.close(); // 关闭连接
}
```

### 3. 总结
#### 核心关键点
1. **接口的核心定位**：
   MyBatis 事务管理的顶层抽象，定义了数据库连接/事务的基础操作（获取、提交、回滚、关闭、超时），屏蔽不同环境下的实现差异。

2. **核心方法的职责**：
    - `getConnection()`：获取连接（所有 SQL 操作的基础）；
    - `commit()`/`rollback()`：事务提交/回滚（核心事务操作）；
    - `close()`：关闭连接（释放资源，避免泄漏）；
    - `getTimeout()`：获取事务超时（控制 SQL 执行时长）。

3. **设计价值**：
    - 解耦：MyBatis 核心逻辑（Executor/SqlSession）只需依赖 `Transaction` 接口，无需关心底层是 JDBC 事务还是 Spring 托管事务；
    - 扩展：自定义事务实现（如分布式事务）只需实现此接口，即可无缝集成到 MyBatis；
    - 规范：统一了事务操作的行为，保证不同实现类的行为一致性。

4. **实现类适配场景**：
    - 独立使用 MyBatis：`JdbcTransaction`（原生 JDBC 事务）；
    - 整合 Spring：`SpringManagedTransaction`（Spring 托管事务）；
    - 容器环境：`ManagedTransaction`（容器管理连接）。

#### 应用价值
- **事务问题排查**：理解 `Transaction` 接口的方法，可快速定位事务相关问题（如提交失败→检查 `commit()` 实现、连接泄漏→检查 `close()` 调用）；
- **自定义事务扩展**：如需实现分布式事务（如 Seata），可通过实现 `Transaction` 接口适配 MyBatis；
- **Spring 整合理解**：MyBatis + Spring 整合的核心是 `SpringManagedTransaction`，通过此接口实现了与 Spring 事务管理器的无缝对接。

