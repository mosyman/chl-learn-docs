



```java
/**
 * 游标契约接口，用于通过迭代器（Iterator）实现数据的懒加载获取。
 * 游标非常适合处理那些无法一次性全部加载到内存中的百万级数据查询场景。
 * 若在结果映射（resultMap）中使用了集合类型，则游标对应的 SQL 查询
 * 必须通过 resultMap 的 id 列进行排序（设置 resultOrdered="true"）。
 *
 * @author Guillaume Darmont / guillaume@dropinocean.com
 */
```

### 关键术语说明（便于你看源码）
- **Cursor**：游标（数据库流式查询核心概念）
- **Lazily fetching**：懒加载
- **Iterator**：迭代器
- **resultMap**：MyBatis 结果映射
- **resultOrdered="true"**：MyBatis 游标查询必须开启的排序开关


### 一、接口整体定位与设计背景
这段代码定义了一个名为 `Cursor<T>` 的泛型接口，核心目的是**以“懒加载”（Lazy Fetching）的方式从数据库中分批获取大量数据**，避免一次性加载数百万条数据到内存导致的内存溢出问题。

它继承了两个核心接口：
- `AutoCloseable`：支持 try-with-resources 语法，确保资源（如数据库连接、游标）能自动关闭，避免泄漏；
- `Iterable<T>`：支持增强 for 循环（`for (T item : cursor)`），可以像遍历集合一样遍历游标中的数据。

注释里还特别强调了使用场景和约束：
1. 适合处理“百万级数据查询”（无法一次性放入内存）；
2. 如果结果映射（resultMap）中包含集合类型，对应的 SQL 查询必须通过 `resultOrdered="true"` 按 resultMap 的 ID 列排序（这是 MyBatis 等 ORM 框架使用游标时的关键约束，确保分批读取的数据不重复、不遗漏）。

### 二、逐行/逐方法详细解释
#### 1. 接口定义
```java
public interface Cursor<T> extends AutoCloseable, Iterable<T>
```
- `Cursor<T>`：泛型接口，`T` 表示游标遍历的元素类型（比如数据库表对应的实体类）；
- `extends AutoCloseable`：强制实现类提供 `close()` 方法，用于释放数据库游标、连接等资源；
- `extends Iterable<T>`：让游标具备“可遍历”特性，能通过 `iterator()` 方法获取迭代器，逐个读取数据。

#### 2. `isOpen()` 方法
```java
/**
 * @return true if the cursor has started to fetch items from database.
 */
boolean isOpen();
```
- **作用**：判断游标是否已“打开”（是否开始从数据库获取数据）；
- **返回值**：`true` = 游标已打开（正在/准备读取数据）；`false` = 游标未打开（或已关闭）；
- **场景**：读取数据前先检查游标是否有效，避免操作已关闭的游标。

#### 3. `isConsumed()` 方法
```java
/**
 * @return true if the cursor is fully consumed and has returned all elements matching the query.
 */
boolean isConsumed();
```
- **作用**：判断游标是否已“消费完毕”（是否读取完所有匹配查询条件的数据）；
- **返回值**：`true` = 所有数据已读取完毕；`false` = 还有未读取的数据；
- **场景**：遍历过程中判断是否还能继续获取数据，比如循环读取时的终止条件。

#### 4. `getCurrentIndex()` 方法
```java
/**
 * Get the current item index. The first item has the index 0.
 *
 * @return -1 if the first cursor item has not been retrieved. The index of the current item retrieved.
 */
int getCurrentIndex();
```
- **作用**：获取当前已读取到的元素的索引（位置）；
- **规则**：
    - 第一个元素的索引是 `0`；
    - 若还未读取任何元素，返回 `-1`；
    - 若已读取第 N 个元素（N≥1），返回 `N-1`；
- **场景**：监控读取进度（比如“已读取第 1000 条数据”），或定位读取异常的位置。

#### 5. `close()` 方法
```java
/**
 * Closes the cursor.
 */
@Override
void close();
```
- **作用**：关闭游标，释放底层资源（如数据库游标、JDBC 连接、ResultSet 等）；
- **重写说明**：`@Override` 表示重写 `AutoCloseable` 接口的 `close()` 方法；
- **注意**：即使遍历未完成，调用 `close()` 后游标会立即失效，`isOpen()` 会返回 `false`，且必须确保最终调用（try-with-resources 会自动调用）。

### 三、使用场景示例（伪代码）
为了让你更直观理解，这里给出一个典型的使用示例（基于 MyBatis 等支持 Cursor 的框架）：
```java
// 1. 获取游标（假设 mapper 是 MyBatis 的 Mapper 接口）
try (Cursor<Order> orderCursor = mapper.queryMillionsOfOrders()) {
    // 2. 检查游标是否打开
    if (orderCursor.isOpen()) {
        // 3. 遍历游标（懒加载，逐个读取数据）
        for (Order order : orderCursor) {
            // 处理单条订单数据（无需加载所有订单到内存）
            processOrder(order);
            
            // 打印当前读取进度
            System.out.println("已读取第 " + (orderCursor.getCurrentIndex() + 1) + " 条订单");
        }
    }
    
    // 4. 检查是否消费完毕
    if (orderCursor.isConsumed()) {
        System.out.println("所有订单已读取完毕");
    }
} // try-with-resources 自动调用 close()，释放资源
```

### 总结
1. **核心定位**：`Cursor<T>` 是处理“海量数据库查询”的懒加载接口，避免内存溢出，核心特性是“分批读取、按需加载”；
2. **关键能力**：通过 `isOpen()`/`isConsumed()` 监控状态，`getCurrentIndex()` 跟踪进度，`close()` 释放资源，且支持 Iterable 遍历、AutoCloseable 自动关闭；
3. **使用约束**：若结果映射包含集合，SQL 查询需按 ID 列排序（`resultOrdered="true"`），确保分批读取的数据完整且不重复。
