

你需要一套能**全方位测试 Spring Boot WebSocket 核心功能**的实战代码，且客户端用 Postman 来测试（Postman 原生支持 WebSocket 协议），我会提供完整的服务端代码 + Postman 测试步骤，覆盖连接、单发/群发消息、连接关闭、异常处理、自定义参数传递等核心场景。

### 一、核心实现思路
1. 服务端：实现支持**纯 WebSocket 协议**（Postman 原生支持）的端点，覆盖连接、消息收发、断开、异常、自定义标识（用户ID）等功能；
2. 测试维度：连接建立、文本消息收发、单发/群发、连接关闭、异常模拟、自定义参数传递；
3. 客户端：基于 Postman 的 WebSocket 客户端完成所有测试，无需前端页面。

### 二、完整服务端代码
#### 1. 引入依赖（pom.xml）
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.2</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.example</groupId>
    <artifactId>springboot-websocket-test</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>springboot-websocket-test</name>
    <description>SpringBoot WebSocket 全功能测试</description>
    <properties>
        <java.version>17</java.version>
    </properties>
    <dependencies>
        <!-- WebSocket 核心依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-websocket</artifactId>
        </dependency>
        <!-- Web 依赖（可选，用于健康检查） -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- 日志增强（方便查看测试日志） -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <!-- 测试依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

#### 2. WebSocket 配置类（支持纯WebSocket协议，适配Postman）
```java
package com.example.websocket.config;

import com.example.websocket.handler.TestWebSocketHandler;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;
import org.springframework.web.socket.server.standard.ServerEndpointExporter;

/**
 * WebSocket 配置类
 * 注意：Postman 测试用纯 WebSocket 协议，无需 SockJS 降级
 */
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    private final TestWebSocketHandler testWebSocketHandler;

    public WebSocketConfig(TestWebSocketHandler testWebSocketHandler) {
        this.testWebSocketHandler = testWebSocketHandler;
    }

    /**
     * 注册WebSocket端点（纯WebSocket协议，ws://开头）
     */
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        // 核心测试端点：ws://localhost:8080/ws/test
        registry.addHandler(testWebSocketHandler, "/ws/test")
                .setAllowedOrigins("*"); // 允许跨域（Postman测试需开启）
    }

    /**
     * 支持JSR-356标准的@ServerEndpoint注解（可选，补充测试）
     */
    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }
}
```

