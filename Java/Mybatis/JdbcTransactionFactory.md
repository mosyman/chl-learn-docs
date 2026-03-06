
## 目录
- [TransactionFactory and JdbcTransactionFactory](#transactionfactory-and-jdbctransactionfactory)


## TransactionFactory and JdbcTransactionFactory

这段代码展示了 **MyBatis 框架中事务管理模块的核心设计**，具体涉及 **工厂模式 (Factory Pattern)** 和 **策略模式 (Strategy Pattern)** 的应用。

它定义了如何创建 `Transaction`（事务）对象，并允许用户通过配置自定义事务行为。

---

### 一、代码功能详解

#### 1. 接口：`TransactionFactory`
这是**抽象层**，定义了创建事务的标准契约。
*   **职责**：无论底层是使用 JDBC 原生事务、JTA 分布式事务，还是 Spring 管理的事务，调用者只需要面对这个接口，不需要关心具体的实现类。
*   **核心方法**：
    *   `setProperties(Properties props)`: **配置入口**。允许用户在 XML 或 Java Config 中传入自定义参数（如 `skipSetAutoCommitOnClose`），工厂根据这些参数调整内部状态。
    *   `newTransaction(Connection conn)`: **基于现有连接创建事务**。通常用于外部已经管理了连接（如 Spring 事务管理器）的场景，MyBatis 只是“借用”这个连接进行操作，不接管连接的开启/关闭。
    *   `newTransaction(DataSource ds, ...)`: **基于数据源创建事务**。MyBatis 需要自己从数据源获取连接，并设置隔离级别 (`level`) 和自动提交 (`autoCommit`) 属性。这是 MyBatis 独立运行时的典型场景。

#### 2. 实现类：`JdbcTransactionFactory`
这是**具体实现层**，专门用于创建基于 **JDBC 原生机制** 的事务对象 (`JdbcTransaction`)。
*   **成员变量 `skipSetAutoCommitOnClose`**:
    *   这是一个布尔标志位，默认 `false`。
    *   **作用**：控制当事务对象关闭（`close()`）时，是否**跳过**将连接的 `autoCommit` 属性重置为 `true` 的操作。
    *   **场景**：在某些连接池或特殊环境下，频繁修改 `autoCommit` 可能会影响性能或导致意外行为，用户可以通过配置 `<property name="skipSetAutoCommitOnClose" value="true"/>` 来禁用该重置操作。
*   **方法实现**：
    *   `setProperties`: 解析配置，设置内部标志位。
    *   `newTransaction(Connection conn)`: 直接包装现有连接。**注意**：这里没有传入 `skipSetAutoCommitOnClose`，意味着对于外部传入的连接，`JdbcTransaction` 内部可能使用默认行为或不处理该标志（需查看 `JdbcTransaction` 构造函数源码确认，通常外部连接模式下由外部控制生命周期）。
    *   `newTransaction(DataSource ds, ...)`: 创建完整的 `JdbcTransaction` 实例，并将配置好的 `skipSetAutoCommitOnClose` 传递进去，以便在事务提交/回滚/关闭时执行相应的逻辑。

---

### 二、背后/底层原理深度剖析

#### 1. 设计模式：工厂模式 (Factory Pattern)
*   **原理**：将对象的**创建逻辑**与**使用逻辑**分离。
*   **在 MyBatis 中的体现**：
    *   `SqlSession` 或 `Executor` 需要使用 `Transaction`，但它们不直接 `new JdbcTransaction()`。
    *   它们持有 `TransactionFactory` 接口引用，调用 `factory.newTransaction(...)`。
*   **优势**：
    *   **解耦**：如果未来 MyBatis 要支持一个新的事务类型（比如 `ReactiveTransaction`），只需新建一个 `ReactiveTransactionFactory` 实现接口，修改配置文件即可，无需修改 `SqlSession` 等核心代码。
    *   **可扩展性**：用户可以编写自己的 `CustomTransactionFactory` 来实现特殊的事务逻辑（如记录事务耗时、自动重试等）。

#### 2. 事务管理的两种模式 (核心原理)

代码中两个 `newTransaction` 方法对应了两种截然不同的事务管理模式，理解这一点对排查并发问题至关重要。

##### 模式 A：自主管理 (Autonomous Management)
*   **调用方法**：`newTransaction(DataSource ds, ...)`
*   **场景**：MyBatis 作为独立框架运行，或者在非 Spring 环境中。
*   **底层行为**：
    1.  **获取连接**：工厂内部调用 `ds.getConnection()`。
    2.  **配置连接**：根据参数调用 `conn.setTransactionIsolation(level)` 和 `conn.setAutoCommit(autoCommit)`。
        *   通常 `autoCommit` 会被设为 `false`，以开启手动事务。
    3.  **生命周期**：`JdbcTransaction` 对象拥有该连接的**完全控制权**。当调用 `transaction.commit()` 或 `rollback()` 时，它直接操作连接。当调用 `transaction.close()` 时，它会**关闭连接** (`conn.close()`) 并归还给连接池。
    4.  **AutoCommit 重置**：在关闭前，如果 `skipSetAutoCommitOnClose` 为 `false`，它会将 `autoCommit` 改回 `true`。这是为了**保护连接池**。因为连接池中取出的连接默认通常是 `autoCommit=true`，如果不重置，下一个使用该连接的组件（可能是非 MyBatis 的代码）可能会误以为自己在自动提交模式下，导致数据不一致。

##### 模式 B：参与式管理 (Participatory Management)
*   **调用方法**：`newTransaction(Connection conn)`
*   **场景**：与 Spring 集成，或在 Servlet 容器中，事务由外部容器（如 Spring 的 `DataSourceTransactionManager`）管理。
*   **底层行为**：
    1.  **接收连接**：直接使用传入的 `conn`。
    2.  **不配置连接**：**不**调用 `setAutoCommit` 或 `setIsolation`。假设外部容器已经配置好了。
    3.  **生命周期**：`JdbcTransaction` **不拥有**连接的所有权。
        *   调用 `commit()`/`rollback()`：通常委托给外部管理器，或者什么都不做（取决于具体实现，MyBatis 的 `JdbcTransaction` 在这种模式下通常**不会**真正 commit/rollback，而是由外部容器决定何时提交）。
        *   调用 `close()`：**绝对不关闭连接**。它只是释放内部引用，连接的生命周期由外部容器管理。
*   **原理意义**：这实现了**事务同步**。Spring 开启事务 -> 获取连接 -> 传给 MyBatis -> MyBatis 执行 SQL -> Spring 决定提交/回滚 -> Spring 关闭连接。MyBatis 只是一个执行者，不干涉事务边界。

#### 3. `skipSetAutoCommitOnClose` 的深层原理
为什么需要这个配置？
*   **默认行为 (`false`)**：
    *   流程：`close()` -> `conn.setAutoCommit(true)` -> `conn.close()`。
    *   **目的**：确保归还给连接池的连接是“干净”的（恢复默认状态）。防止连接池复用时，下一个使用者继承了一个 `autoCommit=false` 的连接，导致它的单条 SQL 无法自动提交，一直锁表直到超时。
*   **特殊行为 (`true`)**：
    *   流程：`close()` -> `conn.close()` (跳过 reset)。
    *   **适用场景**：
        1.  **性能敏感**：某些数据库驱动（如旧版 Oracle Driver）在切换 `autoCommit` 时开销巨大，且应用能保证连接池初始化时已统一设置好。
        2.  **特殊中间件**：某些分布式数据库中间件或代理（Proxy）不支持动态修改 `autoCommit` 属性，强制修改会报错。
        3.  **外部接管**：虽然是在自主模式下，但后续有特殊的过滤器会在连接关闭前统一处理状态。

#### 4. SPI 与配置加载原理
MyBatis 如何通过 XML 加载这个工厂？
*   **XML 配置**：
    ```xml
    <transactionManager type="JDBC">
      <property name="skipSetAutoCommitOnClose" value="true"/>
    </transactionManager>
    ```
*   **底层流程**：
    1.  MyBatis 解析 XML，读取 `type="JDBC"`。
    2.  通过别名机制找到 `JdbcTransactionFactory` 类。
    3.  利用反射实例化该类。
    4.  调用 `setProperties(props)` 方法，将 `<property>` 标签中的内容传入。
    5.  将该工厂实例存储在 `Configuration` 对象中，后续创建 `SqlSession` 时从中获取。

---

### 三、总结与对比

| 特性 | `newTransaction(DataSource...)` | `newTransaction(Connection)` |
| :--- | :--- | :--- |
| **连接来源** | 工厂内部从 DataSource 获取 | 外部传入 |
| **连接所有权** | **拥有** (负责关闭) | **不拥有** (不负责关闭) |
| **AutoCommit 设置** | 根据参数显式设置 | **不设置** (信任外部) |
| **事务边界控制** | MyBatis 控制 (commit/rollback 立即生效) | 外部控制 (MyBatis 通常不实际操作) |
| **典型场景** |  standalone MyBatis, 单元测试 | Spring `@Transactional`, 容器管理事务 |
| **`skipSetAutoCommitOnClose` 影响** | **有效** (决定是否在 close 前重置状态) | **通常无效** (因为不管理连接关闭逻辑) |

**核心价值**：
这段代码体现了 MyBatis 极高的**适应性**。它既能在没有容器的环境下独立管理完整的事务生命周期（包括连接获取、配置、提交、关闭、状态重置），又能完美地融入 Spring 等强大的事务容器中，甘当“配角”，不争夺事务控制权。而 `TransactionFactory` 接口的设计，则为这种灵活性提供了架构基础。


[目录](#目录)
