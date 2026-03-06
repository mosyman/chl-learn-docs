

## 目录
- [1](#1)
- [2](#2)





# 1

`DefaultCursor<T>` 是 MyBatis 框架中用于实现**流式查询（Streaming Query）**的核心组件。它的存在是为了解决传统查询中“一次性将全部结果加载到内存”导致的 OOM（内存溢出）问题，允许应用程序像处理本地集合一样，逐行处理数据库返回的海量数据。

以下是对该类的**详细功能解释**及其**底层实现原理**的深度剖析。

---

### 一、核心功能与设计目标

#### 1. 什么是 Cursor（游标）？
在数据库层面，Cursor 是一种机制，允许客户端逐行检索查询结果集，而不需要一次性获取所有行。
在 MyBatis 层面，`DefaultCursor` 将 JDBC 的 `ResultSet` 包装成了一个 Java `Iterable`/`Iterator`。
*   **传统模式**：`List<User> users = mapper.selectAll();` -> 内存中瞬间生成所有 User 对象，大数据量必挂。
*   **Cursor 模式**：`try (Cursor<User> users = mapper.selectCursor()) { for (User u : users) { ... } }` -> 每次循环只从数据库拉取一行数据，处理完即丢弃，内存占用恒定。

#### 2. 关键特性
*   **懒加载（Lazy Loading）**：只有在调用 `iterator().next()` 或 `hasNext()` 时，才会真正去数据库驱动层读取下一行数据。
*   **资源管理**：实现了 `Closeable` 语义（虽然代码中未直接显示 implements Closeable，但通过 `close()` 方法管理），确保 `ResultSet` 和数据库连接在使用完毕后能被及时释放。
*   **状态机管理**：严格管理生命周期（CREATED -> OPEN -> CONSUMED/CLOSED），防止非法操作（如重复打开迭代器、在关闭后读取）。
*   **RowBounds 支持**：支持逻辑上的分页（Offset/Limit），但在流式场景下，Offset 意味着要“空跑”丢弃前 N 条记录。

---

### 二、底层原理深度剖析

#### 1. 核心架构：适配器模式 + 状态机
`DefaultCursor` 本质上是一个**适配器**，它将 `JDBC 的推模型`（Push Model，`ResultSetHandler` 主动回调）转换成了 `Java 迭代器的拉模型`（Pull Model，用户主动 `next()`）。

##### A. 状态机 (`CursorStatus`)
类内部维护了一个枚举状态，确保操作的合法性：
*   **CREATED**: 刚实例化，尚未接触数据库。
*   **OPEN**: 正在遍历中，`ResultSet` 处于打开状态。
*   **CLOSED**: 用户显式调用 `close()` 或发生异常提前关闭，但未遍历完。
*   **CONSUMED**: 自然遍历结束（所有数据读完），此时自动关闭。

**原理作用**：
*   防止重复创建迭代器：`iteratorRetrieved` 标志位确保一个 Cursor 只能被遍历一次（因为 `ResultSet` 的指针是单向流动的，无法重置）。
*   防止资源泄露：一旦状态变为 CLOSED/CONSUMED，再次调用 `next()` 会直接返回 null 或抛错，并触发 `rs.close()`。

##### B. 数据拉取机制：从“推”到“拉”的转换
这是最精彩的部分。MyBatis 的核心处理逻辑 `ResultSetHandler.handleRowValues` 通常是一个**推模型**：它遍历 `ResultSet`，每读一行就调用 `ResultHandler.handleResult`。

为了适配 `Iterator` 的**拉模型**，`DefaultCursor` 使用了巧妙的**阻塞/信号量机制**：

1.  **自定义 ResultHandler (`ObjectWrapperResultHandler`)**：
    ```java
    protected static class ObjectWrapperResultHandler<T> implements ResultHandler<T> {
        protected T result;
        protected boolean fetched; // 信号旗

        @Override
        public void handleResult(ResultContext<? extends T> context) {
            this.result = context.getResultObject(); // 暂存结果
            context.stop(); // 【关键】立即停止 ResultSetHandler 的后续遍历！
            fetched = true; // 竖起信号旗：我拿到数据了
        }
    }
    ```
    *   **原理**：当 `handleRowValues` 被调用时，它试图循环读取所有行。但我们的 Handler 在读取**第一行**后，立即调用 `context.stop()`。这告诉 `ResultSetHandler`：“停！我只想要这一行，剩下的下次再说”。

2.  **`fetchNextObjectFromDatabase` 流程**：
    *   重置信号旗 `fetched = false`。
    *   调用 `resultSetHandler.handleRowValues(..., objectWrapperResultHandler, ...)`。
    *   `handleRowValues` 启动，读取 JDBC `ResultSet` 的下一行。
    *   触发 `objectWrapperResultHandler.handleResult`。
    *   Handler 保存该行数据到 `result` 字段，设置 `fetched = true`，并调用 `context.stop()` 强行中断 `handleRowValues` 的循环。
    *   控制权回到 `fetchNextObjectFromDatabase`，检查 `fetched`，如果为 true，则返回 `result`。

**结论**：通过这种“调用一次处理逻辑 -> 强制中断 -> 返回单行”的机制，实现了**按需逐行读取**。

#### 2. 迭代器实现 (`CursorIterator`)
内部类 `CursorIterator` 实现了标准的 `java.util.Iterator` 接口。

*   **`hasNext()` 的预取逻辑**：
    ```java
    @Override
    public boolean hasNext() {
        if (!objectWrapperResultHandler.fetched) {
            // 如果还没取到下一行，尝试去数据库拉取
            object = fetchNextUsingRowBound();
        }
        return objectWrapperResultHandler.fetched;
    }
    ```
    *   **原理**：`hasNext()` 不仅仅是检查状态，它实际上承担了**IO 操作**。如果当前缓冲区没有数据，它会触发数据库读取。这意味着 `hasNext()` 可能会阻塞（等待网络 IO）。

*   **`next()` 的消费逻辑**：
    *   如果 `hasNext()` 没被调用过，`next()` 会先调用 `fetchNextUsingRowBound()`。
    *   如果数据有效，清空 `fetched` 标志（为下一次 `hasNext` 做准备），递增索引，返回暂存的 `object`。
    *   **注意**：数据在 `objectWrapperResultHandler` 和 `CursorIterator.object` 之间传递，确保每次只持有一个对象的引用，利于 GC。

#### 3. RowBounds (逻辑分页) 的实现代价
`fetchNextUsingRowBound` 方法处理了 `Offset` 和 `Limit`：
```java
protected T fetchNextUsingRowBound() {
    T result = fetchNextObjectFromDatabase();
    // 如果还没达到 Offset，就一直读并丢弃
    while (objectWrapperResultHandler.fetched && indexWithRowBound < rowBounds.getOffset()) {
        result = fetchNextObjectFromDatabase();
    }
    return result;
}
```
*   **原理**：JDBC 的流式游标通常不支持服务端的光标跳转（Seek）。因此，MyBatis 的实现方式是**客户端空转**。
*   **性能警示**：如果你设置 `Offset=10000`，MyBatis 会从数据库逐行读取 10000 条数据，在 Java 端解析成对象，然后**立即丢弃**，直到第 10001 条才返回给用户。
*   **结论**：在流式查询（Cursor）中使用大的 `Offset` 是极其低效的，应避免。`Limit` 则是正常的，读到指定数量后会自动 `close()`。

#### 4. 线程安全性
*   **明确声明**：`This implementation is not thread safe.`
*   **原因**：
    1.  底层的 JDBC `ResultSet` 通常不是线程安全的。
    2.  状态变量 (`status`, `fetched`, `object`) 没有使用 `volatile` 或锁保护。
    3.  游标是单向的，多线程同时调用 `next()` 会导致状态混乱（例如两个线程都读到了同一行，或者跳过了某行）。
*   **最佳实践**：Cursor 必须在单个线程中使用，且通常配合 try-with-resources 语句块确保关闭。

#### 5. 资源关闭机制
```java
@Override
public void close() {
    // ...
    ResultSet rs = rsw.getResultSet();
    if (rs != null) rs.close(); // 物理关闭 JDBC 资源
    status = CursorStatus.CLOSED;
}
```
*   **自动关闭**：当 `getReadItemsCount() == offset + limit` 时（即读完 Limit 指定的数量），或者 `ResultSet` 遍历到底（`fetched` 为 false），状态会自动变为 `CONSUMED` 并触发 `close()`。
*   **手动关闭**：如果用户在遍历中途 `break` 循环，必须显式调用 `cursor.close()`，否则数据库连接和 ResultSet 会一直持有，导致连接池耗尽。

---

### 三、总结与使用场景

#### 1. 工作流程图解
```text
用户调用 iterator().next()
      ↓
CursorIterator.next()
      ↓
检查是否有缓存数据？(No) -> 调用 fetchNextUsingRowBound()
      ↓
循环检查是否跳过 Offset? (Yes: 丢弃数据继续读)
      ↓
调用 fetchNextObjectFromDatabase()
      ↓
调用 resultSetHandler.handleRowValues()
      ↓
JDBC ResultSet.next() (读取一行)
      ↓
ObjectWrapperResultHandler.handleResult()
      ↓
保存结果到 result 字段
      ↓
调用 context.stop() [阻断 handleRowValues 的循环]
      ↓
返回结果给 Iterator
      ↓
返回给用户
```

#### 2. 核心优势
*   **内存效率**：O(1) 内存复杂度，无论查询 100 万还是 1000 万行，JVM 堆内存占用几乎不变。
*   **响应速度**：第一条数据返回极快，无需等待全量查询结束。

#### 3. 潜在陷阱
*   **长事务**：只要 Cursor 没关闭，数据库事务就一直持有，可能导致数据库锁竞争或 Undo Log 膨胀。务必尽快遍历并关闭。
*   **连接占用**：遍历期间，该数据库连接一直被占用，无法归还连接池。高并发下需谨慎。
*   **Offset 性能**：严禁在流式查询中使用大 Offset 分页。
*   **单次消费**：Iterator 只能遍历一次，不能复用。

`DefaultCursor` 是 MyBatis 对 JDBC 标准游标的高级封装，通过**控制反转（IoC）**和**短路执行**技巧，完美解决了大数据量查询的内存瓶颈问题。


[目录](#目录)


# 2

你希望深入理解 MyBatis 中 `DefaultCursor<T>` 这个核心类的实现细节、底层原理，以及它作为游标（Cursor）的设计思想和工作机制。我会从类的定位、核心成员、核心方法、底层原理、设计亮点五个维度，结合 MyBatis 底层执行流程全面讲解。

### 一、`DefaultCursor` 整体定位
`DefaultCursor<T>` 是 MyBatis 对 `Cursor<T>` 接口的默认实现，核心使命是：
- 提供**按需加载**的结果集遍历能力（区别于 `List` 一次性加载所有结果），解决大数据量查询时的内存溢出问题；
- 基于 JDBC `ResultSet` 实现“懒加载”——遍历到某个元素时才从数据库读取对应行数据，而非一次性将所有结果加载到内存；
- 本质是对 JDBC `ResultSet` 的封装，结合 MyBatis 的结果映射逻辑，对外暴露标准的 `Iterator` 遍历接口。

**核心特点**：
- 非线程安全（类注释明确说明）；
- 只能创建一个迭代器（避免多迭代器同时操作 `ResultSet`）；
- 状态驱动（通过 `CursorStatus` 管理生命周期）；
- 支持 `RowBounds`（分页/偏移量），但底层是“流式”处理而非一次性分页。

---

### 二、核心成员变量解析
先理解类的核心成员，才能看懂后续的方法逻辑：

| 成员变量 | 类型 | 核心作用 |
|----------|------|----------|
| `resultSetHandler` | `DefaultResultSetHandler` | MyBatis 核心组件，负责将 JDBC `ResultSet` 行数据映射为 Java 对象（结果映射的核心） |
| `resultMap` | `ResultMap` | 结果映射规则（字段与实体类属性的映射关系） |
| `rsw` | `ResultSetWrapper` | MyBatis 对 JDBC `ResultSet` 的包装类，简化结果集操作 |
| `rowBounds` | `RowBounds` | 分页/偏移量参数（offset 起始位置，limit 最大条数） |
| `objectWrapperResultHandler` | `ObjectWrapperResultHandler<T>` | 自定义 `ResultHandler`，仅处理单行数据并停止（核心：每次只映射一行） |
| `cursorIterator` | `CursorIterator` | 内部迭代器实现，封装 `ResultSet` 遍历逻辑 |
| `iteratorRetrieved` | `boolean` | 标记是否已创建迭代器，确保“单迭代器”约束 |
| `status` | `CursorStatus` | 游标状态（CREATED/OPEN/CLOSED/CONSUMED），管理生命周期 |
| `indexWithRowBound` | `int` | 基于 `RowBounds` 的内部索引（记录已读取的行数，包含 offset） |

#### 关键内部枚举：`CursorStatus`（生命周期状态）
| 状态 | 含义 | 核心行为 |
|------|------|----------|
| `CREATED` | 初始创建 | 未开始读取 `ResultSet`，游标未激活 |
| `OPEN` | 已打开 | 正在读取 `ResultSet`，迭代器可用 |
| `CLOSED` | 已关闭 | 未完全消费结果集，但主动关闭（如调用 `close()`） |
| `CONSUMED` | 已消费 | 结果集完全遍历完毕，自动关闭 |

---

### 三、核心方法拆解 + 底层原理
#### 1. 构造方法：`DefaultCursor(...)`
```java
public DefaultCursor(DefaultResultSetHandler resultSetHandler, ResultMap resultMap, ResultSetWrapper rsw, RowBounds rowBounds) {
  this.resultSetHandler = resultSetHandler;
  this.resultMap = resultMap;
  this.rsw = rsw;
  this.rowBounds = rowBounds;
}
```
- **作用**：初始化核心依赖，将 MyBatis 结果处理的核心组件注入游标；
- **底层逻辑**：`rsw` 封装了数据库返回的 `ResultSet`，`rowBounds` 定义了遍历的范围（如从第 10 行开始，最多读 100 行），为后续“按需读取”做准备。

#### 2. 状态判断方法：`isOpen()`/`isConsumed()`/`isClosed()`
```java
@Override
public boolean isOpen() {
  return status == CursorStatus.OPEN;
}

@Override
public boolean isConsumed() {
  return status == CursorStatus.CONSUMED;
}

private boolean isClosed() {
  return status == CursorStatus.CLOSED || status == CursorStatus.CONSUMED;
}
```
- **作用**：基于 `status` 枚举判断游标当前状态，是“状态驱动”设计的体现；
- **底层逻辑**：确保游标操作（如读取、关闭）符合生命周期约束（如已关闭的游标不能再读取）。

#### 3. 核心入口：`iterator()`（获取迭代器）
```java
@Override
public Iterator<T> iterator() {
  if (iteratorRetrieved) {
    throw new IllegalStateException("Cannot open more than one iterator on a Cursor");
  }
  if (isClosed()) {
    throw new IllegalStateException("A Cursor is already closed.");
  }
  iteratorRetrieved = true;
  return cursorIterator;
}
```
- **核心约束**：
    1. `iteratorRetrieved` 确保**只能创建一个迭代器**——因为 `ResultSet` 是单向、顺序读取的，多迭代器会导致读取位置混乱；
    2. 已关闭的游标不能创建迭代器；
- **底层逻辑**：返回内部的 `CursorIterator`，所有遍历操作最终由这个迭代器完成。

#### 4. 资源释放：`close()`
```java
@Override
public void close() {
  if (isClosed()) {
    return;
  }

  ResultSet rs = rsw.getResultSet();
  try {
    if (rs != null) {
      rs.close(); // 关闭JDBC ResultSet，释放数据库连接资源
    }
  } catch (SQLException e) {
    // ignore：MyBatis 风格，关闭资源时忽略异常（避免上层感知底层JDBC异常）
  } finally {
    status = CursorStatus.CLOSED; // 最终标记为关闭状态
  }
}
```
- **作用**：关闭底层的 `ResultSet`，释放数据库资源（核心：避免连接泄漏）；
- **底层逻辑**：
    1. 幂等设计（已关闭则直接返回）；
    2. 优先关闭 `ResultSet`（JDBC 资源必须显式关闭）；
    3. 无论是否抛异常，最终标记状态为 `CLOSED`。

#### 5. 核心读取逻辑：`fetchNextObjectFromDatabase()`（从数据库读取下一个对象）
这是游标“按需加载”的核心方法，逐行读取 `ResultSet` 并映射为 Java 对象：
```java
protected T fetchNextObjectFromDatabase() {
  if (isClosed()) {
    return null; // 已关闭则返回null
  }

  try {
    objectWrapperResultHandler.fetched = false; // 重置读取标记
    status = CursorStatus.OPEN; // 标记为已打开
    if (!rsw.getResultSet().isClosed()) {
      // 核心：调用ResultSetHandler处理单行数据（关键：RowBounds.DEFAULT + stop()）
      resultSetHandler.handleRowValues(rsw, resultMap, objectWrapperResultHandler, RowBounds.DEFAULT, null);
    }
  } catch (SQLException e) {
    throw new RuntimeException(e); // 包装为运行时异常（MyBatis 统一异常体系）
  }

  T next = objectWrapperResultHandler.result; // 获取映射后的Java对象
  if (objectWrapperResultHandler.fetched) {
    indexWithRowBound++; // 已读取行数+1
  }
  // 终止条件：1. 无更多数据 2. 达到RowBounds的limit限制
  if (!objectWrapperResultHandler.fetched || getReadItemsCount() == rowBounds.getOffset() + rowBounds.getLimit()) {
    close(); // 关闭游标
    status = CursorStatus.CONSUMED; // 标记为已消费
  }
  objectWrapperResultHandler.result = null; // 清空缓存

  return next;
}
```
- **核心原理**：
    1. **单行映射**：`objectWrapperResultHandler` 的 `handleResult` 方法会在映射一行数据后调用 `context.stop()`，强制 `resultSetHandler` 只处理一行，实现“按需读取”；
    2. **终止条件**：要么 `ResultSet` 已无数据（`fetched=false`），要么达到 `RowBounds` 的总条数（offset+limit），此时关闭游标并标记为 `CONSUMED`；
    3. **状态更新**：读取数据前将状态设为 `OPEN`，确保 `isOpen()` 返回正确。

#### 6. 分页适配：`fetchNextUsingRowBound()`（处理偏移量）
```java
protected T fetchNextUsingRowBound() {
  T result = fetchNextObjectFromDatabase();
  // 跳过offset之前的行：如果已读取行数 < offset，继续读取直到达到offset
  while (objectWrapperResultHandler.fetched && indexWithRowBound < rowBounds.getOffset()) {
    result = fetchNextObjectFromDatabase();
  }
  return result;
}
```
- **作用**：适配 `RowBounds` 的 `offset`（起始偏移量），跳过前 N 行数据；
- **底层逻辑**：比如 `offset=10`，则先循环读取 10 行并丢弃，直到 `indexWithRowBound >= 10`，才返回真正需要的行——这是“流式分页”，区别于 SQL 分页（`LIMIT offset, limit`）。

#### 7. 内部迭代器：`CursorIterator`（核心遍历实现）
这是 `Iterator` 的具体实现，封装了“按需读取”的逻辑：
```java
protected class CursorIterator implements Iterator<T> {
  T object; // 缓存下一个要返回的对象
  int iteratorIndex = -1; // 用户可见的索引（从0开始，不含offset）

  @Override
  public boolean hasNext() {
    if (!objectWrapperResultHandler.fetched) {
      object = fetchNextUsingRowBound(); // 预读取下一个对象
    }
    return objectWrapperResultHandler.fetched; // 返回是否有下一个数据
  }

  @Override
  public T next() {
    T next = object; // 先取缓存的对象

    if (!objectWrapperResultHandler.fetched) {
      next = fetchNextUsingRowBound(); // 未预读取则主动读取
    }

    if (objectWrapperResultHandler.fetched) {
      objectWrapperResultHandler.fetched = false; // 重置标记
      object = null; // 清空缓存
      iteratorIndex++; // 用户索引+1
      return next;
    }
    throw new NoSuchElementException(); // 无数据则抛异常
  }

  @Override
  public void remove() {
    throw new UnsupportedOperationException("Cannot remove element from Cursor");
  }
}
```
- **核心原理**：
    1. **预读取优化**：`hasNext()` 会提前读取下一个对象并缓存到 `object`，避免 `next()` 重复读取；
    2. **索引隔离**：`iteratorIndex` 是用户可见的索引（从0开始），而 `indexWithRowBound` 是包含 `offset` 的内部索引，两者分离让用户无需关心偏移量；
    3. **删除禁用**：`remove()` 直接抛异常——因为游标是基于 `ResultSet` 的单向读取，无法删除数据库中的行，也无法修改已读取的结果。

#### 8. 辅助类：`ObjectWrapperResultHandler`（单行结果处理器）
```java
protected static class ObjectWrapperResultHandler<T> implements ResultHandler<T> {
  protected T result; // 存储单行映射结果
  protected boolean fetched; // 标记是否读取到数据

  @Override
  public void handleResult(ResultContext<? extends T> context) {
    this.result = context.getResultObject(); // 获取映射后的对象
    context.stop(); // 强制停止处理，只处理当前行
    fetched = true; // 标记已读取到数据
  }
}
```
- **核心作用**：MyBatis 的 `ResultHandler` 通常处理多行数据，而这个实现只处理一行并停止，是“按需读取”的关键——确保每次调用 `resultSetHandler.handleRowValues()` 只映射一行。

---

### 四、底层核心原理总结
#### 1. 按需加载（懒加载）的实现逻辑
```mermaid
graph TD
    A[调用iterator()] --> B[返回CursorIterator]
    B --> C[调用hasNext()]
    C --> D{是否已预读取?}
    D -- 否 --> E[调用fetchNextUsingRowBound()]
    E --> F[调用fetchNextObjectFromDatabase()]
    F --> G[ResultSetHandler映射单行数据]
    G --> H[返回映射后的对象并缓存]
    D -- 是 --> I[返回是否有数据]
    I --> J[调用next()]
    J --> K[返回缓存的对象并清空]
```
- 核心：**只有调用 `hasNext()`/`next()` 时，才会从 `ResultSet` 读取一行并映射**，而非一次性加载所有数据，极大节省内存。

#### 2. 与 `List` 结果的核心区别
| 特性 | `DefaultCursor`（游标） | `List`（列表） |
|------|-------------------------|----------------|
| 加载方式 | 按需加载（流式），读一行加载一行 | 一次性加载所有结果到内存 |
| 内存占用 | 低（仅缓存当前行） | 高（所有结果常驻内存） |
| 适用场景 | 大数据量查询（百万级） | 小数据量查询 |
| 遍历特性 | 只能单向遍历，只能创建一个迭代器 | 可随机访问，多次遍历 |
| 资源占用 | 遍历过程中保持 `ResultSet`/数据库连接打开 | 加载完成后关闭 `ResultSet`/连接 |

#### 3. 线程安全问题
类注释明确说明“非线程安全”，原因：
- `ResultSet` 是非线程安全的，多线程同时调用 `next()` 会导致读取位置混乱；
- 核心状态变量（`status`、`indexWithRowBound`、`iteratorRetrieved`）无并发控制；
- `CursorIterator` 的 `object` 缓存是单线程设计。

#### 4. 资源管理逻辑
- 游标未关闭时，`ResultSet` 和数据库连接保持打开状态；
- 遍历完成（`CONSUMED`）或主动调用 `close()` 时，关闭 `ResultSet`；
- MyBatis 会在事务提交/回滚时，确保游标被关闭（避免连接泄漏）。

---

### 五、关键设计亮点
1. **状态驱动的生命周期管理**：通过 `CursorStatus` 枚举清晰管理游标从创建、打开、关闭到消费的全生命周期，避免非法操作；
2. **单迭代器约束**：`iteratorRetrieved` 确保只有一个迭代器操作 `ResultSet`，避免读取位置混乱；
3. **按需加载+分页适配**：结合 `RowBounds` 实现流式分页，既支持大数据量，又兼容 MyBatis 的分页逻辑；
4. **异常封装**：将 `SQLException` 包装为运行时异常，符合 MyBatis 统一的异常体系，简化上层调用；
5. **幂等设计**：`close()` 方法幂等（多次调用无副作用），避免重复关闭资源。

### 总结
1. **核心定位**：`DefaultCursor` 是 MyBatis 对 JDBC `ResultSet` 的流式封装，通过“按需读取单行数据”实现大数据量查询的内存优化，核心是 `fetchNextObjectFromDatabase()` 方法和 `ObjectWrapperResultHandler` 单行映射；
2. **底层原理**：基于迭代器模式，将 `ResultSet` 的单向读取转化为标准的 `Iterator` 接口，通过 `RowBounds` 适配分页，状态枚举管理生命周期，确保资源安全释放；
3. **关键约束**：非线程安全、仅支持单迭代器、遍历过程中保持数据库连接打开，适用于大数据量、低内存占用的查询场景。

掌握这个类的原理，就能理解 MyBatis 为什么推荐在查询百万级数据时使用 `Cursor` 而非 `List`——核心是避免一次性加载所有数据导致的 OOM，同时通过流式读取保证性能。
