
你想全面了解 `docker build` 命令的用法，包括它的核心功能、参数含义、使用场景和最佳实践，我会从基础到进阶，帮你系统梳理这个命令的所有关键知识点。

### 一、`docker build` 核心概念
`docker build` 是 Docker 中用于**从 Dockerfile 和上下文文件构建镜像**的核心命令。简单来说，它的作用是读取你编写的 Dockerfile 中的指令，一步步执行并最终生成一个可运行的 Docker 镜像。

#### 基本语法
```bash
docker build [OPTIONS] PATH | URL | -
```
- `PATH`：本地构建上下文的路径（最常用，比如 `.` 代表当前目录）。
- `URL`：远程仓库地址（比如 GitHub 仓库）。
- `-`：从标准输入读取 Dockerfile（无上下文）。

### 二、核心参数详解（按使用频率排序）
#### 1. 最常用基础参数
| 参数 | 缩写    | 作用 | 示例 |
|------|-------|------|------|
| `--tag` | `-t`  | 为镜像打标签（格式：`名称:版本`），可多次使用打多个标签 | `docker build -t myapp:v1 -t myapp:latest .` |
| `--file` | `-f`  | 指定非默认名称/路径的 Dockerfile（默认是上下文路径下的 `Dockerfile`） | `docker build -f ./docker/Dockerfile.prod .` |
| `--no-cache` | -     | 构建时不使用缓存（强制重新执行所有步骤） | `docker build --no-cache -t myapp:v1 .` |
| `--pull` | -     | 强制拉取最新的基础镜像（即使本地已有） | `docker build --pull -t myapp:v1 .` |

#### 2. 进阶参数（场景化）
##### （1）构建上下文相关
- `--build-context`：指定额外的构建上下文（多上下文构建，Docker 23.0+ 支持）
  ```bash
  # 示例：同时使用本地上下文和远程 GitHub 上下文
  docker build --build-context app=./src --build-context config=https://github.com/xxx/config.git#main .
  ```
- `.dockerignore` 文件：**不是参数，但和上下文强相关**，作用类似 `.gitignore`，指定构建时需要排除的文件/目录（比如 `node_modules`、`.git`），减少上下文体积，加速构建。

##### （2）构建参数（动态传值）
- `--build-arg`：传递构建时参数（对应 Dockerfile 中的 `ARG` 指令）
  ```bash
  # Dockerfile 中定义 ARG APP_VERSION=1.0
  docker build --build-arg APP_VERSION=2.0 -t myapp:v2 .
  ```

##### （3）输出与日志相关
- `--progress`：指定构建进度展示方式（`auto`/`plain`/`tty`），`plain` 适合CI/CD环境
  ```bash
  docker build --progress plain -t myapp:v1 .
  ```
- `--quiet` | `-q`：静默模式，只输出最终镜像ID
  ```bash
  docker build -q -t myapp:v1 .
  ```

##### （4）平台适配（跨架构构建）
- `--platform`：指定构建镜像的目标平台（需开启buildx）
  ```bash
  # 构建arm64架构的镜像
  docker build --platform linux/arm64 -t myapp:v1 .
  ```

##### （5）构建缓存与输出
- `--cache-from`：指定缓存源（加速多机器/CI构建）
  ```bash
  docker build --cache-from myapp:latest -t myapp:v1 .
  ```
- `--output` | `-o`：指定构建结果输出方式（比如导出到本地）
  ```bash
  # 将构建结果导出为tar包
  docker build -o type=tar,dest=myapp.tar .
  ```

### 三、完整使用示例
#### 示例1：基础构建（默认Dockerfile）
```bash
# 进入项目目录
cd /path/to/your/project
# 构建镜像并打标签
docker build -t my-web-app:1.0 .
```

#### 示例2：指定自定义Dockerfile+构建参数
```bash
# Dockerfile路径：./docker/Dockerfile.dev
# 传递构建参数：NODE_ENV=development
docker build \
  -f ./docker/Dockerfile.dev \
  --build-arg NODE_ENV=development \
  -t my-node-app:dev \
  --no-cache \  # 不使用缓存
  .
```

#### 示例3：跨平台构建（需要buildx）
```bash
# 初始化buildx（首次使用）
docker buildx create --use
# 构建并推送多架构镜像到仓库
docker build \
  --platform linux/amd64,linux/arm64 \
  -t username/myapp:latest \
  --push \  # 推送到镜像仓库
  .
```

### 四、常见问题与最佳实践
#### 1. 构建上下文过大？
- 编写 `.dockerignore` 文件，排除无关文件（`node_modules`、`.git`、`logs` 等）
- 尽量将 `COPY/ADD` 指令放在 Dockerfile 后面（利用缓存）
- 使用多阶段构建（`multi-stage build`）减少最终镜像体积

#### 2. 缓存失效？
- `--no-cache` 强制刷新所有缓存
- 单个步骤的指令/文件变化会导致该步骤及后续步骤缓存失效
- 避免在 `COPY . .` 后执行 `RUN apt-get update` 这类频繁变化的指令

#### 3. 多阶段构建示例（优化镜像体积）
```dockerfile
# 阶段1：构建阶段（包含编译工具）
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# 阶段2：运行阶段（仅保留运行依赖）
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
构建命令不变：`docker build -t my-nginx-app:v1 .`

### 总结
1. `docker build` 核心是**读取Dockerfile + 上下文文件**构建镜像，基础语法为 `docker build [OPTIONS] 上下文路径`。
2. 高频参数：`-t`（打标签）、`-f`（指定Dockerfile）、`--build-arg`（传参）、`--no-cache`（禁用缓存）。
3. 最佳实践：用 `.dockerignore` 减小上下文、多阶段构建优化镜像体积、合理利用缓存加速构建。

### 关键点回顾
- 构建上下文：`docker build` 会将指定路径下的所有文件发送给Docker守护进程，`.dockerignore` 可减少上下文体积。
- 缓存机制：Docker会缓存每一步构建结果，指令/文件不变则复用缓存，`--no-cache` 可强制刷新。
- 多阶段构建：分离构建和运行环境，大幅减小最终镜像的体积，是生产环境的标配。

