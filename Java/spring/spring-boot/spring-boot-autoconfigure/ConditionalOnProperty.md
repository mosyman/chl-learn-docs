
## 目录
- [@ConditionalOnProperty](#conditionalonproperty)
- [OnPropertyCondition](#onpropertycondition)



## @ConditionalOnProperty

### 一、注解核心作用
`@ConditionalOnProperty` 是 Spring Boot 提供的一个条件注解，核心作用是：**根据配置文件（如 `application.properties`/`application.yml`）中的指定属性是否存在、是否符合特定值，来决定是否加载某个 Bean（类/方法级别）**。
简单来说：满足属性条件就加载 Bean，不满足就不加载，这是实现「配置驱动 Bean 加载」的核心注解。

### 二、注解整体结构与元注解说明
先看注解的元注解部分，这决定了它的基础行为：
```java
@Retention(RetentionPolicy.RUNTIME)  // 运行时生效，Spring 能反射读取
@Target({ ElementType.TYPE, ElementType.METHOD })  // 可标注在类/方法上（类：控制整个配置类加载；方法：控制单个@Bean方法加载）
@Documented  // 生成文档时包含该注解
@Conditional(OnPropertyCondition.class)  // 核心：真正的条件判断逻辑在 OnPropertyCondition 类中实现
public @interface ConditionalOnProperty { ... }
```

### 三、核心属性详解
注解的每个属性都是控制条件匹配的关键，下面逐个拆解：

#### 1. `value()` - name 属性的别名
- **作用**：等价于 `name()` 属性，是为了简化书写（如果只需要指定属性名，不用写 `name = {"xxx"}`，直接写 `value = {"xxx"}` 甚至省略 `value=`）。
- **默认值**：空数组。
- **示例**：
  ```java
  // 等价于 @ConditionalOnProperty(name = "spring.datasource.enabled")
  @ConditionalOnProperty("spring.datasource.enabled")
  ```

#### 2. `prefix()` - 属性前缀
- **作用**：给 `name()`/`value()` 指定的属性名加统一前缀，前缀会自动补全末尾的 `.`（如果没写的话）。
- **规则**：前缀必须是「点分隔的单词」（如 `acme.system.feature`）。
- **示例**：
  ```java
  // 最终匹配的属性是 "app.config.my-value"（prefix自动补了.）
  @ConditionalOnProperty(prefix = "app.config", name = "my-value")
  ```

#### 3. `name()` - 要检查的属性名
- **核心作用**：指定需要检查的配置属性名（支持多个）。
- **规则**：
    - 如果配置了 `prefix`，属性名会和前缀拼接成完整的 key；
    - 属性名建议用「短横线命名法」（如 `my-long-property`）；
    - 如果指定了多个属性名，**所有属性都满足条件**，注解才会匹配成功。
- **示例**：
  ```java
  // 需同时满足 app.config.enable=true 和 app.config.mode=prod 才匹配
  @ConditionalOnProperty(prefix = "app.config", name = {"enable", "mode"}, havingValue = "true")
  ```

#### 4. `havingValue()` - 属性的预期值
- **作用**：指定属性必须匹配的「目标值」，如果不指定，默认规则是：**属性值不等于 `false` 即匹配**。
- **关键匹配规则（官方表格解读）**：
  
- | 配置文件中的属性值 | havingValue=""（默认） | havingValue="true" | havingValue="false" | havingValue="foo" |
  |--------------------|------------------------|--------------------|---------------------|-------------------|
  | "true"             | 匹配（yes）            | 匹配（yes）        | 不匹配（no）        | 不匹配（no）      |
  | "false"            | 不匹配（no）           | 不匹配（no）       | 匹配（yes）         | 不匹配（no）      |
  | "foo"              | 匹配（yes）            | 不匹配（no）       | 不匹配（no）        | 匹配（yes）       |
- **示例**：
  ```java
  // 要求 app.config.mode 的值必须是 "prod" 才匹配
  @ConditionalOnProperty(prefix = "app.config", name = "mode", havingValue = "prod")
  ```

#### 5. `matchIfMissing()` - 属性缺失时是否匹配
- **作用**：如果配置文件中**完全没有**指定的属性，是否仍然匹配（即是否加载 Bean）。
- **默认值**：`false`（属性缺失则不匹配，不加载 Bean）。
- **场景**：适用于「属性可选，没配置时也默认加载」的场景。
- **示例**：
  ```java
  // 如果 app.config.enable 不存在，也会匹配（加载 Bean）；如果存在，则值不能是 false
  @ConditionalOnProperty(prefix = "app.config", name = "enable", matchIfMissing = true)
  ```

### 四、关键使用注意事项
1. **集合属性不支持**：该注解无法可靠匹配「集合类型」的配置属性（如 `spring.example.values[0]`）。例如：
   ```java
   // 如果配置是 spring.example.values=[1,2,3]，注解能匹配；但如果只配置 spring.example.values[0]=1，注解无法匹配
   @ConditionalOnProperty(prefix = "spring", name = "example.values")
   ```
   这种场景建议自定义 `@Conditional` 注解。

2. **多属性校验规则**：`name()` 指定多个属性时，**所有属性都要满足条件**（相当于「与」逻辑），只要有一个不满足，整体就不匹配。

3. **属性值的类型转换**：配置文件中的值（如 `true`/`false`）会被解析为字符串，因此 `havingValue="true"` 匹配的是字符串 `"true"`，而非布尔类型 `true`（但 Spring 会自动兼容配置文件中的布尔写法）。

### 五、完整使用示例
#### 示例1：基础使用（默认规则）
```java
// 配置文件：app.feature.enable=true → 匹配，加载该配置类
// 配置文件：app.feature.enable=false → 不匹配，不加载
// 配置文件无该属性 → 不匹配（matchIfMissing 默认 false）
@Configuration
@ConditionalOnProperty(prefix = "app.feature", name = "enable")
public class FeatureAutoConfiguration {
    @Bean
    public FeatureService featureService() {
        return new FeatureService();
    }
}
```

#### 示例2：指定预期值 + 缺失时匹配
```java
// 配置文件：app.feature.mode=test → 匹配
// 配置文件无该属性 → 匹配（matchIfMissing=true）
// 配置文件：app.feature.mode=prod → 不匹配
@Bean
@ConditionalOnProperty(
    prefix = "app.feature",
    name = "mode",
    havingValue = "test",
    matchIfMissing = true
)
public TestService testService() {
    return new TestService();
}
```

### 总结
1. `@ConditionalOnProperty` 核心是**根据配置属性控制 Bean 加载**，属性满足条件则加载，否则不加载。
2. 核心属性：`prefix`（属性前缀）、`name`（属性名）、`havingValue`（预期值，默认非 false 即匹配）、`matchIfMissing`（属性缺失时是否匹配，默认 false）。
3. 注意事项：不支持集合属性、多属性是「与」逻辑、属性值以字符串形式匹配。


[目录](#目录)

## OnPropertyCondition
这个核心实现类的源码逻辑，它是 `@ConditionalOnProperty` 注解背后真正执行条件判断的核心组件。我会从类的整体定位、核心方法、内部逻辑一步步拆解，让你彻底理解它是如何完成属性匹配判断的。

### 一、类的核心定位
`OnPropertyCondition` 是 Spring Boot 中 `@ConditionalOnProperty` 注解的**底层实现类**，继承自 `SpringBootCondition`（Spring Boot 封装的条件判断基类），实现了 `Condition` 接口的 `getMatchOutcome` 方法，核心职责是：
根据 `@ConditionalOnProperty` 注解的配置，检查环境（`Environment`）中的属性是否符合条件，最终返回「匹配（match）」或「不匹配（noMatch）」的结果，决定是否加载对应的 Bean。

### 二、关键前置信息
1. **注解与类的关系**：`@ConditionalOnProperty` 上标注了 `@Conditional(OnPropertyCondition.class)`，这意味着 Spring 处理该注解时，会调用 `OnPropertyCondition` 的 `getMatchOutcome` 方法做条件判断。
2. **优先级**：`@Order(Ordered.HIGHEST_PRECEDENCE + 40)` 表示该条件判断的优先级很高（数字越小优先级越高），确保属性检查优先于其他条件执行。
3. **访问修饰符**：`class OnPropertyCondition` 是 `package-private`（默认访问权限），只在当前包内可见，无需外部调用，是框架内部实现。

### 三、核心方法拆解
#### 1. 入口方法：`getMatchOutcome()`
这是 `Condition` 接口的核心方法，负责整体的条件判断流程，返回 `ConditionOutcome`（包含匹配结果 + 提示信息）。

```java
@Override
public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
    // 步骤1：解析目标元素（类/方法）上的所有 @ConditionalOnProperty 注解属性
    List<AnnotationAttributes> allAnnotationAttributes = metadata.getAnnotations()
        .stream(ConditionalOnProperty.class.getName())
        .filter(MergedAnnotationPredicates.unique(MergedAnnotation::getMetaTypes))
        .map(MergedAnnotation::asAnnotationAttributes)
        .toList();
    
    // 步骤2：初始化两个集合，分别存储「不匹配」和「匹配」的提示信息
    List<ConditionMessage> noMatch = new ArrayList<>();
    List<ConditionMessage> match = new ArrayList<>();
    
    // 步骤3：遍历每个 @ConditionalOnProperty 注解，逐个判断是否匹配
    for (AnnotationAttributes annotationAttributes : allAnnotationAttributes) {
        ConditionOutcome outcome = determineOutcome(annotationAttributes, context.getEnvironment());
        // 根据单个注解的判断结果，将提示信息放入对应集合
        (outcome.isMatch() ? match : noMatch).add(outcome.getConditionMessage());
    }
    
    // 步骤4：最终结果判断：只要有一个注解不匹配，整体就不匹配
    if (!noMatch.isEmpty()) {
        return ConditionOutcome.noMatch(ConditionMessage.of(noMatch));
    }
    return ConditionOutcome.match(ConditionMessage.of(match));
}
```

**关键逻辑解读**：
- 支持处理目标元素上**多个** `@ConditionalOnProperty` 注解（虽然实际很少这么用）；
- 「一票否决」规则：只要有一个注解的条件不满足，整体就返回「不匹配」；
- `ConditionMessage` 是 Spring Boot 封装的提示信息类，用于日志/错误提示，方便排查为什么匹配/不匹配。

#### 2. 核心判断方法：`determineOutcome()`
该方法是单个 `@ConditionalOnProperty` 注解的判断逻辑核心，接收注解属性和环境（`PropertyResolver`，可以读取配置属性），返回单个注解的匹配结果。

```java
private ConditionOutcome determineOutcome(AnnotationAttributes annotationAttributes, PropertyResolver resolver) {
    // 步骤1：将注解属性封装为内部 Spec 类（Spec = 规格/配置）
    Spec spec = new Spec(annotationAttributes);
    // 步骤2：初始化两个集合：缺失的属性、值不匹配的属性
    List<String> missingProperties = new ArrayList<>();
    List<String> nonMatchingProperties = new ArrayList<>();
    // 步骤3：核心逻辑：收集缺失/不匹配的属性
    spec.collectProperties(resolver, missingProperties, nonMatchingProperties);
    
    // 步骤4：判断结果
    // 情况1：有缺失的属性 → 不匹配（除非 matchIfMissing=true，该逻辑在 collectProperties 中处理）
    if (!missingProperties.isEmpty()) {
        return ConditionOutcome.noMatch(ConditionMessage.forCondition(ConditionalOnProperty.class, spec)
            .didNotFind("property", "properties")
            .items(Style.QUOTE, missingProperties));
    }
    // 情况2：有值不匹配的属性 → 不匹配
    if (!nonMatchingProperties.isEmpty()) {
        return ConditionOutcome.noMatch(ConditionMessage.forCondition(ConditionalOnProperty.class, spec)
            .found("different value in property", "different value in properties")
            .items(Style.QUOTE, nonMatchingProperties));
    }
    // 情况3：无缺失、无值不匹配 → 匹配
    return ConditionOutcome
        .match(ConditionMessage.forCondition(ConditionalOnProperty.class, spec).because("matched"));
}
```

**关键逻辑解读**：
- `Spec` 是内部静态类，用于封装注解的 `prefix`、`name`、`havingValue`、`matchIfMissing` 等属性，简化后续处理；
- `collectProperties` 是真正检查属性的核心方法，下面重点拆解。

#### 3. 内部核心类：`Spec`
`Spec` 是 `OnPropertyCondition` 的内部静态类，负责：
① 解析 `@ConditionalOnProperty` 的注解属性；
② 检查环境中的属性是否符合条件。

##### 3.1 Spec 构造方法（解析注解属性）
```java
Spec(AnnotationAttributes annotationAttributes) {
    // 步骤1：处理 prefix：如果有值且不以 . 结尾，自动补 .
    String prefix = annotationAttributes.getString("prefix").trim();
    if (StringUtils.hasText(prefix) && !prefix.endsWith(".")) {
        prefix = prefix + ".";
    }
    this.prefix = prefix;
    
    // 步骤2：获取 havingValue 属性
    this.havingValue = annotationAttributes.getString("havingValue");
    
    // 步骤3：获取 name/value 属性（value 是 name 的别名，二者互斥）
    this.names = getNames(annotationAttributes);
    
    // 步骤4：获取 matchIfMissing 属性
    this.matchIfMissing = annotationAttributes.getBoolean("matchIfMissing");
}
```

##### 3.2 `getNames()` 方法（处理 name/value 别名）
```java
private String[] getNames(Map<String, Object> annotationAttributes) {
    String[] value = (String[]) annotationAttributes.get("value");
    String[] name = (String[]) annotationAttributes.get("name");
    
    // 断言1：name 或 value 必须至少指定一个（否则注解无意义）
    Assert.state(value.length > 0 || name.length > 0,
            "The name or value attribute of @ConditionalOnProperty must be specified");
    
    // 断言2：name 和 value 不能同时指定（二者是别名，互斥）
    Assert.state(value.length == 0 || name.length == 0,
            "The name and value attributes of @ConditionalOnProperty are exclusive");
    
    // 返回：有 value 用 value，否则用 name
    return (value.length > 0) ? value : name;
}
```

**关键约束**：
- `@ConditionalOnProperty` 的 `name` 和 `value` 不能同时用，也不能都不用；
- 这解释了为什么使用注解时，要么写 `name = {"xxx"}`，要么写 `value = {"xxx"}`，要么省略 `value=` 直接写 `"xxx"`。

##### 3.3 `collectProperties()` 方法（检查属性核心逻辑）
```java
private void collectProperties(PropertyResolver resolver, List<String> missing, List<String> nonMatching) {
    // 遍历所有需要检查的属性名（name/value 指定的）
    for (String name : this.names) {
        // 拼接完整属性 key：prefix + name（如 prefix=app.config，name=enable → app.config.enable）
        String key = this.prefix + name;
        
        // 情况1：环境中包含该属性 → 检查值是否匹配
        if (resolver.containsProperty(key)) {
            if (!isMatch(resolver.getProperty(key), this.havingValue)) {
                // 值不匹配 → 加入 nonMatching 集合
                nonMatching.add(name);
            }
        }
        // 情况2：环境中不包含该属性 → 检查 matchIfMissing
        else {
            if (!this.matchIfMissing) {
                // matchIfMissing=false（默认）→ 加入 missing 集合（表示属性缺失）
                missing.add(name);
            }
            // 如果 matchIfMissing=true → 忽略，不加入 missing，视为匹配
        }
    }
}
```

##### 3.4 `isMatch()` 方法（值匹配规则）
```java
private boolean isMatch(String value, String requiredValue) {
    // 情况1：指定了 havingValue → 忽略大小写比较（如 "TRUE" 和 "true" 视为匹配）
    if (StringUtils.hasLength(requiredValue)) {
        return requiredValue.equalsIgnoreCase(value);
    }
    // 情况2：未指定 havingValue → 只要值不是 "false"（忽略大小写）就匹配
    return !"false".equalsIgnoreCase(value);
}
```

**核心匹配规则（源码级验证）**：
- 当 `havingValue` 为空（默认）时，属性值只要不是 `false`（包括 `true`、`foo`、`123` 等）都算匹配；
- 当指定 `havingValue` 时，属性值需要和 `havingValue` 忽略大小写完全一致才匹配；
- 这和之前 `@ConditionalOnProperty` 注解文档中的匹配表格完全对应。

### 四、完整执行流程（流程图）
```mermaid
flowchart TD
    A[Spring 扫描到 @ConditionalOnProperty 注解] --> B[调用 OnPropertyCondition.getMatchOutcome()]
    B --> C[解析所有 @ConditionalOnProperty 注解属性]
    C --> D[遍历每个注解属性]
    D --> E[封装为 Spec 类（处理 prefix/name/havingValue 等）]
    E --> F[遍历 Spec 中的属性名]
    F --> G{环境中是否包含该属性？}
    G -->|是| H[调用 isMatch 检查值是否匹配]
    G -->|否| I{matchIfMissing 是否为 true？}
    H -->|不匹配| J[加入 nonMatching 集合]
    H -->|匹配| F
    I -->|否| K[加入 missing 集合]
    I -->|是| F
    J --> L{遍历完所有属性？}
    K --> L
    L -->|是| M{missing/nonMatching 是否为空？}
    M -->|否| N[返回 noMatch + 提示信息]
    M -->|是| O[返回 match + 提示信息]
    N --> P[整体返回 noMatch，不加载 Bean]
    O --> Q[整体返回 match，加载 Bean]
```

### 五、关键细节补充
1. **忽略大小写**：`isMatch` 方法中用了 `equalsIgnoreCase`，意味着配置文件中 `app.config.enable=TRUE` 和 `app.config.enable=true` 效果一样；
2. **PropertyResolver**：是 Spring 读取配置的核心接口，`Environment` 继承了它，因此可以读取 `application.properties`/`yml`、系统属性、环境变量等所有 Spring 能识别的配置；
3. **错误提示**：`ConditionMessage` 会生成友好的提示，比如属性缺失时会提示 `Did not find property 'enable'`，值不匹配时会提示 `Found different value in property 'mode'`，方便排查问题。

### 总结
1. `OnPropertyCondition` 是 `@ConditionalOnProperty` 的底层实现，核心流程是「解析注解属性 → 检查环境属性 → 返回匹配结果」；
2. 核心规则：
    - `prefix` 自动补 `.`，`name/value` 互斥且必填；
    - 属性存在时检查值是否匹配（`havingValue` 为空则非 `false` 即匹配）；
    - 属性缺失时由 `matchIfMissing` 决定是否匹配；
3. 整体是「一票否决」逻辑：只要有一个属性不满足条件，就返回不匹配，不加载对应的 Bean。