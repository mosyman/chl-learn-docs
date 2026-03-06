

## 目录
- [XML11Configuration](#xml11configuration)
- [configureXML11Pipeline](#configurexml11pipeline)


## XML11Configuration
### 一、类的整体定位
`XML11Configuration` 是 Apache Xerces（Java 主流 XML 解析器）中的核心配置类，专门用于**解析 XML 1.0 和 XML 1.1 版本的文档**，继承并实现了 Xerces 内部的配置接口，核心职责是：
1. 管理 XML 1.0/1.1 解析的配置参数（如校验规则、字符编码、实体解析等）；
2. 封装 XML 文档的解析入口，处理解析过程的初始化、版本检测、管道配置和异常统一；
3. 保证解析过程的线程安全（通过 `fParseInProgress` 标记防止并发解析）。

简单来说：这个类是 Xerces 解析器中“XML 1.0/1.1 解析”的配置和执行入口，负责统筹解析前的准备工作、解析中的版本适配，以及解析后的资源清理。

### 二、核心继承/实现关系
```java
public class XML11Configuration extends ParserConfigurationSettings
    implements XMLPullParserConfiguration, XML11Configurable { ... }
```
| 父类/接口                | 核心作用                                                                 |
|--------------------------|--------------------------------------------------------------------------|
| `ParserConfigurationSettings` | 基础配置类，存储解析器的通用配置（如是否开启校验、实体解析器、错误处理器等） |
| `XMLPullParserConfiguration`  | Pull 模式解析器的配置接口，支持“拉取式”XML 解析（按需读取节点）|
| `XML11Configurable`           | XML 1.1 专属配置接口，定义 XML 1.1 解析的特有配置项（如字符集、换行符规则） |

### 三、核心成员变量（隐含，从代码逻辑推导）
代码中出现的关键成员变量（虽未完整展示，但可从方法中推断）：

| 变量名                | 核心作用                                                                 |
|-----------------------|--------------------------------------------------------------------------|
| `fParseInProgress`    | 布尔值，标记是否正在解析（防止同一实例并发调用 `parse()` 方法）|
| `fInputSource`        | 待解析的 XML 输入源（`XMLInputSource`，Xerces 自定义的输入源封装，类似 JDK 的 `InputSource`） |
| `fValidationManager`  | 校验管理器，负责 XML 语法/语义校验（如 DTD/XSD 校验）|
| `fVersionDetector`    | XML 版本检测器，用于判断文档是 XML 1.0 还是 1.1                          |
| `fConfigUpdated`      | 配置更新标记，标记解析器配置是否已更新，用于触发管道重新配置             |
| `fCurrentScanner`     | 当前 XML 扫描器（核心解析组件），负责逐字符读取 XML 并生成解析事件       |
| `PRINT_EXCEPTION_STACK_TRACE` | 调试开关，是否打印异常堆栈（生产环境通常关闭）|

### 四、核心方法详细解析
#### 1. 对外解析入口：`parse(XMLInputSource source)`
这是外部调用的核心解析方法，负责**初始化解析状态、统一异常处理、资源清理**，是解析流程的“总入口”。

```java
public void parse(XMLInputSource source) throws XNIException, IOException {
    // 1. 并发校验：如果正在解析中，抛出异常（防止同一实例并发解析）
    if (fParseInProgress) {
        throw new XNIException("FWK005 parse may not be called while parsing.");
    }
    fParseInProgress = true; // 标记为“正在解析”

    try {
        // 2. 设置输入源（将待解析的 XML 源绑定到当前实例）
        setInputSource(source);
        // 3. 执行核心解析逻辑（参数 true 表示“完整解析文档”）
        parse(true);
    } catch (XNIException ex) { // Xerces 自定义解析异常
        if (PRINT_EXCEPTION_STACK_TRACE)
            ex.printStackTrace();
        throw ex; // 原样抛出，保留异常类型
    } catch (IOException ex) { // IO 异常（如文件读取失败）
        if (PRINT_EXCEPTION_STACK_TRACE)
            ex.printStackTrace();
        throw ex;
    } catch (RuntimeException ex) { // 运行时异常（如空指针）
        if (PRINT_EXCEPTION_STACK_TRACE)
            ex.printStackTrace();
        throw ex;
    } catch (Exception ex) { // 其他所有异常
        if (PRINT_EXCEPTION_STACK_TRACE)
            ex.printStackTrace();
        throw new XNIException(ex); // 包装为 Xerces 统一的 XNIException
    } finally {
        // 4. 无论解析成功/失败，重置状态 + 清理资源
        fParseInProgress = false; // 取消“正在解析”标记
        this.cleanup(); // 关闭 Xerces 打开的所有流（如文件流、网络流）
    }
}
```
##### 关键逻辑解读：
- **并发安全**：`fParseInProgress` 是核心状态标记，确保同一 `XML11Configuration` 实例不会被多线程同时调用 `parse()`，避免解析混乱；
- **异常统一**：将所有异常分类处理，非 `XNIException` 的异常（如 `IOException`、普通 `Exception`）要么原样抛出，要么包装为 `XNIException`，保证外部调用方只需处理 `XNIException` 和 `IOException`；
- **资源安全**：`finally` 块强制重置解析状态并调用 `cleanup()`，即使解析抛出异常，也能关闭所有打开的流，避免资源泄漏。

#### 2. 核心解析逻辑：`parse(boolean complete)`
这是实际执行解析的方法，负责**版本检测、解析管道配置、文档扫描**，是解析流程的“核心执行器”。

```java
public boolean parse(boolean complete) throws XNIException, IOException {
    // 第一步：解析前的初始化（仅当输入源不为空时执行）
    if (fInputSource != null) {
        try {
            // 1. 重置校验管理器（清空上一次解析的校验状态）
            fValidationManager.reset();
            // 2. 重置版本检测器（准备检测当前文档的 XML 版本）
            fVersionDetector.reset(this);
            // 3. 标记配置已更新（触发管道重新配置）
            fConfigUpdated = true;
            // 4. 重置符号表（存储 XML 中的元素/属性名，避免内存泄漏）
            resetSymbolTable();
            // 5. 重置通用解析状态（如错误处理器、实体解析器等）
            resetCommon();

            // 6. 核心：检测 XML 文档版本（1.0 或 1.1）
            short version = fVersionDetector.determineDocVersion(fInputSource);
            if (version == Constants.XML_VERSION_1_1) {
                // 6.1 若是 XML 1.1：初始化 1.1 专属组件（如字符集处理、换行符规则）
                initXML11Components();
                // 6.2 配置 XML 1.1 解析管道（适配 1.1 的语法规则）
                configureXML11Pipeline();
                // 6.3 重置 XML 1.1 解析状态
                resetXML11();
            } else {
                // 6.4 若是 XML 1.0：配置通用解析管道
                configurePipeline();
                // 6.5 重置 XML 1.0 解析状态
                reset();
            }

            // 7. 标记配置已固定（解析过程中不再修改配置）
            fConfigUpdated = false;

            // 8. 启动版本检测器，初始化解析管道并设置版本
            fVersionDetector.startDocumentParsing((XMLEntityHandler) fCurrentScanner, version);
            // 9. 清空输入源（避免重复解析）
            fInputSource = null;
        } catch (IOException | RuntimeException ex) {
            if (PRINT_EXCEPTION_STACK_TRACE)
                ex.printStackTrace();
            throw ex;
        } catch (Exception ex) {
            if (PRINT_EXCEPTION_STACK_TRACE)
                ex.printStackTrace();
            throw new XNIException(ex);
        }
    }

    // 第二步：执行实际的文档扫描
    try {
        // 调用当前扫描器（fCurrentScanner）扫描文档，返回解析状态
        // complete=true：解析整个文档；complete=false：按需解析（Pull 模式）
        return fCurrentScanner.scanDocument(complete);
    } catch (IOException | RuntimeException ex) {
        if (PRINT_EXCEPTION_STACK_TRACE)
            ex.printStackTrace();
        throw ex;
    } catch (Exception ex) {
        if (PRINT_EXCEPTION_STACK_TRACE)
            ex.printStackTrace();
        throw new XNIException(ex);
    }
}
```
##### 关键逻辑解读：
###### (1) 解析前的初始化（核心是“版本适配”）
XML 1.0 和 1.1 有语法差异（如 1.1 支持更多字符集、允许更多换行符类型），因此必须先检测版本，再适配对应的解析组件：
- `fVersionDetector.determineDocVersion(fInputSource)`：读取 XML 文档开头的 `<?xml version="1.0"?>` 或 `<?xml version="1.1"?>`，返回版本标识；
- `initXML11Components()`/`configureXML11Pipeline()`：初始化 XML 1.1 专属的解析组件（如处理 `&#x1;` 这类 1.1 允许的控制字符）；
- `configurePipeline()`：初始化 XML 1.0 的通用解析管道（最常用）。

###### (2) 文档扫描（核心执行）
`fCurrentScanner.scanDocument(complete)` 是真正的解析执行逻辑：
- `fCurrentScanner`：Xerces 内部的 XML 扫描器，负责逐字符读取 XML 输入源，解析出元素、属性、文本等节点，并触发对应的解析事件（如 `startElement`、`endElement`）；
- `complete` 参数：
    - `true`：一次性解析整个 XML 文档（Push 模式，最常用）；
    - `false`：Pull 模式，解析到下一个节点就暂停，等待外部调用继续解析（适合大文档懒加载）；
- 返回值：布尔值，表示“是否还有未解析的内容”（Pull 模式下用于判断是否继续解析）。

###### (3) 异常处理
和外层 `parse()` 方法一致，分类捕获异常并打印堆栈（调试模式），保证异常类型统一。

### 五、核心设计背景与价值
#### 1. XML 1.0/1.1 版本适配
XML 1.1 相比 1.0 有少量但关键的语法差异：
- 1.1 允许更多控制字符（如 `&#x1;`）；
- 1.1 支持更多换行符类型（如 `\x85`、`\u2028`）；
- 1.1 对实体引用的处理规则更宽松。

`XML11Configuration` 通过 `fVersionDetector` 自动检测版本，并切换对应的解析组件，保证解析器能兼容两种版本的 XML 文档。

#### 2. 解析管道（Pipeline）设计
Xerces 采用“管道式”解析架构：
- 解析流程分为多个阶段（字符编码转换 → 语法扫描 → 校验 → 事件生成）；
- `configurePipeline()`/`configureXML11Pipeline()` 负责组装这些阶段的组件，适配不同版本的语法规则；
- 这种设计让解析器的扩展性更强（如新增 XML 1.2 支持时，只需新增 `configureXML12Pipeline()` 即可）。

#### 3. 资源与状态管理
- 每次解析前调用 `resetXxx()` 方法重置状态，避免上一次解析的残留数据影响当前解析；
- `finally` 块强制清理资源，保证解析器实例可复用（多次调用 `parse()`）。

### 六、使用场景（Xerces 解析流程示例）
```java
// 1. 创建 XML 输入源（如文件）
XMLInputSource source = new XMLInputSource(null, "file:///test.xml", null, new FileInputStream("test.xml"), "UTF-8");

// 2. 创建 XML11Configuration 实例
XML11Configuration config = new XML11Configuration();
// 可选：配置解析器（如开启 DTD 校验、设置实体解析器）
config.setValidation(true);
config.setEntityResolver(new MyEntityResolver());

// 3. 执行解析
try {
    config.parse(source); // 解析整个文档（Push 模式）
} catch (XNIException | IOException e) {
    e.printStackTrace();
}
```
这个示例中，`XML11Configuration` 负责统筹所有解析工作：检测 `test.xml` 的版本（1.0/1.1）、配置对应的解析组件、执行扫描，并处理异常和资源清理。

### 总结
1. **核心作用**：`XML11Configuration` 是 Xerces 解析器中 XML 1.0/1.1 文档的配置和解析入口，负责版本检测、解析管道配置、并发控制和资源管理；
2. **关键流程**：
    - 外层 `parse(source)`：校验并发状态 → 设置输入源 → 调用核心解析 → 统一异常 → 清理资源；
    - 内层 `parse(complete)`：重置状态 → 检测 XML 版本 → 适配解析管道 → 执行文档扫描；
3. **核心设计亮点**：
    - 版本自动适配：兼容 XML 1.0/1.1，无需手动指定版本；
    - 并发安全：通过 `fParseInProgress` 防止并发解析；
    - 资源安全：`finally` 块强制清理资源，避免泄漏；
    - 异常统一：将所有异常包装为 Xerces 标准异常，简化外部调用。


[目录](#目录)

## configureXML11Pipeline

你希望我详细解释这段 `configureXML11Pipeline()` 方法的代码逻辑、作用和核心细节，我会从方法整体功能入手，拆解每个代码块的意图、逻辑和作用，让你清楚理解 XML 1.1 解析管道的配置过程。

### 一、方法整体解读
这段代码是 Xerces XML 解析器中**配置 XML 1.1 解析管道（Pipeline）** 的核心方法，“解析管道”可以理解为 XML 解析的“处理链条”——XML 数据会依次经过扫描器（Scanner）、处理器（Processor）、验证器（Validator）等组件，这个方法的作用就是**组装/切换这些组件，使其适配 XML 1.1 规范**，同时重置相关组件的状态。

方法是 `protected` 修饰的，说明它是供子类继承/调用的内部配置方法，不是对外暴露的 API。

### 二、逐段详细解释

#### 1. 方法注释与声明
```java
/**
 *  Configures the XML 1.1 pipeline.
 *  Note: this method also resets the new XML11 components.
 */
protected void configureXML11Pipeline() {
```
- 注释核心：
    - 功能：配置 XML 1.1 解析管道；
    - 注意点：该方法会**重置**新的 XML 1.1 组件（确保组件状态干净，避免旧数据干扰）。
- 访问修饰符 `protected`：仅限当前类和子类访问，属于内部配置逻辑。

#### 2. 第一步：切换 XML 1.1 数据类型验证器工厂
```java
if (fCurrentDVFactory != fXML11DatatypeFactory) {
    fCurrentDVFactory = fXML11DatatypeFactory;
    setProperty(DATATYPE_VALIDATOR_FACTORY, fCurrentDVFactory);
}
```
- 核心意图：确保当前使用的“数据类型验证器工厂”是 XML 1.1 版本的（XML 1.0 和 1.1 的数据类型规则有差异，比如 1.1 支持更多字符）。
- 变量说明：
    - `fCurrentDVFactory`：当前生效的“数据类型验证器工厂”（DV = Datatype Validator）；
    - `fXML11DatatypeFactory`：XML 1.1 专用的数类型验证器工厂；
    - `DATATYPE_VALIDATOR_FACTORY`：属性常量，标识“数据类型验证器工厂”这个配置项。
- 逻辑：如果当前工厂不是 XML 1.1 版本，就切换过去，并通过 `setProperty` 方法将新工厂注册到解析器中。

#### 3. 第二步：切换 XML 1.1 DTD 扫描器和处理器
```java
if (fCurrentDTDScanner != fXML11DTDScanner) {
    fCurrentDTDScanner = fXML11DTDScanner;
    setProperty(DTD_SCANNER, fCurrentDTDScanner);
    setProperty(DTD_PROCESSOR, fXML11DTDProcessor);
}
```
- 核心意图：切换 DTD（文档类型定义）相关的核心组件到 XML 1.1 版本。
- 变量说明：
    - `fCurrentDTDScanner`：当前 DTD 扫描器（负责读取/扫描 DTD 内容）；
    - `fXML11DTDScanner`：XML 1.1 DTD 扫描器；
    - `fXML11DTDProcessor`：XML 1.1 DTD 处理器（负责解析 DTD 语法、处理实体/元素声明等）；
    - `DTD_SCANNER`/`DTD_PROCESSOR`：对应“DTD 扫描器”“DTD 处理器”的属性常量。
- 逻辑：如果当前 DTD 扫描器不是 XML 1.1 版本，就切换扫描器和处理器，并注册到解析器配置中。

#### 4. 第三步：建立 DTD 扫描器 ↔ 处理器的双向关联
```java
fXML11DTDScanner.setDTDHandler(fXML11DTDProcessor);
fXML11DTDProcessor.setDTDSource(fXML11DTDScanner);
fXML11DTDProcessor.setDTDHandler(fDTDHandler);
if (fDTDHandler != null) {
    fDTDHandler.setDTDSource(fXML11DTDProcessor);
}
```
- 核心意图：组装 DTD 处理的“小管道”，让组件之间能传递数据/事件（解析器组件通常通过 `setXxxSource`/`setXxxHandler` 建立双向关联）。
- 关键方法说明：
    - `setDTDHandler`：设置“处理结果的接收者”（扫描器扫完 DTD 内容后，把事件交给处理器）；
    - `setDTDSource`：设置“数据来源”（处理器知道自己的输入来自哪个扫描器）。
- 逐行拆解：
    1. 让 XML 1.1 DTD 扫描器的处理结果交给 XML 1.1 DTD 处理器；
    2. 让 XML 1.1 DTD 处理器知道自己的数据源是 XML 1.1 DTD 扫描器；
    3. 让 XML 1.1 DTD 处理器的处理结果交给外部的 `fDTDHandler`（用户自定义的 DTD 事件处理器）；
    4. 如果外部 DTD 处理器不为空，告诉它数据源是 XML 1.1 DTD 处理器。

#### 5. 第四步：建立 DTD 内容模型的双向关联
```java
fXML11DTDScanner.setDTDContentModelHandler(fXML11DTDProcessor);
fXML11DTDProcessor.setDTDContentModelSource(fXML11DTDScanner);
fXML11DTDProcessor.setDTDContentModelHandler(fDTDContentModelHandler);
if (fDTDContentModelHandler != null) {
    fDTDContentModelHandler.setDTDContentModelSource(fXML11DTDProcessor);
}
```
- 核心意图：和上一步类似，但聚焦于“DTD 内容模型”（比如元素的嵌套规则、属性约束等）的组件关联。
- 变量说明：
    - `fDTDContentModelHandler`：用户自定义的 DTD 内容模型事件处理器；
    - `setDTDContentModelHandler`/`setDTDContentModelSource`：专门用于内容模型的“处理器/数据源”关联方法。
- 逻辑：和 DTD 核心组件的关联逻辑一致，形成“扫描器 → 处理器 → 外部处理器”的内容模型处理链条。

#### 6. 第五步：配置 XML 1.1 文档解析管道（分“命名空间开启/关闭”分支）
这部分是核心，根据是否开启“命名空间（Namespaces）”功能，配置不同的文档解析组件。

##### 分支 1：开启命名空间（NAMESPACES = true）
```java
if (fFeatures.get(NAMESPACES) == Boolean.TRUE) {
    if (fCurrentScanner != fXML11NSDocScanner) {
        fCurrentScanner = fXML11NSDocScanner;
        setProperty(DOCUMENT_SCANNER, fXML11NSDocScanner);
        setProperty(DTD_VALIDATOR, fXML11NSDTDValidator);
    }

    fXML11NSDocScanner.setDTDValidator(fXML11NSDTDValidator);
    fXML11NSDocScanner.setDocumentHandler(fXML11NSDTDValidator);
    fXML11NSDTDValidator.setDocumentSource(fXML11NSDocScanner);
    fXML11NSDTDValidator.setDocumentHandler(fDocumentHandler);

    if (fDocumentHandler != null) {
        fDocumentHandler.setDocumentSource(fXML11NSDTDValidator);
    }
    fLastComponent = fXML11NSDTDValidator;
}
```
- 核心意图：配置**支持命名空间**的 XML 1.1 文档解析管道。
- 变量说明：
    - `fFeatures.get(NAMESPACES)`：获取“是否开启命名空间”的配置（XML 解析的核心特性之一）；
    - `fXML11NSDocScanner`：XML 1.1 命名空间感知的文档扫描器（NS = Namespace）；
    - `fXML11NSDTDValidator`：XML 1.1 命名空间感知的 DTD 验证器；
    - `fDocumentHandler`：用户自定义的文档事件处理器（接收解析后的 XML 节点事件）；
    - `fLastComponent`：记录管道的最后一个组件（方便后续追加其他组件，比如 Schema 验证器）。
- 逻辑：
    1. 切换文档扫描器到 XML 1.1 命名空间版本，并注册 DTD 验证器；
    2. 建立“扫描器 → 命名空间 DTD 验证器 → 外部文档处理器”的关联；
    3. 记录最后一个组件为命名空间 DTD 验证器。

##### 分支 2：关闭命名空间（NAMESPACES = false）
```java
else {
    // create components
    if (fXML11DocScanner == null) {
        // non namespace document pipeline
        fXML11DocScanner = new XML11DocumentScannerImpl();
        addXML11Component(fXML11DocScanner);
        fXML11DTDValidator = new XML11DTDValidator();
        addXML11Component(fXML11DTDValidator);
    }
    if (fCurrentScanner != fXML11DocScanner) {
        fCurrentScanner = fXML11DocScanner;
        setProperty(DOCUMENT_SCANNER, fXML11DocScanner);
        setProperty(DTD_VALIDATOR, fXML11DTDValidator);
    }
    fXML11DocScanner.setDocumentHandler(fXML11DTDValidator);
    fXML11DTDValidator.setDocumentSource(fXML11DocScanner);
    fXML11DTDValidator.setDocumentHandler(fDocumentHandler);

    if (fDocumentHandler != null) {
        fDocumentHandler.setDocumentSource(fXML11DTDValidator);
    }
    fLastComponent = fXML11DTDValidator;
}
```
- 核心意图：配置**不支持命名空间**的 XML 1.1 文档解析管道。
- 关键差异：
    1. 先判断组件是否为空，如果为空则**新建** XML 1.1 文档扫描器和 DTD 验证器（命名空间版本的组件可能已提前初始化，非命名空间版本需要懒加载）；
    2. 使用 `XML11DocumentScannerImpl`（非命名空间扫描器）替代命名空间扫描器；
    3. 组件关联逻辑简化（无需处理命名空间相关的验证）；
    4. 最后一个组件是普通的 XML 1.1 DTD 验证器。
- 辅助方法：`addXML11Component`：将新建的 XML 1.1 组件注册到解析器的组件管理器中（方便统一管理/重置）。

#### 7. 第六步：追加 XML Schema 验证器（如果开启 Schema 验证）
```java
if (fFeatures.get(XMLSCHEMA_VALIDATION) == Boolean.TRUE) {
    // If schema validator was not in the pipeline insert it.
    if (fSchemaValidator == null) {
        fSchemaValidator = new XMLSchemaValidator();
        // add schema component
        setProperty(SCHEMA_VALIDATOR, fSchemaValidator);
        addCommonComponent(fSchemaValidator);
        fSchemaValidator.reset(this);
        // add schema message formatter
        if (fErrorReporter.getMessageFormatter(XSMessageFormatter.SCHEMA_DOMAIN) == null) {
            XSMessageFormatter xmft = new XSMessageFormatter();
            fErrorReporter.putMessageFormatter(XSMessageFormatter.SCHEMA_DOMAIN, xmft);
        }
    }

    fLastComponent.setDocumentHandler(fSchemaValidator);
    fSchemaValidator.setDocumentSource(fLastComponent);
    fSchemaValidator.setDocumentHandler(fDocumentHandler);
    if (fDocumentHandler != null) {
        fDocumentHandler.setDocumentSource(fSchemaValidator);
    }
    fLastComponent = fSchemaValidator;
}
```
- 核心意图：如果开启了“XML Schema 验证”功能，就把 Schema 验证器追加到解析管道的末尾。
- 逻辑拆解：
    1. 检查 `XMLSCHEMA_VALIDATION` 特性是否开启；
    2. 如果 Schema 验证器未初始化，就新建实例，并：
        - 注册到解析器配置中（`setProperty`）；
        - 添加到公共组件管理器（`addCommonComponent`）；
        - 重置验证器状态（`reset`）；
        - 注册 Schema 错误消息格式化器（`XSMessageFormatter`，用于将 Schema 验证错误转换为可读信息）；
    3. 将 Schema 验证器追加到管道末尾：
        - 让管道最后一个组件（DTD 验证器）的输出交给 Schema 验证器；
        - Schema 验证器的输出交给外部文档处理器；
        - 更新 `fLastComponent` 为 Schema 验证器。

#### 8. 方法结束
```java
} // configureXML11Pipeline()
```
标准的方法结束标识，注释提升可读性。

### 三、核心设计思路补充
1. **组件化**：解析器被拆分为扫描器、处理器、验证器等独立组件，通过 `setXxxSource`/`setXxxHandler` 组装成管道；
2. **版本适配**：区分 XML 1.0/1.1 组件，通过该方法切换到 1.1 版本；
3. **特性开关**：根据命名空间、Schema 验证等特性，动态组装不同的管道；
4. **懒加载**：非命名空间的 XML 1.1 组件仅在需要时初始化，节省资源；
5. **状态重置**：确保切换组件时状态干净，避免旧数据干扰。

### 总结
1. `configureXML11Pipeline` 是 Xerces 解析器配置 XML 1.1 解析管道的核心方法，核心逻辑是**切换 XML 1.1 版本的组件，并组装成解析链条**。
2. 方法分步骤配置：切换数据类型工厂 → 配置 DTD 组件 → 按命名空间开关配置文档解析组件 → 按需追加 Schema 验证器。
3. 组件间通过 `setXxxSource`/`setXxxHandler` 建立双向关联，形成“扫描 → 处理 → 验证 → 外部回调”的解析流程，同时支持特性动态开关（命名空间、Schema 验证）。
