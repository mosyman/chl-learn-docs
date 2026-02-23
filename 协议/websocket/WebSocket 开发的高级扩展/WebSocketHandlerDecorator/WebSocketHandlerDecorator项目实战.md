




### 一、核心需求理解
你想了解 `WebSocketHandlerDecorator` 在**企业级项目**中的实际用法，核心是掌握如何基于这个装饰器实现生产环境所需的通用能力（如监控、限流、异常增强、日志脱敏等），而非仅停留在Spring默认提供的基础日志/异常处理层面。

### 二、企业级用法详解
`WebSocketHandlerDecorator` 是Spring基于**装饰器模式**设计的扩展类，企业级场景中核心价值是：**无侵入式增强WebSocket业务逻辑**（无需修改原有`WebSocketHandler`代码），统一封装通用横切逻辑（监控、限流、安全、异常、日志）。

#### 1. 先明确核心基础（企业级开发前提）
在使用装饰器前，先定义一个**业务核心的WebSocketHandler**（企业项目中处理实际消息交互的类）：
```java
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.TextWebSocketHandler;

/**
 * 企业级业务核心的WebSocket处理器
 * 处理实际的消息收发、连接管理等业务逻辑
 */
public class BizWebSocketHandler extends TextWebSocketHandler {

    // 处理客户端发送的文本消息（核心业务逻辑）
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        String payload = message.getPayload();
        // 1. 模拟企业级业务处理：解析消息、调用业务服务、返回响应
        String response = "biz-process-" + payload;
        // 2. 向客户端返回处理结果
        session.sendMessage(new TextMessage(response));
    }

    // 连接建立成功（可做用户认证、连接初始化）
    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        String userId = (String) session.getAttributes().get("userId");
        System.out.println("用户[" + userId + "]建立WebSocket连接，sessionId：" + session.getId());
    }

    // 连接关闭（可做资源清理、离线状态更新）
    @Override
    public void afterConnectionClosed(WebSocketSession session, org.springframework.web.socket.CloseStatus status) throws Exception {
        String userId = (String) session.getAttributes().get("userId");
        System.out.println("用户[" + userId + "]关闭WebSocket连接，状态码：" + status.getCode());
    }
}
```

#### 2. 企业级核心装饰器场景实现
以下是5个最常见的企业级装饰器用法，均基于`WebSocketHandlerDecorator`实现，可单独使用或组合叠加。

##### 场景1：接口耗时监控（核心运维需求）
企业级项目需监控WebSocket接口的响应耗时，定位慢请求，装饰器实现无侵入式监控：
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.WebSocketHandlerDecorator;
import org.springframework.web.socket.handler.WebSocketHandlerDecoratorFactory;

/**
 * 耗时监控装饰器：记录消息处理、连接建立/关闭的耗时
 */
public class TimingWebSocketHandlerDecorator extends WebSocketHandlerDecorator {
    private static final Logger LOGGER = LoggerFactory.getLogger(TimingWebSocketHandlerDecorator.class);

    public TimingWebSocketHandlerDecorator(org.springframework.web.socket.WebSocketHandler delegate) {
        super(delegate);
    }

    // 重写消息处理方法，添加耗时监控
    @Override
    public void handleMessage(WebSocketSession session, org.springframework.web.socket.WebSocketMessage<?> message) throws Exception {
        long startTime = System.currentTimeMillis();
        try {
            // 执行原业务逻辑
            super.handleMessage(session, message);
        } finally {
            long cost = System.currentTimeMillis() - startTime;
            String userId = (String) session.getAttributes().get("userId");
            // 记录耗时日志（企业级可接入监控平台如Prometheus/Grafana）
            LOGGER.info("WebSocket消息处理耗时 | userId: {} | sessionId: {} | 耗时(ms): {} | 消息内容: {}",
                    userId, session.getId(), cost, message.getPayload());
            // 耗时超过阈值告警（企业级可接入告警平台如钉钉/短信）
            if (cost > 500) {
                LOGGER.warn("WebSocket消息处理超时 | userId: {} | 耗时(ms): {}", userId, cost);
            }
        }
    }

