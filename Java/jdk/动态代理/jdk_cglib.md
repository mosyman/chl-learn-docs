
你想深度剖析 JDK 动态代理和 CGLIB 动态代理的**源码实现原理**，我会从“核心原理、源码入口、关键流程、核心差异”四个维度，结合源码片段和执行流程拆解，让你彻底理解两种动态代理的底层逻辑（适合有 Java 反射、字节码基础的开发者）。

### 一、核心前置概念
在看源码前，先明确两个核心概念：

| 概念                | 作用                                                                 |
|---------------------|----------------------------------------------------------------------|
| **动态代理**        | 运行时动态生成代理类，无需手动编写代理类代码，实现对目标对象方法的增强（如日志、事务）。 |
| **JDK 动态代理**    | 基于「接口」实现：代理类继承 `Proxy` 并实现目标接口，依赖 Java 反射机制。|
| **CGLIB 动态代理**  | 基于「继承」实现：代理类继承目标类，通过 ASM 框架动态生成字节码，无需目标类实现接口。|

### 二、JDK 动态代理源码深度剖析
JDK 动态代理的核心类是 `java.lang.reflect.Proxy` 和 `java.lang.reflect.InvocationHandler`，源码基于 JDK 8 分析。

#### 1. 核心入口：`Proxy.newProxyInstance()`
这是创建 JDK 动态代理对象的核心方法，先看完整调用流程：
```java
// 示例：创建 JDK 动态代理对象
public static Object createJdkProxy(Object target) {
    return Proxy.newProxyInstance(
        target.getClass().getClassLoader(), // 类加载器
        target.getClass().getInterfaces(),  // 目标类实现的接口
        new InvocationHandler() {           // 调用处理器（核心增强逻辑）
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                System.out.println("前置增强：" + method.getName());
                Object result = method.invoke(target, args); // 调用目标方法
                System.out.println("后置增强：" + method.getName());
                return result;
            }
        }
    );
}
```

#### 2. `newProxyInstance()` 源码拆解
```java
// java.lang.reflect.Proxy
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h) {
    // 步骤1：校验参数（h 不能为空，interfaces 不能重复）
    Objects.requireNonNull(h);
    final Class<?>[] intfs = interfaces.clone();
    final SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
    }

    // 步骤2：核心！生成代理类的 Class 对象
    Class<?> cl = getProxyClass0(loader, intfs);

    // 步骤3：通过反射获取代理类的构造方法（参数为 InvocationHandler）
    try {
        if (sm != null) {
            checkNewProxyPermission(Reflection.getCallerClass(), cl);
        }
        // 代理类的构造方法签名：Proxy(InvocationHandler h)
        final Constructor<?> cons = cl.getConstructor(InvocationHandler.class);
        final InvocationHandler ih = h;
        // 步骤4：通过构造方法创建代理对象
        return cons.newInstance(new Object[]{ih});
    } catch (Exception e) {
        throw new InternalError(e.toString(), e);
    }
}
```

#### 3. 核心逻辑：`getProxyClass0()` 生成代理类
`getProxyClass0()` 是生成代理类字节码的核心，逻辑如下：
```java
// java.lang.reflect.Proxy
private static Class<?> getProxyClass0(ClassLoader loader, Class<?>... interfaces) {
    // 限制接口数量（最多 65535 个，JVM 限制）
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }
    // 步骤1：从缓存中获取代理类（避免重复生成）
    return proxyClassCache.get(loader, interfaces);
}
```
- `proxyClassCache` 是一个 `WeakCache`（弱引用缓存），key 是「类加载器 + 接口数组」，value 是代理类 Class；
- 缓存未命中时，调用 `ProxyClassFactory` 创建代理类。

