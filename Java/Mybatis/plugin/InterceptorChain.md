



这段代码是 **MyBatis 插件机制（Plugin Mechanism）的核心调度器** —— `InterceptorChain`。

它的设计极其精简，却支撑起了 MyBatis 最强大的扩展能力（如分页插件 PageHelper、性能分析插件、多租户插件等）。
它的核心职责是**管理拦截器列表**，并在目标对象（如 Executor, StatementHandler, ParameterHandler, ResultSetHandler）创建时，**通过责任链模式层层包裹目标对象**，生成动态代理。

---

### 一、代码功能详解

#### 1. 成员变量
```java
private final List<Interceptor> interceptors = new ArrayList<>();
```
*   **作用**：存储所有用户自定义或框架内置的 `Interceptor` 实例。
*   **特性**：
    *   `final`：引用不可变，保证线程安全的基础（虽然列表内容可变，但在 MyBatis 初始化阶段添加，运行时通常只读）。
    *   `ArrayList`：有序集合。**顺序至关重要**，因为拦截器的执行顺序就是添加的顺序（洋葱模型的外层到内层）。

#### 2. 核心方法：`pluginAll(Object target)`
```java
public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
        target = interceptor.plugin(target);
    }
    return target;
}
```
*   **逻辑**：遍历所有拦截器，依次调用每个拦截器的 `plugin()` 方法。
*   **关键操作**：`target = interceptor.plugin(target);`
    *   每一次循环，`target` 都会被重新赋值。
    *   第一次循环：`target` 是原始对象（如 `SimpleExecutor`）。`interceptor1.plugin()` 返回一个**代理对象**（Proxy1），该代理对象包裹了原始 `target`。
    *   第二次循环：`target` 变成了 Proxy1。`interceptor2.plugin()` 返回 **Proxy2**，该代理对象包裹了 Proxy1。
    *   ...
    *   最终返回的是最外层的代理对象。
*   **结果**：形成了一个**嵌套的代理链**。调用最外层的方法时，请求会像穿过洋葱一样，一层层向内传递，直到到达原始目标对象。

#### 3. 辅助方法
*   `addInterceptor(Interceptor interceptor)`: 向链中添加拦截器。通常在 MyBatis 初始化配置阶段调用（解析 XML 中的 `<plugins>` 标签或通过 Java Config 添加）。
*   `getInterceptors()`: 返回一个**不可修改**的列表视图 (`Collections.unmodifiableList`)。这是为了防止外部代码在运行时随意修改拦截器链，破坏系统的稳定性。

---

### 二、背后/底层原理深度剖析

#### 1. 责任链模式 (Chain of Responsibility) 与 装饰器模式 (Decorator Pattern) 的结合
`InterceptorChain` 是这两种模式的典型混合应用。
*   **责任链**：请求（方法调用）沿着拦截器链传递，每个拦截器都有机会处理请求（前置处理）、决定是否继续传递（`proceed`）、或处理后置逻辑。
*   **装饰器**：每个 `plugin()` 调用实际上是在给目标对象“穿衣服”。
    *   原始对象：`Executor`
    *   加一层衣服：`PluginProxy(Executor)`
    *   再加一层：`PluginProxy(PluginProxy(Executor))`
    *   外部调用者面对的是最外层的代理，完全感知不到内部有多少层包装。

#### 2. JDK 动态代理 (JDK Dynamic Proxy) 的核心实现
虽然 `InterceptorChain` 代码很简单，但真正的魔法发生在 `interceptor.plugin(target)` 内部。
*   **标准实现**：MyBatis 提供的基类 `org.apache.ibatis.plugin.Plugin` 实现了 `Interceptor` 接口，其 `plugin` 方法如下：
    ```java
    public static Object wrap(Object target, InterceptorChain interceptorChain) {
      // 获取所有拦截器
      List<Interceptor> interceptors = interceptorChain.getInterceptors();
      for (Interceptor interceptor : interceptors) {
        target = interceptor.plugin(target);
      }
      return target;
    }

    @Override
    public Object plugin(Object target) {
      // 核心逻辑：判断当前拦截器是否关注目标对象的某些方法
      if (interceptorMap.containsKey(target.getClass())) {
        // 如果关注，则创建 JDK 动态代理
        return Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            new InvocationHandler() {
                @Override
                public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                    // 如果该方法在 @Intercepts 注解中声明过，则执行拦截逻辑
                    if (interceptorMap.get(target.getClass()).contains(method)) {
                        return interceptor.intercept(new Invocation(target, method, args));
                    }
                    // 否则，直接反射调用原方法
                    return method.invoke(target, args);
                }
            });
      }
      // 如果不关注，直接返回原对象，避免不必要的代理开销
      return target;
    }
    ```