    // 连接建立耗时监控
    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        long startTime = System.currentTimeMillis();
        try {
            super.afterConnectionEstablished(session);
        } finally {
            long cost = System.currentTimeMillis() - startTime;
            LOGGER.info("WebSocket连接建立耗时 | sessionId: {} | 耗时(ms): {}", session.getId(), cost);
        }
    }

    // 提供装饰器工厂（便于批量应用）
    public static WebSocketHandlerDecoratorFactory factory() {
        return TimingWebSocketHandlerDecorator::new;
    }
}
```

##### 场景2：异常增强处理（生产环境必备）
Spring默认的`ExceptionWebSocketHandlerDecorator`仅返回1011状态码，企业级需：
- 记录详细异常日志（含用户/会话信息）；
- 区分异常类型（业务异常/系统异常），返回不同关闭状态码；
- 异常时清理会话资源、推送告警；
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.socket.CloseStatus;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.ExceptionWebSocketHandlerDecorator;

/**
 * 企业级异常装饰器：增强异常处理逻辑，区分异常类型、记录详细日志、清理资源
 */
public class BizExceptionWebSocketHandlerDecorator extends ExceptionWebSocketHandlerDecorator {
    private static final Logger LOGGER = LoggerFactory.getLogger(BizExceptionWebSocketHandlerDecorator.class);

    public BizExceptionWebSocketHandlerDecorator(org.springframework.web.socket.WebSocketHandler delegate) {
        super(delegate);
    }

    // 重写异常处理逻辑（核心扩展点）
    @Override
    protected void handleUncaughtException(WebSocketSession session, Throwable ex) throws Exception {
        String userId = (String) session.getAttributes().get("userId");
        String sessionId = session.getId();

        // 1. 区分异常类型，记录不同级别日志
        if (ex instanceof IllegalArgumentException) {
            // 业务异常：参数错误，记录WARN日志，返回1002状态码（协议定义：协议错误）
            LOGGER.warn("WebSocket业务异常 | userId: {} | sessionId: {} | 原因: {}", userId, sessionId, ex.getMessage(), ex);
            if (session.isOpen()) {
                session.close(CloseStatus.PROTOCOL_ERROR.withReason(ex.getMessage()));
            }
        } else {
            // 系统异常：记录ERROR日志，返回1011状态码，推送告警
            LOGGER.error("WebSocket系统异常 | userId: {} | sessionId: {}", userId, sessionId, ex);
            // 企业级：调用告警接口（钉钉/短信/邮件）
            // alertService.sendAlert("WebSocket异常", "userId: " + userId + ", 异常: " + ex.getMessage());
            if (session.isOpen()) {
                session.close(CloseStatus.SERVER_ERROR.withReason("服务器内部错误"));
            }
        }

        // 2. 清理会话资源（企业级：释放连接池、更新用户在线状态等）
        cleanSessionResource(session);
    }

    // 清理会话关联资源
    private void cleanSessionResource(WebSocketSession session) {
        String userId = (String) session.getAttributes().get("userId");
        // 示例：移除用户-会话映射
        // sessionManager.removeSession(userId, session.getId());
    }

    // 装饰器工厂
    public static WebSocketHandlerDecoratorFactory factory() {
        return BizExceptionWebSocketHandlerDecorator::new;
    }
}
```

##### 场景3：消息安全控制（日志脱敏+权限校验）
企业级需保证敏感信息不泄露、消息操作符合权限：
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.socket.WebSocketMessage;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.WebSocketHandlerDecorator;

/**
 * 安全装饰器：1. 消息内容脱敏（如手机号/身份证）；2. 操作权限校验
 */
public class SecurityWebSocketHandlerDecorator extends WebSocketHandlerDecorator {
    private static final Logger LOGGER = LoggerFactory.getLogger(SecurityWebSocketHandlerDecorator.class);
    // 权限校验服务（企业级注入真实的权限服务）
    private final PermissionService permissionService;

    public SecurityWebSocketHandlerDecorator(org.springframework.web.socket.WebSocketHandler delegate, PermissionService permissionService) {
        super(delegate);
        this.permissionService = permissionService;
    }

    @Override
    public void handleMessage(WebSocketSession session, WebSocketMessage<?> message) throws Exception {
        String userId = (String) session.getAttributes().get("userId");
        String payload = message.getPayload().toString();

        // 1. 权限校验：校验用户是否有权限发送此类型消息
        if (!permissionService.hasMessagePermission(userId, payload)) {
            throw new IllegalArgumentException("用户[" + userId + "]无权限发送此消息");
        }

        // 2. 消息脱敏：日志中隐藏敏感信息（如手机号138****1234）
        String desensitizedPayload = desensitize(payload);
        LOGGER.info("WebSocket消息接收（脱敏） | userId: {} | 消息: {}", userId, desensitizedPayload);

        // 3. 执行原业务逻辑
        super.handleMessage(session, message);
    }

    // 敏感信息脱敏（企业级可封装为工具类）
    private String desensitize(String payload) {
        // 示例：手机号脱敏
        return payload.replaceAll("(1[3-9]\\d{9})", "$1".replaceAll("(\\d{3})\\d{4}(\\d{4})", "$1****$2"));
    }

    // 装饰器工厂（需传入权限服务）
    public static WebSocketHandlerDecoratorFactory factory(PermissionService permissionService) {
        return delegate -> new SecurityWebSocketHandlerDecorator(delegate, permissionService);
    }

    // 模拟权限服务（企业级替换为真实实现）
    public interface PermissionService {
        boolean hasMessagePermission(String userId, String message);
    }
}
```

##### 场景4：连接限流（防止恶意连接攻击）
企业级需限制单用户/单IP的WebSocket连接数，防止资源耗尽：
```java
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.WebSocketHandlerDecorator;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * 限流装饰器：限制单用户的WebSocket连接数（企业级可扩展为IP限流）
 */
