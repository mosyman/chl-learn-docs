

你想了解 `@SpringBootApplication` 注解默认的包扫描范围、修改扫描范围的所有方式，以及背后的实现原理，我会从「是什么」「怎么改」「为什么」三个维度，把这个核心知识点讲透，帮你彻底理解 Spring Boot 包扫描的逻辑。

### 一、`@SpringBootApplication` 默认包扫描范围
#### 1. 核心结论
`@SpringBootApplication` 注解默认的包扫描范围是：**标注该注解的类所在的包，以及该包下的所有子包**。

#### 2. 举例说明
假设你的启动类如下：
```java
// 包路径：com.example.demo
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```
那么默认扫描范围是：
- `com.example.demo` 包下的所有类；
- `com.example.demo.xxx`（如 `com.example.demo.controller`、`com.example.demo.service`）等子包下的所有类。

如果你的组件（`@Controller`/`@Service`/`@Repository`/`@Component` 等）不在这个范围内，Spring Boot 不会自动扫描并注册这些 Bean。

### 二、修改包扫描范围的所有方式
Spring Boot 提供了 **3类核心方式** 来修改包扫描范围，覆盖「扩大扫描」「缩小扫描」「指定精准扫描」等场景，下面按优先级/常用程度排序：

#### 方式1：通过 `@SpringBootApplication` 的 `scanBasePackages` 属性（推荐）
这是最直接、最常用的方式，通过注解属性指定扫描的基础包，支持多个包路径。
```java
// 场景1：扩大扫描范围（扫描 com.example.demo + com.example.common 两个包）
@SpringBootApplication(scanBasePackages = {"com.example.demo", "com.example.common"})
public class DemoApplication { ... }

// 场景2：指定单个基础包
@SpringBootApplication(scanBasePackages = "com.example")
public class DemoApplication { ... }
```

#### 方式2：通过 `@SpringBootApplication` 的 `scanBasePackageClasses` 属性（类型安全）
通过指定「类/接口」的方式间接指定扫描包（扫描该类所在的包），优点是「类型安全」（包名重构时不会出错），适合严谨的项目。
```java
// 假设 com.example.common.utils.CommonUtils 是目标包下的类
// 扫描范围：com.example.common 包（CommonUtils 所在的包） + DemoApplication 所在的包
@SpringBootApplication(scanBasePackageClasses = {CommonUtils.class})
public class DemoApplication { ... }

// 多个类（扫描多个类所在的包）
@SpringBootApplication(scanBasePackageClasses = {CommonUtils.class, UserController.class})
public class DemoApplication { ... }
```

#### 方式3：显式使用 `@ComponentScan` 注解（灵活控制）
`@SpringBootApplication` 本身包含了 `@ComponentScan`，如果显式添加 `@ComponentScan`，会覆盖默认的扫描规则，支持更精细的配置（如排除某些包/类）。

##### 3.1 基础用法（指定扫描包）
```java
// 覆盖默认扫描，只扫描 com.example.common 包
@SpringBootApplication
@ComponentScan("com.example.common")
public class DemoApplication { ... }

// 多个扫描包 + 排除指定类/包
@SpringBootApplication
@ComponentScan(
    basePackages = {"com.example.demo", "com.example.common"},
    excludeFilters = {
        // 排除 com.example.demo.test 包下的所有组件
        @ComponentScan.Filter(type = FilterType.REGEX, pattern = "com.example.demo.test.*"),
        // 排除指定类
        @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE, value = TestService.class)
    }
)
public class DemoApplication { ... }
```

##### 3.2 缩小扫描范围（只扫描指定类）
```java
// 只扫描 UserController 和 UserService 两个类（精准控制）
@SpringBootApplication
@ComponentScan(basePackageClasses = {UserController.class, UserService.class})
public class DemoApplication { ... }
```

#### 方式4：通过 `SpringApplication` 编程式指定（启动时动态控制）
适合需要「动态决定扫描范围」的场景（如根据配置决定是否扫描某个包），通过 `SpringApplication` 或 `SpringApplicationBuilder` 设置。
```java
public class DemoApplication {
    public static void main(String[] args) {
        // 方式4.1：SpringApplication 直接设置
        SpringApplication app = new SpringApplication(DemoApplication.class);
        // 指定扫描包（底层还是封装了 @ComponentScan 的逻辑）
        app.setScanBasePackages("com.example.demo", "com.example.common");
        app.run(args);
        
        // 方式4.2：SpringApplicationBuilder 链式调用
        /*
        new SpringApplicationBuilder(DemoApplication.class)
            .scanBasePackages("com.example.demo", "com.example.common")
            .run(args);
        */
    }
}
```