#### 3. 全功能WebSocket处理器（覆盖所有测试场景）
```java
package com.example.websocket.handler;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.CloseStatus;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.TextWebSocketHandler;

import java.io.IOException;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * 全功能WebSocket处理器，覆盖：
 * 1. 连接建立/关闭
 * 2. 文本消息收发（单发/群发）
 * 3. 自定义参数解析（如用户ID）
 * 4. 异常处理
 * 5. 连接数统计
 */
@Slf4j
@Component
public class TestWebSocketHandler extends TextWebSocketHandler {

    // 在线连接数统计
    private static final AtomicInteger ONLINE_COUNT = new AtomicInteger(0);
    // 存储会话：key=会话ID，value=WebSocketSession（线程安全）
    private static final Map<String, WebSocketSession> SESSION_MAP = new ConcurrentHashMap<>();

    /**
     * 1. 连接建立成功触发
     */
    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        // 解析客户端传递的自定义参数（Postman可通过URL参数传递，如?userId=1001）
        Map<String, String> params = session.getAttributes();
        String userId = (String) params.get("userId");
        if (userId == null) {
            userId = "anonymous-" + session.getId().substring(0, 6); // 匿名用户
        }
        // 存储会话（用userId作为key更易测试）
        SESSION_MAP.put(userId, session);
        // 连接数+1
        int count = ONLINE_COUNT.incrementAndGet();

        log.info("【连接建立】用户ID：{}，会话ID：{}，当前在线数：{}", userId, session.getId(), count);
        // 向客户端发送连接成功消息（携带用户标识）
        session.sendMessage(new TextMessage(String.format(
                "连接成功！\n- 用户ID：%s\n- 会话ID：%s\n- 当前在线数：%d",
                userId, session.getId(), count
        )));
    }

    /**
     * 2. 接收客户端文本消息（核心测试场景）
     */
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        // 解析用户ID
        String userId = getUserId(session);
        String msg = message.getPayload();
        log.info("【接收消息】用户ID：{}，消息内容：{}", userId, msg);

        // 场景1：单发消息（回复当前客户端）
        session.sendMessage(new TextMessage(String.format("[单发回复] 你发送的消息：%s", msg)));

        // 场景2：群发消息（推送给所有在线客户端）
        if (msg.startsWith("群发：")) {
            String broadcastMsg = msg.substring(3);
            broadcastMessage(userId, broadcastMsg);
        }

        // 场景3：模拟异常（测试异常处理）
        if (msg.equals("触发异常")) {
            throw new RuntimeException("客户端主动触发的测试异常");
        }

        // 场景4：获取连接信息（返回给客户端）
        if (msg.equals("获取连接信息")) {
            StringBuilder info = new StringBuilder();
            info.append("【连接信息】\n");
            info.append("- 会话ID：").append(session.getId()).append("\n");
            info.append("- 用户ID：").append(userId).append("\n");
            info.append("- 在线数：").append(ONLINE_COUNT.get()).append("\n");
            info.append("- 所有在线用户：").append(SESSION_MAP.keySet());
            session.sendMessage(new TextMessage(info.toString()));
        }
    }

    /**
     * 3. 连接关闭触发
     */
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
        String userId = getUserId(session);
        // 移除会话
        SESSION_MAP.remove(userId);
        // 连接数-1
        int count = ONLINE_COUNT.decrementAndGet();

        log.info("【连接关闭】用户ID：{}，会话ID：{}，关闭状态：{}，当前在线数：{}",
                userId, session.getId(), status, count);
        // 广播连接关闭消息
        broadcastMessage("系统", String.format("用户【%s】已断开连接，当前在线数：%d", userId, count));
    }

    /**
     * 4. 异常处理（测试异常场景）
     */
    @Override
    public void handleTransportError(WebSocketSession session, Throwable exception) throws Exception {
        String userId = getUserId(session);
        log.error("【连接异常】用户ID：{}，会话ID：{}，异常信息：{}",
                userId, session.getId(), exception.getMessage(), exception);

        // 异常后关闭连接
        if (session.isOpen()) {
            session.close(CloseStatus.SERVER_ERROR);
        }
        // 清理会话
        SESSION_MAP.remove(userId);
        ONLINE_COUNT.decrementAndGet();
        // 发送异常提示
        broadcastMessage("系统", String.format("用户【%s】连接异常：%s", userId, exception.getMessage()));
    }

    // ===================== 工具方法 =====================
    /**
     * 获取用户ID（从会话属性中）
     */
    private String getUserId(WebSocketSession session) {
        Map<String, String> params = session.getAttributes();
        String userId = (String) params.get("userId");
        return userId == null ? "anonymous-" + session.getId().substring(0, 6) : userId;
    }

    /**
     * 群发消息
     */
    private void broadcastMessage(String senderId, String msg) throws IOException {
        String broadcastContent = String.format("[群发] 【%s】：%s", senderId, msg);
        log.info("【群发消息】发送者：{}，消息：{}，接收人数：{}", senderId, msg, SESSION_MAP.size());

        for (WebSocketSession session : SESSION_MAP.values()) {
            if (session.isOpen()) {
                session.sendMessage(new TextMessage(broadcastContent));
            }
        }
    }

    /**
     * 主动向指定用户发送消息（供外部调用，如接口触发）
     */
    public void sendMsgToUser(String userId, String msg) throws IOException {
        WebSocketSession session = SESSION_MAP.get(userId);
        if (session != null && session.isOpen()) {
            session.sendMessage(new TextMessage(String.format("[服务端主动推送] %s", msg)));
            log.info("【主动推送】用户ID：{}，消息：{}", userId, msg);
        } else {
            log.warn("【主动推送失败】用户ID：{} 未在线", userId);
        }
    }
}
```

#### 4. 补充：REST接口（测试服务端主动推送）
```java
package com.example.websocket.controller;

import com.example.websocket.handler.TestWebSocketHandler;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.io.IOException;

/**
 * REST接口：测试服务端主动向WebSocket客户端推送消息
 */
@Slf4j
@RestController
@RequestMapping("/api/websocket")
@RequiredArgsConstructor
public class WebSocketApiController {

    private final TestWebSocketHandler testWebSocketHandler;

    /**
     * 主动向指定用户推送消息
     * 测试URL：http://localhost:8080/api/websocket/send?userId=1001&msg=测试主动推送
     */
    @PostMapping("/send")
    public ResponseEntity<String> sendMsgToUser(
            @RequestParam String userId,
            @RequestParam String msg) {
        try {
            testWebSocketHandler.sendMsgToUser(userId, msg);
            return ResponseEntity.ok("推送成功！");
        } catch (IOException e) {
            log.error("推送失败", e);
            return ResponseEntity.internalServerError().body("推送失败：" + e.getMessage());
        }
    }
}
```

