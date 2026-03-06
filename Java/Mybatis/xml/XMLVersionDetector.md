
## XMLVersionDetector

### 一、类整体解读
`XMLVersionDetector` 是 Xerces 解析器内部的**XML版本检测与扫描器切换类**，核心作用是：
1. 先通过 XML 1.1 扫描器读取 XML 文档的版本声明（比如 `<?xml version="1.0"?>` 或 `<?xml version="1.1"?>`）；
2. 根据检测到的版本，切换到对应的 XML 1.0/1.1 扫描器，并启动文档的解析流程。

类的访问修饰符是 `public`，但结合 `@xerces.internal` 注解，说明它仅用于 Xerces 内部实现，不建议外部直接使用。

### 二、逐部分详细解释

#### 1. 类级别的注释与注解
```java
/**
 * This class scans the version of the document to determine
 * which scanner to use: XML 1.1 or XML 1.0.
 * The version is scanned using XML 1.1. scanner.
 *
 * @xerces.internal
 *
 * @author Neil Graham, IBM
 * @author Elena Litani, IBM
 */
```
- 核心描述：
    - 功能：扫描文档的版本，决定使用 XML 1.1 还是 1.0 扫描器；
    - 关键细节：**无论文档实际版本是啥，先通过 XML 1.1 扫描器读取版本声明**（因为 XML 1.1 扫描器兼容解析 1.0 的版本声明，反之则不行）。
- `@xerces.internal`：Xerces 内部注解，表明该类仅用于解析器内部，外部依赖可能有版本兼容风险。
- `@author`：标注类的作者，属于代码元信息。

#### 2. 类声明
```java
public class XMLVersionDetector {
```
- `public` 修饰符：允许 Xerces 内部其他包的类访问（比如解析器核心类）；
- 类名 `XMLVersionDetector`：语义明确，即“XML版本检测器”。

#### 3. `startDocumentParsing` 方法注释与声明
```java
/**
 * Reset the reference to the appropriate scanner given the version of the
 * document and start document scanning.
 * @param scanner - the scanner to use
 * @param version - the version of the document (XML 1.1 or XML 1.0).
 */
public void startDocumentParsing(XMLEntityHandler scanner, short version){
```
- 注释核心：
    - 功能：根据文档版本重置对应的扫描器引用，并启动文档扫描；
    - 参数说明：
        - `scanner`：要使用的扫描器（实现了 `XMLEntityHandler` 接口，即你之前问过的实体处理器接口）；
        - `version`：文档的版本（短整型，取值为 1.0 或 1.1 的常量）。
- 方法修饰符 `public`：供解析器内部其他类调用（比如解析器入口类）；
- 参数类型：
    - `XMLEntityHandler`：扫描器实现了该接口，因为扫描器需要处理实体的开始/结束事件；
    - `short`：用短整型存储版本，节省内存（比 int 更轻量）。

#### 4. 第一步：根据版本设置扫描器版本
```java
if (version == Constants.XML_VERSION_1_0){
    fEntityManager.setScannerVersion(Constants.XML_VERSION_1_0);
}
else {
    fEntityManager.setScannerVersion(Constants.XML_VERSION_1_1);
}
```
- 核心意图：将“实体管理器”（`fEntityManager`）中的扫描器版本设置为检测到的文档版本。
- 变量/常量说明：
    - `fEntityManager`：该类的成员变量（未在这段代码中定义，但能推断是 `XMLEntityManager` 类型），负责管理 XML 实体的扫描、解析上下文；
    - `Constants.XML_VERSION_1_0`/`Constants.XML_VERSION_1_1`：Xerces 定义的常量，分别表示 XML 1.0 和 1.1 版本（短整型值，比如 0 代表 1.0，1 代表 1.1）；
    - `setScannerVersion`：`XMLEntityManager` 的方法，用于设置当前使用的扫描器版本。
- 逻辑：简单的分支判断，文档版本是 1.0 就设置 1.0 扫描器版本，否则默认设置为 1.1。

