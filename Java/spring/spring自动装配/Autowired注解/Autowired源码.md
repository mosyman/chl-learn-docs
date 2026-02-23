





### Spring @Autowired 工作原理与源码深度解析
@Autowired 是 Spring 中最核心的依赖注入（DI）注解之一，用于自动装配 Bean 之间的依赖关系。它的底层实现依赖于 Spring 的 **Bean 后置处理器** 和 **依赖解析机制**，下面从注解定义、核心处理器、工作流程、关键源码四个维度进行解析。

---

## 一、@Autowired 注解的源码定义
@Autowired 注解位于 `org.springframework.beans.factory.annotation` 包下，核心作用是标记需要自动注入的依赖点（字段、方法、构造器）。

```java
@Target({ElementType.CONSTRUCTOR, ElementType.METHOD, ElementType.PARAMETER, ElementType.FIELD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Autowired {
    // 标记依赖是否为必填，默认true：找不到依赖时抛出异常
    boolean required() default true;
}
```
- **元注解说明**：
  - `@Target`：支持标注在构造器、方法、参数、字段、注解上；
  - `@Retention(RUNTIME)`：运行时保留，供 Spring 容器反射解析；
  - 除了@Autowired，Spring 还支持 JSR-330 的 `@Inject`、JSR-250 的 `@Resource`，其中@Autowired 是 Spring 原生注解，基于**类型匹配**注入。

---

## 二、处理@Autowired 的核心处理器：`AutowiredAnnotationBeanPostProcessor`
@Autowired 的解析和注入完全由 `AutowiredAnnotationBeanPostProcessor` 这个**Bean 后置处理器**（`BeanPostProcessor` 实现类）负责，它是 Spring 依赖注入的核心执行器。

### 1. 处理器的注册方式
- **Spring Boot 自动注册**：Spring Boot 会通过 `AutoConfigurationPackages` 自动扫描并注册该处理器；
- **XML 配置注册**：在 XML 中配置 `<context:annotation-config/>` 或 `<context:component-scan/>`，Spring 会自动注册该处理器；
- **手动注册**：通过 `@Bean` 注解手动创建该处理器的实例。

### 2. 处理器的初始化逻辑
在 `AutowiredAnnotationBeanPostProcessor` 的构造方法中，会初始化需要处理的注解类型（包括@Autowired、@Value、@Inject）：
```java
public AutowiredAnnotationBeanPostProcessor() {
    this.autowiredAnnotationTypes.add(Autowired.class);
    this.autowiredAnnotationTypes.add(Value.class);
    try {
        this.autowiredAnnotationTypes.add(Class.forName("javax.inject.Inject"));
    } catch (ClassNotFoundException ex) {
        // 忽略JSR-330不存在的情况
    }
}
```

---

## 三、@Autowired 的完整工作流程
@Autowired 的注入发生在 **Bean 创建流程的「属性填充阶段」**，Spring 创建 Bean 的完整流程为：
`实例化 Bean（createBeanInstance）` → `属性填充（populateBean）` → `初始化（initializeBean）` → `Bean 就绪`

其中@Autowired 的处理在 `populateBean` 阶段，具体流程如下：
1. **扫描注入点**：Spring 扫描当前 Bean 的字段、方法，收集所有标注了@Autowired 的依赖点；
2. **解析依赖**：根据依赖的类型，从 Spring 容器中查找匹配的 Bean；
3. **注入依赖**：将找到的 Bean 赋值给字段，或调用方法完成注入；
4. **处理可选依赖**：如果@Autowired 的 `required=false`，且找不到依赖，则注入 null，否则抛出 `NoSuchBeanDefinitionException`。

---

