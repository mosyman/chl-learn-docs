
## InputStreamReader

### 一、整体功能概述
`InputStreamReader` 是 Java IO 体系中<span style="color:#ff6600; font-weight:bold;">字节流到字符流的桥梁</span>，它的核心作用是将底层字节输入流（`InputStream`）读取的字节数据，<span style="color:#ff6600; font-weight:bold;">按照指定的字符编码（Charset）解码成字符</span>，解决了字节和字符之间的转换问题。
这个类继承自 `Reader`（字符输入流的抽象父类），所有<span style="color:#ff6600; font-weight:bold;">核心的读写逻辑都委托给 `StreamDecoder` 实现</span>，自身只做封装和对外提供统一接口。

### 二、逐部分详细解释

#### 1. 类注释解析（核心说明）
注释里的关键信息：
- **核心定位**：字节流 → 字符流的桥梁，通过指定的 `Charset` 解码字节为字符（支持指定编码、显式传入编码、使用系统默认编码）。
- **读取特性**：每次调用 `read()` 方法可能从底层字节流读取1个或多个字节（因为一个字符可能对应多个字节，比如 UTF-8 中中文占3个字节）；为了效率，会提前读取更多字节到缓冲区，而非每次只读满足当前需求的字节。
- **性能建议**：建议包装在 `BufferedReader` 中使用（减少底层IO调用次数），这是IO操作的最佳实践。

#### 2. 核心成员变量
```java
private final StreamDecoder sd;
```
- `StreamDecoder` 是 Java 内部的字符解码工具类（属于 sun.nio.cs 包，不对外暴露），`InputStreamReader` 的所有核心功能（解码、读取、关闭）都通过这个对象完成，这是**委托模式**的典型应用——把复杂的解码逻辑交给专门的类处理，自身只做接口封装。

#### 3. 构造方法（4个重载）
所有构造方法的核心是初始化 `StreamDecoder` 对象，区别仅在于编码的指定方式：

| 构造方法 | 作用 | 注意点 |
|----------|------|--------|
| `InputStreamReader(InputStream in)` | 使用**系统默认编码**创建对象 | 跨平台时编码可能不一致（比如Windows默认GBK，Linux默认UTF-8） |
| `InputStreamReader(InputStream in, String charsetName)` | 通过**编码名称**（如 "UTF-8"、"GBK"）创建 | 编码名称错误会抛出 `UnsupportedEncodingException`；需判空 `charsetName` |
| `InputStreamReader(InputStream in, Charset cs)` | 直接传入 `Charset` 对象（1.4新增） | 类型更安全，避免编码名称拼写错误；需判空 `cs` |
| `InputStreamReader(InputStream in, CharsetDecoder dec)` | 传入自定义的 `CharsetDecoder`（1.4新增） | 可自定义解码规则（如容错策略）；需判空 `dec` |

**示例**：
```java
// 使用系统默认编码
InputStreamReader reader1 = new InputStreamReader(new FileInputStream("test.txt"));
// 指定UTF-8编码（推荐）
InputStreamReader reader2 = new InputStreamReader(new FileInputStream("test.txt"), "UTF-8");
// 显式传入Charset对象（更安全）
InputStreamReader reader3 = new InputStreamReader(new FileInputStream("test.txt"), StandardCharsets.UTF_8);
```

#### 4. 核心方法解析
所有读写方法都直接调用 `StreamDecoder` 的对应方法，自身不做额外逻辑处理：

##### (1) `getEncoding()`
- 作用：返回当前流使用的字符编码名称（历史名称/规范名称）；如果流已关闭，返回 `null`。
- 示例：
  ```java
  InputStreamReader reader = new InputStreamReader(new FileInputStream("test.txt"), "UTF-8");
  System.out.println(reader.getEncoding()); // 输出 UTF8（历史名称）
  reader.close();
  System.out.println(reader.getEncoding()); // 输出 null
  ```

##### (2) 读取方法（3个重载）
| 方法 | 作用 | 返回值说明 |
|------|------|------------|
| `int read()` | 读取单个字符 | 成功返回字符的Unicode值（int类型），到达流末尾返回 `-1` |
| `int read(char[] cbuf, int off, int len)` | 读取字符到数组的指定位置 | 成功返回读取的字符数，到达末尾返回 `-1`；`off` 是数组起始下标，`len` 是读取长度 |
| `int read(CharBuffer target)` | 读取字符到 `CharBuffer`（1.5新增） | 成功返回读取的字符数，到达末尾返回 `-1` |

**读取示例**：
```java
try (InputStreamReader reader = new InputStreamReader(new FileInputStream("test.txt"), StandardCharsets.UTF_8)) {
    char[] buf = new char[1024];
    int len;
    // 循环读取字符到数组，直到末尾
    while ((len = reader.read(buf, 0, buf.length)) != -1) {
        System.out.print(new String(buf, 0, len));
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

##### (3) `ready()`
- 作用：判断流是否准备好读取（缓冲区非空，或底层字节流有可用字节）。
- 注意：返回 `true` 表示**当前**可读取，但不保证后续读取不阻塞；返回 `false` 不代表流已结束。

##### (4) `close()`
- 作用：关闭流，释放底层资源（实际调用 `StreamDecoder.close()`）。
- 最佳实践：使用 `try-with-resources` 自动关闭（Java 7+），避免手动关闭遗漏。

### 三、设计思想与最佳实践
1. **委托模式**：`InputStreamReader` 仅作为“桥梁”，核心解码逻辑交给 `StreamDecoder`，符合“单一职责原则”。
2. **编码灵活性**：支持多种编码指定方式，兼顾易用性（字符串名称）和安全性（`Charset` 对象）。
3. **性能优化**：
    - 内部会提前读取字节到缓冲区，减少底层IO调用；
    - 官方建议搭配 `BufferedReader` 使用（增加字符缓冲区），进一步提升效率：
      ```java
      // 高效读取字符流的标准写法
      try (BufferedReader br = new BufferedReader(
              new InputStreamReader(new FileInputStream("test.txt"), StandardCharsets.UTF_8))) {
          String line;
          while ((line = br.readLine()) != null) {
              System.out.println(line);
          }
      } catch (IOException e) {
          e.printStackTrace();
      }
      ```

### 总结
1. `InputStreamReader` 的核心是**字节→字符的解码桥梁**，依赖 `StreamDecoder` 实现具体解码逻辑。
2. 使用时必须指定明确的字符编码（避免系统默认编码导致跨平台问题），推荐用 `StandardCharsets.UTF_8` 等枚举类。
3. 性能优化的关键是搭配 `BufferedReader`，减少底层IO操作次数，这是字符流读取的标准最佳实践。

