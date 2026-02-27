这是一个典型的 **Postman / API 调试工具** 界面，我帮你逐部分拆解：

---

#### 1. 顶部信息区
- **`mvc-demo / New Request`**：当前请求所属的集合/文件夹名称，以及请求名称。
- **`POST`**：请求方法为 POST，常用于提交数据（如登录、表单提交）。
- **`localhost:8080/api/login`**：请求的目标 URL，指向本地运行的 Spring Boot 应用的登录接口。

---

#### 2. 核心操作区
- **`Send`**：点击后发送请求到服务器。
- **`Save`**：保存当前请求到集合中，方便后续复用。
- **`Share`**：分享这个请求配置给他人。

---

#### 3. 标签页区
- **`Docs`**：为该接口编写文档说明。
- **`Params`**（当前选中）：配置 URL 查询参数（Query Parameters），即 URL 中 `?` 后面的键值对。
- **`Authorization`**：配置认证信息（如 Token、Basic Auth 等）。
- **`Headers`**：配置请求头（当前显示有 8 个，如 Content-Type、User-Agent 等）。
- **`Body`**：配置请求体（绿色点表示已配置，通常用于 POST/PUT 提交 JSON、表单等数据）。
- **`Scripts`**：编写前置/后置脚本（如请求前生成签名，请求后校验响应）。
- **`Settings`**：请求的高级设置。
- **`Cookies`**：管理请求携带的 Cookie。

---

#### 4. Query Params 区
- 这里用于添加 URL 查询参数，例如：
    - Key: `username`，Value: `admin`
    - Key: `password`，Value: `123456`
      最终 URL 会变成：`localhost:8080/api/login?username=admin&password=123456`
- 注意：对于登录接口，**不建议**把密码等敏感信息放在 Query Params 中，因为 URL 会被日志、浏览器历史记录等明文保存，更安全的做法是在 `Body` 中以 JSON 或表单形式提交。

---

#### 5. 典型使用场景（登录接口）
1. 切换到 `Body` 标签页。
2. 选择 `raw` 并指定为 `JSON`。
3. 输入请求体：
   ```json
   {
     "username": "admin",
     "password": "123456"
   }
   ```
4. 点击 `Send` 发送请求，服务器会验证用户名和密码并返回登录结果（如 Token）。

---

## Body 标签页

Body 标签页的请求体格式选项，用于定义 POST/PUT 请求时如何向服务器发送数据。下面逐个解释：

---

#### 1. none
- 含义：不发送任何请求体。
- 适用场景：GET、DELETE 等通常不需要请求体的方法，或明确不需要提交数据的 POST 请求。
- 注意：虽然 HTTP 规范允许 GET 带 Body，但很多服务器和代理会忽略或拒绝，所以不推荐在 GET 中使用 Body。

---

#### 2. form-data
- 含义：对应 `multipart/form-data` 编码格式，是 HTML 表单的原生提交方式之一。
- 特点：
    - 支持键值对文本数据，也支持**文件上传**（File 类型）。
    - 每个字段都会被封装成独立的“部分”（part），适合混合提交文本和二进制文件。
- 适用场景：
    - 上传图片、文档等文件。
    - 提交包含文件的表单（如用户头像上传）。
- 示例：
    - Key: `avatar`，Value: 选择本地图片文件
    - Key: `username`，Value: `zhangsan`

---

#### 3. x-www-form-urlencoded
- 含义：对应 `application/x-www-form-urlencoded` 编码格式，是 HTML 表单默认的提交方式。
- 特点：
    - 键值对会被编码为类似 URL 查询字符串的格式，如 `username=zhangsan&password=123456`。
    - 不支持文件上传，仅支持文本数据。
    - 数据会被 URL 编码（如空格变为 `%20`）。
- 适用场景：
    - 纯文本表单提交（如登录、注册）。
    - 数据量小、结构简单的场景。

---

#### 4. raw
- 含义：原始文本格式，可手动指定内容类型（如 JSON、XML、Text、JavaScript 等）。
- 特点：
    - 完全自定义请求体内容，不受表单格式限制。
    - 最常用于发送 JSON 数据，需手动设置 `Content-Type: application/json`。
- 适用场景：
    - 发送 JSON 或 XML 格式的 API 请求（现代 RESTful API 最常用）。
    - 发送纯文本、HTML 或其他自定义格式数据。
