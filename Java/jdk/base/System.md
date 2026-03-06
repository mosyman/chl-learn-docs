

## System.getProperty(String key)

### 一、方法核心定位
`System.getProperty(String key)`是Java `java.lang.System`类提供的静态方法，核心作用是：**读取JVM中指定“键（key）”对应的系统属性值（value）**。

系统属性（System Properties）是JVM启动时或运行中维护的一组“键值对”，比如`java.version`（JDK版本）、`user.dir`（当前工作目录）、`nacos.standalone`（你之前接触的Nacos单机模式参数）等，是Java程序获取运行环境配置的核心方式。

### 二、注释逐段解读（理解设计规则）
| 注释片段 | 核心含义 |
|----------|----------|
| `First, if there is a security manager, its checkPropertyAccess method is called with the key as its argument. This may result in a SecurityException.` | 安全校验优先：<br>1) 如果JVM中配置了**安全管理器（SecurityManager）**，会先调用其`checkPropertyAccess(key)`方法；<br>2) 若安全管理器禁止访问该系统属性，直接抛出`SecurityException`（保护敏感系统属性不被随意读取）。 |
| `If there is no current set of system properties, a set of system properties is first created and initialized in the same manner as for the getProperties method.` | 懒加载初始化：<br>如果当前JVM还没有初始化系统属性集合（`props`），会先调用`getProperties()`方法创建并初始化（加载JVM默认的系统属性，如`java.home`、`os.name`等）。 |
| `Changing a standard system property may have unpredictable results unless otherwise specified` | 标准系统属性不可随意修改：<br>JDK内置的标准系统属性（如`java.version`、`user.home`）若被修改，可能导致不可预期的问题（比如依赖`java.home`的类加载逻辑出错）。 |
| `@return the string value of the system property, or null if there is no property with that key.` | 返回值规则：<br>1) 存在该key则返回对应的字符串值；<br>2) 不存在则返回`null`（这也是`Boolean.getBoolean()`中解析`null`返回`false`的原因）。 |
| `@throws NullPointerException if key is null.` | 入参key为`null`时，直接抛`NullPointerException`（无容错，需调用方保证key非空）。 |
| `@throws IllegalArgumentException if key is empty.` | 入参key为空字符串（`""`）时，抛`IllegalArgumentException`（禁止空键查询）。 |

### 三、代码逐行解析（理解执行逻辑）
```java
public static String getProperty(String key) {
    // 步骤1：校验入参key的合法性（核心前置校验）
    checkKey(key);
    
    // 步骤2：获取安全管理器（JDK 17+中SecurityManager已标记为废弃，但仍保留逻辑）
    @SuppressWarnings("removal")
    SecurityManager sm = getSecurityManager();
    
    // 步骤3：安全校验（若有安全管理器，检查是否允许访问该属性）
    if (sm != null) {
        sm.checkPropertyAccess(key);
    }

    // 步骤4：从系统属性集合中获取值并返回（不存在则返回null）
    return props.getProperty(key);
}
```

#### 关键子方法/变量解读
##### 1. `checkKey(key)`：入参合法性校验
这是`System`类的私有静态方法，源码逻辑如下（核心作用是校验key是否为`null`或空字符串）：
```java
private static void checkKey(String key) {
    if (key == null) {
        throw new NullPointerException("key can't be null");
    }
    if (key.isEmpty()) {
        throw new IllegalArgumentException("key can't be empty");
    }
}
```
- 这就是注释中提到的`NullPointerException`和`IllegalArgumentException`的触发源头；
- 设计逻辑：系统属性的key必须是“非空非null的字符串”，避免无效查询。

##### 2. `getSecurityManager()`：获取安全管理器
- 安全管理器是JVM的“权限控制中心”，用于限制程序对系统资源的访问（如读取系统属性、创建文件、网络连接等）；
- JDK 17及以上版本中，`SecurityManager`已被废弃（因为大部分场景下不需要这么严格的权限控制），但源码仍保留兼容逻辑；
- 普通开发场景下（如本地调试、后端服务），安全管理器默认是`null`，因此步骤3的安全校验会跳过。

##### 3. `props`：系统属性的底层存储
- `props`是`System`类的私有静态变量，类型为`Properties`（继承自`Hashtable`，线程安全的键值对集合）；
- 初始化时机：第一次调用`getProperty()`或`getProperties()`时，懒加载初始化，加载JVM默认的系统属性；
- 存储内容：包含JDK内置属性（如`java.version`、`os.name`）、JVM启动参数（`-Dkey=value`）、程序运行时通过`System.setProperty()`设置的属性。