## 四、关键源码深度解析
### 1. 核心入口：`postProcessProperties` 方法
`AutowiredAnnotationBeanPostProcessor` 重写了 `postProcessProperties` 方法，这是处理@Autowired 的核心入口：
```java
@Override
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
    // 1. 构建当前Bean的注入元数据（收集所有@Autowired的依赖点）
    InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
    try {
        // 2. 执行依赖注入
        metadata.inject(bean, beanName, pvs);
    } catch (BeanCreationException ex) {
        throw ex;
    } catch (Throwable ex) {
        throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
    }
    return pvs;
}
```

### 2. 收集注入点：`buildAutowiringMetadata` 方法
该方法负责扫描 Bean 的所有字段和方法，收集标注了@Autowired 的依赖点：
```java
private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {
    List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
    Class<?> targetClass = clazz;

    // 遍历当前类及其父类，收集注入点
    do {
        final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();

        // 1. 扫描字段上的@Autowired
        ReflectionUtils.doWithLocalFields(targetClass, field -> {
            MergedAnnotation<?> ann = findAutowiredAnnotation(field);
            if (ann != null) {
                // 静态字段不支持@Autowired
                if (Modifier.isStatic(field.getModifiers())) {
                    logger.info("Autowired annotation is not supported on static fields: " + field);
                    return;
                }
                // 判断是否为必填依赖
                boolean required = determineRequiredStatus(ann);
                currElements.add(new AutowiredFieldElement(field, required));
            }
        });

        // 2. 扫描方法上的@Autowired（如setter方法）
        ReflectionUtils.doWithLocalMethods(targetClass, method -> {
            Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
            if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
                return;
            }
            MergedAnnotation<?> ann = findAutowiredAnnotation(bridgedMethod);
            if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
                // 静态方法不支持@Autowired
                if (Modifier.isStatic(method.getModifiers())) {
                    logger.info("Autowired annotation is not supported on static methods: " + method);
                    return;
                }
                // 无参数的方法不需要@Autowired
                if (method.getParameterCount() == 0) {
                    logger.info("Autowired annotation should only be used on methods with parameters: " + method);
                }
                boolean required = determineRequiredStatus(ann);
                PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
                currElements.add(new AutowiredMethodElement(method, required, pd));
            }
        });

        elements.addAll(0, currElements);
        targetClass = targetClass.getSuperclass();
    } while (targetClass != null && targetClass != Object.class);

    return InjectionMetadata.forElements(elements, clazz);
}
```
- 该方法会遍历当前类及其所有父类，收集字段和方法上的@Autowired 注解；
- 静态字段/方法不支持@Autowired，因为静态成员属于类，不属于 Bean 实例；
- 最终返回 `InjectionMetadata`（注入元数据），包含所有需要注入的依赖点。

### 3. 执行注入：`InjectionMetadata.inject` 方法
`InjectionMetadata` 的 `inject` 方法会遍历所有收集到的注入点，执行实际的注入逻辑：
```java
public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
    Collection<InjectedElement> checkedElements = this.checkedElements;
    Collection<InjectedElement> elementsToIterate =
            (checkedElements != null ? checkedElements : this.injectedElements);
    if (!elementsToIterate.isEmpty()) {
        // 遍历所有注入点，执行注入
        for (InjectedElement element : elementsToIterate) {
            element.inject(target, beanName, pvs);
        }
    }
}
```

### 4. 字段注入的具体实现：`AutowiredFieldElement.inject`
对于字段注入，`AutowiredFieldElement` 的 `inject` 方法会负责解析字段的值并赋值：
```java
@Override
protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
    Field field = (Field) this.member;
    Object value;
    if (this.cached) {
        // 缓存的依赖值，直接使用
        try {
            value = resolvedCachedArgument(beanName, this.cachedFieldValue);
        } catch (NoSuchBeanDefinitionException ex) {
            // 缓存失效，重新解析
            value = resolveFieldValue(field, bean, beanName);
        }
    } else {
        // 解析字段对应的依赖值
        value = resolveFieldValue(field, bean, beanName);
    }
    if (value != null) {
        // 暴力反射访问私有字段
        ReflectionUtils.makeAccessible(field);
        // 为字段赋值
        field.set(bean, value);
    }
}
```