#### 5. 启动类
```java
package com.example.websocket;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringbootWebsocketTestApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringbootWebsocketTestApplication.class, args);
        System.out.println("===== WebSocket测试服务启动成功 =====");
        System.out.println("WebSocket端点：ws://localhost:8080/ws/test");
        System.out.println("主动推送接口：POST http://localhost:8080/api/websocket/send");
    }
}
```

### 三、Postman 测试步骤（全场景覆盖）
#### 前提：启动Spring Boot服务（默认端口8080）

#### 场景1：建立WebSocket连接
1. 打开Postman，点击左侧「New」→ 选择「WebSocket Request」；
2. 在地址栏输入：`ws://localhost:8080/ws/test?userId=1001`（自定义userId=1001，方便标识）；
3. 点击「Connect」按钮，成功后会看到服务端返回的连接成功消息：
   ```
   连接成功！
   - 用户ID：1001
   - 会话ID：xxxxxx
   - 当前在线数：1
   ```
4. 日志控制台会打印：`【连接建立】用户ID：1001，会话ID：xxxxxx，当前在线数：1`。

#### 场景2：单发消息测试
1. 在Postman的「Message」输入框输入：`Hello WebSocket`；
2. 点击「Send」，会收到服务端单发回复：`[单发回复] 你发送的消息：Hello WebSocket`；
3. 日志控制台打印：`【接收消息】用户ID：1001，消息内容：Hello WebSocket`。

#### 场景3：群发消息测试
1. 打开**第二个Postman WebSocket窗口**，连接：`ws://localhost:8080/ws/test?userId=1002`；
2. 在第一个窗口（1001）输入：`群发：大家好！`，点击Send；
3. 两个窗口都会收到群发消息：`[群发] 【1001】：大家好！`；
4. 日志控制台打印群发相关日志。

#### 场景4：异常触发测试
1. 在Postman输入：`触发异常`，点击Send；
2. 会看到连接自动关闭，服务端日志打印异常信息：`【连接异常】用户ID：1001，会话ID：xxxxxx，异常信息：客户端主动触发的测试异常`；
3. 其他在线客户端会收到：`[群发] 【系统】：用户【1001】连接异常：客户端主动触发的测试异常`。

#### 场景5：获取连接信息测试
1. 重新建立连接（userId=1001）；
2. 输入：`获取连接信息`，点击Send；
3. 收到服务端返回的详细连接信息：
   ```
   【连接信息】
   - 会话ID：xxxxxx
   - 用户ID：1001
   - 在线数：2
   - 所有在线用户：[1001, 1002]
   ```

#### 场景6：服务端主动推送测试
1. 保持WebSocket连接（userId=1001）；
2. 打开Postman的「New」→「HTTP Request」，创建POST请求：
    - URL：`http://localhost:8080/api/websocket/send`；
    - 参数：`userId=1001`，`msg=这是服务端主动推送的消息`；
3. 点击Send，WebSocket窗口会收到：`[服务端主动推送] 这是服务端主动推送的消息`；
4. 日志控制台打印：`【主动推送】用户ID：1001，消息：这是服务端主动推送的消息`。

#### 场景7：连接关闭测试
1. 在Postman WebSocket窗口点击「Disconnect」；
2. 服务端日志打印：`【连接关闭】用户ID：1001，会话ID：xxxxxx，关闭状态：NORMAL，当前在线数：1`；
3. 其他在线客户端收到：`[群发] 【系统】：用户【1001】已断开连接，当前在线数：1`。

### 四、关键注意事项
1. **Postman版本**：确保Postman版本≥10.0（低版本可能不支持WebSocket）；
2. **跨域配置**：服务端`setAllowedOrigins("*")`是测试用，生产环境需指定具体域名；
3. **参数传递**：Postman可通过URL参数（如`?userId=1001`）传递自定义参数，服务端从`session.getAttributes()`解析；
4. **线程安全**：使用`ConcurrentHashMap`存储会话，`AtomicInteger`统计连接数，避免多线程问题；
5. **异常处理**：所有IO操作需捕获异常，发送消息前检查`session.isOpen()`。

### 总结
1. 这套代码覆盖了Spring Boot WebSocket的**核心功能**：连接管理、消息收发（单发/群发）、异常处理、主动推送、参数解析；
2. Postman测试步骤对应所有核心场景，可全方位验证WebSocket功能；
3. 代码中添加了详细日志和注释，便于你理解每个功能的实现逻辑，也可直接用于生产环境的基础改造。

如果需要补充「二进制消息测试」「心跳检测」「权限认证」等进阶功能的代码，或者Postman测试的截图指引，我可以继续完善。