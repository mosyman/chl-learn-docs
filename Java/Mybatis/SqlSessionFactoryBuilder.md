

## SqlSessionFactoryBuilder

### 一、类的整体定位
`SqlSessionFactoryBuilder` 是 MyBatis 中负责**构建 `SqlSessionFactory` 实例**的建造者类，遵循“建造者模式”设计。它的核心职责是：读取 MyBatis 的配置信息（XML/Configuration 对象），解析并构建出 `SqlSessionFactory`（SqlSession 工厂）—— 而 `SqlSession` 是 MyBatis 与数据库交互的核心会话对象。

类注释也明确了这一点：`Builds {@link SqlSession} instances.`（构建 SqlSession 实例的工厂的建造者），作者是 MyBatis 核心开发者 Clinton Begin。

### 二、核心设计思路
这个类的设计遵循“重载 + 核心实现分离”的原则：
1. 提供多个重载的 `build()` 方法，适配不同的配置输入方式（Reader/InputStream、指定环境、自定义属性）；
2. 所有重载方法最终都会收敛到两个核心实现（处理 Reader/InputStream），再最终调用 `build(Configuration config)` 生成 `SqlSessionFactory`；
3. 完善的异常处理和资源释放，确保配置文件流/阅读器能正常关闭。

### 三、逐方法详细解释
#### 1. 类定义
```java
public class SqlSessionFactoryBuilder { ... }
```
- 普通的公共类，无继承/实现（仅作为工具类，封装构建逻辑）；
- 无成员变量，所有方法都是静态/实例方法（实际使用时一般 `new SqlSessionFactoryBuilder().build(...)` 调用）。

#### 2. 基于 Reader 的重载方法（处理字符流）
##### (1) 最简重载：`build(Reader reader)`
```java
public SqlSessionFactory build(Reader reader) {
  return build(reader, null, null);
}
```
- **作用**：接收字符流（Reader）形式的配置文件（如 mybatis-config.xml），构建 `SqlSessionFactory`；
- **参数**：`reader` - 读取配置文件的字符流（比如 `new FileReader("mybatis-config.xml")`）；
- **逻辑**：调用核心重载方法，`environment` 和 `properties` 传 `null`（使用配置文件中默认的环境、默认属性）。

##### (2) 指定环境的重载：`build(Reader reader, String environment)`
```java
public SqlSessionFactory build(Reader reader, String environment) {
  return build(reader, environment, null);
}
```
- **作用**：指定要使用的环境（对应配置文件中 `<environments default="xxx">` 的 `environment` 标签 ID）；
- **参数**：`environment` - 环境 ID（如 "dev"/"prod"），用于多环境切换；
- **逻辑**：调用核心方法，`properties` 传 `null`（使用配置文件内的属性）。

##### (3) 自定义属性的重载：`build(Reader reader, Properties properties)`
```java
public SqlSessionFactory build(Reader reader, Properties properties) {
  return build(reader, null, properties);
}
```
- **作用**：传入自定义的属性（Properties），覆盖配置文件中的属性（比如动态设置数据库用户名/密码）；
- **参数**：`properties` - 自定义属性集（如 `properties.setProperty("jdbc.username", "root")`）；
- **逻辑**：调用核心方法，`environment` 传 `null`（使用默认环境）。

##### (4) 核心实现：`build(Reader reader, String environment, Properties properties)`
```java
public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
  try {
    // 1. 创建 XML 配置解析器，传入字符流、环境、自定义属性
    XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
    // 2. 解析配置文件，生成 Configuration 对象（MyBatis 核心配置容器），再构建 SqlSessionFactory
    return build(parser.parse());
  } catch (Exception e) {
    // 3. 异常包装：统一包装成 MyBatis 自定义异常，便于上层处理
    throw ExceptionFactory.wrapException("Error building SqlSession.", e);
  } finally {
    // 4. 重置错误上下文，避免异常信息污染
    ErrorContext.instance().reset();
    try {
      // 5. 释放资源：关闭 Reader 流，避免资源泄漏
      if (reader != null) {
        reader.close();
      }
    } catch (IOException e) {
      // 6. 忽略流关闭异常：优先抛出解析/构建过程中的核心异常
      // Intentionally ignore. Prefer previous error.
    }
  }
}
```
这是处理字符流的核心方法，关键步骤拆解：
- **步骤1**：`XMLConfigBuilder` 是 MyBatis 专门解析核心配置文件的类，负责把 XML 配置解析成内存中的 `Configuration` 对象；
- **步骤2**：`parser.parse()` 解析 XML 配置，生成 `Configuration`（包含环境、数据源、映射器、设置等所有配置），再调用 `build(Configuration config)` 生成工厂；
- **步骤3**：异常处理：用 `ExceptionFactory` 包装原生异常，添加“构建 SqlSession 失败”的提示，便于定位问题；
- **步骤4**：`ErrorContext` 是 MyBatis 的错误上下文工具，重置后避免后续操作受之前异常信息影响；
- **步骤5-6**：finally 块确保 Reader 流关闭，即使解析失败也会释放资源；关闭流的异常被忽略，因为核心异常（解析失败）更重要。