#### 5. 第二步：设置错误报告器的文档定位器
```java
// Make sure the locator used by the error reporter is the current entity scanner.
fErrorReporter.setDocumentLocator(fEntityManager.getEntityScanner());
```
- 注释解释：确保错误报告器使用的“定位器”是当前的实体扫描器（定位器用于报告错误发生的行号、列号等位置信息）。
- 变量/方法说明：
    - `fErrorReporter`：该类的成员变量（`XMLErrorReporter` 类型），负责收集和报告解析过程中的错误；
    - `setDocumentLocator`：`XMLErrorReporter` 的方法，设置错误定位器（`Locator` 接口实现类）；
    - `fEntityManager.getEntityScanner()`：获取实体管理器中当前的实体扫描器（`XMLEntityScanner` 类型），扫描器本身实现了定位器接口，能提供错误位置信息。
- 核心作用：让错误报告器能精准定位到 XML 文档中错误发生的位置（比如“第5行第3列语法错误”）。

#### 6. 第三步：注释说明扫描器重置的注意事项
```java
// Note: above we reset fEntityScanner in the entity manager, thus in startEntity
// in each scanner fEntityScanner field must be reset to reflect the change.
//
```
- 注释核心：
    - 提示：上面的代码重置了实体管理器中的 `fEntityScanner`（实体扫描器），因此每个扫描器的 `startEntity` 方法中，必须重置 `fEntityScanner` 字段，以反映这个变更；
    - 目的：避免扫描器使用旧的实体扫描器实例，导致解析上下文不一致（比如版本不匹配、位置信息错误）。

#### 7. 第四步：设置实体管理器的实体处理器
```java
fEntityManager.setEntityHandler(scanner);
```
- 核心意图：将传入的扫描器（`XMLEntityHandler` 实现类）设置为实体管理器的实体处理器。
- 方法说明：`setEntityHandler` 是 `XMLEntityManager` 的方法，用于指定处理实体事件的处理器（这里就是扫描器本身，因为扫描器实现了 `XMLEntityHandler`）。
- 作用：让实体管理器知道，后续的实体解析事件（比如实体开始/结束）由这个扫描器来处理。

#### 8. 第五步：启动实体扫描（文档解析的入口）
```java
scanner.startEntity(fXMLSymbol, fEntityManager.getCurrentResourceIdentifier(), fEncoding, null);
```
- 核心意图：调用扫描器的 `startEntity` 方法（你之前问过的 `XMLEntityHandler` 接口方法），启动文档实体的解析。
- 参数说明（对应 `startEntity` 方法的参数）：
    1. `fXMLSymbol`：该类的成员变量（`XMLSymbol` 类型），表示文档实体的名称（通常是“[xml]”或空，标识这是XML文档实体）；
    2. `fEntityManager.getCurrentResourceIdentifier()`：获取当前文档的资源标识符（`XMLResourceIdentifier` 类型），比如文档的URL/文件路径；
    3. `fEncoding`：该类的成员变量（`String` 类型），文档的编码格式（比如 UTF-8）；
    4. `null`：`Augmentations` 参数，这里不需要附加信息，传 null。
- 作用：触发扫描器开始解析文档实体，正式启动 XML 文档的解析流程。

#### 9. 方法与类结束
```java
    }
}
```
标准的方法和类结束语法。

### 三、核心设计思路补充
1. **版本兼容**：先通过 XML 1.1 扫描器读取版本声明，保证能兼容解析 1.0 和 1.1 的版本字符串；
2. **上下文统一**：通过实体管理器统一管理扫描器版本、实体处理器，确保解析上下文一致；
3. **错误定位**：将扫描器设置为错误报告器的定位器，保证错误信息的精准性；
4. **接口解耦**：扫描器实现 `XMLEntityHandler` 接口，实体管理器依赖接口而非具体实现，符合“面向接口编程”的设计原则。

### 总结
1. `XMLVersionDetector` 是 Xerces 内部的 XML 版本检测类，核心功能是根据文档版本切换扫描器，并启动文档解析。
2. `startDocumentParsing` 方法是核心入口：设置扫描器版本 → 配置错误定位器 → 绑定实体处理器 → 启动实体扫描。
3. 关键细节：通过实体管理器统一管理扫描器上下文，确保错误定位精准，且扫描器版本切换后上下文一致。

