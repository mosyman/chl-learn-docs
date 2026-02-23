

你想理解这段 Nacos 数据源核心适配类 `DynamicDataSource` 的代码逻辑，我会从**逐行解析、核心设计模式、数据源切换逻辑、实际应用场景**四个维度，把这段代码的作用、设计思路和底层原理讲透。

### 先总结核心作用
这段代码是 Nacos 的**动态数据源适配核心类**，核心功能是：
根据 Nacos 的运行模式（单机/集群），自动切换数据源实现——单机模式默认用「嵌入式本地存储（Derby）」，集群模式默认用「外部数据库（如 MySQL）」，且通过单例模式保证全局数据源实例唯一。

---

## 一、逐行代码解析
```java
/* 版权声明：Apache 2.0 协议，无需关注，重点看业务逻辑 */
package com.alibaba.nacos.persistence.datasource;

import com.alibaba.nacos.persistence.configuration.DatasourceConfiguration;

/**
 * Datasource adapter.  // 数据源适配器（核心注释：适配不同数据源）
 * @author Nacos
 */
public class DynamicDataSource {
    
    // 1. 本地数据源服务（对应嵌入式存储，如 Derby）
    private DataSourceService localDataSourceService = null;
    
    // 2. 基础数据源服务（对应外部数据库，如 MySQL）
    private DataSourceService basicDataSourceService = null;
    
    // 3. 单例实例（饿汉式单例）
    private static final DynamicDataSource INSTANCE = new DynamicDataSource();

    // 4. 私有构造方法（禁止外部实例化，保证单例）
    private DynamicDataSource() {}

    // 5. 获取单例实例的静态方法
    public static DynamicDataSource getInstance() {
        return INSTANCE;
    }
    
    // 6. 核心方法：获取数据源服务（同步方法，保证线程安全）
    public synchronized DataSourceService getDataSource() {
        try {
            // 核心逻辑：判断是否使用嵌入式存储
            // 注释：单机模式默认用嵌入式存储，集群模式默认用外部数据库
            if (DatasourceConfiguration.isEmbeddedStorage()) {
                // 6.1 嵌入式存储逻辑：懒加载 LocalDataSourceServiceImpl
                if (localDataSourceService == null) {
                    localDataSourceService = new LocalDataSourceServiceImpl();
                    localDataSourceService.init(); // 初始化嵌入式数据源（如 Derby）
                }
                return localDataSourceService;
            } else {
                // 6.2 外部数据库逻辑：懒加载 ExternalDataSourceServiceImpl
                if (basicDataSourceService == null) {
                    basicDataSourceService = new ExternalDataSourceServiceImpl();
                    basicDataSourceService.init(); // 初始化外部数据源（如 MySQL 连接池）
                }
                return basicDataSourceService;
            }
        } catch (Exception e) {
            // 初始化失败抛出运行时异常，终止 Nacos 启动
            throw new RuntimeException(e);
        }
    }
    
}
```

### 关键代码拆解
#### 1. 成员变量：两种数据源服务
| 变量名 | 类型 | 作用 | 对应存储 |
|--------|------|------|----------|
| `localDataSourceService` | `DataSourceService` | 本地数据源服务实例 | 嵌入式 Derby 数据库（单机模式） |
| `basicDataSourceService` | `DataSourceService` | 外部数据源服务实例 | MySQL/Oracle 等外部数据库（集群模式） |

`DataSourceService` 是 Nacos 定义的数据源服务接口，核心方法包括 `init()`（初始化）、`getConnection()`（获取连接）等，`LocalDataSourceServiceImpl` 和 `ExternalDataSourceServiceImpl` 是它的两个实现类：
- `LocalDataSourceServiceImpl`：适配嵌入式 Derby，无需配置，开箱即用；
- `ExternalDataSourceServiceImpl`：适配外部数据库，读取 `application.properties` 中的 `db.url`/`db.user` 等配置，初始化数据库连接池。

#### 2. 单例模式：饿汉式实现
```java
private static final DynamicDataSource INSTANCE = new DynamicDataSource();
private DynamicDataSource() {}
public static DynamicDataSource getInstance() { return INSTANCE; }
```
- **设计模式**：饿汉式单例（类加载时就创建实例），保证全局只有一个 `DynamicDataSource` 实例；
- **原因**：数据源是核心资源，全局唯一可避免重复初始化连接池、节省资源；
- **线程安全**：类加载由 JVM 保证线程安全，无需加锁。

#### 3. 核心方法：`getDataSource()`
```java
public synchronized DataSourceService getDataSource() { ... }
```
- `synchronized`：保证多线程环境下，数据源实例只会被初始化一次（避免重复 `new` 和 `init()`）；
- 核心判断：`DatasourceConfiguration.isEmbeddedStorage()`  
  这个方法的底层逻辑是：
  ```java
  // DatasourceConfiguration 核心逻辑（简化版）
  public static boolean isEmbeddedStorage() {
      // 1. 判断是否是单机模式（nacos.mode=standalone）
      boolean isStandalone = "standalone".equals(System.getProperty("nacos.mode"));
      // 2. 单机模式默认用嵌入式存储；集群模式若未配置外部数据库，也会降级为嵌入式（仅测试用）
      return isStandalone && !hasExternalDatasourceConfig();
  }
  
  // 判断是否配置了外部数据库（如 MySQL）
  private static boolean hasExternalDatasourceConfig() {
      return StringUtils.isNotBlank(System.getProperty("db.url.0"));
  }
  ```
  简单说：
    - 单机模式 + 未配置 MySQL → 用 Derby（`LocalDataSourceServiceImpl`）；
    - 集群模式 或 单机模式配置了 MySQL → 用外部数据库（`ExternalDataSourceServiceImpl`）。