#### 4. 代理类生成器：`ProxyClassFactory`
```java
// java.lang.reflect.Proxy.ProxyClassFactory
private static final class ProxyClassFactory
    implements BiFunction<ClassLoader, Class<?>[], Class<?>> {
    // 代理类命名规则：com.sun.proxy.$Proxy + 数字（如 $Proxy0、$Proxy1）
    private static final String proxyClassNamePrefix = "$Proxy";
    private static final AtomicLong nextUniqueNumber = new AtomicLong();

    @Override
    public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {
        // 步骤1：校验接口合法性（不能是重复接口、不能是基本类型）
        Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
        for (Class<?> intf : interfaces) {
            Class<?> interfaceClass = Class.forName(intf.getName(), false, loader);
            if (interfaceClass != intf) {
                throw new IllegalArgumentException(intf + " is not visible from class loader");
            }
            if (!interfaceClass.isInterface()) {
                throw new IllegalArgumentException(interfaceClass.getName() + " is not an interface");
            }
            if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                throw new IllegalArgumentException("repeated interface: " + interfaceClass.getName());
            }
        }

        // 步骤2：生成代理类名称（如 com.sun.proxy.$Proxy0）
        String proxyPkg = null; 
        for (Class<?> intf : interfaces) {
            String pkg = intf.getPackageName();
            if (pkg != null) {
                if (proxyPkg == null) {
                    proxyPkg = pkg;
                } else if (!pkg.equals(proxyPkg)) {
                    throw new IllegalArgumentException("non-public interfaces from different packages");
                }
            }
        }
        if (proxyPkg == null) {
            proxyPkg = "";
        }
        long num = nextUniqueNumber.getAndIncrement();
        String proxyName = proxyPkg + proxyClassNamePrefix + num;

        // 步骤3：核心！生成代理类的字节码
        byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
            proxyName, interfaces, Modifier.PUBLIC | Modifier.FINAL);
        
        // 步骤4：通过类加载器加载字节码，生成 Class 对象
        try {
            return defineClass0(loader, proxyName, proxyClassFile, 0, proxyClassFile.length);
        } catch (ClassFormatError e) {
            throw new IllegalArgumentException(e.toString());
        }
    }
}
```

#### 5. 代理类字节码生成：`ProxyGenerator.generateProxyClass()`
这一步是 JDK 动态代理的底层核心，生成的代理类字节码逻辑如下（反编译后）：
```java
// 反编译后的 $Proxy0 类（示例）
public final class $Proxy0 extends Proxy implements UserService {
    // 静态初始化：反射获取目标接口的方法
    private static Method m1; // equals 方法
    private static Method m2; // toString 方法
    private static Method m3; // 目标方法（如 UserService.addUser）
    private static Method m0; // hashCode 方法

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("com.example.UserService").getMethod("addUser");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException e) {
            throw new NoSuchMethodError(e.getMessage());
        } catch (ClassNotFoundException e) {
            throw new NoClassDefFoundError(e.getMessage());
        }
    }

    // 构造方法：接收 InvocationHandler
    public $Proxy0(InvocationHandler h) {
        super(h);
    }

    // 实现目标接口的方法（核心）
    @Override
    public final void addUser() {
        try {
            // 调用 InvocationHandler.invoke()，实现增强
            super.h.invoke(this, m3, null);
        } catch (RuntimeException | Error e) {
            throw e;
        } catch (Throwable e) {
            throw new UndeclaredThrowableException(e);
        }
    }

    // 重写 Object 的方法（同理）
    @Override
    public final boolean equals(Object obj) {
        // ... 调用 h.invoke(this, m1, new Object[]{obj})
    }
}
```

#### 6. JDK 动态代理核心流程总结
```mermaid
graph TD
    A[Proxy.newProxyInstance()] --> B[校验参数]
    B --> C[getProxyClass0() 获取代理类]
    C --> D{缓存命中？}
    D -- 是 --> E[返回缓存的 Class]
    D -- 否 --> F[ProxyClassFactory 生成代理类名称]
    F --> G[ProxyGenerator 生成字节码]
    G --> H[defineClass0 加载字节码为 Class]
    E & H --> I[反射获取代理类构造方法]
    I --> J[创建代理对象并绑定 InvocationHandler]
    J --> K[调用代理方法时，触发 InvocationHandler.invoke()]
```

### 三、CGLIB 动态代理源码深度剖析
CGLIB 是第三方库（核心包：`cglib-core`、`cglib-proxy`），核心类是 `net.sf.cglib.proxy.Enhancer` 和 `net.sf.cglib.proxy.MethodInterceptor`，源码基于 CGLIB 3.3.0 分析。

