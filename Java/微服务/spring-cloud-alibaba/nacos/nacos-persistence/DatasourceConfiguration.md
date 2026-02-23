

你想理解这段 Nacos 数据源核心配置类 `DatasourceConfiguration` 的代码逻辑，我会从**类定位、核心变量、关键方法、执行流程、设计意图**五个维度，把这段代码的作用、数据源模式判断规则、以及和之前 `DynamicDataSource` 的关联讲透。

### 先总结核心作用
这个类是 Nacos 启动时的**数据源模式初始化器**（实现了 Spring 的 `ApplicationContextInitializer`），核心功能是：
在 Spring 容器初始化阶段，根据「运行模式（单机/集群）」「数据源配置（如 MySQL）」自动判定并设置 Nacos 的数据源模式——是使用「嵌入式存储（Derby）」还是「外部数据库（MySQL）」，为后续 `DynamicDataSource` 提供判断依据。

---

## 一、类基础定位
```java
public class DatasourceConfiguration implements ApplicationContextInitializer<ConfigurableApplicationContext> {
    @Override
    public void initialize(final ConfigurableApplicationContext applicationContext) {
        loadDatasourceConfiguration();
    }
}
```
- **核心接口**：`ApplicationContextInitializer` 是 Spring 提供的扩展接口，作用是**在 Spring 容器初始化（refresh 之前）执行自定义逻辑**；
- **执行时机**：Nacos 启动时，Spring 容器创建后、Bean 初始化前，会调用这个类的 `initialize` 方法，提前完成数据源模式的判定；
- **核心目的**：在所有业务代码（如 `DynamicDataSource`）执行前，先确定数据源模式，避免运行时模式冲突。

---

## 二、核心变量解析
类中定义了两个静态全局变量，是整个 Nacos 数据源模式的「核心开关」：

| 变量名 | 类型 | 初始值 | 核心作用 |
|--------|------|--------|----------|
| `useExternalDb` | boolean | false | 是否使用外部数据库（MySQL/Oracle 等） |
| `embeddedStorage` | boolean | `EnvUtil.getStandaloneMode()` | 是否使用嵌入式存储（Derby）<br/>初始值 = 单机模式（`nacos.standalone=true`）为 true，集群模式为 false |

### 配套的 getter/setter
```java
// 供外部类（如 DynamicDataSource）读取模式
public static boolean isUseExternalDb() { return useExternalDb; }
public static boolean isEmbeddedStorage() { return embeddedStorage; }

// 内部修改模式
public static void setUseExternalDb(boolean useExternalDb) { ... }
public static void setEmbeddedStorage(boolean embeddedStorage) { ... }
```
⚠️ 关键：`DynamicDataSource` 中调用的 `DatasourceConfiguration.isEmbeddedStorage()`，就是读取这个 `embeddedStorage` 变量的值。

---

## 三、核心方法：`loadDatasourceConfiguration()`
这是整个类的核心逻辑，负责「根据配置自动判定数据源模式」，逐行拆解：
```java
private void loadDatasourceConfiguration() {
    // 步骤1：判断是否配置了外部数据源（核心依据：数据源平台是否为非 Derby/非空）
    String platform = DatasourcePlatformUtil.getDatasourcePlatform("");
    boolean useExternalStorage =
            !PersistenceConstant.EMPTY_DATASOURCE_PLATFORM.equalsIgnoreCase(platform) 
            && !PersistenceConstant.DERBY.equalsIgnoreCase(platform);
    setUseExternalDb(useExternalStorage);
    
    // 步骤2：根据外部数据源配置，修正嵌入式存储模式（核心逻辑）
    if (isUseExternalDb()) {
        // 若配置了外部数据源 → 强制关闭嵌入式存储
        setEmbeddedStorage(false);
    } else {
        // 若未配置外部数据源 → 重新计算嵌入式存储模式
        boolean embeddedStorage = isEmbeddedStorage() 
                || Boolean.getBoolean(PersistenceConstant.EMBEDDED_STORAGE);
        setEmbeddedStorage(embeddedStorage);
        
        // 兜底逻辑：若最终嵌入式存储也关闭 → 自动升级为外部数据源（兼容旧版本）
        if (!embeddedStorage) {
            setUseExternalDb(true);
        }
    }
}
```

### 步骤1：判断是否配置外部数据源
#### 关键代码：`DatasourcePlatformUtil.getDatasourcePlatform("")`
这个工具方法的核心逻辑是**读取 Nacos 配置中的数据源类型**，优先级：
1. 读取 `application.properties` 中的 `spring.sql.init.platform`（如 `mysql`）；
2. 若未配置，读取 `db.platform`（旧版配置，兼容用）；
3. 若都未配置，返回空字符串/`derby`。

#### 常量说明：
- `PersistenceConstant.EMPTY_DATASOURCE_PLATFORM`：空字符串（""），表示未配置数据源平台；
- `PersistenceConstant.DERBY`：字符串 "derby"，表示嵌入式存储。

#### 结论：
`useExternalStorage = true` 的条件：
→ 配置了 `spring.sql.init.platform=mysql`（或其他非 derby 平台），且不是空值。

