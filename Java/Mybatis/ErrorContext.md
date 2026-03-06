

## 目录
- [1](#1)
- [2](#2)



## ErrorContext

## 1

这段代码是 **MyBatis** 框架中用于处理错误上下文的核心类 `ErrorContext`。它的主要作用是在复杂的调用链（尤其是涉及 SQL 执行、映射解析等深层嵌套调用）中，**收集并串联分散的错误信息**，最终生成一个清晰、可读性极强的错误报告。

下面我将从**功能详解**和**底层原理**两个维度为你深入剖析。

---

### 一、功能详解：它是做什么的？

在传统的 Java 异常处理中，如果底层抛出一个 `SQLException`，上层捕获后可能会包装成一个新的异常。如果多层嵌套，开发者往往只能看到最外层的报错，或者需要层层剥开 `cause` 才能找到根源（比如哪条 SQL 错了、在哪个 XML 文件里、正在做什么操作）。

`ErrorContext` 解决了这个问题，它像一个`随线程流动的记事本`：

1.  **链式收集信息**：
    通过 `resource()`, `activity()`, `object()`, `sql()`, `message()` 等方法，允许代码在不同层级向这个“记事本”里写入关键信息。
    *   例如：DAO 层写入 "SQL: SELECT * FROM user"，XML 解析层写入 "Resource: UserMapper.xml"，业务层写入 "Activity: querying user by id"。

2.  **上下文隔离与嵌套（Store/Recall）**：
    *   `store()`: 当进入一个新的逻辑作用域（如开始解析一个新的 XML 节点，或执行一个新的 SQL 命令）时，保存当前的上下文状态，并创建一个新的空白上下文。这防止了子任务的信息污染父任务。
    *   `recall()`: 当子任务结束（无论成功还是失败），恢复之前保存的父级上下文。

3.  **统一格式化输出**：
    重写 `toString()` 方法。当异常发生时，将收集到的所有碎片信息（SQL 语句、资源文件、操作步骤、根本原因）按特定格式拼接成一个完整的字符串，直接展示给开发者。

#### 典型输出示例
如果没有 `ErrorContext`，你可能只看到：
> `org.apache.ibatis.exceptions.PersistenceException: Error querying database. Cause: java.sql.SQLException: Column 'name' not found`

有了 `ErrorContext`，你会看到：
```text
### The error may exist in com/example/mapper/UserMapper.xml
### The error may involve com.example.mapper.UserMapper.selectUser
### The error occurred while executing a query
### SQL: SELECT id, name_form_typo FROM user WHERE id = ?
### Cause: java.sql.SQLException: Column 'name_form_typo' not found
```
**价值**：一眼就能看出是哪个文件、哪个方法、哪条 SQL 出了什么问题。

---

### 二、背后/底层原理：它是如何实现的？

这个类的精妙之处在于它结合了 **ThreadLocal**、**原型模式（隐式）** 和 **责任链/栈思想** 来解决多线程环境下的状态管理问题。

#### 1. 核心机制：ThreadLocal (线程隔离)
```java
private static final ThreadLocal<ErrorContext> LOCAL = ThreadLocal.withInitial(ErrorContext::new);
```
*   **原理**：`ThreadLocal` 为每个线程提供独立的变量副本。
*   **为什么需要它？**
    *   MyBatis 通常运行在高并发的 Web 环境中（如 Tomcat），多个请求由不同线程处理。
    *   如果 `ErrorContext` 是静态共享的，线程 A 正在执行 SQL A，线程 B 突然写入了 SQL B 的信息，那么线程 A 报错时就会显示线程 B 的 SQL，导致严重的误导。
    *   使用 `ThreadLocal` 确保了**线程安全**，每个线程都有自己独立的错误记录本，互不干扰。
*   **初始化**：`withInitial(ErrorContext::new)` 保证第一次调用 `LOCAL.get()` 时自动创建一个新实例，无需手动判空。

#### 2. 上下文嵌套机制：手动维护“栈” (Store/Recall)
这是该类最巧妙的设计。通常我们处理嵌套作用域会用递归或显式的 `S tack` 结构，但这里采用了一种**“链表式快照”**的方式。

*   **字段设计**：
    ```java
    private ErrorContext stored; // 指向“上一层”上下文的引用
    ```
*   **`store()` 方法原理**：
    ```java
    public ErrorContext store() {
        ErrorContext newContext = new ErrorContext(); // 1. 创建新本子
        newContext.stored = this;                     // 2. 新本子记录当前本子（this）作为“上一级”
        LOCAL.set(newContext);                        // 3. 将 ThreadLocal 指向新本子
        return LOCAL.get();
    }
    ```
    *   **场景**：假设当前在处理 `UserMapper`，现在要进入 `selectUser` 方法内部。
    *   **动作**：创建一个新的 `ErrorContext` (记为 C2)，让 C2 的 `stored` 指向旧的 (C1)。然后把线程变量指向 C2。
    *   **结果**：现在的操作（写 SQL、写活动）都记录在 C2 上，不会覆盖 C1 中已记录的 `UserMapper` 信息。

*   **`recall()` 方法原理**：
    ```java
    public ErrorContext recall() {
        if (stored != null) {
            LOCAL.set(stored); // 1. 将线程变量指回“上一级”
            stored = null;     // 2. 断开引用，帮助 GC
        }
        return LOCAL.get();
    }
    ```
    *   **场景**：`selectUser` 方法执行结束。
    *   **动作**：从 C2 中取出 `stored` (即 C1)，把线程变量重新指向 C1。
    *   **结果**：恢复到外层上下文。如果此时发生错误，打印的是 C1 的状态（或者如果在 C2 期间发生了异常，通常在 catch 块中会先打印 C2 的完整信息再 recall）。

    > **注意**：这里的 `stored` 字段实际上构成了一个单向链表。`store` 是压栈（Push），`recall` 是出栈（Pop）。这种设计避免了引入额外的 `Stack<ErrorContext>` 对象，利用对象自身的引用关系实现了栈结构，内存开销极小。

#### 3. 构建者模式 (Builder Pattern) 的变体
```java
public ErrorContext sql(String sql) {
    this.sql = sql;
    return this; // 返回自身，支持链式调用
}
```
*   **原理**：所有设置属性的方法都返回 `this`。
*   **目的**：允许流畅的代码风格，如：
    ```java
    ErrorContext.instance()
        .activity("executing update")
        .resource("UserMapper.xml")
        .sql(updateSql)
        .cause(e);
    ```
    这使得在复杂的业务逻辑中插入错误信息采集代码非常简洁，不会打断主逻辑流。

#### 4. 异常信息的组装策略 (`toString`)
*   **防御性编程**：所有的 `if (field != null)` 检查确保即使只记录了部分信息，也不会报空指针异常，且输出依然有意义。
*   **清洗数据**：
    ```java
    sql.replace('\n', ' ').replace('\r', ' ')...
    ```
    SQL 语句通常包含换行和缩进以便阅读代码，但在错误日志中，多行文本会打乱日志格式。这里将其压缩为单行，便于日志系统（如 ELK、Splunk）进行索引和展示。
*   **结构化输出**：使用 `###` 作为前缀，不仅醒目，也方便编写脚本正则提取关键信息。

#### 5. 资源清理 (`reset`)
```java
public ErrorContext reset() {
    // ... 清空字段 ...
    LOCAL.remove(); // 关键！
    return this;
}
```
*   **原理**：`ThreadLocal.remove()`。
*   **为什么重要？**
    *   在线程池环境中（如 Web 容器），线程会被复用。
    *   如果一个请求处理完没有调用 `reset()` (或 `remove()`)，`ThreadLocal` 中残留的 `ErrorContext` 对象及其引用的 `stored` 链表可能不会被垃圾回收（取决于 ThreadLocal 的实现细节和强/弱引用，但在 Tomcat 等容器中极易导致 **内存泄漏**）。
    *   此外，如果不 remove，下一个复用该线程的请求可能会读到上一个请求残留的旧数据（虽然 `store` 会覆盖，但 `instance().xxx()` 直接调用可能会有风险）。
    *   MyBatis 通常在 `SqlSession` 关闭或事务结束时调用此方法。

---

### 三、总结：设计模式与架构思想

| 特性 | 对应设计思想/模式 | 作用 |
| :--- | :--- | :--- |
| **ThreadLocal** | 线程封闭 (Thread Confinement) | 实现多线程下的状态隔离，避免锁竞争，保证高性能和安全。 |
| **store / recall** | 栈 (Stack) / 备忘录 (Memento) | 管理嵌套作用域，保存/恢复现场，实现类似函数调用栈的错误上下文追踪。 |
| **链式 setter** | 构建者 (Builder) | 提高代码可读性和编写效率，便于动态组装错误信息。 |
| **toString** | 模板方法 (Template Method) 思想 | 定义标准的错误报告格式，屏蔽底层数据结构的差异。 |

**一句话总结**：
`ErrorContext` 是一个基于 `ThreadLocal` 实现的、轻量级的、支持嵌套作用域的**线程私有错误信息收集器**，它通过手动维护引用链模拟栈行为，将分散在各层的调试信息串联起来，极大地提升了分布式或复杂系统中排查 SQL 和映射错误的效率。



[目录](#目录)

## 2

### 1. 核心需求理解
MyBatis 核心的 `ErrorContext` 类——这个类是 MyBatis 专门用于**跟踪和格式化错误信息**的上下文工具，
<span style="color:#ff6600; font-weight:bold;">采用 ThreadLocal 保证线程隔离</span>，通过**链式调用**收集错误相关的关键信息（资源、操作、SQL、异常原因等），最终生成人性化的错误描述，是 MyBatis 错误排查的核心辅助类。

### 2. 详细代码解释
#### 2.1 类的整体定位与核心设计
```java
public class ErrorContext {
  // 线程本地存储：每个线程独立的 ErrorContext 实例
  private static final ThreadLocal<ErrorContext> LOCAL = ThreadLocal.withInitial(ErrorContext::new);
  // 错误上下文的存储/回滚引用
  private ErrorContext stored;
  // 错误相关的核心属性
  private String resource; // 出错的资源（如 mapper.xml 文件路径）
  private String activity; // 出错时执行的操作（如 "parsing Mapper XML"、"executing query"）
  private String object;   // 出错关联的对象（如 "com.example.UserMapper.selectById"）
  private String message;  // 错误核心描述
  private String sql;      // 出错的 SQL 语句（格式化后）
  private Throwable cause; // 根异常（原始异常）

  private ErrorContext() {} // 私有构造：仅通过 ThreadLocal 创建
  // ... 核心方法
}
```
- **核心定位**：
    - MyBatis 错误信息的“上下文容器”：收集错误发生时的关键上下文（资源、操作、SQL、异常），避免仅抛出原始异常导致的信息缺失；
    - 线程安全设计：通过 `ThreadLocal` 保证每个线程有独立的 `ErrorContext` 实例，防止多线程环境下错误信息污染；
    - 链式调用设计：所有属性设置方法返回 `this`，支持流式构建错误上下文，代码更简洁。
- **设计目的**：
  解决原始异常“信息碎片化”问题——比如执行 SQL 失败时，不仅抛出 `SQLException`，还能关联到对应的 mapper 文件、执行的 SQL 语句、具体操作，让开发者快速定位错误根源。

#### 2.2 核心静态方法：获取线程独有的实例
```java
public static ErrorContext instance() {
  return LOCAL.get();
}
```
- **核心逻辑**：通过 `ThreadLocal` 获取当前线程的 `ErrorContext` 实例（`ThreadLocal.withInitial(ErrorContext::new)` 保证首次调用时创建新实例）；
- **线程隔离**：每个线程的 `ErrorContext` 完全独立，比如线程 A 执行查询出错的信息，不会被线程 B 的操作覆盖；
- **使用场景**：MyBatis 内部所有错误收集都通过 `ErrorContext.instance()` 获取实例，比如 XML 解析失败、SQL 执行异常时。

#### 2.3 上下文的存储与回滚：`store()` & `recall()`
这两个方法用于**嵌套操作中的错误上下文管理**（比如解析 XML 时嵌套解析子节点，出错后回滚到上层上下文）。
##### （1）`store()`：存储当前上下文，创建新上下文
```java
public ErrorContext store() {
  ErrorContext newContext = new ErrorContext();
  newContext.stored = this; // 新上下文关联当前上下文（作为存储的旧上下文）
  LOCAL.set(newContext);    // 替换 ThreadLocal 中的实例为新上下文
  return LOCAL.get();
}
```
- **核心作用**：保存当前线程的错误上下文，创建新的空上下文用于后续操作，避免嵌套操作的错误信息覆盖原有上下文；
- **使用场景**：MyBatis 解析 mapper.xml 时，解析每个 SQL 节点前会调用 `store()`，解析完成后调用 `recall()` 回滚，保证单个 SQL 节点的错误不影响整个 mapper 的解析上下文。

##### （2）`recall()`：回滚到存储的上下文
```java
public ErrorContext recall() {
  if (stored != null) {
    LOCAL.set(stored); // 恢复为存储的旧上下文
    stored = null;     // 清空存储引用
  }
  return LOCAL.get();
}
```
- **核心作用**：将线程的 `ErrorContext` 恢复为 `store()` 前的实例，完成嵌套操作的上下文回滚；
- **示例流程**：
  ```
  // 初始上下文：ctx1
  ErrorContext ctx1 = ErrorContext.instance();
  ctx1.resource("UserMapper.xml");

  // 存储 ctx1，创建新上下文 ctx2
  ErrorContext ctx2 = ctx1.store();
  ctx2.activity("parsing selectById");

  // 回滚：恢复为 ctx1，ctx2 被丢弃
  ErrorContext ctx3 = ctx2.recall(); // ctx3 == ctx1
  ```

#### 2.4 链式属性设置方法
所有属性设置方法都采用**链式调用**（返回 `this`），支持流式收集错误信息，核心属性含义如下：

| 方法 | 参数 | 核心含义 | 典型值 |
|------|------|----------|--------|
| `resource(String resource)` | 资源路径 | 出错的资源文件 | `com/example/mapper/UserMapper.xml` |
| `activity(String activity)` | 操作描述 | 出错时执行的操作 | `parsing Mapper XML`、`executing update`、`opening sql session` |
| `object(String object)` | 关联对象 | 出错关联的 Mapper 方法/节点 | `com.example.UserMapper.selectById`、`updateUser` |
| `message(String message)` | 错误描述 | 核心错误信息 | `Error parsing SQL Mapper Configuration` |
| `sql(String sql)` | SQL 语句 | 出错的 SQL（格式化后） | `SELECT * FROM user WHERE id = ?` |
| `cause(Throwable cause)` | 根异常 | 原始异常（如 SQLException） | `new SQLException("Table 'user' not found")` |

**链式调用示例**：
```java
ErrorContext.instance()
  .resource("UserMapper.xml")
  .activity("executing selectById")
  .object("UserMapper.selectById")
  .message("Query failed")
  .sql("SELECT * FROM user WHERE id = 1")
  .cause(new SQLException("Table 'user' does not exist"));
```

#### 2.5 重置上下文：`reset()`
```java
public ErrorContext reset() {
  // 清空所有错误属性
  resource = null;
  activity = null;
  object = null;
  message = null;
  sql = null;
  cause = null;
  // 移除 ThreadLocal 中的实例（下次 get() 会创建新实例）
  LOCAL.remove();
  return this;
}
```
- **核心作用**：清空当前线程的错误上下文，移除 ThreadLocal 中的实例，避免错误信息残留；
- **使用场景**：MyBatis 关键操作完成后（如创建 SqlSession、执行 SQL 后），会调用 `reset()` 重置，比如 `DefaultSqlSessionFactory` 的 `openSessionFromDataSource` 方法的 `finally` 块中：
  ```java
  finally {
    ErrorContext.instance().reset();
  }
  ```

#### 2.6 格式化错误信息：`toString()`
```java
@Override
public String toString() {
  StringBuilder description = new StringBuilder();
  // 1. 核心错误消息
  if (this.message != null) {
    description.append(LINE_SEPARATOR).append("### ").append(this.message);
  }
  // 2. 出错资源
  if (resource != null) {
    description.append(LINE_SEPARATOR).append("### The error may exist in ").append(resource);
  }
  // 3. 关联对象
  if (object != null) {
    description.append(LINE_SEPARATOR).append("### The error may involve ").append(object);
  }
  // 4. 执行操作
  if (activity != null) {
    description.append(LINE_SEPARATOR).append("### The error occurred while ").append(activity);
  }
  // 5. SQL 语句（格式化：移除换行/制表符）
  if (sql != null) {
    description.append(LINE_SEPARATOR).append("### SQL: ").append(sql.replace('\n', ' ').replace('\r', ' ').replace('\t', ' ').trim());
  }
  // 6. 根异常
  if (cause != null) {
    description.append(LINE_SEPARATOR).append("### Cause: ").append(cause.toString());
  }
  return description.toString();
}
```
- **核心逻辑**：按“消息→资源→关联对象→操作→SQL→根异常”的顺序拼接错误信息，生成人性化的错误描述；
- **SQL 格式化**：移除换行、回车、制表符，保证 SQL 语句在错误信息中是单行，便于查看；
- **典型输出示例**：
  ```
  ### Query failed
  ### The error may exist in com/example/mapper/UserMapper.xml
  ### The error may involve com.example.UserMapper.selectById
  ### The error occurred while executing selectById
  ### SQL: SELECT * FROM user WHERE id = 1
  ### Cause: java.sql.SQLException: Table 'test.user' doesn't exist
  ```

#### 2.7 典型使用场景（MyBatis 内部）
MyBatis 在关键操作出错时，都会通过 `ErrorContext` 收集信息，比如：
1. **XML 解析失败**：
   ```java
   try {
     // 解析 mapper.xml
   } catch (Exception e) {
     throw new BuilderException(ErrorContext.instance()
             .resource(resource)
             .activity("parsing Mapper XML")
             .object(mapperId)
             .message("Error parsing Mapper XML")
             .cause(e)
             .toString(), e);
   }
   ```
2. **SQL 执行失败**：
   ```java
   try {
     // 执行 SQL
   } catch (SQLException e) {
     throw new PersistenceException(ErrorContext.instance()
             .resource(mapperId)
             .activity("executing query")
             .object(methodId)
             .sql(sql)
             .message("Error querying database")
             .cause(e)
             .toString(), e);
   }
   ```

### 3. 总结
#### 核心关键点
1. **类的核心定位**：
   MyBatis 错误信息的上下文管理器，收集错误相关的资源、操作、SQL、异常等信息，生成人性化的错误描述，是错误排查的核心工具。

2. **核心设计亮点**：
    - **ThreadLocal 隔离**：每个线程独立的错误上下文，避免多线程信息污染；
    - **链式调用**：流式收集错误信息，代码简洁易读；
    - **嵌套上下文管理**：`store()`/`recall()` 支持嵌套操作的上下文存储/回滚，保证错误信息精准；
    - **格式化输出**：`toString()` 按逻辑顺序拼接信息，生成易读的错误描述。

3. **关键方法作用**：
    - `instance()`：获取线程独有的实例；
    - `store()`/`recall()`：管理嵌套操作的上下文；
    - 链式属性方法：收集错误上下文信息；
    - `reset()`：清空上下文，避免信息残留；
    - `toString()`：格式化输出错误信息。

4. **错误排查价值**：
   MyBatis 抛出的异常（如 `BuilderException`、`PersistenceException`）都会包含 `ErrorContext` 生成的字符串，通过这些信息可快速定位：
    - 错误发生在哪个 mapper.xml 文件；
    - 执行的是哪个 Mapper 方法/操作；
    - 对应的 SQL 语句是什么；
    - 原始异常原因是什么。

#### 应用价值
- **错误排查**：理解 `ErrorContext` 的属性和输出格式，能快速从 MyBatis 异常信息中定位问题根源（如 SQL 语法错、mapper 文件路径错、表不存在）；
- **自定义扩展**：开发 MyBatis 插件/自定义组件时，可通过 `ErrorContext` 收集自定义错误信息，保持错误格式的一致性；
- **线程安全理解**：通过 `ThreadLocal` 的使用，加深对 MyBatis 线程隔离设计的理解（如 SqlSession 也是线程私有的）。