#### 3. 基于 InputStream 的重载方法（处理字节流）
这部分方法和 Reader 版本逻辑完全一致，只是输入方式换成了字节流（InputStream），适配不同的文件读取场景（比如从类路径/网络读取配置文件）。

##### (1) 最简重载：`build(InputStream inputStream)`
```java
public SqlSessionFactory build(InputStream inputStream) {
  return build(inputStream, null, null);
}
```
- **作用**：接收字节流形式的配置文件，比如 `Resources.getResourceAsStream("mybatis-config.xml")`（MyBatis 工具类读取类路径下的配置）。

##### (2) 指定环境/自定义属性的重载
```java
public SqlSessionFactory build(InputStream inputStream, String environment) {
  return build(inputStream, environment, null);
}

public SqlSessionFactory build(InputStream inputStream, Properties properties) {
  return build(inputStream, null, properties);
}
```
- 逻辑和 Reader 版本一致，只是参数换成 `InputStream`，适配字节流场景。

##### (3) 核心实现：`build(InputStream inputStream, String environment, Properties properties)`
```java
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
  try {
    // 1. 创建 XML 解析器，传入字节流
    XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
    // 2. 解析 + 构建工厂
    return build(parser.parse());
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error building SqlSession.", e);
  } finally {
    ErrorContext.instance().reset();
    try {
      // 3. 关闭字节流
      if (inputStream != null) {
        inputStream.close();
      }
    } catch (IOException e) {
      // Intentionally ignore. Prefer previous error.
    }
  }
}
```
- 唯一区别：`XMLConfigBuilder` 接收 `InputStream` 而非 `Reader`，其余逻辑（解析、异常、资源释放）完全一致。

#### 4. 最终构建方法：`build(Configuration config)`
```java
public SqlSessionFactory build(Configuration config) {
  return new DefaultSqlSessionFactory(config);
}
```
- **作用**：所有重载方法的最终落脚点，接收解析后的 `Configuration` 对象，创建 `SqlSessionFactory` 的默认实现类；
- **核心**：`DefaultSqlSessionFactory` 是 `SqlSessionFactory` 接口的唯一默认实现，持有 `Configuration` 对象，后续通过它创建 `SqlSession`；
- **设计意图**：把“解析配置”和“创建工厂”分离，即使不通过 XML 配置（比如纯代码构建 `Configuration`），也能调用这个方法创建工厂。

### 四、使用示例（帮助理解）
#### 场景1：通过 XML 配置文件（字节流）构建
```java
// 1. 读取配置文件字节流
InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
// 2. 构建 SqlSessionFactory
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream, "dev");
// 3. 后续通过工厂创建 SqlSession
try (SqlSession session = sqlSessionFactory.openSession()) {
  // 操作数据库
}
```

#### 场景2：纯代码构建（无 XML）
```java
// 1. 手动构建 Configuration 对象
Configuration config = new Configuration();
config.setEnvironment(new Environment("dev", new JdbcTransactionFactory(), 
    new UnpooledDataSource("org.hsqldb.jdbcDriver", "jdbc:hsqldb:mem:test", "sa", "")));
// 2. 注册映射器
config.addMapper(UserMapper.class);
// 3. 构建工厂
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(config);
```

### 总结
1. **核心职责**：`SqlSessionFactoryBuilder` 是建造者类，负责将 MyBatis 配置（XML/Configuration 对象）解析并构建出 `SqlSessionFactory`；
2. **设计特点**：
    - 重载 `build()` 方法适配多种输入方式（Reader/InputStream、指定环境、自定义属性）；
    - 所有重载方法收敛到核心实现，最终调用 `build(Configuration)` 创建 `DefaultSqlSessionFactory`；
    - 完善的异常包装和资源释放，确保配置解析过程的健壮性；
3. **关键流程**：配置输入（Reader/InputStream）→ XMLConfigBuilder 解析 → Configuration 对象 → DefaultSqlSessionFactory 实例。
