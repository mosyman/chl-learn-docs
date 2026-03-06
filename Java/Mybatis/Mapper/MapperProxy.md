

## 目录
- [1](#1)
- [2](#2)


# 1

这段代码是 **MyBatis** 框架中核心类 `MapperProxy` 的源码。它是 MyBatis 实现 **Mapper 接口代理机制** 的关键所在。

简单来说，当你通过 `sqlSession.getMapper(UserMapper.class)` 获取一个 Mapper 对象时，MyBatis 并没有返回一个具体的实现类（因为你没写实现类），
而是利用 JDK 动态代理生成了一个 `MapperProxy` 对象。当你调用 Mapper 接口中的方法（如 `userMapper.selectById(1)`）时，实际执行的是 `MapperProxy` 中的 `invoke` 方法。

下面我将分两部分详细解释：**代码流程详解** 和 **底层原理深度剖析**。

---

### 一、代码流程详解

#### 1. 核心字段
*   `sqlSession`: 用于执行 SQL 的核心会话对象。
*   `mapperInterface`: 用户定义的 Mapper 接口 Class（例如 `UserMapper.class`）。
*   `methodCache`: 缓存 `Method` 到 `MapperMethodInvoker` 的映射。避免每次调用都重新解析方法，提升性能。
*   `privateLookupInMethod`: 这是一个静态初始化的 `Method` 对象，专门用于 Java 9+ 环境下处理接口默认方法（Default Methods）的反射调用。

#### 2. 构造函数
初始化上述字段。注意 `methodCache` 通常是由外部传入的一个 `ConcurrentHashMap`，保证线程安全。

#### 3. 静态代码块 (Java 9+ 兼容性)
```java
static {
    try {
        // 尝试获取 Java 9 引入的 MethodHandles.privateLookupIn 方法
        privateLookupInMethod = MethodHandles.class.getMethod("privateLookupIn", Class.class, MethodHandles.Lookup.class);
    } catch (NoSuchMethodException e) {
        // 如果是 Java 8，没有这个方法，抛出异常（但在实际运行逻辑中，如果是 Java 8 且调用了默认方法，会在 later 阶段处理或报错，这里主要是预检查）
        throw new IllegalStateException(...);
    }
}
```
**作用**：MyBatis 支持接口中的 `default` 方法。在 Java 8 中，动态代理调用 default 方法比较麻烦；Java 9 引入了 `MethodHandles.privateLookupIn` 使得获取私有访问权限的 `MethodHandle` 变得容易。这段代码是为了后续能兼容调用接口的 default 方法做准备。

#### 4. 核心入口：`invoke` 方法
这是 JDK 动态代理的拦截器入口。任何对 Mapper 接口的调用都会进入这里。

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
        // 【关键点 1】处理 Object 类的方法
        if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, args);
        }
        // 【关键点 2】处理 Mapper 接口定义的方法
        return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
    } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
    }
}
```
*   **Object 类方法处理**：如果调用的是 `toString()`, `hashCode()`, `equals()` 等 `Object` 类自带的方法，直接反射调用当前代理对象 (`this`) 的这些方法，不走 SQL 逻辑。
*   **业务方法处理**：如果是接口定义的方法（如 `selectUser`），则通过 `cachedInvoker` 获取执行器，然后执行。

#### 5. 缓存与执行器获取：`cachedInvoker`
```java
private MapperMethodInvoker cachedInvoker(Method method) throws Throwable {
    return methodCache.computeIfAbsent(method, m -> {
        if (!m.isDefault()) {
            // 【情况 A】普通接口方法（抽象方法）
            // 创建 PlainMethodInvoker，内部封装了 MapperMethod（包含 SQL ID、参数映射、结果映射等元数据）
            return new PlainMethodInvoker(new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
        }
        // 【情况 B】接口默认方法 (default method)
        try {
            return new DefaultMethodInvoker(getMethodHandleJava9(method));
        } catch (...) { ... }
    });
}
```
*   **缓存机制**：使用 `computeIfAbsent`，确保同一个方法只解析一次，后续直接从 Map 取。
*   **分支逻辑**：
    *   **非 default 方法**：这是最常见的情况。它会构建一个 `MapperMethod` 对象。这个对象里解析了注解（`@Select`）或 XML 中的 SQL 语句、参数类型、返回值类型等。然后包裹在 `PlainMethodInvoker` 中。
    *   **default 方法**：如果接口里有 `default void log() { ... }` 这种写法。MyBatis 需要调用接口中这段具体的代码实现。它使用 `MethodHandle` 技术来绑定并调用这个默认方法。

#### 6. 两种执行器实现

**A. `PlainMethodInvoker` (普通方法)**
```java
public Object invoke(...) {
    // 核心：委托给 MapperMethod 执行
    return mapperMethod.execute(sqlSession, args);
}
```
*   `MapperMethod.execute` 是真正的干活地方。它会根据方法签名判断是 `SELECT`, `INSERT`, `UPDATE` 还是 `DELETE`，然后调用 `sqlSession` 的对应方法（如 `sqlSession.selectOne`），最终通过 JDBC 执行 SQL。

**B. `DefaultMethodInvoker` (默认方法)**
```java
public Object invoke(...) {
    // 核心：利用 MethodHandle 绑定到代理对象 proxy，然后.invokeWithArguments
    return methodHandle.bindTo(proxy).invokeWithArguments(args);
}
```
*   这里利用了 Java 7+ 的 `MethodHandle` 机制。`bindTo(proxy)` 将调用的目标对象绑定为当前的代理对象 `proxy`，从而让接口中的 default 方法代码能够正确执行（因为 default 方法可能需要访问接口的其他方法，而这些其他方法已经被代理了）。

---

### 二、背后/底层原理深度剖析

#### 1. JDK 动态代理 (JDK Dynamic Proxy)
*   **原理**：`MapperProxy` 实现了 `InvocationHandler` 接口。MyBatis 使用 `Proxy.newProxyInstance(classLoader, interfaces, handler)` 生成代理对象。
*   **为什么用 JDK 代理而不是 CGLIB？**
    *   MyBatis 的 Mapper 必须是**接口**。JDK 动态代理专门用于代理接口，无需第三方库（CGLIB 需要字节码生成库，虽然现在 Spring 等也常用，但 MyBatis 原生设计倾向于轻量级和标准 JDK 特性）。
    *   接口代理天然符合 "面向接口编程" 的规范。

#### 2. 元数据解析与缓存 (Meta-Data & Caching)
*   **痛点**：每次调用方法都去解析注解或查找 XML SQL 定义是非常耗性能的。
*   **解决方案**：`methodCache` (`Map<Method, MapperMethodInvoker>`)。
    *   **首次调用**：解析方法上的 `@Select` 注解，或者根据方法名去 XML 中找对应的 `<select id="...">` 节点。构建 `MapperMethod` 对象（包含 `SqlCommandType`, `SqlSource` 等重型对象）。
    *   **后续调用**：直接从 Map 中取出 `MapperMethodInvoker`，跳过解析过程，直接执行。
    *   **并发安全**：`methodCache` 通常是 `ConcurrentHashMap`，`computeIfAbsent` 保证了在多线程环境下的安全懒加载。

#### 3. 接口默认方法 (Default Methods) 的处理难点
这是 MyBatis 3.4+ 引入的高级特性。
*   **背景**：Java 8 允许接口有方法体 (`default`)。如果用户在 Mapper 接口写了 default 方法，期望执行其中的 Java 逻辑，而不是 SQL。
*   **挑战**：JDK 动态代理的 `invoke` 方法会拦截**所有**接口方法调用，包括 default 方法。如果直接放行，会导致死循环（因为 default 方法内部可能调用其他接口方法，又回到 `invoke`）。
*   **底层黑科技 (`MethodHandles`)**：
    *   在 Java 8 中，调用 default 方法非常繁琐，通常需要构造 `MethodHandles.Lookup` 并欺骗权限。
    *   在 Java 9 中，引入了 `MethodHandles.privateLookupIn(callingClass, lookup)`。这允许我们获取一个具有**私有访问权限**的 Lookup 对象，即使是在代理类中，也能合法地找到并调用接口内部的 default 方法实现。
    *   代码中的 `getMethodHandleJava9` 就是利用反射调用这个 API，获取 `MethodHandle`，然后通过 `bindTo(proxy)` 将调用上下文绑定回代理对象。这样，当 default 方法内部调用 `this.otherMethod()` 时，依然会走代理逻辑（即执行 SQL），实现了完美的混合编程。

#### 4. 异常解包 (Exception Unwrapping)
```java
throw ExceptionUtil.unwrapThrowable(t);
```
*   **原理**：反射调用（`method.invoke` 或 `methodHandle.invoke`）抛出的异常通常会被包装在 `InvocationTargetException` 或 `UndeclaredThrowableException` 中。
*   **目的**：为了让用户感受不到反射的存在，MyBatis 会将这些包装壳剥掉，直接抛出原始的业务异常（如 `PersistenceException` 或数据库层面的 `SQLException`），保持调用栈的清晰。

#### 5. 执行链路总结
1.  **用户调用**: `userMapper.selectById(1)`
2.  **JDK 代理拦截**: 进入 `MapperProxy.invoke`
3.  **缓存查找**: 检查 `methodCache` 是否有该方法的执行器。
    *   无 -> 解析注解/XML -> 创建 `MapperMethod` -> 封装为 `PlainMethodInvoker` -> 放入缓存。
    *   有 -> 直接获取。
4.  **执行逻辑**:
    *   调用 `PlainMethodInvoker.invoke`。
    *   进入 `MapperMethod.execute`。
    *   判断 SQL 类型 (SELECT/INSERT...)。
    *   参数转换 (将 Java 对象转为 JDBC 参数)。
    *   调用 `SqlSession.selectOne/insert...`。
    *   `SqlSession` 委托给 `Executor`。
    *   `Executor` 处理缓存、事务，最终通过 `StatementHandler` -> `ParameterHandler` -> JDBC Driver 执行 SQL。
    *   `ResultSetHandler` 将结果集映射为 Java 对象。
5.  **返回结果**: 层层返回给用户。

### 总结
`MapperProxy` 是 MyBatis "魔法" 的源头。它巧妙地结合了：
1.  **JDK 动态代理** 实现无侵入式接口调用。
2.  **缓存机制** 保证高性能。
3.  **MethodHandles** 解决 Java 8/9+ 接口默认方法的调用难题。
4.  **元数据驱动** 将 Java 方法映射到 SQL 语句。

这使得开发者只需要定义接口和 SQL，完全不需要编写 boilerplate 的实现代码。






[目录](#目录)

# 2

你想要深入理解 MyBatis 核心组件 `MapperProxy` 类的详细实现逻辑，以及它背后的底层原理，这是理解 MyBatis 动态代理机制的关键。下面我会从**功能定位、核心代码逐行解析、底层原理、设计思想**四个维度，完整拆解这个类。

### 一、核心定位：MapperProxy 是什么？
`MapperProxy` 是 MyBatis 实现 **Mapper 接口动态代理** 的核心类，它实现了 JDK 动态代理的 `InvocationHandler` 接口，是连接「Mapper 接口方法调用」和「实际 SQL 执行」的桥梁。
简单来说：你在代码中调用 `UserMapper.selectById(1)` 时，真正执行的是这个类的 `invoke` 方法，它会帮你找到对应的 SQL 并执行。

### 二、核心代码逐行解析（带原理说明）
#### 1. 类定义与成员变量
```java
public class MapperProxy<T> implements InvocationHandler, Serializable {
  // 序列化版本号，Serializable 接口要求
  private static final long serialVersionUID = -4724728412955527868L;
  // Java 9+ 用于访问私有方法的 Method 对象（底层：MethodHandles.privateLookupIn）
  private static final Method privateLookupInMethod;
  // MyBatis 的核心会话对象，用于执行 SQL（底层：封装了 Connection、Executor 等）
  private final SqlSession sqlSession;
  // 被代理的 Mapper 接口（比如 UserMapper.class）
  private final Class<T> mapperInterface;
  // 方法缓存：缓存 Method -> MapperMethodInvoker，避免重复创建（底层：空间换时间）
  private final Map<Method, MapperMethodInvoker> methodCache;
}
```
- **底层原理**：
    - `InvocationHandler`：JDK 动态代理的核心接口，所有代理对象的方法调用都会转发到 `invoke` 方法。
    - `methodCache`：缓存的核心目的是**避免每次调用 Mapper 方法都重新解析注解/XML 生成 MapperMethod**，提升性能。

#### 2. 静态代码块
```java
static {
  try {
    // 获取 MethodHandles.privateLookupIn 方法（Java 9+ 新增，用于突破访问权限）
    privateLookupInMethod = MethodHandles.class.getMethod("privateLookupIn", Class.class, MethodHandles.Lookup.class);
  } catch (NoSuchMethodException e) {
    throw new IllegalStateException(
        "There is no 'privateLookupIn(Class, Lookup)' method in java.lang.invoke.MethodHandles.", e);
  }
}
```
- **核心作用**：提前获取 `privateLookupIn` 方法的引用，用于后续调用 Mapper 接口的**默认方法（default method）**。
- **底层原理**：
    - Java 8 开始接口支持默认方法，但默认方法属于接口的私有实现，常规反射无法直接调用；
    - Java 9 提供 `MethodHandles.privateLookupIn` 突破访问权限检查，能直接调用接口的默认方法（替代 Java 8 的 hack 方式）。

#### 3. 构造方法
```java
public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethodInvoker> methodCache) {
  this.sqlSession = sqlSession;
  this.mapperInterface = mapperInterface;
  this.methodCache = methodCache;
}
```
- **核心作用**：初始化三大核心依赖：
    1. `sqlSession`：提供 SQL 执行能力；
    2. `mapperInterface`：标记当前代理的是哪个 Mapper 接口；
    3. `methodCache`：方法缓存（由 MapperProxyFactory 创建，多个 MapperProxy 共享）。

#### 4. 核心方法：invoke（JDK 动态代理入口）
```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  try {
    // 1. 过滤 Object 类的方法（toString、hashCode 等），直接调用自身实现
    if (Object.class.equals(method.getDeclaringClass())) {
      return method.invoke(this, args);
    }
    // 2. 核心逻辑：获取缓存的 Invoker，执行方法
    return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
  } catch (Throwable t) {
    // 解包异常（比如将 InvocationTargetException 转为底层真实异常）
    throw ExceptionUtil.unwrapThrowable(t);
  }
}
```
- **核心逻辑拆解**：
    1. **过滤 Object 方法**：避免代理 `toString()`、`hashCode()` 等通用方法，直接执行原生逻辑；
    2. **缓存+执行**：通过 `cachedInvoker` 获取方法执行器，再调用 `invoke` 执行实际逻辑；
    3. **异常解包**：MyBatis 自定义的异常处理，把反射调用的包装异常（如 `InvocationTargetException`）转为用户能理解的异常。
- **底层原理**：
    - JDK 动态代理的核心机制：当你调用 `UserMapper.selectById()` 时，JVM 会自动将调用转发到 `InvocationHandler.invoke()` 方法；
    - 这里的 `proxy` 是动态生成的 Mapper 代理对象，`method` 是当前调用的方法（如 `selectById`），`args` 是方法入参。

#### 5. 缓存方法执行器：cachedInvoker
```java
private MapperMethodInvoker cachedInvoker(Method method) throws Throwable {
  try {
    // computeIfAbsent：缓存不存在则创建，存在则直接返回
    return methodCache.computeIfAbsent(method, m -> {
      // 1. 非默认方法（常规 Mapper 方法，如 selectById）
      if (!m.isDefault()) {
        // 创建 PlainMethodInvoker，封装 MapperMethod（核心：解析 SQL 相关信息）
        return new PlainMethodInvoker(new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
      }
      // 2. 默认方法（接口的 default 方法）
      try {
        return new DefaultMethodInvoker(getMethodHandleJava9(method));
      } catch (NoSuchMethodException | IllegalAccessException | InvocationTargetException e) {
        throw new RuntimeException(e);
      }
    });
  } catch (RuntimeException re) {
    // 解包异常，抛出真实原因
    Throwable cause = re.getCause();
    throw cause == null ? re : cause;
  }
}
```
- **核心逻辑**：
    - 利用 `Map.computeIfAbsent` 实现「懒加载+缓存」：第一次调用方法时创建 `MapperMethodInvoker`，后续直接从缓存取；
    - 区分两种方法类型：
        - 常规 Mapper 方法（非 default）：创建 `PlainMethodInvoker`；
        - 接口默认方法（default）：创建 `DefaultMethodInvoker`。
- **底层原理**：
    - `MapperMethod` 是 MyBatis 解析 Mapper 方法的核心类：它会解析注解（如 `@Select`）或 XML 中的 SQL、参数映射、返回值映射等信息，封装成可执行的对象；
    - `computeIfAbsent` 是 ConcurrentHashMap 的高效方法（methodCache 实际是 ConcurrentHashMap），保证线程安全的同时实现懒加载。

#### 6. 获取默认方法的 MethodHandle：getMethodHandleJava9
```java
private MethodHandle getMethodHandleJava9(Method method)
    throws NoSuchMethodException, IllegalAccessException, InvocationTargetException {
  final Class<?> declaringClass = method.getDeclaringClass();
  // 1. 调用 privateLookupIn 方法，获取能访问私有方法的 Lookup 对象
  // 2. 通过 findSpecial 找到接口的默认方法的 MethodHandle
  return ((Lookup) privateLookupInMethod.invoke(null, declaringClass, MethodHandles.lookup())).findSpecial(
      declaringClass, method.getName(), 
      MethodType.methodType(method.getReturnType(), method.getParameterTypes()),
      declaringClass);
}
```
- **核心作用**：获取接口默认方法的 `MethodHandle`（Java 7 引入的「方法句柄」，比反射更轻量、高效）。
- **底层原理**：
    - `MethodHandles.Lookup`：用于查找和访问方法的句柄，`privateLookupIn` 能突破访问权限，访问接口的默认方法；
    - `findSpecial`：查找类的私有/默认方法（区别于 `findVirtual` 查找公共方法）；
    - `MethodType`：描述方法的签名（返回值+参数类型），保证找到的方法和目标方法一致。

#### 7. 内部类：PlainMethodInvoker（常规 Mapper 方法执行器）
```java
private static class PlainMethodInvoker implements MapperMethodInvoker {
  private final MapperMethod mapperMethod;

  public PlainMethodInvoker(MapperMethod mapperMethod) {
    this.mapperMethod = mapperMethod;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args, SqlSession sqlSession) throws Throwable {
    // 核心：调用 MapperMethod.execute 执行 SQL
    return mapperMethod.execute(sqlSession, args);
  }
}
```
- **核心逻辑**：将方法调用转发给 `MapperMethod.execute`，这是真正执行 SQL 的入口。
- **底层原理**：
    - `MapperMethod.execute` 会根据 SQL 类型（SELECT/INSERT/UPDATE/DELETE）调用 `SqlSession` 的对应方法（如 `sqlSession.selectOne()`）；
    - `SqlSession` 最终会通过 `Executor` 执行 SQL，完成结果集映射。

#### 8. 内部类：DefaultMethodInvoker（默认方法执行器）
```java
private static class DefaultMethodInvoker implements MapperMethodInvoker {
  private final MethodHandle methodHandle;

  public DefaultMethodInvoker(MethodHandle methodHandle) {
    this.methodHandle = methodHandle;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args, SqlSession sqlSession) throws Throwable {
    // 核心：绑定代理对象，执行默认方法
    return methodHandle.bindTo(proxy).invokeWithArguments(args);
  }
}
```
- **核心逻辑**：通过 `MethodHandle` 执行接口的默认方法。
- **底层原理**：
    - `bindTo(proxy)`：将方法句柄绑定到代理对象（确保默认方法中的 `this` 指向代理对象）；
    - `invokeWithArguments(args)`：传入参数执行方法，比反射的 `method.invoke()` 性能更高。

### 三、底层核心原理总结
#### 1. JDK 动态代理原理
- MyBatis 不直接创建 Mapper 接口的实现类，而是通过 JDK 动态代理（`Proxy.newProxyInstance()`）生成代理对象；
- 代理对象的所有方法调用都会转发到 `MapperProxy.invoke()` 方法，实现「无实现类却能执行方法」的效果。

#### 2. 缓存原理
- `methodCache` 是线程安全的 `ConcurrentHashMap`，缓存 `Method` 到 `MapperMethodInvoker` 的映射；
- 避免每次调用方法都重新解析 Mapper 注解/XML，提升执行效率（空间换时间）。

#### 3. 方法分发原理
- 区分「Object 类方法」「接口默认方法」「常规 Mapper 方法」，分别走不同的执行逻辑；
- 常规方法通过 `MapperMethod` 执行 SQL，默认方法通过 `MethodHandle` 直接执行，兼顾灵活性和性能。

#### 4. 权限突破原理（Java 9+）
- 接口默认方法属于私有实现，常规反射无法调用；
- 通过 `MethodHandles.privateLookupIn` 突破访问权限检查，获取 `MethodHandle` 执行默认方法。

### 四、执行流程可视化（Mermaid 流程图）
```mermaid
graph TD
    A[用户调用 UserMapper.selectById(1)] --> B[JDK 动态代理转发到 MapperProxy.invoke()]
    B --> C{是否是 Object 类方法？}
    C -->|是| D[直接调用 method.invoke(this, args)]
    C -->|否| E[cachedInvoker 获取执行器]
    E --> F{是否是默认方法？}
    F -->|是| G[创建 DefaultMethodInvoker，通过 MethodHandle 执行]
    F -->|否| H[创建 PlainMethodInvoker，封装 MapperMethod]
    H --> I[MapperMethod.execute 执行 SQL]
    I --> J[SqlSession 调用 Executor 执行 SQL，返回结果]
    G & D & J --> K[返回最终结果给用户]
```

### 总结
1. **核心作用**：`MapperProxy` 是 MyBatis 动态代理的核心，承接 Mapper 接口方法调用，转发到实际的 SQL 执行逻辑；
2. **核心优化**：通过 `methodCache` 缓存方法执行器，避免重复解析 Mapper 配置，提升性能；
3. **核心适配**：区分常规方法和默认方法，分别通过 `MapperMethod` 和 `MethodHandle` 执行，兼顾 SQL 执行和接口默认方法的兼容性。

这个类的设计充分体现了「懒加载」「缓存」「接口隔离」等设计思想，也是 MyBatis 「无 XML/注解配置即可执行 SQL」的核心实现。




[目录](#目录)


