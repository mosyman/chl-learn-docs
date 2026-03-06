




# 资料
1, [Spring自动装配方式有哪些-源码解析](https://www.mianshiya.com/question/1797568244295532545)

2, [小李哥编程](https://zhuanlan.zhihu.com/p/689955166)






# 详解 @Autowired 注解
作为 Spring 最核心的注解之一，`@Autowired` 是实现**依赖注入（DI）** 的关键工具。下面我会从「核心作用→使用场景→核心规则→特殊注意事项」的逻辑，由浅入深讲解这个注解，帮你彻底理解它的用法和底层逻辑。

## 一、核心作用（一句话看懂）
`@Autowired` 的本质是：**告诉 Spring 容器“自动帮我找匹配的 Bean，注入到标注的位置（构造方法/字段/方法/参数）”**。
- 它是 JSR-330 规范中 `@Inject` 注解的增强版，核心新增了 `required` 属性（控制依赖是否“必需”）；
- 注解的生命周期是 `RUNTIME`（运行时生效），作用范围支持构造方法、方法、参数、字段、注解类型。

## 二、核心使用场景（按常用度排序）
### 1. 字段注入（最简洁，新手常用）
#### 核心规则
- 注入时机：Bean 实例化后 → 配置方法执行前，完成字段赋值；
- 访问修饰符：无需 `public`，`private` 也能注入（Spring 通过反射实现）。

#### 代码示例
```java
@Component
public class UserService {
    // 私有字段也能注入，Spring 自动找容器中的 UserDao Bean
    @Autowired
    private UserDao userDao;
    
    public void query() {
        userDao.query();
    }
}

// 被依赖的 Bean
@Component
public class UserDao {
    public void query() {
        System.out.println("执行查询");
    }
}
```

### 2. 构造方法注入（Spring 官方推荐）
#### 核心规则
这是最规范的注入方式（保证 Bean 初始化时依赖已完全注入），关键规则：
- 若 `required = true`：一个类只能有**一个**构造方法标注 `@Autowired`；
- 若 `required = false`：多个构造方法可标注，Spring 选“能匹配最多依赖”的那个；
- 兜底逻辑：无标注的构造方法 → 优先用“主/默认构造方法（无参）”；若类只有一个构造方法，哪怕不标注也会自动使用；
- 访问修饰符：无需 `public`。

#### 代码示例
```java
@Component
public class OrderService {
    private final OrderDao orderDao;
    
    // 推荐：构造方法注入（依赖不可变，初始化时完成）
    @Autowired
    public OrderService(OrderDao orderDao) {
        this.orderDao = orderDao;
    }
}

// 简化：Spring 4.3+ 后，若类只有一个构造方法，可省略 @Autowired
@Component
public class OrderService {
    private final OrderDao orderDao;
    
    // 自动注入，无需注解
    public OrderService(OrderDao orderDao) {
        this.orderDao = orderDao;
    }
}
```

### 3. 方法注入（含 setter 方法，适合需自定义逻辑的场景）
#### 核心规则
- 方法名任意（不一定是 setter），参数数量无限制；
- 每个参数都会从容器中匹配 Bean 注入；
- setter 方法是方法注入的“特殊案例”；
- 访问修饰符：无需 `public`。

#### 代码示例
```java
@Component
public class CartService {
    private CartDao cartDao;
    private RedisClient redisClient;
    
    // 普通方法注入（多个参数，自定义逻辑）
    @Autowired
    public void init(CartDao cartDao, RedisClient redisClient) {
        this.cartDao = cartDao;
        this.redisClient = redisClient;
        // 可添加初始化逻辑，比如初始化连接池
        redisClient.init();
    }
    
    // setter 方法注入（特殊的方法注入）
    @Autowired
    public void setCartDao(CartDao cartDao) {
        this.cartDao = cartDao;
    }
}
```

### 4. 参数注入（极少用，仅测试场景生效）
#### 核心规则
- Spring 5.0+ 支持标注在单个方法/构造方法参数上；
- 框架大部分模块忽略此用法，**仅 Spring Test（JUnit 测试）** 支持；
- 实际开发中几乎不用，了解即可。

#### 代码示例（测试场景）
```java
@SpringBootTest
public class UserTest {
    @Test
    public void test(@Autowired UserDao userDao) {
        // 测试中参数级注入生效，直接使用 userDao
        userDao.query();
    }
}
```

## 三、关键进阶规则
### 1. `required` 语义（必需/可选依赖）
- `required = true`（默认）：依赖必须存在，否则抛 `NoSuchBeanDefinitionException`；
- `required = false`：依赖不存在时，注入 `null`，不报错；
- 覆盖规则：单个参数可通过 `Optional`/`@Nullable` 覆盖全局 `required` 语义。

#### 代码示例
```java
@Component
public class PayService {
    // 全局 required = true，但参数用 Optional 覆盖 → 依赖不存在时注入 Optional.empty()
    @Autowired
    public PayService(Optional<AlipayClient> alipayClient) {
        alipayClient.ifPresent(client -> client.init());
    }
    
    // required = false → 依赖不存在时，wechatClient = null
    @Autowired(required = false)
    private WechatClient wechatClient;
    
    // 参数标注 @Nullable → 依赖不存在时，unionPayClient = null
    @Autowired
    public void setUnionPayClient(@Nullable UnionPayClient unionPayClient) {
        this.unionPayClient = unionPayClient;
    }
}
```

### 2. 数组/集合/Map 的注入规则
这是新手容易踩坑的点，核心逻辑：**注入“所有匹配类型”的 Bean，而非单个 Bean**。

| 类型                   | 注入规则                                                                 |
|----------------------|----------------------------------------------------------------------|
| 数组/Collection        | 注入容器中所有匹配泛型类型的 Bean，按 `@Order`/`Ordered` 排序，无则按注册顺序 |
| Map                  | Key = Bean 名称（String 类型），Value = 匹配类型的 Bean，排序规则同上       |

#### 代码示例
```java
// 定义多个同类型 Bean
@Component("alipay")
@Order(1)
public class Alipay implements PayProcessor {}

@Component("wechat")
@Order(2)
public class WechatPay implements PayProcessor {}

// 注入集合
@Component
public class PayProcessorManager {
    // 注入所有 PayProcessor 类型的 Bean → 包含 Alipay、WechatPay
    @Autowired
    private List<PayProcessor> payProcessors;
    
    // 注入 Map → key=alipay/wechat，value=对应 Bean
    @Autowired
    private Map<String, PayProcessor> payProcessorMap;
    
    public void printAll() {
        // 输出：[Alipay, WechatPay]（按 @Order 排序）
        System.out.println(payProcessors);
        // 输出：{alipay=Alipay, wechat=WechatPay}
        System.out.println(payProcessorMap);
    }
}
```

### 3. 禁用场景：BeanPostProcessor/BeanFactoryPostProcessor
#### 核心原因
Spring 的注入操作是通过 `BeanPostProcessor`（Bean 后置处理器）实现的，而 `BeanPostProcessor` 本身的初始化优先级高于普通 Bean。如果给 `BeanPostProcessor` 类型的 Bean 用 `@Autowired`，此时注入逻辑还未初始化，会导致依赖注入失败。

#### 反例（禁止这么写）
```java
// 错误：BeanPostProcessor 不能用 @Autowired 注入依赖
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {
    // 注入会失败！
    @Autowired
    private UserDao userDao;
    
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }
}
```

## 四、总结（核心知识点）
1. `@Autowired` 核心是“自动匹配 Bean 并注入”，支持构造方法/字段/方法/参数，推荐优先用**构造方法注入**；
2. `required` 属性控制依赖是否必需，可通过 `Optional`/`@Nullable` 覆盖单个参数的语义；
3. 数组/集合/Map 注入时，会获取“所有匹配类型”的 Bean，而非单个；
4. 禁止在 `BeanPostProcessor`/`BeanFactoryPostProcessor` 中使用 `@Autowired`，会导致注入失败。

掌握这些规则，你就能正确、规范地使用 `@Autowired`，避开新手常见的注入失败、空指针等问题。