#### 1. 核心入口：`Enhancer.create()`
先看 CGLIB 代理的使用示例：
```java
// 示例：创建 CGLIB 动态代理对象
public static Object createCglibProxy(Object target) {
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(target.getClass()); // 设置父类（目标类）
    enhancer.setCallback(new MethodInterceptor() { // 方法拦截器（核心增强）
        @Override
        public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
            System.out.println("前置增强：" + method.getName());
            Object result = proxy.invokeSuper(obj, args); // 调用目标方法（注意：不是 method.invoke）
            System.out.println("后置增强：" + method.getName());
            return result;
        }
    });
    return enhancer.create(); // 生成并返回代理对象
}
```

#### 2. `Enhancer.create()` 源码拆解
```java
// net.sf.cglib.proxy.Enhancer
public Object create() {
    classOnly = false;
    argumentTypes = null;
    return createHelper();
}

private Object createHelper() {
    // 步骤1：校验参数（父类不能为空）
    preValidate();
    // 步骤2：获取类加载器（默认使用目标类的类加载器）
    ClassLoader loader = getClassLoader();
    // 步骤3：生成代理类的 Class 对象
    Class<?> key = getKey();
    Object result = null;
    // 步骤3.1：从缓存中获取（CGLIB 也有缓存，避免重复生成）
    if (useCache) {
        result = cache.get(loader, key);
    }
    // 步骤3.2：缓存未命中，生成代理类
    if (result == null) {
        try {
            // 核心：生成代理类字节码并加载
            Class<?> gen = generate(loader, key);
            // 步骤4：反射创建代理对象
            result = createUsingReflection(gen);
        } catch (RuntimeException e) {
            throw e;
        } catch (Exception e) {
            throw new CodeGenerationException(e);
        }
    }
    return result;
}
```

#### 3. 核心逻辑：`generate()` 生成代理类
```java
// net.sf.cglib.proxy.Enhancer
protected Class<?> generate(ClassLoader loader, Object key) {
    // 步骤1：创建 ClassGenerator（类生成器）
    ClassGenerator cg = createClassGenerator(loader);
    // 步骤2：设置生成策略（默认 DefaultGeneratorStrategy）
    GeneratorStrategy strategy = getGeneratorStrategy();
    // 步骤3：生成字节码
    byte[] b = strategy.generate(cg);
    // 步骤4：加载字节码为 Class 对象
    Class<?> clazz = ReflectUtils.defineClass(getClassName(), b, loader);
    // 步骤5：注册回调（绑定 MethodInterceptor）
    registerCallbacks(clazz);
    return clazz;
}
```

#### 4. 字节码生成核心：`ClassGenerator`
CGLIB 基于 ASM 框架生成字节码，`ClassGenerator` 的核心实现是 `net.sf.cglib.core.DefaultGeneratorStrategy`：
```java
// net.sf.cglib.core.DefaultGeneratorStrategy
public byte[] generate(ClassGenerator cg) throws Exception {
    // 创建 ASM 的 ClassWriter，生成字节码
    ClassWriter cw = getClassWriter();
    // 核心：ClassGenerator 生成类的字节码（继承父类 + 实现回调接口）
    cg.generateClass(cw);
    // 可选：修改字节码（如添加方法、修改访问修饰符）
    return transform(cw.toByteArray());
}
```

#### 5. 代理类字节码逻辑（反编译后）
CGLIB 生成的代理类继承目标类，并重写所有非 final 方法，核心逻辑如下：
```java
// 反编译后的 CGLIB 代理类（示例）
public class UserService$$EnhancerByCGLIB$$5f2d786c extends UserService implements Factory {
    // 回调数组（存储 MethodInterceptor）
    private MethodInterceptor CGLIB$CALLBACK_0;
    // 方法代理对象（缓存 MethodProxy）
    private static final MethodProxy CGLIB$addUser$0$Proxy;
    // 静态初始化：创建 MethodProxy
    static {
        CGLIB$STATICHOOK1();
    }

    private static void CGLIB$STATICHOOK1() {
        // 初始化 MethodProxy，绑定目标方法
        CGLIB$addUser$0$Proxy = MethodProxy.create(
            UserService.class, 
            UserService$$EnhancerByCGLIB$$5f2d786c.class, 
            "()V", // 方法签名
            "addUser", // 目标方法名
            "CGLIB$addUser$0" // 代理方法名
        );
    }

    // 重写目标方法
    @Override
    public final void addUser() {
        MethodInterceptor tmp = CGLIB$CALLBACK_0;
        if (tmp != null) {
            // 调用 MethodInterceptor.intercept()，实现增强
            tmp.intercept(this, CGLIB$addUser$0$Method, null, CGLIB$addUser$0$Proxy);
        } else {
            super.addUser(); // 无增强时直接调用父类方法
        }
    }

    // Factory 接口方法：设置回调
    @Override
    public void setCallback(int index, Callback callback) {
        if (index == 0) {
            CGLIB$CALLBACK_0 = (MethodInterceptor)callback;
        }
    }
}
```

