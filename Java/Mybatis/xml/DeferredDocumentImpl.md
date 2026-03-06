

## 目录




## 资料

[xerces-j-api](https://xerces.apache.org/xerces-j/apiDocs/org/apache/xerces/dom/DeferredDocumentImpl.html)




## DeferredDocumentImpl

### 1. 核心需求理解
你希望我详细解释这段 Java 代码的含义、功能和核心设计思路。这段代码是 Xerces 解析器中 `DeferredDocumentImpl` 类的实现，用于**延迟加载（Deferred）** 方式处理 XML/HTML 文档的 DOM 树构建，是 DOM 解析的底层核心实现。

### 2. 详细代码解释
#### 2.1 类的整体定位与设计背景
```java
public class DeferredDocumentImpl
    extends DocumentImpl
    implements DeferredNode {
```
- **继承关系**：
    - 继承 `DocumentImpl`：`DocumentImpl` 是 DOM 规范中 `Document` 接口的基础实现，提供了文档节点的核心能力。
    - 实现 `DeferredNode`：这是 Xerces 内部的延迟节点接口，核心是**延迟加载**——解析 XML 时<span style="color:#ff6600; font-weight:bold;">不立即创建完整的 Java 对象树，而是先把节点数据存入数组表</span>，等到真正访问节点时才实例化对象，大幅降低内存占用。
- **核心作用**：作为 DOM 文档的“延迟实现”，是整个延迟 DOM 树的根节点，负责管理所有延迟节点的存储、创建和数据访问。

#### 2.2 注释与基础说明
开头的 Javadoc 注释明确了：
- `Document` 接口代表整个 HTML/XML 文档，是文档树的根，提供文档数据的核心访问入口。
- 节点（元素、文本、注释等）不能脱离 Document 存在，因此 Document 还需提供创建这些节点的工厂方法。
- 该类从 DOM Level 1 开始支持，最后修改于 2019 年，且是 Xerces 内部实现（`@xerces.internal`）。

#### 2.3 常量定义（核心是分块存储设计）
```java
// 分块移位值：用于计算节点属于哪个分块
protected static final int CHUNK_SHIFT = 8;           // 2^8 = 256
// 每个分块的大小：256 个节点
protected static final int CHUNK_SIZE = (1 << CHUNK_SHIFT);
// 分块掩码：用于计算节点在分块内的索引
protected static final int CHUNK_MASK = CHUNK_SIZE - 1;
// 初始分块数量：32 个分块（32*256=8192 个初始节点容量）
protected static final int INITIAL_CHUNK_COUNT = (1 << (13 - CHUNK_SHIFT));
```
- **设计目的**：为了高效管理大量节点，采用**二维数组分块存储**（而非一维数组）。
    - 一维数组的问题：如果初始容量太小，频繁扩容会影响性能；如果太大，闲置时浪费内存。
    - 分块方案：把节点数据<span style="color:#ff6600; font-weight:bold;">拆分成多个大小为 256 的“块”</span>，按需扩容（新增块），兼顾内存效率和性能。
    - 计算逻辑：
        - 节点全局索引 `nodeIndex` → 分块索引 `chunk = nodeIndex >> 8`（右移 8 位，等价于除以 256）。
        - 节点在分块内的索引 `index = nodeIndex & 0xFF`（与 255 按位与，等价于取模 256）。

#### 2.4 核心数据结构（延迟节点的存储表）
这些二维数组是<span style="color:#ff6600; font-weight:bold;">延迟 DOM 的核心</span>——所有节点数据先存在这里，<span style="color:#ff6600; font-weight:bold;">而非直接创建 Node 对象</span>：
```java
/** 节点总数 */
protected transient int fNodeCount = 0;
/** 节点类型（如 DOCUMENT_NODE、ELEMENT_NODE 等） */
protected transient int fNodeType[][];
/** 节点名称（如元素名、属性名） */
protected transient Object fNodeName[][];
/** 节点值（如文本节点内容、属性值） */
protected transient Object fNodeValue[][];
/** 父节点的索引 */
protected transient int fNodeParent[][];
/** 最后一个子节点的索引 */
protected transient int fNodeLastChild[][];
/** 前一个兄弟节点的索引 */
protected transient int fNodePrevSib[][];
/** 节点的命名空间 URI */
protected transient Object fNodeURI[][];
/** 额外数据的索引（存储节点的扩展信息） */
protected transient int fNodeExtra[][];

/** ID 相关：ID 名称数组 */
protected transient int fIdCount;
protected transient String fIdName[];
/** ID 相关：ID 对应的元素索引 */
protected transient int fIdElement[];

/** 是否启用命名空间支持 */
protected boolean fNamespacesEnabled = false;

// 字符串缓存相关：优化内存，减少字符串对象创建
private transient final StringBuilder fBufferStr = new StringBuilder();
private transient final List<String> fStrChunks = new ArrayList<>();
```
- **关键特性**：
    - `transient` 修饰：这些数据不需要序列化（DOM 节点的序列化有单独逻辑）。
    - 二维数组 `[chunk][index]`：每个维度对应<span style="color:#ff6600; font-weight:bold;">“分块索引”和“块内索引”</span>，比如 `fNodeType[2][10]` 表示第 2 个分块中第 10 个节点的类型。
    - 字符串优化：`fBufferStr` 和 `fStrChunks` 用于批量处理字符串，避免频繁创建小字符串对象，降低内存开销。

#### 2.5 构造方法
```java
// 无参构造：默认不启用命名空间
public DeferredDocumentImpl() { this(false); }

// 带命名空间开关的构造
public DeferredDocumentImpl(boolean namespacesEnabled) {
    this(namespacesEnabled, false);
}

// 核心构造：启用命名空间 + 语法访问
public DeferredDocumentImpl(boolean namespaces, boolean grammarAccess) {
    super(grammarAccess);
    // 标记需要同步数据/子节点（延迟加载的关键：访问时才同步）
    needsSyncData(true);
    needsSyncChildren(true);
    fNamespacesEnabled = namespaces;
}
```
- **核心逻辑**：
    - 调用父类 `DocumentImpl` 的构造，初始化基础文档能力。
    - `needsSyncData(true)`/`needsSyncChildren(true)`：标记该文档的节点数据和子节点关系需要“延迟同步”——直到用户调用 `getNodeValue()`、`getChildNodes()` 等方法时，才从二维数组中加载数据并创建对应的 Node 对象。

#### 2.6 核心方法
##### （1）获取 DOM 实现能力
```java
public DOMImplementation getImplementation() {
    return DeferredDOMImplementationImpl.getDOMImplementation();
}
```
- 返回单例的 `DeferredDOMImplementationImpl`，描述当前 DOM 实现的能力（比如是否支持 DOM Level 2、命名空间等）。

##### （2）命名空间开关
```java
boolean getNamespacesEnabled() { return fNamespacesEnabled; }
void setNamespacesEnabled(boolean enable) { fNamespacesEnabled = enable; }
```
- 控制当前文档是否处理 XML 命名空间（解析时是否解析 `xmlns` 相关属性）。

##### （3）延迟节点创建工厂方法（核心）
这类方法不直接创建 Node 对象，而是把节点数据存入二维数组，并返回**节点索引**（后续通过索引访问/创建对象）。

###### ① 创建文档节点
```java
public int createDeferredDocument() {
    int nodeIndex = createNode(Node.DOCUMENT_NODE);
    return nodeIndex;
}
```
- `createNode(Node.DOCUMENT_NODE)`：内部逻辑是分配新的节点索引，初始化该索引对应的 `fNodeType` 为 DOCUMENT_NODE，返回全局索引。

###### ② 创建文档类型（DOCTYPE）节点
```java
public int createDeferredDocumentType(String rootElementName,
                                      String publicId, String systemId) {
    // 1. 创建节点，分配索引
    int nodeIndex = createNode(Node.DOCUMENT_TYPE_NODE);
    int chunk     = nodeIndex >> CHUNK_SHIFT; // 计算分块
    int index     = nodeIndex & CHUNK_MASK;   // 计算块内索引

    // 2. 存储节点数据到二维数组
    setChunkValue(fNodeName, rootElementName, chunk, index); // 根元素名
    setChunkValue(fNodeValue, publicId, chunk, index);      // 公共ID
    setChunkValue(fNodeURI, systemId, chunk, index);        // 系统ID

    return nodeIndex;
}
```
- `setChunkValue`：内部实现是确保目标分块存在（不存在则创建），然后把值存入 `[chunk][index]` 位置。

###### ③ 设置 DOCTYPE 内部子集
```java
public void setInternalSubset(int doctypeIndex, String subset) {
    int chunk     = doctypeIndex >> CHUNK_SHIFT;
    int index     = doctypeIndex & CHUNK_MASK;

    // 创建额外数据节点存储内部子集（避免主节点数据溢出）
    int extraDataIndex = createNode(Node.DOCUMENT_TYPE_NODE);
    int echunk = extraDataIndex >> CHUNK_SHIFT;
    int eindex = extraDataIndex & CHUNK_MASK;
    
    // 关联主节点和额外数据节点
    setChunkIndex(fNodeExtra, extraDataIndex, chunk, index);
    // 存储内部子集内容
    setChunkValue(fNodeValue, subset, echunk, eindex);
}
```
- 设计思路：DOCTYPE 的内部子集（比如 `<!ELEMENT ...>`）内容较长，单独用一个“额外数据节点”存储，通过 `fNodeExtra` 关联到主 DOCTYPE 节点。

###### ④ 创建符号（Notation）节点
```java
public int createDeferredNotation(String notationName,
                                  String publicId, String systemId, String baseURI) {
    // 1. 创建主节点
    int nodeIndex = createNode(Node.NOTATION_NODE);
    int chunk     = nodeIndex >> CHUNK_SHIFT;
    int index     = nodeIndex & CHUNK_MASK;

    // 2. 创建额外数据节点（存储baseURI）
    int extraDataIndex = createNode(Node.NOTATION_NODE);
    int echunk = extraDataIndex >> CHUNK_SHIFT;
    int eindex = extraDataIndex & CHUNK_MASK;

    // 3. 存储主节点数据
    setChunkValue(fNodeName, notationName, chunk, index); // 符号名
    setChunkValue(fNodeValue, publicId, chunk, index);    // 公共ID
    setChunkValue(fNodeURI, systemId, chunk, index);      // 系统ID

    // 4. 关联额外数据节点 + 存储baseURI
    setChunkIndex(fNodeExtra, extraDataIndex, chunk, index);
    setChunkValue(fNodeName, baseURI, echunk, eindex);

    return nodeIndex;
}
```

###### ⑤ 创建实体（Entity）节点
```java
public int createDeferredEntity(String entityName, String publicId,
                                String systemId, String notationName,
                                String baseURI) {
    // 1. 创建主节点
    int nodeIndex = createNode(Node.ENTITY_NODE);
    int chunk     = nodeIndex >> CHUNK_SHIFT;
    int index     = nodeIndex & CHUNK_MASK;

    // 2. 创建第一个额外数据节点（存储notation、版本、编码）
    int extraDataIndex = createNode(Node.ENTITY_NODE);
    int echunk = extraDataIndex >> CHUNK_SHIFT;
    int eindex = extraDataIndex & CHUNK_MASK;

    // 3. 存储主节点数据
    setChunkValue(fNodeName, entityName, chunk, index);   // 实体名
    setChunkValue(fNodeValue, publicId, chunk, index);    // 公共ID
    setChunkValue(fNodeURI, systemId, chunk, index);      // 系统ID
    setChunkIndex(fNodeExtra, extraDataIndex, chunk, index); // 关联额外节点

    // 4. 第一个额外节点存储 notation、版本、编码
    setChunkValue(fNodeName, notationName, echunk, eindex); // 关联的符号名
    setChunkValue(fNodeValue, null, echunk, eindex);        // 版本（L3，暂未实现）
    setChunkValue(fNodeURI, null, echunk, eindex);          // 编码（L3，暂未实现）

    // 5. 创建第二个额外数据节点（存储baseURI）
    int extraDataIndex2 = createNode(Node.ENTITY_NODE);
    int echunk2 = extraDataIndex2 >> CHUNK_SHIFT;
    int eindex2 = extraDataIndex2 & CHUNK_MASK;
    setChunkIndex(fNodeExtra, extraDataIndex2, echunk, eindex); // 关联到第一个额外节点
    setChunkValue(fNodeName, baseURI, echunk2, eindex2);       // 存储baseURI

    return nodeIndex;
}
```
- 实体节点的属性更多（名称、公共ID、系统ID、符号名、baseURI、版本、编码），因此用**两级额外数据节点**存储，通过 `fNodeExtra` 链式关联。

### 3. 总结
#### 核心关键点
1. **延迟加载核心设计**：
    - 解析 XML 时不直接创建 Node 对象，而是将节点数据存入**分块二维数组**，仅记录节点索引，大幅降低内存占用（尤其适合大文档）。
    - 只有当用户访问节点（如 `getNodeName()`）时，才根据索引从数组中加载数据，实例化对应的 Node 对象。

2. **分块存储优化**：
    - 采用 `CHUNK_SIZE=256` 的分块二维数组，而非一维数组，兼顾内存利用率（按需扩容）和访问性能（数组随机访问）。

3. **数据扩展设计**：
    - 通过 `fNodeExtra` 关联“额外数据节点”，存储超长/扩展属性（如 DOCTYPE 内部子集、实体的 baseURI），避免主节点数据结构臃肿。

#### 应用场景
该类是 Xerces 解析器的核心底层实现，主要用于：
- 解析大体积 XML 文档时的内存优化。
- 对性能/内存敏感的场景（如服务器端解析大量 XML）。
- 符合 DOM 规范的延迟节点访问需求。