*   **原理细节**：
    1.  **签名匹配**：MyBatis 解析用户拦截器上的 `@Intercepts(@Signature(...))` 注解，建立 `Class -> Set<Method>` 的映射表。只有当目标对象的方法在映射表中时，才创建代理。这极大地优化了性能，避免了为不相关的方法创建代理。
    2.  **InvocationHandler**：生成的代理对象的所有方法调用都会路由到 `invoke` 方法。
    3.  **拦截逻辑**：在 `invoke` 中，再次检查方法是否需要拦截。如果需要，调用用户的 `interceptor.intercept(Invocation invocation)` 方法。
    4.  **放行**：用户在 `intercept` 方法中调用 `invocation.proceed()`，实际上是反射调用 `method.invoke(target, args)`，将请求传递给链中的下一个节点（可能是下一个代理，也可能是原始对象）。

#### 3. “洋葱模型”执行流程
假设我们有两个插件：`PluginA` (先添加) 和 `PluginB` (后添加)，目标是 `Executor`。
生成的结构是：`ProxyB(ProxyA(Executor))`。

当调用 `executor.query()` 时：
1.  **进入 ProxyB**：执行 `PluginB` 的前置逻辑。
2.  **调用 proceed()**：进入 `ProxyA`。
3.  **进入 ProxyA**：执行 `PluginA` 的前置逻辑。
4.  **调用 proceed()**：进入原始 `Executor`。
5.  **原始执行**：执行真实的 SQL 查询逻辑。
6.  **返回 ProxyA**：执行 `PluginA` 的后置逻辑。
7.  **返回 ProxyB**：执行 `PluginB` 的后置逻辑。
8.  **最终返回**：结果回到调用者。

**注意顺序**：
*   **前置逻辑执行顺序**：与添加顺序一致 (A -> B)。
*   **后置逻辑执行顺序**：与添加顺序相反 (B -> A)。
*   这与 Servlet Filter 或 Spring AOP 的逻辑是一致的。

#### 4. 为什么只在特定接口上生效？
MyBatis 的插件只能拦截以下四个核心接口的方法：
1.  `Executor` (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)
2.  `ParameterHandler` (getParameterObject, setParameters)
3.  `ResultSetHandler` (handleResultSets, handleOutputParameters)
4.  `StatementHandler` (prepare, parameterize, batch, update, query)

**原理**：`Plugin.wrap` 方法在创建代理时，使用了 `target.getClass().getInterfaces()`。这意味着目标对象必须实现了接口，JDK 动态代理才能工作。MyBatis 的核心组件设计都严格遵循接口编程，正是为了支持这种代理机制。如果用户尝试拦截一个没有实现这些接口的类，插件将无效。

#### 5. 性能考量
*   **按需代理**：如前所述，只有当 `@Signature` 匹配时才创建代理。如果一个插件只拦截 `query` 方法，那么调用 `commit` 方法时，代理的 `invoke` 方法会直接反射调用目标，几乎没有额外开销（除了极小的反射成本，现代 JVM 对反射优化得很好）。
*   **链式调用开销**：每多一个插件，就多一层方法调用和栈帧。在极高并发场景下，过多的插件（尤其是那些不做任何判断直接 `proceed` 的插件）会累积微小的性能损耗。因此，插件应遵循“最小干扰原则”。

---

### 三、总结与应用场景

`InterceptorChain` 是 MyBatis **可扩展性架构的枢纽**。

1.  **解耦**：它将“核心执行逻辑”与“横切关注点”（Cross-Cutting Concerns）完全分离。核心代码不需要知道分页、审计、监控的存在。
2.  **灵活编排**：通过简单的列表顺序，控制了多个切面的执行次序。
3.  **经典应用场景**：
    *   **分页 (PageHelper)**：拦截 `Executor.query`，解析分页参数，重写 SQL 添加 `LIMIT/OFFSET`，并自动追加 Count 查询。
    *   **多租户 (Multi-tenant)**：拦截 `StatementHandler.prepare` 或 `Executor.query`，在 SQL 中自动拼接 `WHERE tenant_id = ?`。
    *   **SQL 性能监控**：拦截 `Executor.update/query`，记录开始时间，`proceed` 后记录结束时间，计算耗时并打印慢 SQL。
    *   **字段自动填充**：拦截 `ParameterHandler.setParameters`，自动填充 `create_time`, `update_user` 等字段。

**一句话总结**：`InterceptorChain` 利用 **JDK 动态代理** 和 **责任链模式**，将一组拦截器层层包裹在 MyBatis 核心组件之外，构建了一个非侵入式、可灵活编排的 AOP（面向切面编程）框架，使得 MyBatis 能够在不修改源码的情况下无限扩展功能。
