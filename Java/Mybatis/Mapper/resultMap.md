

## 目录
- [1](#1)
- [2](#2)





# 1

这段代码是 **MyBatis 框架核心配置文件 DTD (Document Type Definition)** 的一部分。它严格定义了 XML 映射文件（Mapper XML）中 `<resultMap>` 标签及其子标签的**语法结构**、**允许的属性**以及**嵌套规则**。

`<resultMap>` 是 MyBatis 最强大也是最复杂的元素，用于解决`数据库列名 与 Java 对象属性名`不一致的问题，以及处理复杂的关系映射（如一对一、一对多）。

以下是对这段 DTD 定义的详细解释及其背后的使用原理。

---

### 1. DTD 语法详细解读

DTD 使用特定的语法来约束 XML 的结构。我们将逐行拆解：

#### A. `<resultMap>` 根元素定义
```xml
<!ELEMENT resultMap (constructor?,id*,result*,association*,collection*, discriminator?)>
```
*   **含义**: 定义 `<resultMap>` 标签内部可以包含哪些子元素，以及它们的出现顺序和次数。
*   **顺序与基数**:
    *   `constructor?`: `<constructor>` 元素可选（0 或 1 次），必须放在最前面。用于通过构造函数实例化对象。
    *   `id*`: `<id>` 元素可出现多次（0 或更多）。用于`标记主键列`。
    *   `result*`: `<result>` 元素可出现多次（0 或更多）。用于`映射普通列`。
    *   `association*`: `<association>` 元素可出现多次。用于映射“一对一”或“多对一”关系（如 User 关联 Department）。
    *   `collection*`: `<collection>` 元素可出现多次。用于映射“一对多”关系（如 User 关联 List）。
    *   `discriminator?`: `<discriminator>` 元素可选（0 或 1 次），必须放在最后。用于根据列值动态选择子类映射（类似 Java 的 `switch-case`）。
*   **关键点**: 这个顺序是**严格**的。如果你在 XML 中把 `<association>` 写在 `<id>` 前面，XML 解析器会直接报错，因为不符合 DTD 定义。

```xml
<!ATTLIST resultMap
    id CDATA #REQUIRED
    type CDATA #REQUIRED
    extends CDATA #IMPLIED
    autoMapping (true|false) #IMPLIED
>
```
*   **含义**: 定义 `<resultMap>` 标签必须的属性和可选属性。
*   **属性详解**:
    *   `id` (**必填**): 当前 `ResultMap 的唯一标识符`，供 `<select>` 标签通过 `resultMap="..."` 引用。
    *   `type` (**必填**): 映射的`目标 Java 类的全限定名`（如 `com.example.User`）。MyBatis 将把查询结果映射到这个类的实例。
    *   `extends` (可选): 允许继承另一个已定义的 ResultMap 的映射规则，避免重复配置。
    *   `autoMapping` (可选): 值为 `true` 或 `false`。如果开启，MyBatis 会自动匹配列名和属性名相同的字段，未匹配的再由显式定义的 `<id>` 或 `<result>` 处理。

#### B. `<id>` 元素定义 (主键映射)
```xml
<!ELEMENT id EMPTY>
<!ATTLIST id
    property CDATA #IMPLIED
    javaType CDATA #IMPLIED
    column CDATA #IMPLIED
    jdbcType CDATA #IMPLIED
    typeHandler CDATA #IMPLIED
>
```
*   **`EMPTY`**: 表示 `<id>` 是一个空标签，不能有子元素（如 `<id>...</id>` 是非法的），只能有属性。
*   **作用**: 标记哪一列是`主键`。
    *   **性能优化**: MyBatis 利用主键信息来优化缓存策略（判断对象是否相同）以及加速嵌套查询的处理。
    *   **属性**:
        *   `property`: Java 对象的属性名。
        *   `column`: 数据库表的列名。
        *   `javaType`: 指定属性的 Java 类型（通常可省略，MyBatis 会反射推断）。
        *   `jdbcType`: 指定数据库列类型（在处理 `NULL` 值时非常重要，如 `jdbcType=VARCHAR`）。
        *   `typeHandler`: 自定义类型转换器。

#### C. `<result>` 元素定义 (普通列映射)
```xml
<!ELEMENT result EMPTY>
<!ATTLIST result
    property CDATA #IMPLIED
    javaType CDATA #IMPLIED
    column CDATA #IMPLIED
    jdbcType CDATA #IMPLIED
    typeHandler CDATA #IMPLIED
>
```
*   **结构**: 与 `<id>` 完全相同，也是空标签。
*   **作用**: 映射`非主键的普通字段`。
*   **区别**: 虽然属性一样，但在 MyBatis 内部逻辑中，被标记为 `<id>` 的字段会被特殊对待（用于对象身份识别），而 `<result>` 仅用于赋值。

---

### 2. 使用原理与底层机制

理解 DTD 只是第一步，关键在于 MyBatis 如何利用这些定义在运行时工作。

#### A. 为什么需要 `<resultMap>`？(解决阻抗不匹配)
*   **问题**: 数据库遵循关系模型（表、列），Java 遵循对象模型（类、属性）。
    *   数据库列名常用下划线命名法 (`user_name`)。
    *   Java 属性常用驼峰命名法 (`userName`)。
    *   或者存在复杂的关联关系，单张表无法直接映射到一个对象。
*   **原理**: `<resultMap>` 充当了**元数据地图**。MyBatis 在执行 SQL 获取 `ResultSet` 后，不会盲目地按索引赋值，而是查阅这张“地图”，知道 `ResultSet` 中的 `user_name` 列应该调用 Java 对象的 `setUserName()` 方法。

#### B. 映射执行流程 (Runtime Workflow)

当 MyBatis 执行一个配置了 `resultMap` 的查询时，内部发生以下步骤：

1.  **解析阶段 (Initialization)**:
    *   MyBatis 启动时解析 XML，读取 DTD 验证合法性。
    *   构建 `ResultMap` 对象树。每个 `<id>` 和 `<result>` 被解析为 `ResultMapping` 对象，存储在内存中。
    *   如果有 `<association>` 或 `<collection>`，会递归构建嵌套的 `ResultMap`。

2.  **执行阶段 (Execution)**:
    *   JDBC 执行 SQL，返回 `java.sql.ResultSet`。
    *   MyBatis 的 `ResultSetWrapper` 包装原始 ResultSet，提供列名到索引的快速查找。

3.  **实例化与赋值 (Object Creation & Population)**:
    *   **实例化**: MyBatis 根据 `type` 属性，通过反射（或工厂模式）创建目标 Java 对象（如果是 `<constructor>` 则调用构造器）。
    *   **主键识别**: 遍历 `<id>` 定义的列。MyBatis 记录这些值，用于判断在嵌套查询或缓存中，当前行是否对应一个已存在的对象实例（避免重复创建）。
    *   **属性填充**:
        *   遍历 `<result>` 列表。
        *   根据 `column` 从 ResultSet 取值。
        *   如果配置了 `typeHandler`（例如将数据库的 `BOOLEAN` 'Y'/'N' 转为 Java `boolean`），则调用处理器转换数据。
        *   通过反射调用对应的 `setter` 方法将值注入对象。
    *   **关联处理**:
        *   遇到 `<association>`：MyBatis 可能会发起额外的 SQL 查询（嵌套查询）或利用已有的 ResultSet 列（嵌套结果）来填充关联对象。
        *   遇到 `<collection>`：创建一个集合（List/Set），循环处理多行数据并添加到集合中。

#### C. 关键属性的底层作用

1.  **`<id>` vs `<result>` 的本质区别**:
    *   如果你把所有列都写成 `<result>`，功能上通常也能正常工作（数据能赋值）。
    *   **但是**，如果不标记 `<id>`，MyBatis 在某些场景下（特别是开启二级缓存或使用 `<collection>` 进行嵌套结果映射时）无法正确判断两行数据是否属于同一个父对象。
    *   **原理**: 在“一对多”映射中，数据库结果集会有多行具有相同的主键（父对象数据重复，子对象数据不同）。MyBatis 依靠 `<id>` 标记的列值来判断：“哦，这几行的 ID 一样，它们属于同一个 User 对象，我只需要创建一次 User，然后把后面的 Order 加到他的列表里”。如果没有 `<id>`，MyBatis 可能会为每一行创建一个全新的 User 对象，导致数据重复且内存浪费。

2.  **`jdbcType` 的重要性**:
    *   当数据库列值为 `NULL` 时，某些 JDBC 驱动（如 Oracle）无法推断列的类型，会抛出异常。
    *   显式声明 `jdbcType="VARCHAR"` 告诉驱动：“即使这里是 NULL，也请把它当作 VARCHAR 处理”，从而避免异常。

3.  **`autoMapping` 的原理**:
    *   当 `autoMapping="true"` 时，MyBatis 会扫描 ResultSet 的所有列名。
    *   它会将列名（自动转换为驼峰风格）与 Java Bean 的 setter 方法名进行匹配。
    *   **优先级**: 显式定义的 `<id>` 和 `<result>` 优先级高于自动映射。如果两者冲突，以显式定义为准。

### 3. 实际使用示例

基于你提供的 DTD，一个标准的用法如下：

```xml
<!-- 定义 resultMap -->
<resultMap id="UserDetailMap" type="com.example.User" autoMapping="true">
    <!-- 1. 主键映射 (必须，用于去重和缓存) -->
    <id property="userId" column="USER_ID" jdbcType="INTEGER"/>
    
    <!-- 2. 普通列映射 (覆盖自动映射或处理名称不一致) -->
    <result property="userName" column="USER_NAME" jdbcType="VARCHAR"/>
    <result property="status" column="STATUS_CODE" typeHandler="com.example.StatusTypeHandler"/>
    
    <!-- 3. 一对一关联 (Association) -->
    <association property="department" javaType="com.example.Department">
        <id property="deptId" column="DEPT_ID"/>
        <result property="deptName" column="DEPT_NAME"/>
    </association>
    
    <!-- 4. 一对多集合 (Collection) -->
    <collection property="orders" ofType="com.example.Order">
        <id property="orderId" column="ORDER_ID"/>
        <result property="amount" column="AMOUNT"/>
    </collection>
    
    <!-- 5. 鉴别器 (Discriminator) - 根据 type 列决定映射到 Employee 或 Manager -->
    <discriminator javaType="string" column="TYPE">
        <case value="EMPLOYEE" resultMap="EmployeeMap"/>
        <case value="MANAGER" resultMap="ManagerMap"/>
    </discriminator>
</resultMap>

<!-- 在 Select 中使用 -->
<select id="selectUser" resultMap="UserDetailMap">
    SELECT * FROM users u
    LEFT JOIN departments d ON u.dept_id = d.id
    LEFT JOIN orders o ON u.id = o.user_id
</select>
```

### 总结

这段 DTD 代码是 MyBatis ORM 能力的**骨架**。
*   **结构上**：它强制规定了映射配置的顺序（Constructor -> Id -> Result -> Association/Collection -> Discriminator），确保了解析的逻辑性。
*   **功能上**：它定义了如何通过 `<id>` 和 `<result>` 将关系型数据“翻译”成对象图。
*   **核心价值**：通过 `<id>` 标记主键，MyBatis 能够智能地处理数据库结果集中的重复数据（在 Join 查询中常见），正确构建对象之间的引用关系，而不是简单地生成重复的对象实例。这是 MyBatis 区别于简单 JDBC 封装的关键特性之一。

[目录](#目录)

# 2

你想深入理解MyBatis中`resultMap`相关的DTD（文档类型定义）语法，以及这些定义背后的使用规则和底层原理。这段DTD代码是MyBatis XML映射文件中`resultMap`标签的官方语法规范，规定了`resultMap`及其子标签（`id`、`result`）的结构、属性和使用约束。

### 一、DTD语法基础说明
首先要明确：DTD（Document Type Definition）是XML的“语法校验规则”，MyBatis通过这套DTD定义了`resultMap`标签**允许包含哪些子标签**、**有哪些属性**、**属性是否必填**等。
- `<!ELEMENT 标签名 (子标签规则)>`：定义标签的子元素结构。
- `<!ATTLIST 标签名 属性名 类型 约束>`：定义标签的属性（类型、是否必填等）。
- 约束规则：`#REQUIRED`（必填）、`#IMPLIED`（可选）、`(值1|值2)`（只能选其中一个）。
- `EMPTY`：表示该标签是“空标签”（无内容，仅属性），如`<id column="id" property="id"/>`。

### 二、逐段解析DTD定义
#### 1. resultMap 核心定义
```dtd
<!ELEMENT resultMap (constructor?,id*,result*,association*,collection*, discriminator?)>
<!ATTLIST resultMap
id CDATA #REQUIRED
type CDATA #REQUIRED
extends CDATA #IMPLIED
autoMapping (true|false) #IMPLIED
>
```
##### （1）`<!ELEMENT resultMap (...)>`：子标签规则
- `constructor?`：`constructor`标签（构造函数映射）**可选**（`?`表示0或1个）。
- `id*`：`id`标签（主键映射）**任意个**（`*`表示0、1、多个）。
- `result*`：`result`标签（普通字段映射）**任意个**。
- `association*`：`association`标签（一对一关联映射）**任意个**。
- `collection*`：`collection`标签（一对多关联映射）**任意个**。
- `discriminator?`：`discriminator`标签（鉴别器/多态映射）**可选**。
- 顺序要求：子标签必须按`constructor → id → result → association → collection → discriminator`的顺序编写（DTD严格规定，MyBatis会校验）。

##### （2）`<!ATTLIST resultMap ...>`：属性规则
| 属性名       | 类型   | 约束       | 含义                                                                 |
|--------------|--------|------------|----------------------------------------------------------------------|
| `id`         | CDATA  | #REQUIRED  | `resultMap`的唯一标识（必填），供`<select resultMap="xxx">`引用。     |
| `type`       | CDATA  | #REQUIRED  | 映射的目标Java类型（必填），可以是全类名（如`com.example.User`）或别名（如`User`）。 |
| `extends`    | CDATA  | #IMPLIED   | 继承其他`resultMap`的ID（可选），实现`resultMap`复用（子类会覆盖父类同名映射）。 |
| `autoMapping`| true\|false | #IMPLIED | 是否开启自动映射（可选）：<br>- `true`：自动映射未显式配置的字段；<br>- `false`：仅映射显式配置的字段；<br>- 默认值：全局`autoMappingBehavior`配置（通常为`PARTIAL`）。 |

##### （3）使用示例（对应DTD规则）
```xml
<!-- 符合DTD规则的resultMap：id/type必填，extends/autoMapping可选 -->
<resultMap id="userResultMap" type="com.example.User" extends="baseResultMap" autoMapping="true">
    <id column="id" property="userId"/> <!-- 主键映射 -->
    <result column="name" property="userName"/> <!-- 普通字段映射 -->
</resultMap>
```

#### 2. id 标签定义（主键映射）
```dtd
<!ELEMENT id EMPTY>
<!ATTLIST id
property CDATA #IMPLIED
javaType CDATA #IMPLIED
column CDATA #IMPLIED
jdbcType CDATA #IMPLIED
typeHandler CDATA #IMPLIED
>
```
##### （1）核心规则
- `<!ELEMENT id EMPTY>`：`id`是**空标签**（无文本内容，仅属性），必须写成`<id .../>`，不能写成`<id>内容</id>`。
- `<!ATTLIST id ...>`：属性均为`#IMPLIED`（可选），但实际使用中`property`和`column`几乎必配（否则映射无意义）。

##### （2）属性含义
| 属性名       | 含义                                                                 |
|--------------|----------------------------------------------------------------------|
| `property`   | Java对象的属性名（如`userId`），MyBatis会通过反射设置该属性值。       |
| `javaType`   | Java属性的类型（可选，MyBatis可自动推断，如`java.lang.Long`）。       |
| `column`     | 数据库查询结果的列名（如`id`），或列的别名（如`user_id`）。           |
| `jdbcType`   | 数据库列的JDBC类型（可选，如`INTEGER`、`VARCHAR`，解决NULL值映射问题）。 |
| `typeHandler`| 自定义类型处理器（可选），指定将JDBC类型转换为Java类型的处理器（如`com.example.MyTypeHandler`）。 |

##### （3）底层原理：`id`标签的特殊作用
`id`标签标记的是**主键字段**，MyBatis在处理关联查询（如`association`/`collection`）时，会通过`id`字段判断“是否是同一个对象”，从而避免重复创建对象（提升性能）。
- 例如：一对多查询用户和订单时，MyBatis会通过用户的`id`字段判断“是否是同一个用户”，从而将多个订单归到同一个用户对象下。

#### 3. result 标签定义（普通字段映射）
```dtd
<!ELEMENT result EMPTY>
<!ATTLIST result
property CDATA #IMPLIED
javaType CDATA #IMPLIED
column CDATA #IMPLIED
jdbcType CDATA #IMPLIED
typeHandler CDATA #IMPLIED
>
```
##### （1）核心规则
- 与`id`标签的DTD定义**完全一致**（空标签、相同属性），唯一区别是**语义不同**：
    - `id`：标记主键字段（MyBatis有特殊处理逻辑）；
    - `result`：标记普通非主键字段（无特殊处理）。

##### （2）使用示例
```xml
<resultMap id="userResultMap" type="com.example.User">
    <id property="id" column="user_id" jdbcType="BIGINT"/> <!-- 主键 -->
    <result property="name" column="user_name" javaType="java.lang.String"/> <!-- 普通字段 -->
    <result property="createTime" column="create_time" typeHandler="org.apache.ibatis.type.LocalDateTimeTypeHandler"/> <!-- 自定义类型处理器 -->
</resultMap>
```

### 三、底层使用原理（DTD → MyBatis执行流程）
MyBatis解析`resultMap`的核心流程，本质是“将DTD定义的XML结构转换为Java对象，再通过反射完成数据映射”：
```mermaid
graph TD
    A[加载XML映射文件] --> B[校验XML是否符合DTD规则]
    B --> C[解析resultMap标签为ResultMap对象]
    C --> D[解析id/result标签为ResultMapping对象]
    D --> E[执行SQL，获取JDBC ResultSet]
    E --> F[遍历ResultSet行数据]
    F --> G[根据ResultMapping匹配列名(column)和属性名(property)]
    G --> H[使用typeHandler转换JDBC类型→Java类型]
    H --> I[通过反射设置Java对象属性值]
    I --> J[返回映射后的Java对象]
```

#### 关键原理细节
1. **DTD校验阶段**：
   MyBatis启动时会加载`mybatis-3-mapper.dtd`，并校验所有XML映射文件是否符合DTD规则（如`resultMap`必须有`id`和`type`属性、子标签顺序是否正确），校验失败会直接抛出启动异常。

2. **映射核心逻辑**：
    - `id`/`result`的`column`属性对应`ResultSet.getString(column)`/`ResultSet.getLong(column)`等方法，从JDBC结果集中获取数据；
    - `property`属性对应Java对象的`setter`方法（如`setProperty(value)`），MyBatis通过反射调用setter完成赋值；
    - 若未指定`javaType`，MyBatis会根据`property`的类型（通过反射获取）自动推断；若未指定`jdbcType`，会根据数据库返回的字段类型自动适配。

3. **autoMapping原理**：
   当`autoMapping="true"`时，MyBatis会遍历`ResultSet`的所有列，对于未在`id`/`result`中显式配置的列，会按“列名（下划线转驼峰）→ 属性名”的规则自动映射（如列`user_name`→属性`userName`）。

### 四、常见错误与DTD约束的关系
| 错误场景                          | 违反的DTD规则                          | 报错示例                                  |
|-----------------------------------|----------------------------------------|-------------------------------------------|
| `resultMap`缺少`id`属性           | `id CDATA #REQUIRED`                   | The content of element type "resultMap" must match... |
| `resultMap`子标签顺序错误（如`result`在`id`前） | `(constructor?,id*,result*,...)`       | XML document structure error              |
| `id`标签写成`<id>内容</id>`       | `<!ELEMENT id EMPTY>`                  | Element type "id" must be empty           |
| `autoMapping`赋值为`yes`          | `autoMapping (true|false) #IMPLIED`    | Attribute "autoMapping" with value "yes" must have a value from the list "true false" |

### 总结
1. **DTD核心作用**：定义了`resultMap`/`id`/`result`的语法规则（必填属性、子标签顺序、空标签约束），MyBatis启动时会校验XML是否符合规则，避免运行时错误。
2. **`id`与`result`的区别**：语法完全一致，但`id`标记主键字段，MyBatis会利用主键优化关联查询的对象复用；`result`仅映射普通字段。
3. **底层映射原理**：MyBatis将DTD解析后的`resultMap`转换为`ResultMap`/`ResultMapping`对象，通过JDBC的`ResultSet`获取列值，再通过反射调用Java对象的setter方法完成赋值，`autoMapping`则是自动匹配未显式配置的字段。