#### 方式5：排除自动配置（间接缩小范围，可选）
不是修改「包扫描」，而是排除 Spring Boot 自动配置的组件，适合需要精准控制 Bean 加载的场景：
```java
// 排除指定的自动配置类
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
public class DemoApplication { ... }
```

### 三、包扫描的底层实现原理
要理解包扫描原理，核心是拆解 `@SpringBootApplication` 的构成，以及 Spring Boot 启动时的扫描流程。

#### 1. `@SpringBootApplication` 的注解构成（核心）
`@SpringBootApplication` 是一个「组合注解」，核心包含以下3个注解，其中 `@ComponentScan` 是包扫描的关键：
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration // 等价于 @Configuration，标记配置类
@EnableAutoConfiguration // 开启自动配置
@ComponentScan(excludeFilters = { // 核心：包扫描注解
    @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
    @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class)
})
public @interface SpringBootApplication {
    // 省略其他属性...
    
    // scanBasePackages 本质是 @ComponentScan 的 basePackages 属性的别名
    String[] scanBasePackages() default {};
    
    // scanBasePackageClasses 本质是 @ComponentScan 的 basePackageClasses 属性的别名
    Class<?>[] scanBasePackageClasses() default {};
}
```

#### 2. 包扫描的核心流程（启动时）
Spring Boot 启动时，包扫描的核心逻辑在 `SpringApplication` 的 `refreshContext` 方法中，简化流程如下：
```mermaid
flowchart TD
    A[启动类执行 SpringApplication.run()] --> B[创建 ApplicationContext（应用上下文）]
    B --> C[解析 @SpringBootApplication 注解]
    C --> D{判断是否指定了 scanBasePackages/scanBasePackageClasses？}
    D -->|是| E[使用指定的包路径作为扫描基础]
    D -->|否| F[使用启动类所在的包作为默认扫描基础]
    E --> G[执行 @ComponentScan 逻辑]
    F --> G
    G --> H[遍历指定包下的所有 .class 文件]
    H --> I[筛选带有 @Component/@Controller/@Service 等注解的类]
    I --> J[将这些类注册为 BeanDefinition（Bean 定义）]
    J --> K[ApplicationContext 初始化这些 Bean]
```

#### 3. 关键细节解读
##### 3.1 `@ComponentScan` 的默认规则
`@ComponentScan` 如果不指定 `basePackages`/`basePackageClasses`，默认会以「标注该注解的类所在的包」作为扫描基础——这就是 `@SpringBootApplication` 默认扫描范围的来源。

##### 3.2 扫描的核心过滤器
`@ComponentScan` 内置了过滤器，只扫描带有以下注解的类：
- `@Component`（核心，`@Controller`/`@Service`/`@Repository` 都继承了 `@Component`）；
- `@Configuration`；
- 自定义的 `@Component` 派生注解。

##### 3.3 `scanBasePackages` 的优先级
如果指定了 `scanBasePackages`/`scanBasePackageClasses`，会**覆盖**默认的扫描包，而非「追加」。例如：
- 启动类在 `com.example.demo`，指定 `scanBasePackages = "com.example.common"` → 只扫描 `com.example.common` 包，不会扫描 `com.example.demo`。
- 如果需要同时扫描默认包 + 自定义包，必须把默认包也加入 `scanBasePackages`：
  ```java
  @SpringBootApplication(scanBasePackages = {"com.example.demo", "com.example.common"})
  ```

##### 3.4 编程式指定的底层逻辑
`SpringApplication.setScanBasePackages()` 本质是在启动时动态设置 `@ComponentScan` 的 `basePackages` 属性，最终还是通过 `ComponentScanAnnotationParser` 解析扫描规则，和注解方式的底层逻辑一致。

#### 4. 常见问题的原理解释
- **为什么不在扫描范围内的组件无法注入？**  
  因为 Spring Boot 只会将扫描范围内的组件注册为 Bean，不在范围内的类不会被解析，自然无法通过 `@Autowired`/`@Resource` 注入。
- **为什么 `scanBasePackages` 能覆盖默认范围？**  
  `ComponentScanAnnotationParser` 解析时，会优先使用显式指定的 `basePackages`，只有当 `basePackages` 为空时，才会使用注解所在类的包作为默认值。

### 总结
1. **默认扫描范围**：`@SpringBootApplication` 标注类所在的包 + 所有子包，核心由 `@ComponentScan` 的默认规则决定；
2. **修改方式**：核心有5种（按常用度）——`scanBasePackages`、`scanBasePackageClasses`、显式 `@ComponentScan`、编程式 `setScanBasePackages`、排除自动配置；
3. **核心原理**：`@SpringBootApplication` 组合了 `@ComponentScan`，启动时 Spring Boot 会解析该注解的扫描规则，遍历指定包下的类，将带有组件注解的类注册为 Bean，显式指定的扫描包会覆盖默认规则。
