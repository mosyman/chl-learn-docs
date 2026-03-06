

## 目录
- [MyBatis 中的 `XPathParser` 类](#mybatis-中的-xpathparser-类)
- [evalNode方法](#evalnode方法)


## MyBatis 中的 `XPathParser` 类

### 一、类的整体定位
`XPathParser` 是 MyBatis 自定义的 **XML 解析工具类**，基于 JDK 自带的 XPath 规范实现，核心作用是：
1. 将 XML 数据源（字符串、Reader、InputStream、Document）解析为 DOM 文档（`org.w3c.dom.Document`）；
2. 提供简洁的 API，通过 XPath 表达式从 XML 文档中提取指定数据（字符串、布尔值、数字、节点列表等）；
3. 支持 XML 校验、变量替换、自定义实体解析（如离线 DTD 解析），是 MyBatis 解析核心配置文件（mybatis-config.xml）和映射文件（Mapper.xml）的基础工具。

简单来说：MyBatis 读取 XML 配置后，通过 `XPathParser` 快速定位和提取 `<configuration>`、`<environment>`、`<mapper>`、`<select>` 等标签的内容，是 XML 解析的核心入口。

### 二、核心成员变量
```java
private final Document document;       // 解析后的 DOM 文档对象（核心，所有 XPath 查询都基于它）
private boolean validation;            // 是否开启 XML 校验（DTD/XSD 校验）
private EntityResolver entityResolver; // 实体解析器（如 MyBatis 的 XMLMapperEntityResolver，离线解析 DTD）
private Properties variables;          // 变量配置（用于解析 XML 中的占位符，如 ${jdbc.url}）
private XPath xpath;                   // JDK 原生 XPath 对象，执行 XPath 表达式查询
```
| 变量名          | 核心作用                                                                 |
|-----------------|--------------------------------------------------------------------------|
| `document`      | 存储解析后的 XML 文档树，是所有 XPath 查询的“根对象”，不可变（final）|
| `validation`    | 控制解析 XML 时是否开启 DTD/XSD 语法校验（true=开启，false=关闭）|
| `entityResolver`| 自定义实体解析器，替换默认的网络 DTD 加载逻辑（MyBatis 中用于离线解析）|
| `variables`     | 存储全局变量（如数据库连接参数），用于替换 XML 中的 `${变量名}` 占位符 |
| `xpath`         | JDK 原生的 XPath 执行器，负责解析 XPath 表达式并返回结果               |

### 三、构造方法（重载设计）
`XPathParser` 提供了 12 个重载构造方法，核心逻辑是：**适配不同的 XML 输入源 + 统一初始化公共配置**，可分为 4 类输入源：
1. 字符串（`String xml`）；
2. 字符流（`Reader reader`）；
3. 字节流（`InputStream inputStream`）；
4. 已解析的 DOM 文档（`Document document`）。

每个输入源又支持 4 种参数组合：
- 仅输入源；
- 输入源 + 校验开关（`validation`）；
- 输入源 + 校验开关 + 变量（`variables`）；
- 输入源 + 校验开关 + 变量 + 实体解析器（`entityResolver`）。

#### 核心构造逻辑示例（以字符串输入为例）
```java
public XPathParser(String xml, boolean validation, Properties variables, EntityResolver entityResolver) {
    // 1. 公共初始化：设置校验、变量、实体解析器，创建 XPath 对象
    commonConstructor(validation, variables, entityResolver);
    // 2. 将字符串转为 InputSource，解析为 DOM 文档并赋值给 document
    this.document = createDocument(new InputSource(new StringReader(xml)));
}
```
所有构造方法都遵循“先调用 `commonConstructor` 初始化，再创建/赋值 `document`”的逻辑。

### 四、核心方法解析
#### 1. 公共初始化：`commonConstructor()`
```java
private void commonConstructor(boolean validation, Properties variables, EntityResolver entityResolver) {
    this.validation = validation;       // 设置 XML 校验开关
    this.entityResolver = entityResolver; // 设置自定义实体解析器
    this.variables = variables;         // 设置全局变量
    // 创建 JDK 原生 XPathFactory，生成 XPath 对象（核心执行器）
    XPathFactory factory = XPathFactory.newInstance();
    this.xpath = factory.newXPath();
}
```
- 作用：统一初始化所有构造方法的公共参数，避免重复代码；
- 关键：`XPathFactory.newInstance()` 创建默认的 XPath 工厂，生成可执行 XPath 表达式的 `XPath` 对象。

#### 2. DOM 文档创建：`createDocument()`
```java
private Document createDocument(InputSource inputSource) {
    try {
        // 1. 创建 DocumentBuilderFactory（DOM 解析工厂）
        DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
        // 开启安全处理（防止 XML 注入攻击）
        factory.setFeature(XMLConstants.FEATURE_SECURE_PROCESSING, true);
        // 2. 设置 XML 校验开关（是否校验 DTD/XSD）
        factory.setValidating(validation);

        // 3. 配置 DOM 解析规则（MyBatis 固定配置）
        factory.setNamespaceAware(false);          // 关闭命名空间支持（MyBatis 不使用 XML 命名空间）
        factory.setIgnoringComments(true);         // 忽略 XML 注释
        factory.setIgnoringElementContentWhitespace(false); // 不忽略元素内容中的空白字符
        factory.setCoalescing(false);              // 不合并相邻的文本节点
        factory.setExpandEntityReferences(false);  // 不展开实体引用（安全防护）

        // 4. 创建 DocumentBuilder（DOM 解析器）
        DocumentBuilder builder = factory.newDocumentBuilder();
        builder.setEntityResolver(entityResolver); // 设置自定义实体解析器（离线 DTD）
        // 5. 设置错误处理器：错误/致命错误直接抛出异常，警告忽略
        builder.setErrorHandler(new ErrorHandler() {
            @Override
            public void error(SAXParseException exception) throws SAXException {
                throw exception; // 语法错误直接抛出
            }
            @Override
            public void fatalError(SAXParseException exception) throws SAXException {
                throw exception; // 致命错误直接抛出
            }
            @Override
            public void warning(SAXParseException exception) throws SAXException {
                // NOP：忽略警告
            }
        });
        // 6. 解析 InputSource 为 DOM 文档并返回
        return builder.parse(inputSource);
    } catch (Exception e) {
        // 包装为 MyBatis 自定义异常，便于定位问题
        throw new BuilderException("Error creating document instance.  Cause: " + e, e);
    }
}
```
这是将 `InputSource` 转为 DOM 文档的核心方法，关键配置解读：
- `XMLConstants.FEATURE_SECURE_PROCESSING`：开启安全处理，防止 XXE（XML External Entity）注入攻击；
- `setValidating(validation)`：是否开启 DTD 校验（MyBatis 解析配置文件时通常关闭，提升性能）；
- `setEntityResolver(entityResolver)`：关联自定义实体解析器（如 `XMLMapperEntityResolver`），实现离线 DTD 解析；
- 错误处理器：严格处理错误（语法错误直接抛异常），保证 XML 配置的合法性。

#### 3. XPath 执行核心：`evaluate()`
```java
private Object evaluate(String expression, Object root, QName returnType) {
    try {
        // 调用 JDK 原生 XPath 执行器，执行表达式并返回指定类型结果
        return xpath.evaluate(expression, root, returnType);
    } catch (Exception e) {
        // 包装为 MyBatis 构建异常，添加“XPath 解析失败”提示
        throw new BuilderException("Error evaluating XPath.  Cause: " + e, e);
    }
}
```
- 作用：封装 JDK 原生 XPath 执行逻辑，统一异常处理；
- 参数说明：
    - `expression`：XPath 表达式（如 `"/configuration/environments/environment[@id='dev']"`）；
    - `root`：查询的根节点（可以是 `Document`、`Node` 等）；
    - `returnType`：返回值类型（如 `XPathConstants.STRING`、`XPathConstants.NODE`）；
- 所有对外的 `evalXxx()` 方法最终都调用这个方法。

#### 4. 对外查询 API（evalXxx 系列）
提供多种类型的查询方法，分为两类：
- 基于整个文档的查询（默认根节点为 `document`）；
- 基于指定根节点的查询（传入 `Object root`，可以是任意 Node）。

##### (1) 字符串查询：`evalString()`
```java
public String evalString(Object root, String expression) {
    // 1. 执行 XPath，获取原始字符串结果
    String result = (String) evaluate(expression, root, XPathConstants.STRING);
    // 2. 替换结果中的 ${变量名} 占位符（核心！）
    return PropertyParser.parse(result, variables);
}
```
- 关键：`PropertyParser.parse()` 会将结果中的 `${xxx}` 替换为 `variables` 中的值（如 `${jdbc.url}` 替换为实际的数据库连接地址）。

##### (2) 布尔值查询：`evalBoolean()`
```java
public Boolean evalBoolean(Object root, String expression) {
    // 直接执行 XPath，返回布尔值（无需变量替换）
    return (Boolean) evaluate(expression, root, XPathConstants.BOOLEAN);
}
```
- 适用场景：查询 XML 中的布尔属性（如 `<setting name="cacheEnabled" value="true"/>`）。

##### (3) 数字类型查询（evalShort/Integer/Long/Float/Double）
- 短整型/整型/长整型/浮点型：基于 `evalString()` 结果转换（先取字符串，再转数字）；
- 双精度浮点型：直接通过 `XPathConstants.NUMBER` 获取（JDK 原生支持）。

示例（长整型）：
```java
public Long evalLong(Object root, String expression) {
    return Long.valueOf(evalString(root, expression));
}
```

##### (4) 节点列表查询：`evalNodes()`
```java
public List<XNode> evalNodes(Object root, String expression) {
    List<XNode> xnodes = new ArrayList<>();
    // 1. 执行 XPath，获取 NodeList（DOM 节点列表）
    NodeList nodes = (NodeList) evaluate(expression, root, XPathConstants.NODESET);
    // 2. 将 DOM 节点包装为 MyBatis 自定义的 XNode（简化节点操作）
    for (int i = 0; i < nodes.getLength(); i++) {
      xnodes.add(new XNode(this, nodes.item(i), variables));
    }
    return xnodes;
}
```
- 关键：将 JDK 原生的 `NodeList` 转为 MyBatis 自定义的 `XNode` 列表，`XNode` 封装了节点的属性、文本、子节点等操作，更易用。

##### (5) 单个节点查询：`evalNode()`
```java
public XNode evalNode(Object root, String expression) {
    // 1. 执行 XPath，获取单个 Node
    Node node = (Node) evaluate(expression, root, XPathConstants.NODE);
    if (node == null) {
      return null;
    }
    // 2. 包装为 XNode 返回
    return new XNode(this, node, variables);
}
```
- 适用场景：查询唯一节点（如 `/configuration/environments`）。

#### 5. 变量设置：`setVariables()`
```java
public void setVariables(Properties variables) {
    this.variables = variables;
}
```
- 作用：动态设置/更新全局变量（如解析配置文件后，动态替换数据库连接参数）。

### 四、实际使用场景（MyBatis 源码示例）
MyBatis 在 `XMLConfigBuilder` 中解析核心配置文件时，会使用 `XPathParser`：
```java
// XMLConfigBuilder 中的核心代码
public class XMLConfigBuilder extends BaseBuilder {
    private final XPathParser parser;

    public XMLConfigBuilder(InputStream inputStream, String environment, Properties props) {
        // 1. 创建 XPathParser，传入字节流、关闭校验、全局变量、自定义实体解析器
        this.parser = new XPathParser(inputStream, false, props, new XMLMapperEntityResolver());
        // 2. 解析 <configuration> 节点
        parseConfiguration(parser.evalNode("/configuration"));
    }

    private void parseConfiguration(XNode root) {
        try {
            // 3. 用 XPathParser 解析各个子节点
            parseEnvironments(root.evalNode("environments")); // 解析环境配置
            parseMappers(root.evalNode("mappers")); // 解析映射器
            // ... 其他节点解析
        } catch (Exception e) {
            throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
        }
    }
}
```
这个场景中：
1. `XPathParser` 负责将配置文件字节流解析为 DOM 文档；
2. 通过 `evalNode("/configuration")` 定位根节点；
3. 再通过 `evalNode("environments")` 等方法提取子节点，完成配置解析。

### 五、核心设计亮点
1. **重载构造方法**：适配所有常见的 XML 输入源，使用灵活；
2. **统一异常处理**：将 JDK 原生异常包装为 MyBatis 自定义的 `BuilderException`，便于上层统一处理；
3. **变量替换**：内置 `PropertyParser` 支持 `${变量名}` 占位符，实现配置动态化；
4. **安全防护**：开启安全处理、关闭实体引用展开，防止 XML 注入攻击；
5. **简化节点操作**：将 DOM 节点包装为 `XNode`，提供更易用的节点操作 API。

### 总结
1. **核心作用**：`XPathParser` 是 MyBatis 封装的 XML 解析工具，基于 XPath 实现 XML 文档的解析和数据提取，是解析配置文件/映射文件的核心；
2. **关键流程**：
    - 输入源（字符串/流/Document）→ `createDocument()` 转为 DOM 文档；
    - 通过 `evalXxx()` 方法执行 XPath 表达式，提取指定类型数据；
    - 支持变量替换、离线 DTD 解析、安全防护；
3. **核心方法**：
    - `commonConstructor()`：初始化公共配置和 XPath 执行器；
    - `createDocument()`：解析 InputSource 为 DOM 文档，配置安全和校验规则；
    - `evaluate()`：封装 XPath 执行逻辑，统一异常处理；
    - `evalXxx()`：对外提供类型化的查询 API，支持变量替换和节点封装。


[目录](#目录)

## evalNode方法


### 1. 核心需求理解
你希望我详细解释这段 MyBatis 中 `XPathParser` 类的核心方法（`evalNode`/`evaluate`）——这些方法是 MyBatis 解析 XML 配置文件的底层核心，负责通过 XPath 语法定位 XML 节点，并封装为 MyBatis 自定义的 `XNode` 对象，是 XML 配置解析的基础。

### 2. 详细代码解释
#### 2.1 方法整体定位
这三个方法属于 `XPathParser` 类（MyBatis 封装的 XML 解析工具），核心作用是：
- 基于 **XPath 表达式** 从 XML 文档/节点中定位目标节点；
- 将原生 DOM 节点（`org.w3c.dom.Node`）封装为 MyBatis 自定义的 `XNode` 对象（简化节点操作）；
- 统一处理 XPath 解析异常，封装为 MyBatis 标准的 `BuilderException`。

#### 2.2 方法1：无参根节点的 evalNode(String expression)
```java
public XNode evalNode(String expression) {
  return evalNode(document, expression);
}
```
- **方法作用**：简化调用——以<span style="color:#ff6600; font-weight:bold;">整个 XML 文档</span>（`document` 是 `XPathParser` 的成员变量，代表解析后的 XML 文档根）为起点，执行 XPath 表达式。
- **参数说明**：
    - `expression`：XPath 表达式（如 `/configuration/properties`），用于定位 XML 节点；
- **底层调用**：直接转发到重载方法 `evalNode(Object root, String expression)`，传入“整个文档”作为根节点。
- **使用场景**：解析 XML 配置的根级别节点（如 `<configuration>` 下的 `<properties>`），是最常用的调用方式。

#### 2.3 方法2：指定根节点的 evalNode(Object root, String expression)
```java
public XNode evalNode(Object root, String expression) {
  // 1. 执行 XPath 表达式，获取原生 DOM 节点（Node）
  Node node = (Node) evaluate(expression, root, XPathConstants.NODE);
  // 2. 若节点不存在，返回 null
  if (node == null) {
    return null;
  }
  // 3. 将原生 Node 封装为 MyBatis 自定义的 XNode 对象
  return new XNode(this, node, variables);
}
```
- **核心逻辑拆解**：
  ##### 步骤1：执行 XPath 表达式
    - `evaluate(expression, root, XPathConstants.NODE)`：调用底层 `evaluate` 方法，传入三个关键参数：
        - `expression`：XPath 表达式（如 `properties`）；
        - `root`：解析的起点节点（可以是整个文档，也可以是某个子节点，如 `<configuration>`）；
        - `XPathConstants.NODE`：指定返回类型为“单个 DOM 节点”（XPath 支持返回节点、节点集、字符串等类型）。
    - 类型转换：将 `evaluate` 返回的 `Object` 强转为原生 DOM 的 `Node` 对象（W3C 标准的 XML 节点）。

  ##### 步骤2：空值处理
    - 若 XPath 表达式未匹配到任何节点（返回 `null`），直接返回 `null`，避免后续空指针异常。

  ##### 步骤3：封装为 XNode
    - `new XNode(this, node, variables)`：将原生 `Node` 封装为 MyBatis 自定义的 `XNode` 对象，核心参数：
        - `this`：当前 `XPathParser` 实例（供 `XNode` 后续解析子节点时复用）；
        - `node`：原生 DOM 节点（核心数据）；
        - `variables`：全局变量（如 `<properties>` 解析后的参数，用于替换节点中的 `${xxx}` 占位符）。
    - **封装目的**：原生 `Node` 操作繁琐（如获取属性、子节点需写大量重复代码），`XNode` 是 MyBatis 对其的“易用性封装”，提供 `getStringAttribute()`、`getChildrenAsProperties()` 等便捷方法。

#### 2.4 方法3：底层核心 evaluate 方法
```java
private Object evaluate(String expression, Object root, QName returnType) {
  try {
    // 调用 JDK 原生 XPath 解析器执行表达式
    return xpath.evaluate(expression, root, returnType);
  } catch (Exception e) {
    // 统一封装为 MyBatis 标准异常
    throw new BuilderException("Error evaluating XPath.  Cause: " + e, e);
  }
}
```
- **核心作用**：封装 JDK 原生 XPath 解析逻辑，统一异常处理。
- **关键细节**：
  ##### 1. 核心依赖：`xpath` 成员变量
    - `xpath` 是 `XPathParser` 初始化时创建的 `javax.xml.xpath.XPath` 实例（JDK 原生 XPath 解析器），已配置好命名空间、变量解析器等。
  ##### 2. 参数说明：
    - `expression`：XPath 表达式（如 `/configuration/environments`）；
    - `root`：解析起点（可以是 `org.w3c.dom.Document`/`org.w3c.dom.Node`）；
    - `returnType`：返回类型（`QName`），如 `XPathConstants.NODE`（单个节点）、`XPathConstants.NODESET`（节点集合）、`XPathConstants.STRING`（字符串）。
  ##### 3. 异常处理：
    - 捕获所有 XPath 解析异常（如表达式语法错误、节点不存在、命名空间错误等）；
    - 封装为 MyBatis 标准的 `BuilderException`，并补充“Error evaluating XPath”上下文，便于定位问题（如 XML 配置文件的 XPath 表达式写错）。

#### 2.5 关键辅助说明
##### （1）XPath 表达式示例（MyBatis 常用）
| 表达式                | 含义                                  |
|-----------------------|---------------------------------------|
| `/configuration`      | 定位 XML 根节点 `<configuration>`     |
| `configuration/properties` | 定位 `<configuration>` 下的 `<properties>` 节点 |
| `environments/environment[@id='dev']` | 定位 `<environments>` 下 id 为 dev 的 `<environment>` 节点 |

##### （2）XNode 封装的价值
原生 `Node` 获取属性需要写：
```java
// 原生 DOM 获取属性
String resource = ((Element) node).getAttribute("resource");
```
而 `XNode` 提供了便捷方法：
```java
// MyBatis XNode 获取属性
String resource = xnode.getStringAttribute("resource");
```
还支持直接解析子节点为 `Properties`：
```java
Properties props = xnode.getChildrenAsProperties();
```
大幅简化了 XML 解析代码，是 MyBatis 封装 `XNode` 的核心目的。

##### （3）变量替换能力
`XNode` 初始化时传入的 `variables`（全局属性），会在获取节点文本/属性时自动替换 `${xxx}` 占位符，例如：
```xml
<property name="url" value="${jdbc.url}"/>
```
`XNode` 会自动将 `${jdbc.url}` 替换为 `variables` 中的 `jdbc.url` 值，这是 MyBatis 配置文件占位符的底层实现。

### 3. 总结
#### 核心关键点
1. **方法调用链路**：
   `evalNode(String)` → `evalNode(Object, String)` → `evaluate(...)` → JDK 原生 XPath 解析 → 封装为 `XNode`。

2. **核心设计目的**：
    - **简化调用**：提供无参根节点的 `evalNode` 方法，无需每次传入文档根节点；
    - **易用性封装**：将原生 DOM `Node` 转为 `XNode`，提供便捷的属性/子节点操作方法；
    - **异常统一化**：将 XPath 解析异常封装为 `BuilderException`，便于上层统一处理；
    - **支持变量替换**：`XNode` 关联全局变量，实现配置文件中 `${xxx}` 占位符的解析。

3. **核心依赖**：
    - JDK 原生 `javax.xml.xpath.XPath`：负责底层 XPath 表达式执行；
    - MyBatis 自定义 `XNode`：简化 XML 节点操作，支持变量替换。

#### 应用场景
这些方法是 MyBatis 解析所有 XML 配置的基础：
- 解析 `mybatis-config.xml`：通过 `evalNode("/configuration")` 定位根节点，再逐层解析子节点；
- 解析 Mapper XML：通过 `evalNode("mapper/select[@id='selectById']")` 定位 SQL 节点；
- 自定义 XML 解析：若扩展 MyBatis 自定义配置节点，可复用这些方法解析 XML。