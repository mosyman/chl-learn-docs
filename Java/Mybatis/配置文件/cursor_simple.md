
解释这份 MyBatis 的核心配置文件（mybatis-config.xml）的含义、结构和每个配置项的作用
### 一、文件整体定位
这是一份标准的 **MyBatis 核心配置文件**（mybatis-config.xml），主要作用是配置 MyBatis 的运行环境（数据库连接、事务管理器）和映射器（Mapper）路径，是 MyBatis 框架的核心配置入口。这份配置是针对“cursor_simple”（游标简单示例）场景的测试配置，使用的是内存型 HSQLDB 数据库。

### 二、逐部分详细解释
#### 1. XML 文档头 & 版权声明
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!--
       Copyright 2009-2022 the original author or authors.

       Licensed under the Apache License, Version 2.0 (the "License");
       you may not use this file except in compliance with the License.
       You may obtain a copy of the License at

          https://www.apache.org/licenses/LICENSE-2.0

       Unless required by applicable law or agreed to in writing, software
       distributed under the License is distributed on an "AS IS" BASIS,
       WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
       See the License for the specific language governing permissions and
       limitations under the License.
-->
```
- `<?xml version="1.0" encoding="UTF-8" ?>`：XML 文档的声明，指定 XML 版本为 1.0，编码格式为 UTF-8（确保配置文件中的中文/特殊字符不出现乱码）；
- 注释部分：Apache 开源许可证声明，说明该文件遵循 Apache 2.0 协议，可自由使用、修改（需遵守协议条款）。

#### 2. DOCTYPE 声明（约束文件）
```xml
<!DOCTYPE configuration
    PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
    "https://mybatis.org/dtd/mybatis-3-config.dtd">
```
- 作用：指定 MyBatis 配置文件的 **DTD（文档类型定义）约束**，MyBatis 会根据这个 DTD 校验配置文件的语法是否正确（比如标签是否合法、属性是否存在）；
- 核心含义：
    - `configuration`：根标签名称；
    - `"-//mybatis.org//DTD Config 3.0//EN"`：MyBatis 配置文件的 DTD 标识；
    - `"https://mybatis.org/dtd/mybatis-3-config.dtd"`：DTD 文件的网络地址，MyBatis 会加载这个文件做语法校验。

#### 3. 根标签 `<configuration>`
```xml
<configuration>
    <!-- 所有配置项都包裹在这个根标签内 -->
</configuration>
```
- MyBatis 配置文件的唯一根标签，所有核心配置（环境、映射器、设置等）都必须放在这个标签内。

#### 4. 环境配置 `<environments>`
```xml
<environments default="development">
    <environment id="development">
        <transactionManager type="JDBC">
            <property name="" value="" />
        </transactionManager>
        <dataSource type="UNPOOLED">
            <property name="driver" value="org.hsqldb.jdbcDriver" />
            <property name="url" value="jdbc:hsqldb:mem:cursor_simple" />
            <property name="username" value="sa" />
        </dataSource>
    </environment>
</environments>
```
这是配置数据库连接和事务管理的核心部分，逐标签解释：

##### (1) `<environments default="development">`
- `default="development"`：指定默认使用的环境 ID（这里是 `development`，即开发环境）；
- 作用：支持多环境配置（比如开发、测试、生产环境），只需修改 `default` 值即可切换环境，无需改其他配置。

##### (2) `<environment id="development">`
- `id="development"`：环境的唯一标识，与 `<environments>` 的 `default` 属性对应；
- 一个 `<environments>` 可以包含多个 `<environment>`，对应不同环境的数据库配置。

##### (3) `<transactionManager type="JDBC">`
- 事务管理器配置，`type="JDBC"` 表示使用 **JDBC 事务管理器**：
    - 原理：直接使用 JDBC 的 `Connection` 对象的事务管理（`commit()`/`rollback()`），由应用程序控制事务；
    - 其他可选值：`MANAGED`（交由容器管理事务，比如 Spring 容器）；
    - `<property>` 标签：这里为空，因为 JDBC 事务管理器无需额外配置。

##### (4) `<dataSource type="UNPOOLED">`
- 数据源配置，`type="UNPOOLED"` 表示使用 **非池化数据源**：
    - 原理：每次请求数据库时创建新的连接，用完后直接关闭，不复用连接（适合测试/轻量场景，生产环境一般用 `POOLED` 或第三方数据源如 Druid）；
    - 子标签 `<property>` 是数据库连接的核心参数：
        - `driver`：JDBC 驱动类全限定名（这里是 HSQLDB 内存数据库的驱动）；
        - `url`：数据库连接地址（`jdbc:hsqldb:mem:cursor_simple` 表示 HSQLDB 的内存数据库，数据库名是 `cursor_simple`，内存数据库重启后数据会丢失，适合测试）；
        - `username`：数据库用户名（HSQLDB 默认用户名是 `sa`，无密码）。

#### 5. 映射器配置 `<mappers>`
```xml
<mappers>
    <mapper resource="org/apache/ibatis/submitted/cursor_simple/Mapper.xml" />
</mappers>
```
- 作用：告诉 MyBatis 去哪里找 SQL 映射文件（Mapper.xml），MyBatis 会加载这些文件解析 SQL 语句；
- `<mapper resource="xxx" />`：通过 **类路径** 加载映射文件（这里的路径是 `org/apache/ibatis/submitted/cursor_simple/Mapper.xml`，对应项目中 `resources` 目录下的同名路径）；
- 其他加载方式：`url`（绝对路径）、`class`（接口全类名）、`package`（扫描整个包）。

### 三、配置文件的使用场景
这份配置是 MyBatis 官方的 **游标（Cursor）功能测试配置**，特点：
1. 使用内存型 HSQLDB 数据库：无需安装数据库，启动即创建，测试完数据自动销毁，适合单元测试；
2. 非池化数据源：测试场景下无需连接池，简化配置；
3. JDBC 事务管理器：测试时手动控制事务，逻辑简单；
4. 加载指定的 Mapper.xml：对应游标功能的 SQL 映射文件，用于测试百万级数据的懒加载查询。

### 总结
1. **核心作用**：这份配置是 MyBatis 连接数据库、加载 SQL 映射文件的基础配置，专为“游标（Cursor）功能测试”设计；
2. **关键配置项**：
    - 环境配置：指定数据库（HSQLDB 内存库）、事务管理器（JDBC）、数据源（非池化）；
    - 映射器配置：加载游标功能对应的 SQL 映射文件；
3. **适用场景**：MyBatis 游标功能的单元测试，而非生产环境（生产需改用池化数据源、正式数据库）。