### 四、核心规则+实际使用示例
#### 1. 核心规则总结
| 场景 | 执行结果 |
|------|----------|
| 传入`key=null` | 调用`checkKey()`抛`NullPointerException` |
| 传入`key=""`（空字符串） | 调用`checkKey()`抛`IllegalArgumentException` |
| 安全管理器禁止访问该key | 抛`SecurityException` |
| key存在 | 返回对应的字符串值 |
| key不存在 | 返回`null` |
| 系统属性集合未初始化 | 先调用`getProperties()`初始化，再查询 |

#### 2. 实际使用示例
##### （1）读取JDK内置系统属性
```java
public class SystemPropertyDemo {
    public static void main(String[] args) {
        // 读取JDK版本
        String javaVersion = System.getProperty("java.version");
        System.out.println("JDK版本：" + javaVersion); // 输出：17.0.8（取决于你的JDK版本）
        
        // 读取操作系统名称
        String osName = System.getProperty("os.name");
        System.out.println("操作系统：" + osName); // 输出：Mac OS X / Windows 10 / Linux
        
        // 读取当前工作目录
        String userDir = System.getProperty("user.dir");
        System.out.println("当前工作目录：" + userDir); // 输出：你的项目根目录
        
        // 读取不存在的属性（返回null）
        String nonExist = System.getProperty("non.exist.key");
        System.out.println("不存在的属性：" + nonExist); // 输出：null
        
        // 传入null（抛NPE）
        try {
            System.getProperty(null);
        } catch (NullPointerException e) {
            System.out.println("入参为null：抛NullPointerException");
        }
        
        // 传入空字符串（抛IllegalArgumentException）
        try {
            System.getProperty("");
        } catch (IllegalArgumentException e) {
            System.out.println("入参为空字符串：抛IllegalArgumentException");
        }
    }
}
```

##### （2）读取JVM启动参数（和你之前的Nacos场景关联）
启动Java程序时，通过`-Dkey=value`设置系统属性：
```bash
# 启动Nacos时设置单机模式参数
java -Dnacos.standalone=true -jar nacos-server.jar
```
程序中读取该参数：
```java
// Nacos源码中读取单机模式参数
boolean isStandalone = Boolean.getBoolean("nacos.standalone");
// 等价于：
String standaloneValue = System.getProperty("nacos.standalone");
boolean isStandalone2 = Boolean.parseBoolean(standaloneValue);
```

### 五、异常与安全机制详解
#### 1. 异常类型及触发条件
| 异常类型 | 触发条件 | 处理建议 |
|----------|----------|----------|
| `NullPointerException` | 入参key为`null` | 调用前先判空：`if (key != null) { ... }` |
| `IllegalArgumentException` | 入参key为空字符串 | 调用前校验：`if (key != null && !key.isEmpty()) { ... }` |
| `SecurityException` | 安全管理器禁止访问该属性 | 1) 调整安全管理器的权限配置；<br>2) 避免访问受保护的系统属性（如`java.home`） |

#### 2. 安全管理器的作用（补充）
安全管理器的`checkPropertyAccess(key)`方法会校验：
- 是否允许访问该系统属性（比如某些敏感属性`sun.boot.path`默认禁止普通程序访问）；
- 若不允许，直接抛出`SecurityException`，阻止读取；
- 普通开发场景下，安全管理器默认关闭（`sm=null`），因此不会触发该异常；仅在安全严格的环境（如Applet、沙箱运行的程序）中才会启用。

### 六、和`Boolean.getBoolean()`的关联（回顾你之前的问题）
`Boolean.getBoolean(name)`的核心逻辑是：
```java
result = parseBoolean(System.getProperty(name));
```
两者的调用链路：
```
Boolean.getBoolean("nacos.standalone") → 
System.getProperty("nacos.standalone") → 
checkKey("nacos.standalone") → 
安全校验（跳过） → 
props.getProperty("nacos.standalone") → 
返回"true" → 
Boolean.parseBoolean("true") → 返回true
```
- 若`System.getProperty()`返回`null`（属性不存在），`parseBoolean(null)`返回`false`；
- 若`System.getProperty()`抛异常（如key为null），`Boolean.getBoolean()`会捕获异常并返回`false`。

### 总结
1. **核心功能**：`System.getProperty(key)`是读取JVM系统属性的核心方法，返回指定key对应的字符串值，不存在则返回`null`；
2. **前置校验**：先通过`checkKey()`校验key非null/非空，再通过安全管理器校验访问权限；
3. **异常规则**：key为null抛NPE、为空字符串抛IllegalArgumentException、权限不足抛SecurityException；
4. **使用场景**：读取JDK内置环境信息、JVM启动参数（如Nacos的单机模式）、自定义系统属性；
5. **关联知识点**：`Boolean.getBoolean()`是基于该方法的上层封装，专门用于将系统属性值解析为布尔值。

简单来说：这个方法就是“向JVM查询配置”——告诉JVM“我要查名为XXX的配置项”，JVM先校验你查的“名字”是否合法、你是否有权限查，然后返回配置值（没有就返回null）。