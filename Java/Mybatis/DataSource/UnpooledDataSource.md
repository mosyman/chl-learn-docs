
## 目录
- [MyBatis 中 UnpooledDataSource](#mybatis-中的-unpooleddatasource)
- [doGetConnection(String username, String password)](#dogetconnectionstring-username-string-password)
- [doGetConnection(Properties properties)](#dogetconnectionproperties-properties)


## MyBatis 中的 UnpooledDataSource

### 一、类的整体定位与核心作用
`UnpooledDataSource` 是 MyBatis 提供的**非池化数据源实现类**，实现了 JDBC 标准的 `javax.sql.DataSource` 接口。
- 核心特点：每次调用 `getConnection()` 都会创建一个全新的数据库连接，使用完后直接关闭（不会复用），没有连接池的复用机制。
- 适用场景：简单测试、低并发场景，或作为自定义连接池的基础组件。
- 对比：与 `PooledDataSource`（池化数据源）相反，它不维护连接池，无连接复用、无最大连接数限制，实现更简单但性能较低。

### 二、核心成员变量解析
先理解类中定义的关键属性，这些是创建数据库连接的核心配置：

| 变量名 | 作用 |
|--------|------|
| `driverClassLoader` | 加载 JDBC 驱动类的类加载器（自定义类加载场景用） |
| `driverProperties` | 驱动相关的额外属性（如连接参数） |
| `registeredDrivers` | 静态并发 Map，缓存已注册的 JDBC Driver 实例（避免重复加载） |
| `driver` | JDBC 驱动类全限定名（如 `com.mysql.cj.jdbc.Driver`） |
| `url` | 数据库连接 URL（如 `jdbc:mysql://localhost:3306/test`） |
| `username`/`password` | 数据库认证信息 |
| `autoCommit` | 连接的自动提交模式（null 则使用数据库默认） |
| `defaultTransactionIsolationLevel` | 默认事务隔离级别（如 READ_COMMITTED） |
| `defaultNetworkTimeout` | 数据库操作的网络超时时间（毫秒） |

### 三、关键代码逻辑拆解
#### 1. 静态代码块：初始化已注册的 Driver
```java
static {
  Enumeration<Driver> drivers = DriverManager.getDrivers();
  while (drivers.hasMoreElements()) {
    Driver driver = drivers.nextElement();
    registeredDrivers.put(driver.getClass().getName(), driver);
  }
}
```
- 作用：类加载时，从 `DriverManager` 中获取所有已注册的 JDBC 驱动，缓存到 `registeredDrivers` 中，避免重复加载。
- 为什么要做？JDBC 驱动通常会在类加载时自动注册到 `DriverManager`，这里提前缓存，后续创建连接时直接复用。

#### 2. 构造方法：初始化连接配置
提供了多个重载构造方法，核心是接收 `driver`、`url`、`username/password` 或 `driverProperties` 等配置，本质是给成员变量赋值，满足不同场景的初始化需求（比如自定义类加载器、自定义驱动属性）。

#### 3. 核心方法：`getConnection()`
这是 `DataSource` 接口的核心方法，用于获取数据库连接，底层调用 `doGetConnection()`：
```java
@Override
public Connection getConnection() throws SQLException {
  return doGetConnection(username, password);
}

@Override
public Connection getConnection(String username, String password) throws SQLException {
  return doGetConnection(username, password);
}
```
- 两个重载方法：一个使用类中配置的用户名密码，一个允许临时传入用户名密码，灵活性更高。

#### 4. 核心实现：`doGetConnection()`
这是创建连接的核心逻辑，分三步：
```java
private Connection doGetConnection(Properties properties) throws SQLException {
  // 步骤1：初始化/加载 JDBC 驱动
  initializeDriver();
  // 步骤2：通过 DriverManager 获取全新连接（无池化，每次新建）
  Connection connection = DriverManager.getConnection(url, properties);
  // 步骤3：配置连接（自动提交、事务隔离级别、网络超时）
  configureConnection(connection);
  return connection;
}
```

##### 步骤1：`initializeDriver()` 加载驱动
```java
private void initializeDriver() throws SQLException {
  try {
    // computeIfAbsent：如果 driver 未在 registeredDrivers 中，则执行 Lambda 加载
    registeredDrivers.computeIfAbsent(driver, x -> {
      Class<?> driverType;
      try {
        // 加载驱动类（优先用自定义类加载器，否则用 MyBatis 的 Resources 工具类）
        if (driverClassLoader != null) {
          driverType = Class.forName(x, true, driverClassLoader);
        } else {
          driverType = Resources.classForName(x);
        }
        // 创建驱动实例，并通过 DriverProxy 代理后注册到 DriverManager
        Driver driverInstance = (Driver) driverType.getDeclaredConstructor().newInstance();
        DriverManager.registerDriver(new DriverProxy(driverInstance));
        return driverInstance;
      } catch (Exception e) {
        throw new RuntimeException("Error setting driver on UnpooledDataSource.", e);
      }
    });
  } catch (RuntimeException re) {
    throw new SQLException("Error setting driver on UnpooledDataSource.", re.getCause());
  }
}
```
- 核心逻辑：
    1. 检查 `registeredDrivers` 中是否已有该驱动，没有则加载；
    2. 用指定类加载器加载驱动类，创建实例；
    3. 通过 `DriverProxy` 代理驱动后注册到 `DriverManager`（代理的目的是解耦，避免直接依赖具体驱动实现）；
    4. 异常转换：将运行时异常转为 SQLException，符合 JDBC 规范。

##### 步骤2：`DriverManager.getConnection()`
通过 JDBC 标准的 `DriverManager` 创建全新连接——这是“非池化”的核心体现：**每次调用都会新建连接，而非复用已有连接**。

##### 步骤3：`configureConnection()` 配置连接
```java
private void configureConnection(Connection conn) throws SQLException {
  // 设置网络超时（如果配置了）
  if (defaultNetworkTimeout != null) {
    conn.setNetworkTimeout(Executors.newSingleThreadExecutor(), defaultNetworkTimeout);
  }
  // 设置自动提交（如果配置了，且与当前连接默认值不同）
  if (autoCommit != null && autoCommit != conn.getAutoCommit()) {
    conn.setAutoCommit(autoCommit);
  }
  // 设置事务隔离级别（如果配置了）
  if (defaultTransactionIsolationLevel != null) {
    conn.setTransactionIsolation(defaultTransactionIsolationLevel);
  }
}
```
- 作用：对新建的连接做统一配置，保证连接的行为符合预期（比如统一的自动提交策略、事务隔离级别）。

#### 5. 内部类：`DriverProxy`（驱动代理）
```java
private static class DriverProxy implements Driver {
  private final Driver driver;

  DriverProxy(Driver d) {
    this.driver = d;
  }

  // 所有方法都委托给真正的 Driver 实例实现
  @Override
  public boolean acceptsURL(String u) throws SQLException {
    return this.driver.acceptsURL(u);
  }
  // ... 其他方法（connect、getMajorVersion 等）都是简单委托
}
```
- 本质：这是一个**静态内部代理类**，实现了 `Driver` 接口，所有方法都委托给真正的驱动实例。
- 目的：
    1. 解耦：MyBatis 不直接依赖具体的 JDBC 驱动实现；
    2. 安全：避免驱动类的直接暴露；
    3. 兼容：统一驱动的调用方式。

#### 6. 其他方法：实现 `DataSource` 接口的冗余方法
`unwrap()`、`isWrapperFor()`、`getParentLogger()` 等方法是 `DataSource` 接口的要求，但 `UnpooledDataSource` 没有实现包装/解包逻辑，因此：
- `unwrap()` 直接抛出 SQLException；
- `isWrapperFor()` 返回 false；
- `getParentLogger()` 返回全局日志器（兼容 JDBC 4.0+）。

另外，`setLoginTimeout()`、`getLogWriter()` 等方法直接委托给 `DriverManager` 实现，复用 JDBC 标准的配置。

### 四、使用示例（辅助理解）
```java
// 创建 UnpooledDataSource 实例
UnpooledDataSource dataSource = new UnpooledDataSource();
dataSource.setDriver("com.mysql.cj.jdbc.Driver");
dataSource.setUrl("jdbc:mysql://localhost:3306/test?useSSL=false");
dataSource.setUsername("root");
dataSource.setPassword("123456");
dataSource.setAutoCommit(false); // 关闭自动提交

// 获取连接（每次都是新连接）
try (Connection conn = dataSource.getConnection()) {
  // 执行数据库操作
  System.out.println("连接是否自动提交：" + conn.getAutoCommit()); // false
} catch (SQLException e) {
  e.printStackTrace();
}
```
- 输出：`连接是否自动提交：false`，验证了 `configureConnection` 的配置生效。
- 注意：每次调用 `getConnection()` 都会新建连接，使用后需手动关闭（或用 try-with-resources 自动关闭）。

### 总结
1. `UnpooledDataSource` 是 MyBatis 的**非池化数据源**，实现 `DataSource` 接口，每次获取连接都会新建，无复用机制；
2. 核心逻辑：加载驱动 → 创建新连接 → 配置连接属性（自动提交、事务隔离级别等）；
3. 关键设计：用 `registeredDrivers` 缓存驱动避免重复加载，用 `DriverProxy` 代理驱动解耦，所有配置最终委托给 `DriverManager` 创建连接。

这个类的设计核心是**简单、通用**，虽然性能不如池化数据源，但胜在实现简单、无额外依赖，是 MyBatis 数据源体系的基础组件。



[目录](#目录)


## doGetConnection(String username, String password)

这段代码通常出现在 Java 数据库连接池（如 **MyBatis 的 `PooledDataSource`** 或类似的自定义数据源实现）中。它的主要作用是**构建包含认证信息的属性集，并委托给另一个方法来建立实际的数据库连接**。

```java
private Connection doGetConnection(String username, String password) throws SQLException {
    // 1. 创建一个 Properties 对象，用于存储连接数据库所需的配置键值对
    Properties props = new Properties();
    
    // 2. 如果当前数据源对象中已经预定义了一些驱动属性 (driverProperties)
    //    (例如: url, driver, autoCommit, poolTimeout 等)，将它们全部复制到 props 中
    if (driverProperties != null) {
      props.putAll(driverProperties);
    }
    
    // 3. 如果调用者传入了 username，则将其放入 props，键名为 "user"
    //    注意：JDBC 标准中，用户名对应的 key 通常是 "user" 而不是 "username"
    if (username != null) {
      props.setProperty("user", username);
    }
    
    // 4. 如果调用者传入了 password，则将其放入 props，键名为 "password"
    if (password != null) {
      props.setProperty("password", password);
    }
    
    // 5. 调用重载方法 doGetConnection(Properties props)，传入组装好的属性集
    //    真正的建立连接逻辑（如调用 DriverManager.getConnection）通常在那个方法里
    return doGetConnection(props);
}
```

#### 关键点说明：
*   **`Properties props = new Properties()`**: `Properties` 是 Java 中专门用于处理配置文件的类，继承自 `Hashtable`。在 JDBC 中，`DriverManager` 和 `DataSource` 广泛使用它来传递连接参数。
*   **`props.putAll(driverProperties)`**: 这里体现了**配置合并**的思想。`driverProperties` 通常是在初始化数据源时配置的全局默认值（比如数据库 URL `jdbc:mysql://...`）。
*   **`props.setProperty("user", username)`**: 这是一个非常关键的细节。JDBC 规范规定，当通过 `Properties` 对象传递认证信息时，**用户名的键必须是 `"user"`**，而不是 `"username"`。如果写错，数据库驱动可能会忽略该参数导致认证失败。
*   **`return doGetConnection(props)`**: 这是**方法重载（Overloading）**的应用。这个接受 `(String, String)` 的方法只是一个“适配器”或“便利方法”，它将字符串参数转换为 `Properties` 对象，然后交给核心逻辑处理，避免代码重复。

---

### 二、背后原理与设计思想

这段代码虽然简单，但蕴含了几个重要的软件设计和 JDBC 工作原理：

#### 1. JDBC 驱动的参数传递机制
Java 的 `java.sql.DriverManager` 在建立连接时，主要有两种方式传递参数：
*   **URL 拼接方式**: `DriverManager.getConnection("jdbc:mysql://host/db?user=abc&password=123")`
*   **Properties 对象方式**: `DriverManager.getConnection(url, props)`

**为什么代码中要用 Properties？**
*   **安全性与清晰性**: 将密码放在 URL 中可能会在日志中泄露。使用 `Properties` 对象可以将敏感信息与连接 URL 分离。
*   **灵活性**: `Properties` 可以容纳任意数量的驱动特定参数（如 `ssl`, `characterEncoding`, `connectTimeout` 等），而不仅仅是 user 和 password。
*   **优先级覆盖**: 代码中的逻辑允许**动态参数覆盖静态配置**。如果 `driverProperties` 里已经有了一个 "user"，但方法调用时传入了新的 `username`，后面的 `setProperty` 会覆盖前面的值。这允许在全局默认配置的基础上，为特定连接临时指定不同的用户。

#### 2. 策略模式与配置合并 (Configuration Merging)
*   **默认配置 (`driverProperties`)**: 代表“全局默认值”。通常在应用启动时读取配置文件（如 `mybatis-config.xml` 或 `.properties` 文件）设定。
*   **运行时参数 (`username`, `password`)**: 代表“临时覆盖值”。可能在运行时根据当前登录用户动态决定。
*   **合并逻辑**: 代码先 `putAll` (加载默认)，再 `setProperty` (应用临时)。这是一种经典的**配置层级合并**策略，确保了灵活性和默认值的便利性。

#### 3. 适配器模式与方法重载
*   该方法是一个典型的**Facade（外观）**或**Adapter（适配器）**角色。
*   系统内部的核心逻辑可能只认 `Properties` 对象（因为 JDBC 底层驱动这么要求）。
*   但是，为了方便开发者调用（有时候开发者手里只有明文的用户名密码字符串，不想手动 new 一个 Properties 去 put），提供了这个接受 `(String, String)` 的便捷方法。
*   它负责将“外部接口格式”转换为“内部核心格式”，保持了核心代码 `doGetConnection(Properties)` 的单一职责。

#### 4. 空指针安全与健壮性
*   代码中使用了 `if (driverProperties != null)` 和 `if (username != null)` 判断。
*   这防止了当全局配置未初始化，或者调用者只想用默认用户（不传参）时抛出 `NullPointerException`。
*   如果 `username` 为 `null`，代码不会强行放入一个 `null` 值到 properties 中，而是依赖 `driverProperties` 中已有的配置（如果有）或让数据库驱动使用默认行为。

### 总结

这段代码是 **JDBC 连接建立过程中的“参数组装车间”**。

1.  它收集**全局默认配置**。
2.  它融合**运行时传入的认证信息**（并修正键名为 JDBC 标准的 `user`）。
3.  它将组装好的完整配置包传递给底层执行者去创建物理连接。



##### 步骤1：初始化 `Properties` 参数容器
- **代码逻辑**：创建空的 `Properties` 对象，作为连接参数的统一容器；
- **背后原理**：
  JDBC 规范定义了**基于 `Properties` 传递连接参数**的标准——`DriverManager.getConnection(url, props)` 或驱动原生的连接创建方法，都支持通过 `Properties` 传递参数（如 user、password、characterEncoding 等）。
  `Properties` 是 Java 内置的键值对容器（继承 `Hashtable`），兼容字符串类型的参数传递，且支持加载/存储配置，是 JDBC 标准的参数传递方式。

##### 步骤2：合并驱动自定义参数（`driverProperties`）
- **代码逻辑**：如果数据源初始化时配置了驱动自定义参数（如 `useSSL=false`、`serverTimezone=UTC`），则合并到当前参数容器；
- **背后原理**：
  不同数据库驱动有大量自定义参数（如 MySQL 的 `rewriteBatchedStatements=true`、Oracle 的 `defaultRowPrefetch=100`），这些参数需要随连接一起传递给驱动，才能生效。
  `driverProperties` 通常是数据源初始化时通过配置文件/代码设置的全局参数（如 HikariCP 的 `dataSourceProperties`），合并到当前参数容器后，所有参数可一次性传递给驱动，避免参数分散。

##### 步骤3-4：设置用户名/密码（覆盖逻辑）
- **代码逻辑**：如果方法传入了 `username`/`password`，则设置到 `Properties` 中（key 固定为 `user`/`password`）；
- **背后原理**：
  ① **JDBC 标准参数名**：JDBC 规范规定，用户名的参数名是 `user`（而非 `username`），密码是 `password`，所有数据库驱动都遵循此规范；
  ② **覆盖优先级**：方法传入的用户名/密码 > 驱动全局参数（`driverProperties`）中的 `user`/`password`——这是合理的设计：临时传入的认证信息（如多租户场景）应覆盖全局配置；
  ③ **空值兼容**：若 `username`/`password` 为 null，则不覆盖，使用 `driverProperties` 中的默认值（或驱动的默认配置）。

##### 步骤5：转发到带 `Properties` 的 `doGetConnection(props)`
- **代码逻辑**：将封装好的参数容器传递给真正的连接创建方法；
- **背后原理**：
  这是**职责分离**的设计模式——当前方法仅负责“参数封装”，不负责“连接创建”，真正的连接创建逻辑（如通过驱动创建连接、从连接池获取连接）封装在 `doGetConnection(Properties)` 中：
  ```java
  // 真正的连接创建方法（示例：基础实现）
  private Connection doGetConnection(Properties props) throws SQLException {
    // 1. 获取驱动类（如 com.mysql.cj.jdbc.Driver）
    Driver driver = DriverManager.getDriver(url);
    // 2. 通过驱动创建连接（传递 URL + 参数）
    return driver.connect(url, props);
  }
  ```
  这种设计的好处：
    - 复用参数封装逻辑：无论调用 `getConnection()`（无参）还是 `getConnection(user, pwd)`（带参），最终都转发到同一个连接创建方法；
    - 便于扩展：参数封装逻辑变更时，只需修改当前方法，无需修改连接创建逻辑。

#### 2.2 核心设计原理
##### （1）参数标准化：遵循 JDBC 规范
JDBC 规范为连接参数传递定义了统一标准：
- 参数容器：优先使用 `Properties`（键值对），而非自定义 Map；
- 核心参数名：`user`（用户名）、`password`（密码）是所有驱动都必须支持的标准 key；
- 扩展参数：驱动自定义参数（如 `useSSL`）通过相同的 `Properties` 传递，驱动内部自行解析。

这段代码严格遵循该规范，保证了跨数据库驱动的兼容性（如 MySQL、Oracle、PostgreSQL 驱动都能识别 `user`/`password` 参数）。

##### （2）参数优先级：临时参数 > 全局参数
参数优先级设计符合“就近原则”：
```
方法传入的 username/password > driverProperties 中的 user/password > 驱动默认配置
```
示例场景：
- 全局配置：`driverProperties` 中设置了 `user=default_user`；
- 方法调用：`doGetConnection("admin", "123456")`；
- 最终生效：`user=admin`、`password=123456`（覆盖全局配置）。

这种设计满足了“全局默认配置 + 临时自定义”的通用需求（如多租户系统中，不同租户使用不同数据库账号）。

##### （3）职责分离：参数封装与连接创建解耦
通过两个重载的 `doGetConnection` 方法实现职责分离：

| 方法 | 职责 |
|------|------|
| `doGetConnection(String, String)` | 仅封装参数（合并全局参数 + 覆盖用户名/密码） |
| `doGetConnection(Properties)` | 真正创建连接（驱动调用/连接池获取） |

解耦的好处：
- 可维护性：参数封装逻辑变更（如新增参数、调整优先级），不影响连接创建逻辑；
- 可复用性：所有获取连接的入口（如无参 `getConnection()`、带参 `getConnection(user, pwd)`）都可复用该参数封装逻辑；
- 可测试性：参数封装逻辑可单独测试（如验证优先级、参数合并是否正确）。

##### （4）空值兼容：鲁棒性设计
代码中所有 `if (xxx != null)` 的判断，是典型的鲁棒性设计：
- 若 `driverProperties` 为 null（未配置全局参数），则跳过合并，避免空指针；
- 若 `username/password` 为 null（未传入临时参数），则不覆盖，使用全局配置；
- 这种设计保证了方法在“部分参数缺失”的情况下仍能正常运行，符合“最小依赖”原则。

#### 2.3 典型应用场景与扩展
##### （1）基础数据源实现（如 MySQLDataSource）
MySQL 官方的 `MysqlDataSource` 中，`getConnection(user, pwd)` 方法的底层逻辑与这段代码完全一致：
```java
// MySQLDataSource 源码（简化）
public Connection getConnection(String username, String password) throws SQLException {
  Properties props = new Properties();
  props.putAll(this.getConnectionProperties()); // 对应 driverProperties
  if (username != null) {
    props.setProperty("user", username);
  }
  if (password != null) {
    props.setProperty("password", password);
  }
  return getConnection(props);
}
```

##### （2）连接池实现（如 HikariCP）
HikariCP 的 `HikariDataSource` 中，`doGetConnection` 方法同样遵循此逻辑，只是在 `doGetConnection(Properties)` 中增加了连接池的逻辑：
```java
// HikariCP 核心逻辑（简化）
private Connection doGetConnection(Properties props) throws SQLException {
  // 1. 从连接池获取空闲连接
  PoolEntry poolEntry = pool.borrowConnection(loginTimeout);
  // 2. 验证连接有效性（如 ping 数据库）
  if (!poolEntry.isValid()) {
    poolEntry = pool.replaceConnection(poolEntry);
  }
  // 3. 返回连接（包装为池化连接）
  return poolEntry.createProxyConnection();
}
```

##### （3）多租户场景扩展
基于此方法，可轻松扩展多租户支持：
```java
// 扩展：根据租户 ID 获取对应账号
private Connection doGetConnection(String tenantId) throws SQLException {
  // 1. 从租户配置中获取账号
  TenantConfig config = tenantConfigMap.get(tenantId);
  // 2. 复用原有参数封装逻辑
  return doGetConnection(config.getUsername(), config.getPassword());
}
```

#### 2.4 常见问题与优化点
##### （1）参数覆盖的“陷阱”
若 `driverProperties` 中已设置 `user=root`，而方法传入 `username=null`，则 `props` 中仍保留 `user=root`（不会被清空）。
- 问题场景：想“清空”用户名，使用驱动默认配置；
- 解决方案：若需清空，需显式调用 `props.remove("user")`：
  ```java
  if (username != null) {
    props.setProperty("user", username);
  } else {
    props.remove("user"); // 清空全局配置的 user
  }
  ```

##### （2）线程安全问题
`Properties` 是线程不安全的（继承 `Hashtable`，但 `putAll`/`setProperty` 是单线程操作），但这段方法是**私有方法**，且每次调用都创建新的 `Properties` 对象，因此不存在线程安全问题。

##### （3）参数编码问题
若密码包含特殊字符（如中文、空格），需提前编码：
```java
props.setProperty("password", URLEncoder.encode(password, "UTF-8"));
```
（不同驱动对特殊字符的处理不同，需结合驱动文档调整）

### 3. 总结
#### 核心关键点
1. **代码核心作用**：
   封装数据库连接的认证参数和驱动自定义参数，遵循 JDBC 规范合并参数优先级（临时参数 > 全局参数），并转发到真正的连接创建方法，是数据源获取连接的“参数预处理”核心逻辑。

2. **背后核心原理**：
    - 标准化：遵循 JDBC 规范，使用 `Properties` 传递参数，核心参数名（`user`/`password`）符合跨驱动标准；
    - 优先级：临时传入的用户名/密码覆盖全局配置，满足灵活配置需求；
    - 解耦：参数封装与连接创建分离，提升代码可维护性和复用性；
    - 鲁棒性：空值判断避免空指针，保证方法在部分参数缺失时仍能运行。

3. **设计价值**：
    - 兼容性：跨数据库驱动通用（MySQL/Oracle/PostgreSQL 都支持）；
    - 灵活性：支持全局默认配置 + 临时自定义参数；
    - 可扩展：便于新增参数、适配多租户等场景。

#### 应用价值
- **源码理解**：掌握该逻辑，能快速读懂主流数据源（如 HikariCP、Druid）的连接创建源码；
- **问题排查**：参数传递错误（如用户名密码不生效）时，可通过该逻辑定位优先级/覆盖问题；
- **自定义扩展**：实现自定义数据源时，可复用该参数封装逻辑，保证符合 JDBC 规范；
- **性能优化**：理解参数封装的轻量级特性（每次创建新 `Properties`），无需担心性能损耗（连接创建的性能瓶颈在数据库握手，而非参数封装）。

[目录](#目录)



## doGetConnection(Properties properties)

这段代码是 **MyBatis `UnpooledDataSource`**（非池化数据源）的核心实现逻辑。它展示了 Java 应用程序如何**手动加载数据库驱动**、**建立物理连接**以及**标准化连接配置**。

与之前那个方法不同，这里是真正的“执行者”，负责与 JDBC 驱动层直接交互。

---

### 一、代码逐行详细解释

#### 1. 核心入口：`doGetConnection(Properties properties)`
```java
private Connection doGetConnection(Properties properties) throws SQLException {
    // 1. 确保数据库驱动已加载并注册到 DriverManager
    initializeDriver();
    
    // 2. 使用 URL 和属性集请求建立连接
    //    这是 JDBC 标准 API，底层会遍历已注册的驱动，找到能处理该 URL 的驱动并创建连接
    Connection connection = DriverManager.getConnection(url, properties);
    
    // 3. 对刚建立的连接进行标准化配置（如事务隔离级别、自动提交等）
    configureConnection(connection);
    
    // 4. 返回配置好的连接对象
    return connection;
}
```
*   **作用**：串联整个连接流程：准备驱动 -> 获取连接 -> 调优连接 -> 返回。

#### 2. 驱动初始化：`initializeDriver()`
这是最复杂的部分，涉及类加载和驱动注册机制。
```java
private void initializeDriver() throws SQLException {
    try {
      // 1. 使用 ConcurrentHashMap 的 computeIfAbsent 保证驱动只加载一次（线程安全且懒加载）
      //    registeredDrivers 是一个静态或实例的 Map，用于缓存已加载的驱动
      registeredDrivers.computeIfAbsent(driver, x -> {
        Class<?> driverType;
        try {
          // 2. 动态加载驱动类
          if (driverClassLoader != null) {
            // 如果指定了类加载器（常用于 OSGi 或复杂容器环境），使用指定的加载器
            driverType = Class.forName(x, true, driverClassLoader);
          } else {
            // 否则使用 MyBatis 的工具类 Resources 加载（通常委托给当前线程上下文类加载器）
            driverType = Resources.classForName(x);
          }
          
          // 3. 实例化驱动对象
          //    通过反射调用无参构造函数创建 Driver 实例
          Driver driverInstance = (Driver) driverType.getDeclaredConstructor().newInstance();
          
          // 4. 注册驱动到 DriverManager
          //    关键：这里没有直接注册 driverInstance，而是包装成了 DriverProxy
          DriverManager.registerDriver(new DriverProxy(driverInstance));
          
          return driverInstance;
        } catch (Exception e) {
          throw new RuntimeException("Error setting driver on UnpooledDataSource.", e);
        }
      });
    } catch (RuntimeException re) {
      // 将运行时异常转换为 SQL 异常，符合 JDBC 规范
      throw new SQLException("Error setting driver on UnpooledDataSource.", re.getCause());
    }
}
```

#### 3. 连接配置：`configureConnection(Connection conn)`
```java
private void configureConnection(Connection conn) throws SQLException {
    // 1. 设置网络超时时间 (JDBC 4.1+)
    if (defaultNetworkTimeout != null) {
      // 注意：setNetworkTimeout 需要一个 Executor，这里临时创建了一个单线程池
      conn.setNetworkTimeout(Executors.newSingleThreadExecutor(), defaultNetworkTimeout);
    }
    
    // 2. 设置自动提交模式 (AutoCommit)
    //    只有当配置值不为 null 且与当前连接状态不一致时才修改，减少不必要的调用
    if (autoCommit != null && autoCommit != conn.getAutoCommit()) {
      conn.setAutoCommit(autoCommit);
    }
    
    // 3. 设置事务隔离级别 (Transaction Isolation Level)
    if (defaultTransactionIsolationLevel != null) {
      conn.setTransactionIsolation(defaultTransactionIsolationLevel);
    }
}
```

---

### 二、背后原理与深度解析

#### 1. 为什么需要手动 `initializeDriver`？（类加载机制）
在早期的 JDBC 规范中，开发者必须写 `Class.forName("com.mysql.cj.jdbc.Driver")` 来触发驱动的静态代码块，从而完成自我注册。
*   **现代 JDBC (Java 6+)**: 引入了 `ServiceLoader` 机制，驱动 jar 包中包含 `META-INF/services/java.sql.Driver` 文件，JVM 会自动加载并注册驱动。
*   **MyBatis 为什么要手动做？**
    *   **兼容性**: 为了支持旧版驱动或某些特殊环境（如 OSGi、特定的应用服务器），自动加载可能失效。
    *   **可控性**: MyBatis 希望明确知道哪个驱动被加载了，并且可以自定义类加载器 (`driverClassLoader`)。这在模块化开发或热部署场景中至关重要。
    *   **懒加载**: 使用 `computeIfAbsent` 确保驱动只在第一次请求连接时加载，避免启动时的不必要开销。

#### 2. `DriverProxy` 的作用是什么？（内存泄漏防护）
代码中有一行非常特殊的代码：
```java
DriverManager.registerDriver(new DriverProxy(driverInstance));
```
为什么不直接 `registerDriver(driverInstance)`？
*   **原理**: `DriverManager` 内部维护了一个静态的 `List<DriverInfo>`。如果直接注册驱动实例，`DriverManager`（属于 JDK  bootstrap classloader）会持有该驱动对象的强引用。
*   **问题**: 如果驱动类是由应用服务器的自定义 ClassLoader 加载的（例如 Tomcat 部署的 war 包中的驱动），当应用卸载（Undeploy）时，由于 `DriverManager` 仍然持有驱动实例的引用，导致驱动类的 ClassLoader 无法被垃圾回收（GC）。这就是经典的 **JDBC 驱动内存泄漏**。
*   **解决方案**: MyBatis 使用 `DriverProxy` 包装真实的驱动。`DriverProxy` 通常是一个轻量级代理，或者在反注册时更容易清理。更重要的是，这是一种防御性编程，旨在解耦 JDK 静态管理类与应用动态加载类之间的强依赖关系（尽管在现代 JDK 和应用服务器中，这个问题已有改善，但框架仍保留此习惯以确保安全）。

#### 3. `computeIfAbsent` 的并发安全性
*   `registeredDrivers` 通常是一个 `ConcurrentHashMap`。
*   `computeIfAbsent` 是原子操作。在高并发场景下，如果有 100 个线程同时第一次请求连接，该 lambda 表达式只会执行一次，其他线程会等待或使用已计算出的结果。这避免了重复加载驱动类和重复注册驱动的开销及潜在错误。

#### 4. 连接的“标准化” (`configureConnection`)
数据库驱动创建的默认连接状态取决于驱动实现的默认值（例如 MySQL 默认 `autoCommit=true`，Oracle 可能不同）。
*   **目的**: 应用程序通常希望屏蔽底层数据库的差异，统一行为。
*   **原理**:
    *   **事务控制**: 强制设置 `autoCommit` 和 `isolation level` 是为了确保业务逻辑在预期的事务边界内运行。如果不设置，可能导致脏读或隐式提交。
    *   **网络超时**: `setNetworkTimeout` 是 JDBC 4.1 引入的特性，用于防止因网络故障导致查询无限期挂起，占用连接资源。
    *   **性能优化**: 代码中 `if (val != null && val != current)` 的判断非常重要。频繁调用 `setAutoCommit` 或 `setTransactionIsolation` 可能会触发与数据库的额外网络交互（Round-trip）。只有在必要时才修改，能提升性能。

#### 5. `Executors.newSingleThreadExecutor()` 的隐患（代码审查视角）
在 `configureConnection` 中：
```java
conn.setNetworkTimeout(Executors.newSingleThreadExecutor(), defaultNetworkTimeout);
```
*   **原理**: `setNetworkTimeout` 需要一个 `Executor` 来处理超时回调。
*   **潜在问题**: 这里每次调用都 `new` 了一个线程池（包含一个线程）。
    *   如果这个方法被频繁调用（每次获取连接都调用），会创建大量线程池对象。虽然线程池内部线程可能复用，但 `ExecutorService` 对象本身如果没有被 `shutdown`，在某些实现中可能会导致资源累积。
    *   **最佳实践**: 通常应该将这个 `Executor` 作为数据源的成员变量复用，而不是在每次配置连接时新建。这可能是 MyBatis 早期版本的一个实现细节或简化处理，但在高并发生产环境中值得优化。

### 总结

这段代码是 **JDBC 规范的高级封装**，体现了以下核心思想：

1.  **生命周期管理**: 通过懒加载和缓存 (`computeIfAbsent`) 管理驱动的生命周期。
2.  **类加载隔离**: 支持自定义 `ClassLoader`，适应复杂部署环境。
3.  **资源安全**: 通过 `DriverProxy` 尝试避免 ClassLoader 泄漏。
4.  **配置一致性**: 在返回连接给用户前，强制抹平不同数据库驱动的默认行为差异（事务、超时等），提供统一的开发体验。
5.  **防御性编程**: 对空值、异常类型转换做了细致处理，确保符合 `SQLException` 规范。

它是 MyBatis 能够“一次配置，到处运行”且稳定管理数据库连接的基石之一。

[目录](#目录)



