
## 目录
- [1](#1)
- [2](#2)


# 1

`MapperRegistry` 是 MyBatis 框架中** Mapper 接口管理的核心注册中心**。它的主要职责是`维护 Mapper 接口与具体实现`之间的`映射关系`，并在`运行时动态`生成 Mapper 接口的`代理对象`。

它是 MyBatis 实现“接口编程”（即只需定义接口，无需编写实现类）的关键枢纽。

---

### 一、核心功能详解

#### 1. 角色定位
*   **注册表**：存储所有已知的 Mapper 接口（`knownMappers`）。
*   **工厂**：负责根据 Mapper 接口类型，创建对应的代理对象（`MapperProxy`）。
*   **解析器触发器**：在注册 Mapper 接口时，触发对该接口注解或 XML 配置的解析。

#### 2. 关键成员变量
```java
private final Configuration config;
// 核心缓存：Key 是 Mapper 接口 Class 对象，Value 是对应的工厂类
private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new ConcurrentHashMap<>();
```
*   **`ConcurrentHashMap`**：保证了在高并发环境下（多个线程同时获取 Mapper）的线程安全性。
*   **`MapperProxyFactory`**：这是一个工厂类，专门用于创建 `MapperProxy` 实例。注意，注册表中存的是**工厂**，而不是直接存代理对象。代理对象是每次请求时动态创建的（或者在某些优化场景下缓存，但 MyBatis 默认是每次 `getMapper` 都通过工厂生成新的 Proxy，而 Proxy 内部持有 SqlSession）。

#### 3. 核心方法解析

| 方法 | 作用 | 关键点 |
| :--- | :--- | :--- |
| **`addMapper(Class<T> type)`** | 注册一个 Mapper 接口。 | 1. 检查是否为接口。2. 放入工厂到 Map。3. **立即解析**该接口的 SQL 配置（注解或 XML）。 |
| **`getMapper(Class<T>, SqlSession)`** | 获取 Mapper 代理对象。 | 1. 从 Map 取出工厂。2. 调用工厂的 `newInstance` 创建代理。3. 返回 JDK 动态代理对象。 |
| **`addMappers(String packageName)`** | 批量扫描包路径注册。 | 利用 `ResolverUtil` 扫描指定包下的所有类，过滤出接口的子类型，逐个调用 `addMapper`。 |

---

### 二、底层原理深度剖析

#### 1. 为什么需要 `MapperProxyFactory`？（工厂模式 + 延迟代理）
`MapperRegistry` 并不直接存储 `MapperProxy` 实例，而是存储 `MapperProxyFactory`。
*   **原理**：
    *   `MapperProxy` 是无状态的（或者说状态依赖于传入的 `SqlSession`）。
    *   每次调用 `sqlSession.getMapper(UserMapper.class)` 时，理论上应该返回一个新的代理对象，该代理对象绑定当前的 `SqlSession`。
    *   如果直接缓存 `MapperProxy`，会导致代理对象与特定的 `SqlSession` 绑定，无法复用。
    *   **工厂模式**解耦了“注册”和“实例化”。注册时只记录“谁能生产”，获取时才真正“生产”并注入当前的 `SqlSession`。

#### 2. `addMapper` 中的“先注册后解析”策略
这是 `addMapper` 方法中最精妙的设计细节：
```java
// 1. 先放入工厂
knownMappers.put(type, new MapperProxyFactory<>(type));

// 2. 再解析配置
MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
parser.parse();
```
**为什么要这个顺序？**
*   **防止循环依赖/重复解析**：在解析过程中（`parser.parse()`），MyBatis 可能会处理 `<mapper>` 标签或者扫描关联的其他 Mapper。如果解析逻辑中再次尝试注册同一个接口，`hasMapper(type)` 会返回 `true`，从而跳过重复注册。
*   **原子性与回滚**：代码使用了 `try-finally` 块。如果 `parser.parse()` 抛出异常（例如 SQL 语法错误），`loadCompleted` 为 `false`，`finally` 块会将刚才放入的工厂移除。
    *   **意义**：保证注册表的**一致性**。要么成功注册且配置正确，要么完全不注册。避免注册了一个“半残”的 Mapper 导致后续运行时报错难以排查。

#### 3. `getMapper` 的动态代理生成流程
当用户调用 `sqlSession.getMapper(UserMapper.class)` 时：
1.  **查找工厂**：`MapperRegistry` 从 `knownMappers` 中获取 `UserMapper` 对应的 `MapperProxyFactory`。
2.  **创建代理**：调用 `factory.newInstance(sqlSession)`。
3.  **JDK 动态代理**：在 `MapperProxyFactory` 内部，执行：
    ```java
    // 伪代码
    return (T) Proxy.newProxyInstance(
        mapperInterface.getClassLoader(),
        new Class[] { mapperInterface },
        new MapperProxy<>(sqlSession, mapperInterface, methodCache)
    );
    ```
4.  **结果**：返回一个实现了 `UserMapper` 接口的代理对象。当调用 `userMapper.selectById(1)` 时，请求会被拦截到 `MapperProxy.invoke()` 方法，进而转换为 SQL 执行。

#### 4. 包扫描原理 (`ResolverUtil`)
`addMappers(String packageName)` 方法展示了 MyBatis 如何自动发现 Mapper：
*   **工具类**：`ResolverUtil` 是 MyBatis 内置的一个轻量级类扫描器。
*   **过程**：
    1.  获取当前线程的 `ClassLoader`。
    2.  将包名转换为资源路径（如 `com.example.mapper` -> `com/example/mapper`）。
    3.  调用 `classLoader.getResources()` 获取所有匹配的 `.class` 文件 URL。
    4.  遍历文件，加载类。
    5.  **过滤**：使用 `ResolverUtil.IsA(superType)` 测试类是否是指定类型的子类（或接口实现）。对于 Mapper，通常检查是否是接口。
*   **底层限制**：这种扫描方式依赖于类路径（Classpath）。在模块化系统（JPMS）或某些特殊的容器环境中，可能需要额外的配置才能扫描到。

#### 5. 线程安全设计
*   **注册阶段**：通常在 MyBatis 启动初始化阶段（单线程）完成，但 `ConcurrentHashMap` 保证了即使在多线程初始化下也不会崩溃。
*   **运行阶段**：`getMapper` 是高频调用。
    *   `knownMappers.get(type)` 是读操作，`ConcurrentHashMap` 提供高效的无锁读取（Java 8+）。
    *   `MapperProxyFactory.newInstance` 是无状态的，多线程调用安全。
    *   生成的 `MapperProxy` 对象虽然包含 `methodCache`（缓存 Method 对象），但该缓存也是线程安全的（通常使用 `ConcurrentHashMap` 或不可变集合）。

---

### 三、常见面试题与陷阱

#### Q1: Mapper 接口需要在 XML 中声明吗？
*   **A**: 不需要在 XML 中显式声明 `<mapper class="..."/>` 如果使用 `addMappers` 扫描包。但如果使用 `<mappers><mapper class="..."/></mappers>` 配置，底层最终也是调用 `config.addMapper()`，进而调用 `MapperRegistry.addMapper()`。

#### Q2: 为什么 `getMapper` 每次返回的对象可能不同？
*   **A**: 因为 `getMapper` 每次都会调用 `factory.newInstance(sqlSession)` 创建一个新的 `MapperProxy` 实例。这个新实例绑定了当前传入的 `SqlSession`。
    *   **注意**：虽然 Proxy 对象不同，但它们指向的**接口定义**和**SQL 映射元数据**（存在 `Configuration` 中）是共享的。

#### Q3: 如果 Mapper 接口有语法错误，会发生什么？
*   **A**: 在 `addMapper` 阶段，`parser.parse()` 会抛出 `BindingException` 或其他运行时异常。由于 `try-finally` 的存在，该 Mapper 不会被注册到 `knownMappers` 中。应用启动时会报错并停止（取决于 Spring 或容器的配置），不会留下隐患。

#### Q4: `MapperRegistry` 和 `Configuration` 的关系？
*   **A**: `MapperRegistry` 是 `Configuration` 的一个组件（持有 `config` 引用）。
    *   `Configuration` 是全局上下文，保存所有环境信息（数据源、类型别名、插件等）。
    *   `MapperRegistry` 专注于管理 Mapper 接口的元数据和工厂。
    *   解析 Mapper 时需要的 `Config` 信息（如数据库方言、类型处理器）通过 `MapperRegistry` 透传给 `MapperAnnotationBuilder`。

### 四、总结

`MapperRegistry` 是 MyBatis 魔法的起点之一。它通过：
1.  **注册表模式**管理接口元数据。
2.  **工厂模式**解耦接口定义与代理实例创建。
3.  **动态代理**实现接口方法的拦截与 SQL 执行。
4.  **严谨的状态控制**（先注册后解析、异常回滚）确保系统稳定性。

它将静态的 Java 接口变成了动态的、可执行的数据库操作对象，是 MyBatis“简洁 API”背后的坚实基石。




[目录](#目录)


# 2

你希望深入理解 MyBatis 核心组件 `MapperRegistry` 的实现细节、底层原理，以及它在 MyBatis 框架中扮演的核心角色。我会从类的定位、核心成员、核心方法拆解、底层设计原理、与其他组件的关联五个维度，全面且细致地讲解。

### 一、`MapperRegistry` 整体定位
`MapperRegistry` 是 MyBatis 的**Mapper 接口注册中心/工厂**，核心使命是：
- 管理所有 Mapper 接口的注册、存储和实例化；
- 作为 MyBatis 中“接口式编程”的核心枢纽——将 Mapper 接口（如 `UserMapper`）与底层的 SQL 执行逻辑绑定；
- 生成 Mapper 接口的动态代理对象，让开发者可以直接调用接口方法执行 SQL，无需编写实现类。

简单来说：`MapperRegistry` 是 MyBatis 中 Mapper 接口的“总管家”，负责 Mapper 接口的“注册（add）- 查询（get）- 校验（has）”全生命周期管理，是动态代理模式在 MyBatis 中的核心体现。

---

### 二、核心成员变量解析
先理解类的核心成员，才能看懂后续方法逻辑：

| 成员变量 | 类型 | 核心作用 |
|----------|------|----------|
| `config` | `Configuration` | MyBatis 全局配置对象，包含所有核心配置（环境、插件、映射规则等），是 MyBatis 的“配置中枢” |
| `knownMappers` | `ConcurrentHashMap<Class<?>, MapperProxyFactory<?>>` | 核心存储容器：<br>1. Key：Mapper 接口的 Class 对象（如 `UserMapper.class`）；<br>2. Value：对应接口的动态代理工厂（`MapperProxyFactory`）；<br>3. 采用 `ConcurrentHashMap` 保证多线程环境下的线程安全 |

#### 关键依赖：`MapperProxyFactory`
这是生成 Mapper 接口动态代理对象的工厂类（后续会关联讲解），核心作用是为指定的 Mapper 接口创建 `MapperProxy`（动态代理类）实例，而 `MapperProxy` 会拦截接口方法调用，最终执行对应的 SQL 逻辑。

---

### 三、核心方法拆解 + 底层原理
#### 1. 构造方法：`MapperRegistry(Configuration config)`
```java
public MapperRegistry(Configuration config) {
  this.config = config;
}
```
- **作用**：注入 MyBatis 全局配置对象，让 `MapperRegistry` 能访问全局配置（如环境、映射规则、插件等）；
- **底层逻辑**：`Configuration` 是 MyBatis 的核心上下文，所有组件（包括 `MapperRegistry`）都依赖它获取全局配置，这是“依赖注入”思想的体现。

#### 2. 核心方法：`getMapper(Class<T> type, SqlSession sqlSession)`（获取 Mapper 实例）
这是开发者最常用的方法（如 `sqlSession.getMapper(UserMapper.class)`），核心是生成 Mapper 接口的动态代理对象：
```java
@SuppressWarnings("unchecked")
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  // 1. 从容器中获取对应接口的代理工厂
  final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
  if (mapperProxyFactory == null) {
    // 未注册则抛异常，提示接口未被识别
    throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
  }
  try {
    // 2. 通过代理工厂创建动态代理实例（核心）
    return mapperProxyFactory.newInstance(sqlSession);
  } catch (Exception e) {
    throw new BindingException("Error getting mapper instance. Cause: " + e, e);
  }
}
```
- **核心原理**：
    1. **查找代理工厂**：根据 Mapper 接口的 Class 对象，从 `knownMappers` 中获取对应的 `MapperProxyFactory`；
    2. **生成动态代理对象**：`mapperProxyFactory.newInstance(sqlSession)` 会创建：
        - `MapperProxy`（实现 `InvocationHandler`）：拦截接口方法调用；
        - 通过 `Proxy.newProxyInstance()` 生成 Mapper 接口的动态代理实例；
    3. **异常封装**：将创建过程中的异常包装为 `BindingException`（MyBatis 绑定层异常），统一异常体系。
- **关键细节**：
    - `SqlSession` 传入代理工厂：让动态代理对象能通过 `SqlSession` 执行 SQL（`SqlSession` 是 MyBatis 与数据库交互的核心入口）；
    - 泛型强转：`@SuppressWarnings("unchecked")` 是因为 `knownMappers` 的 Value 是通配符类型，需强转为对应接口的代理工厂。

#### 3. 校验方法：`hasMapper(Class<T> type)`
```java
public <T> boolean hasMapper(Class<T> type) {
  return knownMappers.containsKey(type);
}
```
- **作用**：校验指定的 Mapper 接口是否已注册；
- **底层逻辑**：直接调用 `ConcurrentHashMap` 的 `containsKey()`，保证线程安全。

#### 4. 核心方法：`addMapper(Class<T> type)`（注册 Mapper 接口）
这是 Mapper 接口注册的核心方法，MyBatis 启动时会扫描 Mapper 接口并调用此方法注册：
```java
public <T> void addMapper(Class<T> type) {
  // 1. 仅注册接口（Mapper必须是接口，不能是类）
  if (type.isInterface()) {
    // 2. 校验是否已注册，避免重复注册
    if (hasMapper(type)) {
      throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
    }
    boolean loadCompleted = false;
    try {
      // 3. 先将接口和代理工厂存入容器（关键：先存后解析）
      knownMappers.put(type, new MapperProxyFactory<>(type));
      // 4. 解析Mapper接口上的注解（如@Select、@Insert）或XML映射文件
      MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
      parser.parse();
      // 5. 标记加载完成
      loadCompleted = true;
    } finally {
      // 6. 异常回滚：解析失败则从容器中移除，保证容器一致性
      if (!loadCompleted) {
        knownMappers.remove(type);
      }
    }
  }
}
```
- **核心原理**：
    1. **接口校验**：仅处理接口类型——MyBatis 的 Mapper 必须是接口（因为动态代理只能代理接口）；
    2. **先存后解析**：注释明确说明“先将接口加入容器，再执行解析”，避免解析过程中重复尝试注册；
    3. **注解/XML解析**：`MapperAnnotationBuilder.parse()` 会：
        - 解析接口上的注解（`@Select`、`@Insert` 等）；
        - 加载对应的 XML 映射文件（如 `UserMapper.xml`）；
        - 将解析后的 SQL 语句、参数映射、结果映射等存入 `Configuration`；
    4. **异常回滚**：`finally` 块中如果 `loadCompleted` 为 `false`（解析失败），则从 `knownMappers` 中移除该接口，保证容器中只保留“注册且解析成功”的 Mapper 接口。
- **线程安全**：`knownMappers` 是 `ConcurrentHashMap`，`put`/`remove` 操作线程安全，避免多线程扫描 Mapper 时出现数据不一致。

#### 5. 批量注册：`addMappers(String packageName)` / `addMappers(String packageName, Class<?> superType)`
这两个方法用于**批量扫描指定包下的所有 Mapper 接口**（如 `addMappers("com.example.mapper")`）：
```java
public void addMappers(String packageName, Class<?> superType) {
  // 1. 工具类：扫描指定包下继承/实现superType的所有类
  ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<>();
  resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
  Set<Class<? extends Class<?>>> mapperSet = resolverUtil.getClasses();
  // 2. 遍历扫描结果，逐个注册
  for (Class<?> mapperClass : mapperSet) {
    addMapper(mapperClass);
  }
}

public void addMappers(String packageName) {
  // 重载方法：扫描指定包下所有类（superType为Object）
  addMappers(packageName, Object.class);
}
```
- **核心原理**：
    1. **包扫描工具**：`ResolverUtil` 是 MyBatis 内置的类扫描工具，通过类加载器扫描指定包下的所有类；
    2. **类型过滤**：`ResolverUtil.IsA(superType)` 过滤出继承/实现 `superType` 的类（重载方法中 `superType` 为 `Object`，即扫描所有类）；
    3. **批量注册**：遍历扫描到的类，调用 `addMapper()` 逐个注册，实现“一键扫描包下所有 Mapper”。
- **实际应用**：Spring 整合 MyBatis 时，`@MapperScan("com.example.mapper")` 底层就是调用这个方法批量注册 Mapper 接口。

#### 6. 辅助方法：`getMappers()`
```java
public Collection<Class<?>> getMappers() {
  // 返回不可修改的集合，避免外部修改容器中的Key集合
  return Collections.unmodifiableCollection(knownMappers.keySet());
}
```
- **作用**：获取所有已注册的 Mapper 接口；
- **底层逻辑**：`Collections.unmodifiableCollection()` 包装集合，保证只读性——外部只能查看，不能修改，避免破坏容器的一致性。

---

### 四、底层核心原理总结
#### 1. 动态代理模式的核心体现
`MapperRegistry` 是 MyBatis 动态代理模式的“入口”，完整流程如下：
```mermaid
graph TD
    A[开发者调用sqlSession.getMapper(UserMapper.class)] --> B[MapperRegistry.getMapper()]
    B --> C[从knownMappers获取UserMapper的MapperProxyFactory]
    C --> D[MapperProxyFactory创建MapperProxy（InvocationHandler）]
    D --> E[Proxy.newProxyInstance()生成UserMapper动态代理对象]
    E --> F[开发者调用userMapper.selectById(1)]
    F --> G[MapperProxy.invoke()拦截方法调用]
    G --> H[解析方法名，找到对应的SQL]
    H --> I[通过SqlSession执行SQL并返回结果]
```
- 核心：开发者调用的 `UserMapper` 方法，本质是动态代理对象拦截后，通过 `SqlSession` 执行对应的 SQL，无需编写任何实现类。

#### 2. 线程安全设计
- `knownMappers` 采用 `ConcurrentHashMap`：保证多线程环境下的注册、查询、移除操作线程安全；
- `addMapper()` 方法的“先存后解析+异常回滚”：保证注册过程中即使解析失败，也不会在容器中留下无效的 Mapper 接口。

#### 3. 配置驱动的设计思想
- `MapperRegistry` 依赖 `Configuration`：所有解析后的 Mapper 规则（SQL、参数、结果映射）最终存入 `Configuration`，是 MyBatis “配置中枢”设计的体现；
- `MapperAnnotationBuilder` 解析注解/XML：将开发者编写的 Mapper 接口/XML 转化为 MyBatis 可执行的内部规则，是“约定优于配置”的体现。

#### 4. 异常处理原则
- 统一异常体系：所有异常包装为 `BindingException`（绑定层异常），便于上层捕获和处理；
- 失败回滚：`addMapper()` 的 `finally` 块保证解析失败时移除已注册的接口，避免容器数据不一致。

---

### 五、与其他核心组件的关联
| 组件 | 与 `MapperRegistry` 的关联 |
|------|----------------------------|
| `Configuration` | `MapperRegistry` 是 `Configuration` 的核心成员（`configuration.getMapperRegistry()`），所有 Mapper 注册最终归属到全局配置； |
| `SqlSession` | `getMapper()` 需要传入 `SqlSession`，动态代理对象通过 `SqlSession` 执行 SQL； |
| `MapperProxyFactory` | `MapperRegistry` 存储接口与代理工厂的映射关系，是生成动态代理对象的工厂； |
| `MapperAnnotationBuilder` | `addMapper()` 调用其 `parse()` 方法，解析 Mapper 注解/XML 映射规则； |
| `ResolverUtil` | 批量扫描包下的 Mapper 接口，实现批量注册。 |

---

### 六、关键设计亮点
1. **容器化管理**：通过 `ConcurrentHashMap` 集中管理所有 Mapper 接口，实现“注册-查询-校验”的标准化；
2. **先存后解析**：避免解析过程中重复注册，保证注册流程的原子性；
3. **异常回滚**：`finally` 块的回滚逻辑，保证容器中只保留有效 Mapper 接口；
4. **只读封装**：`getMappers()` 返回不可修改的集合，防止外部篡改容器；
5. **批量扫描**：`addMappers()` 支持包扫描，简化大量 Mapper 接口的注册流程。

### 总结
1. **核心定位**：`MapperRegistry` 是 MyBatis 的 Mapper 接口注册中心，负责 Mapper 接口的注册、存储和动态代理对象的生成，是“接口式编程”的核心；
2. **底层原理**：基于 `ConcurrentHashMap` 实现线程安全的容器管理，通过 `MapperProxyFactory` 生成动态代理对象，拦截接口方法调用并执行 SQL；
3. **关键设计**：先存后解析+异常回滚保证注册一致性，批量扫描简化使用，依赖 `Configuration` 实现全局配置驱动。

掌握 `MapperRegistry` 的原理，就能理解 MyBatis 为什么“只写接口不写实现类就能执行 SQL”——核心是动态代理模式的应用，而 `MapperRegistry` 是这个模式的“总入口”。
