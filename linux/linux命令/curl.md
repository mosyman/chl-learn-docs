你希望我详细讲解 Linux 中 `curl` 命令的**全面用法**，包括基础、进阶、实战场景和冷门但实用的功能，我会从易到难、分类清晰地帮你梳理，让你能覆盖日常开发/运维中所有用到 `curl` 的场景。

`curl` 全称 "Client URL"，是一个跨平台的命令行工具，核心能力是**在客户端与服务器之间传输数据**，支持 HTTP/HTTPS/FTP/SCP 等几十种协议，是 Linux 下测试网络、调试接口、传输文件的「瑞士军刀」。

---

## 一、基础语法与核心参数
### 1. 核心语法
```bash
curl [选项] [URL]
```
- 无选项：默认以 `GET` 方式请求 URL，将响应内容输出到终端
- 选项：控制请求行为（如请求方法、头信息、输出方式等）
- URL：支持多个 URL（空格分隔），按顺序执行

### 2. 必记基础参数（高频）
| 参数 | 作用 | 示例 |
|------|------|------|
| `-o <文件名>` | 将响应内容保存到指定文件（自定义文件名） | `curl -o baidu.html https://www.baidu.com` |
| `-O` | 保存响应内容，文件名与 URL 末尾的文件名一致 | `curl -O https://example.com/test.zip` |
| `-i` | 输出包含**响应头 + 响应体** | `curl -i https://www.baidu.com` |
| `-I`/`--head` | 只请求响应头（等价于 `HEAD` 方法） | `curl -I https://www.baidu.com` |
| `-v`/`--verbose` | 显示详细的请求/响应过程（调试神器） | `curl -v https://www.baidu.com` |
| `-s`/`--silent` | 静默模式（不输出进度、错误等冗余信息） | `curl -s https://www.baidu.com` |
| `-S` | 配合 `-s` 使用，只输出错误信息（静默但不隐藏错误） | `curl -sS https://www.baidu.com` |
| `-L`/`--location` | 跟随 HTTP 重定向（3xx 状态码） | `curl -L https://shorturl.example.com` |

## 二、HTTP 请求方法控制（接口调试核心）
`curl` 默认用 `GET`，可通过参数指定任意 HTTP 方法（GET/POST/PUT/DELETE/PATCH 等）。

### 1. GET 请求（默认）
- 基础：`curl https://api.example.com/user?id=123`
- 带查询参数（空格需转义或加引号）：
  ```bash
  # 方式1：直接拼在 URL 后（空格转义为 %20）
  curl https://api.example.com/search?keyword=linux%20curl
  # 方式2：用引号包裹（更易读）
  curl "https://api.example.com/search?keyword=linux curl"
  ```

### 2. POST 请求（最常用）
#### （1）表单格式（application/x-www-form-urlencoded）
```bash
# -X POST：指定请求方法；-d：传递表单数据（多个参数用 & 分隔）
curl -X POST -d "username=admin&password=123456" https://api.example.com/login

# 数据也可从文件读取（适合大量参数）
curl -X POST -d @data.txt https://api.example.com/login
# data.txt 内容：username=admin&password=123456
```

#### （2）JSON 格式（application/json）
```bash
curl -X POST \
  -H "Content-Type: application/json" \  # 必须指定 Content-Type
  -d '{"username":"admin","password":"123456"}' \  # JSON 字符串（单引号包裹）
  https://api.example.com/login

# 从文件读取 JSON 数据（推荐，避免转义麻烦）
curl -X POST -H "Content-Type: application/json" -d @data.json https://api.example.com/login
# data.json 内容：{"username":"admin","password":"123456"}
```

#### （3）文件上传（multipart/form-data）
```bash
# -F：模拟表单上传文件（form-data 格式）
curl -X POST -F "file=@/home/test.txt" -F "desc=测试文件" https://api.example.com/upload
# @ 表示「读取本地文件」，/home/test.txt 是本地文件路径
```

### 3. PUT/DELETE/PATCH 请求
```bash
# PUT（更新资源）
curl -X PUT -H "Content-Type: application/json" -d '{"name":"test"}' https://api.example.com/user/123

# DELETE（删除资源）
curl -X DELETE https://api.example.com/user/123

# PATCH（部分更新）
curl -X PATCH -H "Content-Type: application/json" -d '{"age":20}' https://api.example.com/user/123
```

## 三、请求头/认证/Cookie 控制（进阶）
### 1. 自定义请求头（-H/--header）
```bash
# 单个头信息
curl -H "Authorization: Bearer token123" https://api.example.com/user

# 多个头信息（多个 -H）
curl -H "Content-Type: application/json" -H "Token: abc123" -H "User-Agent: MyCurl/1.0" https://api.example.com

# 移除默认头信息（头名前加 -）
curl -H "-User-Agent" https://api.example.com  # 不发送 User-Agent 头
```

### 2. Cookie 操作
#### （1）发送 Cookie
```bash
# 方式1：通过请求头
curl -H "Cookie: sessionid=abc123; uid=1001" https://api.example.com

# 方式2：专用参数 --cookie
curl --cookie "sessionid=abc123; uid=1001" https://api.example.com

# 从文件读取 Cookie（格式同浏览器 Cookie 文件）
curl --cookie cookies.txt https://api.example.com
```