- 示例（JSON）：
  ```json
  {
    "username": "zhangsan",
    "password": "123456"
  }
  ```

---

#### 5. binary
- 含义：二进制格式，用于发送非文本的二进制数据。
- 特点：
    - 直接发送文件的二进制流，不进行任何编码。
    - 通常需要手动设置 `Content-Type`（如 `image/png`、`application/octet-stream`）。
- 适用场景：
    - 上传单个二进制文件（如图片、音频、视频）。
    - 发送原始字节流数据。

---

#### 6. GraphQL
- 含义：专门用于发送 GraphQL 查询的格式。
- 特点：
    - 提供专门的编辑器，支持 GraphQL 查询语法高亮和变量定义。
    - 自动设置 `Content-Type: application/json`，并将查询和变量封装成标准的 GraphQL 请求格式。
- 适用场景：
    - 与 GraphQL API 交互，发送查询（query）或变更（mutation）。

---

#### 对比总结表

| 格式 | 编码/类型 | 支持文件 | 典型用途 |
| :--- | :--- | :--- | :--- |
| form-data | `multipart/form-data` | ✅ | 文件上传 + 文本表单 |
| x-www-form-urlencoded | `application/x-www-form-urlencoded` | ❌ | 纯文本表单提交 |
| raw | 自定义（如 JSON/XML） | ❌ | REST API JSON/XML 请求 |
| binary | 二进制流 | ✅ | 单文件二进制上传 |
| GraphQL | `application/json` | ❌ | GraphQL 查询 |


---

# Spring MVC 接收不同 Body 格式数据示例
## 前置准备
确保 Spring Boot 项目已引入 Web 依赖（pom.xml）：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

## 1. 接收 x-www-form-urlencoded 格式
### 控制器代码
```java
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/api/form")
public class FormUrlEncodedController {

    /**
     * 接收 x-www-form-urlencoded 格式数据
     * 对应 Postman Body → x-www-form-urlencoded
     */
    @PostMapping("/login")
    public Map<String, Object> login(
            // 单个参数接收（必填，可加 defaultValue 设置默认值）
            @RequestParam String username,
            @RequestParam(required = false, defaultValue = "123456") String password) {
        
        Map<String, Object> result = new HashMap<>();
        result.put("code", 200);
        result.put("msg", "登录成功");
        result.put("data", Map.of("username", username, "password", password));
        return result;
    }
}
```

### Postman 测试说明
1. 请求方法：POST
2. URL：`http://localhost:8080/api/form/login`
3. Body 选择：`x-www-form-urlencoded`
4. 添加参数：
    - Key: `username`，Value: `admin`
    - Key: `password`，Value: `123456`
5. 发送请求，返回 JSON 结果。

### 核心注解
- `@RequestParam`：接收 URL 查询参数或 `x-www-form-urlencoded` 格式的表单参数；
- `required = false`：表示参数非必填；
- `defaultValue`：参数未传时的默认值。

## 2. 接收 form-data 格式（含文件上传）
### 控制器代码
```java
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.multipart.MultipartFile;

import java.io.File;
import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/api/form-data")
public class FormDataController {

    /**
     * 接收 form-data 格式数据（文本 + 文件）
     * 对应 Postman Body → form-data
     */
    @PostMapping("/upload")
    public Map<String, Object> uploadFile(
            @RequestParam String username,          // 文本参数
            @RequestParam MultipartFile avatar) {   // 文件参数
        
        Map<String, Object> result = new HashMap<>();
        try {
            // 保存文件到本地（示例路径，可根据实际修改）
            String filePath = "uploads/" + avatar.getOriginalFilename();
            File destFile = new File(filePath);
            if (!destFile.getParentFile().exists()) {
                destFile.getParentFile().mkdirs();
            }
            avatar.transferTo(destFile);

            result.put("code", 200);
            result.put("msg", "文件上传成功");
            result.put("data", Map.of(
                    "username", username,
                    "fileName", avatar.getOriginalFilename(),
                    "fileSize", avatar.getSize() + " bytes",
                    "filePath", filePath
            ));
        } catch (IOException e) {
            result.put("code", 500);
            result.put("msg", "文件上传失败：" + e.getMessage());
        }
        return result;
    }
}
```

