





## l2 相关资料 

### Spring框架文档
[链接](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/index.html)

### Web Servlet    
Spring MVC、WebSocket、SockJS、STOMP 消息传递。[链接](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/web.html#websocket-server-handshake)

### Web Reactive 
Spring WebFlux, WebClient, WebSocket, RSocket.
[链接](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/web-reactive.html#spring-web-reactive)

### Spring官网
websocket-api用法
[链接](https://docs.spring.io/spring-framework/reference/web/websocket/server.html)

STOMP
[链接](https://docs.spring.io/spring-framework/reference/web/websocket/stomp.html)

### 教程：

[掘金-Spring Boot 如何集成使用 WebSocket](https://juejin.cn/post/7474375249149984787)




## l2 WebSocketConfigurer接口

```java
/**
 * Defines callback methods to configure the WebSocket request handling
 * via {@link org.springframework.web.socket.config.annotation.EnableWebSocket @EnableWebSocket}.
 * <p>定义回调方法，用于通过 {@link org.springframework.web.socket.config.annotation.EnableWebSocket @EnableWebSocket} 注解配置 WebSocket 请求的处理逻辑。
 *
 * @author Rossen Stoyanchev
 * @since 4.0
 * <p>@作者 罗森·斯托扬切夫
 * @始于 Spring 框架 4.0 版本
 */
public interface WebSocketConfigurer {

	/**
	 * Register {@link WebSocketHandler WebSocketHandlers} including SockJS fallback options if desired.
	 * <p>注册 {@link WebSocketHandler WebSocket处理器}，如需支持 SockJS 降级方案也可一并配置。
	 */
	void registerWebSocketHandlers(WebSocketHandlerRegistry registry);

}
```

### 补充关键术语解释（辅助理解）
| 英文术语 | 中文翻译 | 补充说明 |
|----------|----------|----------|
| `callback methods` | 回调方法 | 指由框架触发执行的方法，开发者通过实现该接口重写方法完成自定义配置 |
| `@EnableWebSocket` | 启用WebSocket注解 | Spring 注解，标注在配置类上以开启 WebSocket 功能支持 |
| `SockJS fallback` | SockJS 降级方案 | SockJS 是 WebSocket 的降级兼容方案，当浏览器不支持 WebSocket 时，自动切换为 XHR/JSONP 等方式通信 |
| `WebSocketHandlerRegistry` | WebSocket处理器注册器 | Spring 提供的注册表对象，用于注册 WebSocketHandler 并配置映射路径、拦截器、SockJS 等 |







## l2 SprigBoot中如何使用 websocket
你想知道在 Spring Boot 中如何使用 WebSocket 实现客户端与服务端的双向实时通信，我会给你一套完整、可直接运行的示例代码，从依赖引入到功能实现一步到位。

### 一、核心实现思路
1. 引入 Spring Boot 提供的 WebSocket 起步依赖
2. 配置 WebSocket 核心处理器（注册端点、设置消息处理器）
3. 编写消息处理的业务逻辑（处理连接、消息收发、断开连接）
4. 编写前端页面测试实时通信

### 二、完整代码实现
#### 1. 引入依赖（pom.xml）
```xml
<!-- Spring Boot Web 依赖（基础web环境） -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!-- Spring Boot WebSocket 依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

#### 2. WebSocket 配置类
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;
import org.springframework.web.socket.handler.TextWebSocketHandler;

/**
 * WebSocket 核心配置类
 * @EnableWebSocket：开启WebSocket支持
 */
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    // 注入自定义的消息处理器
    private final MyWebSocketHandler myWebSocketHandler;

    public WebSocketConfig(MyWebSocketHandler myWebSocketHandler) {
        this.myWebSocketHandler = myWebSocketHandler;
    }

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        // 注册WebSocket端点：/ws/chat
        // setAllowedOrigins("*")：允许跨域（生产环境建议指定具体域名）
        // withSockJS()：降级支持（当浏览器不支持WebSocket时，用SockJS模拟）
        registry.addHandler(myWebSocketHandler, "/ws/chat")
                .setAllowedOrigins("*")
                .withSockJS();
    }
}
```

#### 3. 自定义 WebSocket 消息处理器
```java
import org.springframework.stereotype.Component;
import org.springframework.web.socket.CloseStatus;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.TextWebSocketHandler;

import java.io.IOException;
import java.util.concurrent.ConcurrentHashMap;

/**
 * 自定义WebSocket消息处理器
 * 处理连接建立、消息接收、连接关闭等事件
 */
@Component
public class MyWebSocketHandler extends TextWebSocketHandler {

    // 存储在线会话（ConcurrentHashMap保证线程安全）
    private static final ConcurrentHashMap<String, WebSocketSession> SESSION_MAP = new ConcurrentHashMap<>();

    /**
     * 连接建立成功时触发
     */
    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        // 获取客户端标识（可从session中获取参数，如用户ID）
        String clientId = session.getId();
        SESSION_MAP.put(clientId, session);
        System.out.println("客户端[" + clientId + "]已连接，当前在线数：" + SESSION_MAP.size());
        // 向客户端发送连接成功消息
        session.sendMessage(new TextMessage("连接成功！你的ID：" + clientId));
    }

    /**
     * 接收客户端发送的消息
     */
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        String clientId = session.getId();
        String receiveMsg = message.getPayload();
        System.out.println("收到客户端[" + clientId + "]消息：" + receiveMsg);

        // 1. 单发：回复当前客户端
        session.sendMessage(new TextMessage("服务端已收到：" + receiveMsg));

        // 2. 群发：向所有在线客户端推送消息
        for (WebSocketSession s : SESSION_MAP.values()) {
            if (s.isOpen()) {
                s.sendMessage(new TextMessage("客户端[" + clientId + "]：" + receiveMsg));
            }
        }
    }

    /**
     * 连接关闭时触发
     */
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
        String clientId = session.getId();
        SESSION_MAP.remove(clientId);
        System.out.println("客户端[" + clientId + "]已断开连接，当前在线数：" + SESSION_MAP.size());
    }

    /**
     * 发生错误时触发
     */
    @Override
    public void handleTransportError(WebSocketSession session, Throwable exception) throws Exception {
        String clientId = session.getId();
        System.out.println("客户端[" + clientId + "]连接出错：" + exception.getMessage());
        if (session.isOpen()) {
            session.close();
        }
        SESSION_MAP.remove(clientId);
    }

    /**
     * 主动向指定客户端发送消息
     */
    public void sendMsgToClient(String clientId, String msg) throws IOException {
        WebSocketSession session = SESSION_MAP.get(clientId);
        if (session != null && session.isOpen()) {
            session.sendMessage(new TextMessage("服务端主动推送：" + msg));
        }
    }
}
```

#### 4. 前端测试页面（static/chat.html）
在 `resources/static` 目录下创建 `chat.html`（Spring Boot 会自动映射静态资源）：
```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>WebSocket 测试</title>
</head>
<body>
<h3>WebSocket 实时聊天</h3>
<div id="msgBox" style="width: 500px; height: 300px; border: 1px solid #ccc; padding: 10px; overflow-y: auto;"></div>
<input type="text" id="msgInput" placeholder="请输入消息" style="width: 400px;">
<button onclick="sendMsg()">发送</button>

<!-- 引入SockJS（降级支持） -->
<script src="https://cdn.jsdelivr.net/npm/sockjs-client@1.6.1/dist/sockjs.min.js"></script>
<script>
    // 1. 创建WebSocket连接（使用SockJS降级）
    const socket = new SockJS('http://localhost:8080/ws/chat');

    // 2. 监听连接成功
    socket.onopen = function () {
        console.log("连接成功");
        appendMsg("【系统】连接成功！");
    };

    // 3. 监听接收消息
    socket.onmessage = function (e) {
        appendMsg(e.data);
    };

    // 4. 监听连接关闭
    socket.onclose = function () {
        appendMsg("【系统】连接已断开！");
    };

    // 5. 监听错误
    socket.onerror = function (err) {
        console.error("连接出错：", err);
        appendMsg("【系统】连接出错：" + err.message);
    };

    // 发送消息
    function sendMsg() {
        const input = document.getElementById('msgInput');
        const msg = input.value.trim();
        if (msg) {
            socket.send(msg);
            input.value = "";
        }
    }

    // 追加消息到页面
    function appendMsg(msg) {
        const msgBox = document.getElementById('msgBox');
        const p = document.createElement('p');
        p.innerText = new Date().toLocaleString() + "：" + msg;
        msgBox.appendChild(p);
        // 滚动到底部
        msgBox.scrollTop = msgBox.scrollHeight;
    }
</body>
</html>
```

### 三、测试步骤
1. 启动 Spring Boot 应用（默认端口 8080）
2. 打开浏览器访问：`http://localhost:8080/chat.html`（可打开多个标签页模拟多客户端）
3. 在输入框输入消息并点击“发送”，可看到：
    - 服务端控制台打印收到的消息
    - 当前客户端收到回复
    - 所有在线客户端收到群发消息

### 四、关键知识点说明
1. **核心注解**：
    - `@EnableWebSocket`：开启 WebSocket 支持
    - `WebSocketConfigurer`：配置 WebSocket 端点和处理器
2. **核心类**：
    - `TextWebSocketHandler`：处理文本类型的 WebSocket 消息（还有 `BinaryWebSocketHandler` 处理二进制消息）
    - `WebSocketSession`：代表单个客户端的连接会话，可获取客户端信息、发送消息
3. **SockJS**：
    - 作用：当浏览器不支持 WebSocket 协议（如低版本IE）时，自动降级为轮询等方式
    - 使用：前端引入 SockJS 库，后端配置 `withSockJS()`
4. **线程安全**：
    - 用 `ConcurrentHashMap` 存储会话，避免多线程并发问题
    - 发送消息前需检查 `session.isOpen()`，防止向已关闭的连接发消息

### 总结
1. Spring Boot 集成 WebSocket 核心是引入起步依赖 + 配置端点 + 实现消息处理器；
2. `TextWebSocketHandler` 是处理文本消息的核心类，需重写连接、消息、关闭等核心方法；
3. 生产环境需注意：限制跨域域名、添加认证授权、处理会话超时、监控连接状态。

如果需要实现**基于STOMP协议的WebSocket（更规范的消息订阅/发布模式）**，或者**WebSocket + Spring Security 权限控制**，我可以补充对应的代码示例。