#### （2）保存响应中的 Cookie 到文件
```bash
# -c/--cookie-jar：将服务器返回的 Cookie 保存到文件
curl -c cookies.txt https://api.example.com/login
```

### 3. 身份认证
#### （1）Basic 认证（基础用户名密码）
```bash
# -u <用户名:密码>：Basic 认证
curl -u admin:123456 https://api.example.com/admin
```

#### （2）Digest 认证
```bash
curl --digest -u admin:123456 https://api.example.com/admin
```

#### （3）Bearer Token（JWT 常用）
```bash
curl -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." https://api.example.com/user
```

## 四、文件传输与网络控制（下载/限速/断点续传）
### 1. 断点续传（大文件下载必备）
```bash
# -C -：自动识别断点位置，继续下载（- 表示「从上次中断的位置开始」）
curl -C - -O https://example.com/bigfile.iso
```

### 2. 限速下载（避免占满带宽）
```bash
# --limit-rate <速率>：限制下载速度（单位：K/M/G）
curl --limit-rate 2M -O https://example.com/bigfile.iso  # 限速 2MB/s
```

### 3. 分段下载（并行加速）
```bash
# 下载文件的 0-1000 字节
curl -r 0-1000 -o part1.iso https://example.com/bigfile.iso
# 下载 1001-2000 字节
curl -r 1001-2000 -o part2.iso https://example.com/bigfile.iso
# 合并分段（Linux 下用 cat）
cat part1.iso part2.iso > bigfile.iso
```

### 4. 上传文件（FTP/SCP 示例）
```bash
# FTP 上传
curl -T localfile.txt ftp://user:pass@ftp.example.com/remote/

# SCP 上传（需服务器开启 SSH）
curl -T localfile.txt scp://user@example.com/remote/
```

### 5. 批量下载（通配符/循环）
```bash
# 通配符匹配多个 URL
curl -O https://example.com/file[1-5].txt  # 下载 file1.txt 到 file5.txt

# 循环下载（结合 shell 循环）
for i in {1..5}; do curl -O https://example.com/file$i.txt; done
```

## 五、冷门但实用的功能
### 1. 测试接口响应时间
```bash
# -w：自定义输出格式（%{time_total} 表示总响应时间，单位秒）
curl -o /dev/null -s -w "总响应时间：%{time_total}s\n" https://www.baidu.com
```
常用的 `-w` 变量：
- `%{http_code}`：HTTP 状态码（如 200、404）
- `%{time_connect}`：建立连接的时间
- `%{time_starttransfer}`：第一个字节传输的时间
- `%{size_download}`：下载的字节数

### 2. 忽略 SSL 证书错误（测试内网 HTTPS 接口）
```bash
# -k/--insecure：跳过 SSL 证书验证
curl -k https://localhost:8080/api
```

### 3. 设置超时时间（避免无限等待）
```bash
# --connect-timeout：连接超时（秒）；--max-time：总超时（秒）
curl --connect-timeout 5 --max-time 10 https://api.example.com
```

### 4. 模拟不同请求方法/版本
```bash
# 模拟 HTTP/2 请求
curl --http2 https://example.com

# 模拟 HTTP/1.0 请求
curl --http1.0 https://example.com

# 发送 OPTIONS 请求（查看接口支持的方法）
curl -X OPTIONS -i https://api.example.com/user
```

### 5. 代理访问
```bash
# HTTP 代理
curl -x http://proxy.example.com:8080 https://www.baidu.com

# SOCKS5 代理
curl --socks5 socks5://proxy.example.com:1080 https://www.baidu.com

# 带认证的代理
curl -x http://user:pass@proxy.example.com:8080 https://www.baidu.com
```

## 六、实战场景模板（直接复制用）
### 1. 调试接口（显示完整请求/响应）
```bash
curl -v -H "Content-Type: application/json" \
  -d '{"id":123,"name":"test"}' \
  https://api.example.com/user
```

### 2. 测试接口可用性（只看状态码）
```bash
curl -o /dev/null -s -w "%{http_code}\n" https://api.example.com/user
```

### 3. 下载大文件（断点续传+限速+静默）
```bash
curl -C - --limit-rate 5M -O -sS https://example.com/bigfile.iso
```

### 4. 上传文件到接口（form-data 格式）
```bash
curl -X POST -F "file=@/home/test.jpg" -F "type=image" \
  -H "Token: abc123" \
  https://api.example.com/upload
```

### 5. 保存 Cookie 并复用（模拟登录）
```bash
# 第一步：登录，保存 Cookie 到 cookie.txt
curl -c cookie.txt -X POST -d "username=admin&password=123456" https://api.example.com/login

# 第二步：用保存的 Cookie 访问需要登录的接口
curl -b cookie.txt https://api.example.com/user/info
```

---

### 总结
1. `curl` 核心是「传输数据」，基础用法记住 `-o/-O`（保存）、`-i/-I/-v`（调试）、`-L`（重定向）即可覆盖 80% 场景；
2. 接口调试重点掌握 `POST` 的两种格式（表单/JSON），配合 `-H` 自定义头信息；
3. 高级场景记住断点续传（`-C -`）、超时控制（`--max-time`）、Cookie 操作（`-c/-b`）、代理（`-x`），能解决复杂的网络/文件传输需求。

如果有具体的使用场景（比如调试某个特定协议的接口、解决下载失败问题），可以告诉我，我会针对性给出命令模板。