#### 6. CGLIB 动态代理核心流程总结
```mermaid
graph TD
    A[Enhancer.create()] --> B[createHelper()]
    B --> C[preValidate() 校验参数]
    C --> D[getClassLoader() 获取类加载器]
    D --> E{缓存命中？}
    E -- 是 --> F[返回缓存的 Class]
    E -- 否 --> G[generate() 生成代理类字节码]
    G --> H[ASM 框架生成继承目标类的字节码]
    H --> I[ReflectUtils.defineClass 加载字节码为 Class]
    F & I --> J[registerCallbacks 绑定 MethodInterceptor]
    J --> K[createUsingReflection 创建代理对象]
    K --> L[调用代理方法时，触发 MethodInterceptor.intercept()]
```

### 四、JDK 动态代理 vs CGLIB 动态代理核心差异
| 维度                | JDK 动态代理                          | CGLIB 动态代理                          |
|---------------------|---------------------------------------|-----------------------------------------|
| **实现方式**        | 基于接口 + 反射                       | 基于继承 + ASM 字节码生成               |
| **目标类要求**      | 必须实现接口                          | 无需实现接口，不能是 final 类           |
| **方法要求**        | 只能增强接口中的方法                  | 可增强所有非 final 方法                 |
| **性能**            | 反射调用，效率较低（JDK 8 后优化）    | 字节码直接调用，效率更高                |
| **核心类**          | Proxy、InvocationHandler              | Enhancer、MethodInterceptor             |
| **方法调用方式**    | method.invoke(target, args)           | proxy.invokeSuper(obj, args)            |
| **缓存机制**        | WeakCache 缓存代理类 Class            | 内置缓存，默认开启                      |
| **依赖**            | JDK 内置，无需额外依赖                | 第三方库（cglib），依赖 ASM             |

### 五、关键源码细节补充
1. **JDK 动态代理的 `invoke` 方法参数**：
    - `proxy`：代理对象本身（注意：不要在 `invoke` 中调用 `proxy` 的方法，会导致死循环）；
    - `method`：当前调用的目标方法；
    - `args`：方法参数。

2. **CGLIB 的 `invokeSuper` vs `invoke`**：
    - `proxy.invokeSuper(obj, args)`：调用父类（目标类）的方法，避免死循环；
    - `method.invoke(obj, args)`：调用代理对象的方法，会再次触发 `intercept`，导致死循环。

3. **缓存的核心作用**：
   无论是 JDK 还是 CGLIB，都对生成的代理类 Class 做缓存，因为「生成字节码 + 加载类」是耗时操作，缓存可大幅提升性能。

### 总结
1. **JDK 动态代理**：核心是 `Proxy.newProxyInstance()` 生成实现目标接口的代理类，通过 `InvocationHandler` 拦截方法调用，依赖反射，仅支持接口增强；
2. **CGLIB 动态代理**：核心是 `Enhancer.create()` 生成继承目标类的代理类，通过 `MethodInterceptor` 拦截方法调用，基于 ASM 字节码，支持类增强；
3. **核心差异**：JDK 基于接口、反射效率低；CGLIB 基于继承、字节码效率高，但受限于 final 类/方法。

如果需要，我可以补充「两种代理的性能对比测试代码」或「Spring AOP 中如何选择 JDK/CGLIB 代理」的源码分析，帮你关联实际框架的使用场景。