public class RateLimitWebSocketHandlerDecorator extends WebSocketHandlerDecorator {
    // 存储用户-连接数映射（企业级可替换为Redis分布式存储）
    private static final Map<String, AtomicInteger> USER_CONNECTION_COUNT = new ConcurrentHashMap<>();
    // 单用户最大连接数（企业级可配置化）
    private static final int MAX_CONNECTION_PER_USER = 3;

    public RateLimitWebSocketHandlerDecorator(org.springframework.web.socket.WebSocketHandler delegate) {
        super(delegate);
    }

    // 连接建立前校验限流规则
    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        String userId = (String) session.getAttributes().get("userId");
        if (userId == null) {
            session.close(org.springframework.web.socket.CloseStatus.POLICY_VIOLATION.withReason("未认证用户禁止连接"));
            return;
        }

        // 1. 累加用户连接数
        AtomicInteger count = USER_CONNECTION_COUNT.computeIfAbsent(userId, k -> new AtomicInteger(0));
        int currentCount = count.incrementAndGet();

        // 2. 超过阈值则关闭连接
        if (currentCount > MAX_CONNECTION_PER_USER) {
            session.close(org.springframework.web.socket.CloseStatus.POLICY_VIOLATION.withReason("单用户连接数超过上限：" + MAX_CONNECTION_PER_USER));
            // 回减计数
            count.decrementAndGet();
            return;
        }

        // 3. 执行原逻辑
        super.afterConnectionEstablished(session);
    }

    // 连接关闭时清理计数
    @Override
    public void afterConnectionClosed(WebSocketSession session, org.springframework.web.socket.CloseStatus status) throws Exception {
        String userId = (String) session.getAttributes().get("userId");
        if (userId != null) {
            AtomicInteger count = USER_CONNECTION_COUNT.get(userId);
            if (count != null) {
                int currentCount = count.decrementAndGet();
                if (currentCount <= 0) {
                    USER_CONNECTION_COUNT.remove(userId);
                }
            }
        }
        super.afterConnectionClosed(session, status);
    }

    // 装饰器工厂
    public static WebSocketHandlerDecoratorFactory factory() {
        return RateLimitWebSocketHandlerDecorator::new;
    }
}
```

#### 3. 企业级配置：装饰器组合与注册
企业项目中需将多个装饰器**叠加使用**（按优先级排序），通过Spring配置注册：
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;
import org.springframework.web.socket.handler.WebSocketHandlerDecoratorFactory;

/**
 * 企业级WebSocket配置：注册核心处理器+叠加装饰器
 */
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    // 模拟权限服务（企业级替换为真实Bean）
    @Bean
    public SecurityWebSocketHandlerDecorator.PermissionService permissionService() {
        return (userId, message) -> {
            // 示例：仅允许userId为"admin"的用户发送消息
            return "admin".equals(userId);
        };
    }

    // 注册WebSocket处理器+装饰器
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        // 1. 创建核心业务处理器
        BizWebSocketHandler bizHandler = new BizWebSocketHandler();

        // 2. 注册处理器，配置路径和跨域
        registry.addHandler(bizHandler, "/ws/biz")
                .setAllowedOrigins("https://your-domain.com") // 企业级严格限制跨域
                // 3. 叠加装饰器（顺序：限流→安全→耗时→异常，按优先级从外到内）
                .addDecoratorFactory(RateLimitWebSocketHandlerDecorator.factory())
                .addDecoratorFactory(SecurityWebSocketHandlerDecorator.factory(permissionService()))
                .addDecoratorFactory(TimingWebSocketHandlerDecorator.factory())
                .addDecoratorFactory(BizExceptionWebSocketHandlerDecorator.factory());
    }
}
```

#### 4. 企业级最佳实践补充
1. **装饰器顺序**：建议按「限流→安全→监控→异常」的顺序叠加，先做前置校验（限流/安全），再做过程监控，最后做异常兜底；
2. **分布式场景**：限流、连接计数等需替换为Redis等分布式存储，避免单机内存存储的问题；
3. **配置化**：将最大连接数、耗时阈值、脱敏规则等配置到Nacos/Apollo等配置中心，支持动态调整；
4. **优雅关闭**：装饰器中需处理`session.close()`的异常，避免关闭已断开的连接导致报错；
5. **日志规范**：统一日志格式（如包含traceId），便于链路追踪（企业级可接入SkyWalking/Zipkin）；
6. **资源释放**：装饰器中需确保`finally`块清理资源，避免内存泄漏（如连接计数、会话映射）。

### 三、总结
`WebSocketHandlerDecorator` 在企业级项目中的核心价值是**无侵入式增强**，关键用法可总结为3点：
1. **核心场景**：基于装饰器实现限流、安全（脱敏/权限）、监控（耗时）、异常增强等通用能力，不侵入业务逻辑；
2. **配置方式**：通过`addDecoratorFactory`叠加多个装饰器，按「前置校验→过程监控→异常兜底」的顺序排列；
3. **企业级扩展**：适配分布式场景（Redis限流）、配置化（动态阈值）、链路追踪（日志traceId），保证高可用和可维护性。

通过这种方式，既能保证WebSocket业务逻辑的简洁性，又能统一封装生产环境所需的通用能力，符合企业级项目的可扩展性和可维护性要求。