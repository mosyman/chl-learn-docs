
## 目录
- [1](#1)
- [2](#2)


# 1

`MapperProxyFactory` 是 MyBatis 框架中**动态代理机制的工厂类**。它的核心职责非常明确：**利用 JDK 动态代理技术，为一个 Mapper 接口生成一个代理对象实例**。

这个代理对象是 MyBatis 实现“面向接口编程”的关键：开发者只需定义接口，无需编写实现类，MyBatis 会在运行时自动生成实现逻辑，将接口方法调用转换为 SQL 执行。

---

### 一、核心功能详解

#### 1. 类结构解析
```java
public class MapperProxyFactory<T> {
  // 1. 目标接口：用户定义的 Mapper 接口（如 UserMapper.class）
  private final Class<T> mapperInterface;
  
  // 2. 方法缓存：缓存接口方法与具体执行器（Invoker）的映射关系
  private final Map<Method, MapperMethodInvoker> methodCache = new ConcurrentHashMap<>();
}
```
*   **泛型 `<T>`**：保证类型安全，确保生成的代理对象可以被强制转换回原始的 Mapper 接口类型。
*   **`mapperInterface`**：保存了接口的 `Class` 对象，用于后续创建代理时指定“我要代理哪个接口”。
*   **`methodCache`**：这是一个性能优化点。
    *   **作用**：存储接口中的每个 `Method` 对象对应的 `MapperMethodInvoker`。
    *   **`MapperMethodInvoker`**：这是 MyBatis 3.12+ 引入的新特性（之前是直接缓存 `MapperMethod`）。它是一个函数式接口，负责真正执行 SQL 逻辑。
    *   **为什么缓存**：解析一个接口方法对应的 SQL 语句（查找 XML 或注解、解析参数映射、构建 `MappedStatement`）是一个相对耗时的过程。通过缓存，同一个接口方法在多次调用时只需解析一次。

#### 2. 核心方法流程

##### A. 构造函数
```java
public MapperProxyFactory(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
}
```
*   由 `MapperRegistry` 在启动阶段调用，仅仅保存接口类型，此时**不创建任何代理对象**，也**不解析 SQL**。这体现了**懒加载**的思想。

##### B. `newInstance(SqlSession sqlSession)` —— 核心入口
```java
public T newInstance(SqlSession sqlSession) {
    // 1. 创建拦截器核心：MapperProxy
    // 注意：这里传入了当前的 sqlSession，意味着每个代理对象都绑定了一个特定的会话
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    
    // 2. 调用底层的 JDK 代理生成方法
    return newInstance(mapperProxy);
}
```
*   **关键点**：每次调用 `getMapper` 都会创建一个新的 `MapperProxy` 实例。
    *   为什么？因为 `MapperProxy` 内部持有 `SqlSession` 引用。不同的线程或不同的事务上下文需要不同的 `SqlSession`。如果复用 `MapperProxy`，会导致线程安全问题（多个线程共用一个 `SqlSession`）。

##### C. `newInstance(MapperProxy<T> mapperProxy)` —— 魔法发生地
```java
protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(
        mapperInterface.getClassLoader(), // 1. 类加载器
        new Class[] { mapperInterface },  // 2. 要代理的接口数组
        mapperProxy                       // 3. InvocationHandler 实现类
    );
}
```
*   这是标准的 **JDK 动态代理** 调用。
*   返回的对象是一个内存中动态生成的类（通常命名为 `$Proxy0` 等），它实现了 `mapperInterface` 定义的所有方法。
*   当用户调用 `userMapper.selectById(1)` 时，实际上是在调用这个动态生成类的方法，而该方法的内部逻辑被委托给了 `mapperProxy.invoke()`。

---

### 二、底层原理深度剖析

#### 1. JDK 动态代理机制 (The Magic of `Proxy.newProxyInstance`)
这是 Java 反射机制的高级应用。
*   **生成过程**：
    1.  JVM 在内存中动态生成一个字节码类。这个类实现了你定义的所有 Mapper 接口。
    2.  这个生成的类继承自 `java.lang.reflect.Proxy`。
    3.  生成的类中，所有接口方法的实现逻辑都被统一重写为：
        ```java
        // 伪代码：生成的代理类内部逻辑
        public Object selectById(Object id) throws Throwable {
            return this.h.invoke(this, m_selectById, new Object[]{id});
        }
        ```
        其中 `h` 就是传入的 `MapperProxy` 实例，`m_selectById` 是反射获取的 `Method` 对象。
*   **执行流**：
    `用户调用` -> `动态代理类方法` -> `MapperProxy.invoke()` -> `解析 SQL 并执行` -> `返回结果`。

#### 2. 缓存策略与并发安全 (`methodCache`)
*   **缓存内容**：`Map<Method, MapperMethodInvoker>`。
    *   **Key**: `java.lang.reflect.Method` 对象。代表接口中的某个方法（如 `User selectById(int id)`）。
    *   **Value**: `MapperMethodInvoker`。这是一个轻量级的执行器，内部通常持有一个 `MapperMethod` 对象（包含 SQL ID、参数类型、返回类型等元数据）。
*   **为什么用 `ConcurrentHashMap`**？
    *   `MapperProxyFactory` 是单例的（在 `MapperRegistry` 中每个接口对应一个工厂），会被多线程共享。
    *   当第一个线程调用某个方法时，缓存为空，需要解析 SQL 并放入缓存。
    *   当第二个线程同时调用同一个方法时，必须保证不会重复解析，也不会看到半成品的缓存数据。
    *   `ConcurrentHashMap` 提供了高效的线程安全读写。
*   **生命周期**：缓存的生命周期与 `MapperProxyFactory` 相同，即**应用运行期间永久有效**（除非重启）。这意味着 SQL 解析的开销只在第一次调用时产生。

#### 3. `MapperProxy` 的角色 (InvocationHandler)
虽然代码片段中没有展示 `MapperProxy` 的内部，但理解工厂必须理解它生成的产物。
*   `MapperProxy` 实现了 `InvocationHandler` 接口。
*   它的 `invoke` 方法逻辑大致如下：
    1.  检查是否是 `Object` 类的方法（如 `toString`, `hashCode`），如果是，直接反射调用，不走 SQL 逻辑。
    2.  从 `methodCache` 中获取当前方法对应的 `MapperMethodInvoker`。如果没有，则创建并缓存。
    3.  调用 `invoker.invoke(sqlSession, args)`。
    4.  `MapperMethod` 会根据配置，决定调用 `sqlSession.selectOne`, `sqlSession.insert` 等方法。

#### 4. 为什么每次 `getMapper` 都要 `new` 一个 Proxy？
这是一个常见的面试考点。
*   **错误观念**：为了性能，应该缓存 Proxy 对象，单例模式最好。
*   **正确原理**：
    *   `MapperProxy` 内部持有了 `SqlSession` 引用。
    *   `SqlSession` **不是线程安全的**，它代表了数据库的一次会话（包含连接、事务状态、一级缓存等）。
    *   如果在 Web 应用中缓存了 `MapperProxy`（即复用了 `SqlSession`），那么多个请求线程会共用同一个 `SqlSession`，导致：
        1.  **事务混乱**：A 线程提交事务影响了 B 线程。
        2.  **脏数据**：一级缓存串号。
        3.  **并发异常**：`ConcurrentModificationException` 或数据库锁错误。
    *   **结论**：`MapperProxyFactory` 是单例的（存元数据和缓存），但生成的 `MapperProxy`（及背后的 `SqlSession` 绑定关系）必须是**每次请求新建**的。

#### 5. MyBatis 3.12+ 的演进：`MapperMethodInvoker`
在旧版本中，`methodCache` 存储的是 `MapperMethod`。
*   **变化**：新版本引入了 `MapperMethodInvoker` 接口。
*   **原因**：为了支持更灵活的执行策略和解耦。`MapperMethod` 变得臃肿，包含了很多执行逻辑。通过提取 `Invoker` 接口，MyBatis 可以针对不同类型的操作（如普通查询、默认方法、流式查询）使用不同的执行器实现，甚至允许用户自定义执行逻辑，而无需修改核心的 `MapperMethod` 类。这是一种**策略模式**的应用。

---

### 三、总结：从代码到执行的完整链路

1.  **启动阶段**：
    *   `MapperRegistry` 扫描包，发现 `UserMapper` 接口。
    *   创建 `MapperProxyFactory<UserMapper>` 并存入注册表。
    *   解析 `UserMapper` 的注解/XML，但不创建代理对象。

2.  **运行阶段 (请求到来)**：
    *   用户调用 `sqlSession.getMapper(UserMapper.class)`。
    *   `MapperRegistry` 找到对应的 `MapperProxyFactory`。
    *   工厂调用 `newInstance(sqlSession)`。
    *   **关键步骤**：
        1.  创建 `MapperProxy` 实例（绑定当前 `sqlSession`）。
        2.  调用 `Proxy.newProxyInstance` 生成动态代理对象 `$Proxy0`。
    *   返回 `$Proxy0` 给用户。

3.  **方法调用阶段**：
    *   用户调用 `proxy.selectById(1)`。
    *   JVM 路由到 `MapperProxy.invoke()`。
    *   `MapperProxy` 查缓存 `methodCache` 拿到 `Invoker`。
    *   `Invoker` 执行 SQL，通过 `sqlSession` 与数据库交互。
    *   结果返回给用户。

`MapperProxyFactory` 虽短小精悍，却是连接**静态接口定义**与**动态 SQL 执行**的桥梁，完美融合了**工厂模式**、**代理模式**、**缓存策略**和**并发控制**。




[目录](#目录)


# 2


你希望深入理解 MyBatis 中 `MapperProxyFactory<T>` 这个核心工厂类的实现细节、底层原理，以及它在动态代理生成 Mapper 接口实例过程中的核心作用。我会从类的定位、核心成员、核心方法拆解、底层设计原理、与其他组件的关联五个维度，全面且细致地讲解。

### 一、`MapperProxyFactory` 整体定位
`MapperProxyFactory<T>` 是 MyBatis 中**生成 Mapper 接口动态代理对象的工厂类**，核心使命是：
- 为指定的 Mapper 接口（如 `UserMapper`）创建对应的动态代理实例；
- 缓存 Mapper 接口中方法对应的执行器（`MapperMethodInvoker`），避免重复解析方法；
- 封装 JDK 动态代理的创建逻辑，隔离底层代理实现细节，对外提供简洁的 `newInstance()` 接口。

简单来说：`MapperProxyFactory` 是“动态代理对象的生产线”，`MapperRegistry` 中存储的是“生产线”（`MapperProxyFactory`），而开发者最终拿到的 Mapper 实例，是这个生产线生产出来的“产品”（动态代理对象）。

---

### 二、核心成员变量解析
先理解类的核心成员，才能看懂后续方法逻辑：

| 成员变量 | 类型 | 核心作用 |
|----------|------|----------|
| `mapperInterface` | `Class<T>` | 目标 Mapper 接口的 Class 对象（如 `UserMapper.class`），标记当前工厂为哪个接口生产代理对象 |
| `methodCache` | `ConcurrentHashMap<Method, MapperMethodInvoker>` | 方法缓存容器：<br>1. Key：Mapper 接口中的方法对象（如 `UserMapper.selectById(int)` 对应的 `Method`）；<br>2. Value：方法对应的执行器（`MapperMethodInvoker`），封装 SQL 执行逻辑；<br>3. 采用 `ConcurrentHashMap` 保证多线程下的缓存读写安全 |

#### 关键依赖：`MapperMethodInvoker`
这是 MyBatis 3.5+ 引入的接口，核心作用是封装 Mapper 方法的执行逻辑，不同类型的方法（如普通 CRUD、默认方法、批量方法）对应不同的实现类（如 `PlainMethodInvoker`、`DefaultMethodInvoker`）。缓存这个对象的核心目的是**避免每次调用方法时重复解析方法注解/XML 规则**，提升性能。

---

### 三、核心方法拆解 + 底层原理
#### 1. 构造方法：`MapperProxyFactory(Class<T> mapperInterface)`
```java
public MapperProxyFactory(Class<T> mapperInterface) {
  this.mapperInterface = mapperInterface;
}
```
- **作用**：初始化工厂，绑定到指定的 Mapper 接口；
- **底层逻辑**：工厂与接口一一对应，一个 `MapperProxyFactory` 只能为一个 Mapper 接口生产代理对象，符合“单一职责原则”。

#### 2. 基础 getter 方法：`getMapperInterface()` / `getMethodCache()`
```java
public Class<T> getMapperInterface() {
  return mapperInterface;
}

public Map<Method, MapperMethodInvoker> getMethodCache() {
  return methodCache;
}
```
- **作用**：对外暴露工厂的核心属性，方便其他组件（如 `MapperProxy`）访问；
- **设计细节**：`getMethodCache()` 直接返回缓存容器的引用（而非只读包装），因为 `MapperProxy` 需要向缓存中写入 `MapperMethodInvoker`，这是“工厂-代理”协作的关键。

#### 3. 核心方法：`newInstance(MapperProxy<T> mapperProxy)`（创建动态代理对象）
```java
@SuppressWarnings("unchecked")
protected T newInstance(MapperProxy<T> mapperProxy) {
  return (T) Proxy.newProxyInstance(
    mapperInterface.getClassLoader(), // 1. 类加载器：使用Mapper接口的类加载器
    new Class[] { mapperInterface },   // 2. 代理接口：只代理目标Mapper接口
    mapperProxy                        // 3. 调用处理器：核心拦截逻辑
  );
}
```
- **核心原理（JDK 动态代理）**：
  JDK 动态代理的核心是 `Proxy.newProxyInstance()` 方法，它需要三个参数：
    1. **类加载器**：必须与被代理接口的类加载器一致（这里用 `mapperInterface.getClassLoader()`），否则会抛出类加载异常；
    2. **代理接口数组**：指定代理对象要实现的接口（这里只传目标 Mapper 接口，保证代理对象只能调用该接口的方法）；
    3. **InvocationHandler**：即 `MapperProxy`，是方法调用的“拦截器”——所有对代理对象方法的调用，都会转发到 `MapperProxy.invoke()` 方法。
- **泛型强转**：`@SuppressWarnings("unchecked")` 是因为 `Proxy.newProxyInstance()` 返回的是 `Object` 类型，需要强转为目标 Mapper 接口类型；
- **访问控制**：`protected` 修饰符——允许子类重写代理对象的创建逻辑（如自定义类加载器、添加额外接口），符合“开闭原则”。

#### 4. 对外入口：`newInstance(SqlSession sqlSession)`（工厂核心入口）
```java
public T newInstance(SqlSession sqlSession) {
  // 1. 创建MapperProxy（InvocationHandler），传入核心依赖
  final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
  // 2. 调用重载方法创建动态代理对象
  return newInstance(mapperProxy);
}
```
- **核心逻辑**：
    1. **创建调用处理器**：`MapperProxy` 是真正的“逻辑执行者”，它持有：
        - `SqlSession`：用于执行 SQL 的核心入口；
        - `mapperInterface`：目标 Mapper 接口；
        - `methodCache`：方法执行器缓存，避免重复解析；
    2. **创建代理对象**：调用 `newInstance(MapperProxy)` 生成最终的 Mapper 代理实例；
- **设计细节**：对外只暴露 `newInstance(SqlSession)` 方法，隐藏了 `MapperProxy` 的创建细节，符合“封装性原则”。

---

### 四、底层核心原理总结
#### 1. JDK 动态代理的完整流程（MyBatis 视角）
`MapperProxyFactory` 是 JDK 动态代理在 MyBatis 中的核心落地，完整调用链路如下：
```mermaid
graph TD
    A[MapperRegistry.getMapper()] --> B[MapperProxyFactory.newInstance(sqlSession)]
    B --> C[创建MapperProxy（InvocationHandler）]
    C --> D[Proxy.newProxyInstance()生成代理对象]
    D --> E[开发者调用userMapper.selectById(1)]
    E --> F[JDK将调用转发给MapperProxy.invoke()]
    F --> G{方法是否在缓存中?}
    G -- 否 --> H[解析方法，创建MapperMethodInvoker并存入缓存]
    G -- 是 --> I[从缓存获取MapperMethodInvoker]
    H --> I
    I --> J[执行MapperMethodInvoker，通过SqlSession执行SQL]
    J --> K[返回SQL执行结果]
```
- 核心：`MapperProxyFactory` 只负责“生产”代理对象，而方法的拦截和 SQL 执行逻辑，全部交给 `MapperProxy` 和 `MapperMethodInvoker` 处理，符合“单一职责原则”。

#### 2. 缓存设计的核心价值
`methodCache`（`ConcurrentHashMap`）的核心作用是**性能优化**：
- Mapper 接口的方法（如 `selectById`）在第一次调用时，需要解析注解/XML 生成 `MapperMethodInvoker`（包含 SQL 信息、参数映射、结果映射）；
- 后续调用该方法时，直接从缓存中获取 `MapperMethodInvoker`，无需重复解析，大幅提升执行效率；
- `ConcurrentHashMap` 保证多线程下缓存的读写安全（如多个线程同时调用同一个 Mapper 方法，不会重复创建 `MapperMethodInvoker`）。

#### 3. 接口隔离与封装
- **工厂职责隔离**：`MapperProxyFactory` 只负责创建代理对象，不关心方法的具体执行逻辑（执行逻辑由 `MapperProxy` 处理）；
- **对外封装细节**：开发者只需要调用 `newInstance(SqlSession)`，无需关心 `MapperProxy` 的创建、JDK 动态代理的参数等底层细节；
- **扩展预留**：`newInstance(MapperProxy)` 是 `protected` 修饰，允许自定义工厂子类扩展代理对象的创建逻辑（如添加额外的拦截器、修改类加载器）。

#### 4. 线程安全设计
- `methodCache` 采用 `ConcurrentHashMap`：多线程环境下，缓存的读写操作（如添加、查询 `MapperMethodInvoker`）是线程安全的；
- `MapperProxy` 是无状态的（每次创建新实例）：每个代理对象对应一个 `MapperProxy`，避免多线程共享导致的线程安全问题。

---

### 五、与其他核心组件的关联
| 组件 | 与 `MapperProxyFactory` 的关联 |
|------|-------------------------------|
| `MapperRegistry` | `MapperRegistry` 存储 `Class<T>` 与 `MapperProxyFactory<T>` 的映射关系，是工厂的“管理者”； |
| `MapperProxy` | `MapperProxyFactory` 创建 `MapperProxy` 并将其传入 `Proxy.newProxyInstance()`，是调用处理器的“生产者”； |
| `SqlSession` | `newInstance(SqlSession)` 接收 `SqlSession`，并传递给 `MapperProxy`，让代理对象能执行 SQL； |
| `MapperMethodInvoker` | `methodCache` 缓存 `Method` 与 `MapperMethodInvoker` 的映射，`MapperProxy` 从缓存中获取执行器执行 SQL； |
| `Configuration` | `MapperProxy` 最终通过 `SqlSession` 访问 `Configuration` 中的 SQL 映射规则，工厂间接依赖全局配置； |

---

### 六、关键设计亮点
1. **工厂模式 + 动态代理模式**：结合两种设计模式，工厂模式封装代理对象的创建逻辑，动态代理模式实现接口方法的无侵入式拦截；
2. **缓存优化**：`methodCache` 缓存方法执行器，避免重复解析 Mapper 方法规则，提升性能；
3. **开闭原则**：`protected newInstance(MapperProxy)` 允许子类扩展代理创建逻辑，无需修改核心代码；
4. **职责单一**：工厂只负责创建代理对象，拦截逻辑交给 `MapperProxy`，执行逻辑交给 `MapperMethodInvoker`，符合“单一职责原则”；
5. **线程安全**：基于 `ConcurrentHashMap` 的缓存设计，保证多线程环境下的安全运行。

### 总结
1. **核心定位**：`MapperProxyFactory` 是 MyBatis 生成 Mapper 接口动态代理对象的工厂类，封装了 JDK 动态代理的创建逻辑，是“接口式编程”的核心工厂；
2. **底层原理**：基于 JDK 动态代理，通过 `Proxy.newProxyInstance()` 创建代理对象，`MapperProxy` 作为调用处理器拦截方法调用，`methodCache` 缓存方法执行器提升性能；
3. **关键设计**：工厂模式封装创建逻辑，ConcurrentHashMap 保证缓存线程安全，protected 方法预留扩展空间，职责单一原则隔离创建与执行逻辑。

掌握 `MapperProxyFactory` 的原理，就能彻底理解 MyBatis “只定义接口不写实现类就能执行 SQL”的底层逻辑——核心是 JDK 动态代理的应用，而这个工厂类是动态代理对象的“生产者”。

