
# @RequestMapping 在源码中的解析位置

## 一、核心结论
`@RequestMapping` 注解的解析分为两个阶段：
1. **初始化阶段（容器启动时）**：扫描所有 `@Controller` / `@RequestMapping` 类与方法，解析注解属性（URL、请求方法、参数、请求头等），并缓存成「请求路径 → 处理器方法」映射关系。
2. **请求阶段（HTTP 请求时）**：根据请求 URL、请求方法等，从缓存中匹配对应的处理器方法。

**核心解析类**：
`RequestMappingHandlerMapping`

---

## 二、初始化阶段：注解解析与注册

### 1. 入口方法：afterPropertiesSet()
```java
// AbstractHandlerMethodMapping.java
@Override
public void afterPropertiesSet() {
    initHandlerMethods();
}
```

### 2. 扫描所有处理器：initHandlerMethods()
```java
// AbstractHandlerMethodMapping.java
protected void initHandlerMethods() {
    String[] beanNames = getApplicationContext().getBeanNamesForType(Object.class);

    for (String beanName : beanNames) {
        Class<?> beanType = getApplicationContext().getType(beanName);
        if (beanType != null && isHandler(beanType)) {
            detectHandlerMethods(beanName);
        }
    }
}
```

### 3. 判断是否为处理器：isHandler()
```java
// RequestMappingHandlerMapping.java
@Override
protected boolean isHandler(Class<?> beanType) {
    return AnnotatedElementUtils.hasAnnotation(beanType, Controller.class)
        || AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class);
}
```

### 4. 解析当前类里的所有映射方法：detectHandlerMethods()
```java
// AbstractHandlerMethodMapping.java
protected void detectHandlerMethods(Object handler) {
    Class<?> handlerType = ...;
    Map<Method, RequestMappingInfo> methods = MethodIntrospector.selectMethods(handlerType,
        method -> getMappingForMethod(method, handlerType)
    );
    methods.forEach((method, mapping) -> {
        registerHandlerMethod(handler, method, mapping);
    });
}
```

### 5. 最核心：解析单个方法的 @RequestMapping
```java
// RequestMappingHandlerMapping.java
@Override
protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
    // 解析方法上的 @RequestMapping
    RequestMappingInfo info = createRequestMappingInfo(method);
    if (info != null) {
        // 解析类上的 @RequestMapping（前缀）
        RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);
        if (typeInfo != null) {
            info = typeInfo.combine(info);
        }
    }
    return info;
}
```

### 6. 解析注解并构建 RequestMappingInfo
```java
private RequestMappingInfo createRequestMappingInfo(AnnotatedElement element) {
    RequestMapping requestMapping = AnnotatedElementUtils.findMergedAnnotation(element, RequestMapping.class);
    return createRequestMappingInfo(requestMapping, condition);
}
```

### 7. 注册到映射注册表
```java
// AbstractHandlerMethodMapping.java
protected void registerHandlerMethod(Object handler, Method method, RequestMappingInfo mapping) {
    this.mappingRegistry.register(mapping, handler, method);
}
```

---

## 三、请求阶段：根据请求匹配映射

### 入口：getHandlerInternal()
```java
// AbstractHandlerMethodMapping.java
@Override
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
    String lookupPath = initLookupPath(request);
    try {
        // 根据路径 + 请求方法等条件匹配
        return lookupHandlerMethod(lookupPath, request);
    } finally {
        // 释放锁
    }
}
```

---

## 四、完整流程总结
```
容器启动
  ↓
afterPropertiesSet()
  ↓
initHandlerMethods() 扫描所有 Bean
  ↓
isHandler() 判断是否为 @Controller / @RequestMapping
  ↓
detectHandlerMethods() 遍历方法
  ↓
getMappingForMethod() 解析注解
  ↓
合并类级别 + 方法级别路径
  ↓
registerHandlerMethod() 注册到 mappingRegistry
  ↓
请求进来
  ↓
getHandlerInternal()
  ↓
lookupHandlerMethod() 匹配
  ↓
返回 HandlerMethod
```

---

## 五、关键类总结
- **`RequestMappingHandlerMapping`**：`@RequestMapping` 解析核心类
- **`AbstractHandlerMethodMapping`**：提供通用扫描与注册逻辑
- **`RequestMappingInfo`**：注解解析后的结构化对象
- **`mappingRegistry`**：存储所有 URL → 方法映射关系