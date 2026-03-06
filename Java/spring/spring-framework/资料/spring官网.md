


## SpringBoot

[SpringBoot官网](https://spring.io/projects/spring-boot)

[SpringBoot 4.0.2 Reference Doc](https://docs.spring.io/spring-boot/index.html)





你想要系统地研究 Spring 框架具体类的原理和源码，核心需求是获取一套**可落地的研究方法**，以及**权威、全面的资料来源**（尤其是官方资料），从而能深入理解 Spring 核心类的设计思想和实现逻辑。

### 一、研究 Spring 源码的落地方法（分步骤）
#### 步骤 1：环境准备（基础前提）
首先要搭建能流畅阅读、调试 Spring 源码的环境，避免因环境问题阻碍研究：
1. **获取纯净源码**
    - 官方仓库：[Spring Framework GitHub](https://github.com/spring-projects/spring-framework)（核心源码）
    - 版本选择：优先选稳定版（如 6.1.x、5.3.x），避免快照版；新手建议从 5.3.x 入手（兼容 Java 8，文档更全）。
    - 编译源码：参考官方编译指南 [Building Spring Framework](https://github.com/spring-projects/spring-framework/wiki/Building-from-Source)，核心命令：
      ```bash
      # 预编译（需安装 Gradle）
      ./gradlew :spring-oxm:compileTestJava
      # 完整编译
      ./gradlew build
      ```
2. **IDE 配置**
    - 推荐 IntelliJ IDEA（对 Gradle 支持更好），导入源码后需：
        - 配置 JDK（5.3.x 用 JDK 8，6.x 用 JDK 17+）；
        - 关闭 IDEA 的“省电模式”，确保源码索引正常；
        - 为常用类（如 `ApplicationContext`、`BeanFactory`）添加书签，方便快速定位。
3. **调试环境搭建**
    - 新建一个简单的 Spring Boot 项目（通过 [Spring Initializr](https://start.spring.io/)），将 Spring 源码作为“依赖源码”关联到项目中；
    - 打断点调试核心流程（如 Bean 创建、依赖注入、AOP 织入），观察调用栈和变量值。

#### 步骤 2：确定研究方向（避免盲目）
Spring 源码庞大，先聚焦具体模块/类，再逐步扩展，新手推荐优先级：
1. **核心容器（最基础）**：`BeanFactory`、`ApplicationContext`、`DefaultListableBeanFactory`、`AbstractBeanFactory`（Bean 的创建、解析、注册）；
2. **注解驱动**：`ConfigurationClassPostProcessor`、`ComponentScanAnnotationParser`（你之前问的 `@ComponentScan` 解析类）、`AutowiredAnnotationBeanPostProcessor`；
3. **AOP 模块**：`DefaultAopProxyFactory`、`JdkDynamicAopProxy`、`CglibAopProxy`；
4. **事务模块**：`TransactionInterceptor`、`DataSourceTransactionManager`；
5. **Web 模块**：`DispatcherServlet`、`RequestMappingHandlerMapping`（Spring MVC 核心）。

#### 步骤 3：源码研究核心技巧（从“读”到“懂”）
1. **先理清“宏观流程”，再钻“细节实现”**
    - 比如研究 `@ComponentScan`：
      ① 先画流程图：Spring 启动 → 解析 `@ComponentScan` → 扫描指定包 → 过滤类 → 注册 BeanDefinition；
      ② 再找核心类：`ComponentScanAnnotationParser`（解析注解）→ `ClassPathBeanDefinitionScanner`（扫描类）→ `TypeFilter`（过滤规则）；
      ③ 最后看关键方法：`ComponentScanAnnotationParser#parse()`、`ClassPathBeanDefinitionScanner#doScan()`。
2. **通过“问题驱动”研究**
   带着具体问题找源码，效率更高，比如：
    - 问题 1：`@ComponentScan` 的 `excludeFilters` 是如何生效的？→ 找 `ClassPathScanningCandidateComponentProvider#isCandidateComponent()`；
    - 问题 2：Spring 如何解决循环依赖？→ 找 `DefaultSingletonBeanRegistry#singletonFactories`（三级缓存）；
    - 问题 3：`@Autowired` 是如何实现依赖注入的？→ 找 `AutowiredAnnotationBeanPostProcessor#postProcessProperties()`。
3. **跟踪调用栈（调试关键）**
    - 在核心方法（如 `SpringApplication.run()`）打端点，按“步入（Step Into）”跟踪调用链，记录每一步的核心类/方法；
    - 忽略非核心代码（如日志、异常处理），聚焦“核心逻辑分支”。
4. **对比阅读（加深理解）**
    - 比如对比 `BeanFactory`（基础接口）和 `ApplicationContext`（扩展接口）的方法差异，理解“接口分层设计”；
    - 对比 `JdkDynamicAopProxy` 和 `CglibAopProxy` 的实现，理解 JDK 动态代理 vs CGLIB 代理的适用场景。

#### 步骤 4：总结与验证（巩固成果）
1. **手写简化版实现**：比如手写一个简易的 `ComponentScan` 解析器，仅实现“扫描指定包 + 过滤注解”，验证自己是否理解核心逻辑；
2. **写笔记/博客**：将研究过程中的关键类、方法、设计模式（如工厂模式、代理模式、观察者模式）整理成笔记，强迫自己梳理逻辑；
3. **参与社区讨论**：在 [Spring 官方论坛](https://forum.spring.io/) 或 Stack Overflow 回答相关问题，检验自己的理解是否正确。

### 二、Spring 核心资料收集（官方+优质第三方）
#### 1. 官方权威资料（优先级最高）
| 资料类型 | 链接 | 适用场景 |
|----------|------|----------|
| Spring 框架官方文档 | [Spring Framework Documentation](https://docs.spring.io/spring-framework/reference/) | 学习核心概念、API 用法、设计思想（6.x 对应 JDK 17+，5.3.x 对应 [5.3 文档](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/)） |
| Spring 源码注释 | 源码中的 Javadoc（如 `@ComponentScan`、`BeanFactory` 的注释） | 理解类/方法的设计目的、参数含义、使用注意事项（最权威的“源码说明书”） |
| Spring 官方博客 | [Spring Blog](https://spring.io/blog) | 了解新特性设计背景、核心模块的实现思路（如“Spring 6 的 AOT 编译”） |
| Spring 官方 GitHub Wiki | [Spring Framework Wiki](https://github.com/spring-projects/spring-framework/wiki) | 编译源码、贡献代码、核心设计决策（如“为什么选择 Gradle 而非 Maven”） |
| Spring 规格文档 | [Spring Specifications](https://github.com/spring-projects/spring-framework/tree/main/docs/src/main/asciidoc) | 深入理解核心模块的设计规格（如 Bean 生命周期、AOP 规范） |

#### 2. 官方辅助资料（补充理解）
| 资料 | 链接 | 说明 |
|------|------|------|
| Spring Boot 官方文档 | [Spring Boot Docs](https://docs.spring.io/spring-boot/docs/current/reference/html/) | 研究 Spring Boot 自动配置（如 `AutoConfiguration`）时必备 |
| Spring API 文档（Javadoc） | [Spring API Doc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/) | 快速查询类/方法的详细说明（可按包检索） |
| Spring 官方示例 | [Spring Samples](https://github.com/spring-projects/spring-samples) | 官方提供的源码示例，可对比理解核心类的使用方式 |

#### 3. 优质第三方资料（辅助深入）
| 类型 | 推荐资源 | 适用场景 |
|------|----------|----------|
| 源码解析书籍 | 《Spring 源码深度解析》（郝佳）、《Spring 核心技术与最佳实践》 | 新手入门，梳理核心流程（注意版本匹配，优先选覆盖 5.x 的版本） |
| 视频教程 | 尚硅谷 Spring 源码解析、黑马程序员 Spring 源码课 | 可视化理解调用流程，适合“听觉型”学习者 |
| 技术博客 | - [Spring 官方团队博客](https://spring.io/blog) <br> - [Baeldung Spring 专栏](https://www.baeldung.com/spring-tutorial) <br> - 芋道源码（[www.iocoder.cn](https://www.iocoder.cn/)） | 深入解析具体类/模块的实现细节（如循环依赖、AOP 源码） |
| 社区问答 | [Stack Overflow Spring 标签](https://stackoverflow.com/questions/tagged/spring) | 解决研究过程中遇到的具体问题（如“某个方法的作用”） |





