### Postman 测试说明
1. 请求方法：POST
2. URL：`http://localhost:8080/api/form-data/upload`
3. Body 选择：`form-data`
4. 添加参数：
    - Key: `username`，Type: `Text`，Value: `zhangsan`
    - Key: `avatar`，Type: `File`，Value: 选择本地图片/文档文件
5. 发送请求，返回文件上传结果。

### 核心注解/类
- `MultipartFile`：接收文件参数，Spring 自动封装上传的文件；
- `transferTo()`：将上传的文件保存到指定路径；
- 注意：需确保项目有 `uploads` 目录，或修改为绝对路径（如 `D:/uploads/`）。

## 3. 接收 raw 格式（JSON 为主）
### 步骤1：创建实体类（接收 JSON 数据）
```java
import lombok.Data;

// 需引入 lombok 依赖，或手动写 getter/setter
@Data
public class UserLoginDTO {
    private String username;
    private String password;
    private Integer age; // 可选字段
}
```

### 步骤2：控制器代码
```java
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/api/raw")
public class RawJsonController {

    /**
     * 接收 raw → JSON 格式数据
     * 对应 Postman Body → raw → JSON
     */
    @PostMapping("/login")
    public Map<String, Object> login(@RequestBody UserLoginDTO loginDTO) {
        Map<String, Object> result = new HashMap<>();
        result.put("code", 200);
        result.put("msg", "JSON 数据接收成功");
        result.put("data", loginDTO);
        return result;
    }
}
```

### Postman 测试说明
1. 请求方法：POST
2. URL：`http://localhost:8080/api/raw/login`
3. Body 选择：`raw` → 右侧下拉选择 `JSON`
4. 输入 JSON 数据：
   ```json
   {
     "username": "admin",
     "password": "123456",
     "age": 20
   }
   ```
5. 发送请求，返回接收的 JSON 数据。

### 核心注解
- `@RequestBody`：将请求体的 JSON 数据自动绑定到实体类对象；
- 注意：需确保请求头 `Content-Type` 为 `application/json`（Postman 选择 JSON 后会自动设置）。

## 4. 接收 binary 格式（二进制文件）
### 控制器代码
```java
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestPart;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.multipart.MultipartFile;

import java.io.File;
import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/api/binary")
public class BinaryController {

    /**
     * 接收 binary 格式二进制文件
     * 对应 Postman Body → binary
     */
    @PostMapping("/upload-file")
    public Map<String, Object> uploadBinaryFile(@RequestPart MultipartFile file) {
        Map<String, Object> result = new HashMap<>();
        try {
            String filePath = "binary-uploads/" + System.currentTimeMillis() + "_" + file.getOriginalFilename();
            File destFile = new File(filePath);
            if (!destFile.getParentFile().exists()) {
                destFile.getParentFile().mkdirs();
            }
            file.transferTo(destFile);

            result.put("code", 200);
            result.put("msg", "二进制文件上传成功");
            result.put("data", Map.of(
                    "fileName", file.getOriginalFilename(),
                    "fileSize", file.getSize() + " bytes",
                    "contentType", file.getContentType(),
                    "filePath", filePath
            ));
        } catch (IOException e) {
            result.put("code", 500);
            result.put("msg", "文件上传失败：" + e.getMessage());
        }
        return result;
    }
}
```

### Postman 测试说明
1. 请求方法：POST
2. URL：`http://localhost:8080/api/binary/upload-file`
3. Body 选择：`binary` → 点击 `Select File` 选择本地二进制文件（如图片、视频）
4. 发送请求，返回文件上传结果。

### 核心说明
- `binary` 格式本质是单文件的二进制流，Spring 仍用 `MultipartFile` 接收；
- `@RequestPart`：等同于 `@RequestParam`，更语义化表示“请求部分”，适合文件接收。

---

### 总结
1. **x-www-form-urlencoded**：用 `@RequestParam` 接收纯文本表单，无文件场景首选；
2. **form-data**：用 `@RequestParam + MultipartFile` 接收文本+文件混合数据；
3. **raw（JSON）**：用 `@RequestBody + 实体类` 接收结构化 JSON 数据（REST API 主流）；
4. **binary**：用 `@RequestPart + MultipartFile` 接收单文件二进制流；
5. 核心注解：`@RequestParam`（表单/查询参数）、`@RequestBody`（JSON 体）、`MultipartFile`（文件）。

如果需要测试 GraphQL 格式的接收示例，或者想了解如何统一处理请求参数校验，我可以继续补充。