### 5. 依赖解析的核心：`resolveDependency` 方法
依赖的查找和解析最终会调用 `BeanFactory` 的 `resolveDependency` 方法，该方法在 `DefaultListableBeanFactory` 中实现，核心逻辑包括：
- 根据依赖类型查找所有候选 Bean；
- 处理 `@Qualifier`、`@Primary`、`@Priority` 等注解，筛选出最终的 Bean；
- 处理可选依赖（`required=false`）；
- 处理依赖的类型转换；
- 解决循环依赖（通过 Spring 三级缓存）。

核心源码片段：
```java
@Override
@Nullable
public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
        @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

    descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());
    // 处理Optional类型的依赖
    if (Optional.class == descriptor.getDependencyType()) {
        return createOptionalDependency(descriptor, requestingBeanName);
    }
    // 处理ObjectFactory/ObjectProvider类型的依赖（延迟注入）
    else if (ObjectFactory.class == descriptor.getDependencyType() ||
            ObjectProvider.class == descriptor.getDependencyType()) {
        return new DependencyObjectProvider(descriptor, requestingBeanName);
    }
    // 处理JSR-330的Provider类型
    else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
        return new Jsr330Factory().createDependencyProvider(descriptor, requestingBeanName);
    } else {
        // 尝试获取懒加载代理
        Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
                descriptor, requestingBeanName);
        if (result == null) {
            // 实际解析依赖
            result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
        }
        return result;
    }
}
```

---

## 五、关键细节与注意事项
### 1. 注入优先级
Spring 中依赖注入的优先级为：`构造器注入 > 方法注入 > 字段注入`：
- 构造器注入：Spring 4.3+ 之后，若类只有一个构造器，无需加@Autowired，Spring 会自动注入构造器参数；
- 方法注入：通常用于 setter 方法，Spring 会解析方法参数并注入；
- 字段注入：最简洁，但不便于单元测试（无法手动注入依赖）。

### 2. required 属性的处理
- 当 `required=true`（默认）时，若找不到匹配的 Bean，Spring 会抛出 `NoSuchBeanDefinitionException`；
- 当 `required=false` 时，若找不到匹配的 Bean，Spring 会注入 null，不会抛出异常。

### 3. 循环依赖的处理
@Autowired 支持处理**字段注入的循环依赖**，通过 Spring 的三级缓存解决：
- 一级缓存（singletonObjects）：存放已完全初始化的 Bean；
- 二级缓存（earlySingletonObjects）：存放提前暴露的 Bean 代理（未完成属性填充）；
- 三级缓存（singletonFactories）：存放 Bean 工厂，用于创建提前暴露的代理。
- 注意：**构造器注入的循环依赖无法解决**，因为构造器注入发生在实例化阶段，此时 Bean 尚未被放入缓存。

### 4. 与@Resource 的区别
| 特性                | @Autowired                          | @Resource                          |
|---------------------|-------------------------------------|-------------------------------------|
| 依赖匹配方式        | 基于类型（Type）                    | 基于名称（Name），找不到再按类型    |
| 所属规范            | Spring 原生注解                     | JSR-250 规范                        |
| 支持的注入点        | 构造器、方法、字段、参数            | 字段、方法（不支持构造器）          |
| 支持的属性          | required                            | name、type                          |

---

## 总结
@Autowired 的本质是通过 `AutowiredAnnotationBeanPostProcessor` 这个后置处理器，在 Bean 实例化后的属性填充阶段，扫描并解析标注了@Autowired 的依赖点，通过 Spring 的 BeanFactory 查找匹配的 Bean，最终通过反射完成依赖注入。整个过程完全基于 Spring 的 Bean 生命周期和依赖解析机制，是 Spring 控制反转（IoC）的核心体现。