### 步骤2：修正嵌入式存储模式（核心规则）
这一步是「兜底+兼容」，确保数据源模式唯一且不冲突，分两种情况：
#### 情况1：配置了外部数据源（`useExternalDb=true`）
→ 强制关闭嵌入式存储（`embeddedStorage=false`），避免同时启用两种存储。

#### 情况2：未配置外部数据源（`useExternalDb=false`）
→ 重新计算嵌入式存储模式：
```java
boolean embeddedStorage = isEmbeddedStorage() || Boolean.getBoolean(PersistenceConstant.EMBEDDED_STORAGE);
```
- `isEmbeddedStorage()`：初始值是「单机模式=true，集群模式=false」；
- `Boolean.getBoolean(PersistenceConstant.EMBEDDED_STORAGE)`：读取系统属性 `nacos.embedded.storage`（手动强制开启嵌入式存储，优先级更高）。

→ 再兜底：若最终 `embeddedStorage=false`（比如集群模式+未配置外部数据源），则强制开启外部数据源（`useExternalDb=true`），避免无数据源可用（兼容旧版逻辑）。

---

## 四、完整执行流程（结合 Nacos 启动）
```mermaid
graph TD
    A[Nacos 启动] --> B[Spring 容器初始化]
    B --> C[调用 DatasourceConfiguration.initialize()]
    C --> D[执行 loadDatasourceConfiguration()]
    D --> D1[读取数据源平台配置（spring.sql.init.platform）]
    D1 --> D2[判断是否配置外部数据源（useExternalDb）]
    D2 -->|是（如 mysql）| E[设置 embeddedStorage=false]
    D2 -->|否| F[计算嵌入式存储模式（单机模式/手动开启）]
    F --> F1[embeddedStorage=true → 最终用 Derby]
    F --> F2[embeddedStorage=false → 强制 useExternalDb=true]
    E & F1 & F2 --> G[Spring 容器继续初始化]
    G --> H[DynamicDataSource 读取 embeddedStorage/useExternalDb 变量]
    H --> I[返回对应数据源实例（Derby/MySQL）]
```

### 核心规则总结（最终数据源模式判定）
| 场景 | useExternalDb | embeddedStorage | 最终存储类型 |
|------|---------------|-----------------|--------------|
| 单机模式 + 未配置 MySQL | false | true | Derby |
| 单机模式 + 配置 MySQL | true | false | MySQL |
| 集群模式 + 配置 MySQL | true | false | MySQL |
| 集群模式 + 未配置 MySQL | true | false | 强制要求 MySQL（初始化失败） |
| 任意模式 + 手动指定 `nacos.embedded.storage=true` | false | true | Derby |

---

## 五、关键补充（和 DynamicDataSource 的关联）
1. **执行顺序**：
   `DatasourceConfiguration` 先执行（Spring 容器初始化前）→ 确定 `embeddedStorage` 变量 → `DynamicDataSource.getDataSource()` 读取该变量 → 初始化对应数据源。

2. **核心依赖**：
   `DynamicDataSource` 中的 `DatasourceConfiguration.isEmbeddedStorage()`，就是读取这个类中设置的 `embeddedStorage` 变量，二者是「配置判定」和「实例化」的关系：
    - `DatasourceConfiguration`：负责「判定模式」；
    - `DynamicDataSource`：负责「根据模式创建数据源实例」。

3. **兼容旧版逻辑**：
   代码中 `if (!embeddedStorage) { setUseExternalDb(true); }` 是为了兼容 Nacos 1.x 版本的逻辑——旧版集群模式下即使未配置 MySQL，也会强制尝试连接外部数据库（而非降级到 Derby），避免集群数据不一致。

---

## 六、设计意图（为什么这么写？）
1. **提前初始化**：
   利用 Spring 的 `ApplicationContextInitializer` 在容器初始化前判定数据源模式，避免后续业务代码初始化时出现模式冲突（比如 Bean 依赖数据源，但模式未确定）。

2. **模式自动适配**：
   无需手动指定「用 Derby 还是 MySQL」，根据「运行模式+配置」自动判定，降低用户使用门槛（单机开箱即用，集群强制 MySQL）。

3. **兼容与兜底**：
    - 兼容旧版配置（如 `db.platform`）；
    - 兜底逻辑避免「无数据源可用」的情况，集群模式下未配置 MySQL 会强制开启 `useExternalDb=true`，最终初始化 MySQL 失败（抛出异常），而非静默使用 Derby（避免集群数据不一致）。

---

## 总结（核心关键点）
1. **类定位**：Spring 容器初始化器，在 Nacos 启动初期判定数据源模式；
2. **核心变量**：`useExternalDb`（是否用外部数据库）、`embeddedStorage`（是否用 Derby），是全局数据源模式开关；
3. **核心规则**：
    - 配置 MySQL → 强制用外部数据库，关闭 Derby；
    - 未配置 MySQL + 单机模式 → 用 Derby；
    - 未配置 MySQL + 集群模式 → 强制要求 MySQL（初始化失败）；
4. **和 DynamicDataSource 的关系**：前者「判定模式」，后者「创建实例」，共同实现数据源动态适配；
5. **核心目的**：自动适配不同部署场景，保证集群数据一致性，降低用户配置成本。

如果需要，我可以帮你整理一份「Nacos 数据源模式判定的完整测试用例」，包含单机/集群、配置/未配置 MySQL 等场景的实际运行结果，需要吗？