#### 4. 懒加载初始化
```java
if (localDataSourceService == null) {
    localDataSourceService = new LocalDataSourceServiceImpl();
    localDataSourceService.init();
}
```
- **懒加载**：第一次调用 `getDataSource()` 时才初始化数据源，而非类加载时初始化；
- **`init()` 方法作用**：
    - `LocalDataSourceServiceImpl.init()`：初始化 Derby 嵌入式数据库，创建默认表结构；
    - `ExternalDataSourceServiceImpl.init()`：读取 MySQL 配置，初始化 Druid/HikariCP 连接池，验证数据库连接。

---

## 二、核心设计思路（为什么要这么写？）
### 1. 适配不同部署模式
- **单机模式**：优先用 Derby（无需配置，开箱即用），降低开发/测试门槛；
- **集群模式**：强制用外部数据库（MySQL），保证集群数据一致性（Derby 不支持集群共享）。

### 2. 单例 + 懒加载 + 线程安全
- **单例**：全局唯一数据源实例，避免重复创建连接池（连接池创建成本高）；
- **懒加载**：启动时不初始化数据源，第一次使用时才初始化，加快 Nacos 启动速度；
- **synchronized**：多线程场景下（如 Nacos 启动时多个模块同时获取数据源），保证数据源只初始化一次。

### 3. 接口隔离（面向接口编程）
通过 `DataSourceService` 接口封装不同数据源的实现，上层业务代码只需调用 `getDataSource()` 获取接口实例，无需关心底层是 Derby 还是 MySQL，实现「业务与数据源解耦」。

---

## 三、实际应用场景（代码怎么被调用？）
Nacos 中所有数据库操作（如配置存储、服务注册信息存储）都会通过这个类获取数据源，比如：
```java
// 配置模块读取数据的示例（简化版）
public class ConfigDao {
    public ConfigInfo getConfig(String dataId) {
        // 1. 获取全局数据源实例
        DataSourceService dataSourceService = DynamicDataSource.getInstance().getDataSource();
        // 2. 获取数据库连接（底层自动适配 Derby/MySQL）
        Connection conn = dataSourceService.getConnection();
        // 3. 执行 SQL 查询（上层无需关心数据库类型）
        PreparedStatement pstmt = conn.prepareStatement("SELECT content FROM config_info WHERE data_id = ?");
        pstmt.setString(1, dataId);
        ResultSet rs = pstmt.executeQuery();
        // 4. 处理结果...
    }
}
```

### 关键场景：数据源切换
1. **默认单机模式**：
   启动 Nacos 单机版（`-m standalone`），未配置 MySQL → `isEmbeddedStorage()=true` → 使用 Derby；
2. **单机模式切换为 MySQL**：
   在 `application.properties` 中配置 `db.url.0`/`db.user`/`db.password` → `hasExternalDatasourceConfig()=true` → `isEmbeddedStorage()=false` → 使用 MySQL；
3. **集群模式**：
   启动 Nacos 集群版 → `isEmbeddedStorage()=false` → 必须配置 MySQL，否则初始化失败（抛出 `RuntimeException`）。

---

## 四、避坑点 & 核心补充
### 1. 集群模式必须配置外部数据库
如果集群模式未配置 MySQL，`ExternalDataSourceServiceImpl.init()` 会失败，抛出 `RuntimeException`，导致 Nacos 启动失败——这是因为 Derby 不支持集群数据共享，集群模式必须用 MySQL 保证数据一致性。

### 2. 数据源实例只初始化一次
`localDataSourceService`/`basicDataSourceService` 一旦初始化，后续调用 `getDataSource()` 会直接返回已有实例，不会重复 `init()`；若需切换数据源（如运行时修改配置），需重启 Nacos。

### 3. 异常处理逻辑
初始化数据源时若出错（如 MySQL 连接失败），会抛出 `RuntimeException`，直接终止 Nacos 启动——数据源是核心组件，初始化失败无法提供服务。

---

## 总结（核心关键点）
1. **核心功能**：单例模式 + 动态数据源适配，根据运行模式（单机/集群）自动切换 Derby/MySQL；
2. **核心判断**：`DatasourceConfiguration.isEmbeddedStorage()` 决定数据源类型，单机默认 Derby，集群/配置 MySQL 则用外部数据库；
3. **设计模式**：饿汉式单例（全局唯一）+ 懒加载（首次使用初始化）+ 同步方法（线程安全）；
4. **核心解耦**：通过 `DataSourceService` 接口隔离底层数据源实现，上层业务无需关心数据库类型；
5. **生产要求**：集群模式必须配置 MySQL，单机模式推荐配置 MySQL（Derby 仅用于测试）。

如果需要，我可以帮你补充 `LocalDataSourceServiceImpl` 和 `ExternalDataSourceServiceImpl` 的核心初始化代码，以及 Nacos 切换 MySQL 数据源的完整配置步骤，需要吗？















