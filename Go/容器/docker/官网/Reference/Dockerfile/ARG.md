
你希望我详细解释 Dockerfile 中的 `WORKDIR` 和 `ARG` 两个核心指令，包括它们的语法、核心作用、使用规则、相互配合方式，以及和 `ENV` 的关键区别，我会把这两个指令拆分成独立模块讲解，再补充它们的协作场景和避坑要点，帮你彻底掌握。

### 一、`WORKDIR` 指令：设置工作目录
#### 1. 核心定义
`WORKDIR` 用于**为 Dockerfile 后续的 `RUN`/`CMD`/`ENTRYPOINT`/`COPY`/`ADD` 指令设置「当前工作目录」**，相当于在容器内执行 `cd` 命令，但更规范、更可控。
- 若指定的目录不存在，Docker 会自动创建（即使后续指令不用这个目录）；
- 所有后续指令的相对路径都会基于 `WORKDIR` 解析，避免硬编码绝对路径。

#### 2. 基本语法
```dockerfile
WORKDIR /path/to/workdir
```
- `/path/to/workdir`：容器内的绝对路径（推荐），也支持相对路径（基于上一次 `WORKDIR`）。

#### 3. 核心特性与规则
##### （1）多次使用的叠加效果
`WORKDIR` 可在 Dockerfile 中多次使用，若传入相对路径，会基于「上一次 `WORKDIR` 的路径」拼接：
```dockerfile
WORKDIR /a       # 当前工作目录：/a
WORKDIR b        # 相对路径，拼接后：/a/b
WORKDIR c        # 相对路径，拼接后：/a/b/c
RUN pwd          # 输出：/a/b/c
```

##### （2）支持解析 `ENV` 定义的环境变量
`WORKDIR` 可引用之前通过 `ENV` 设置的变量（仅支持 Dockerfile 中显式定义的 `ENV`，不支持 `ARG`）：
```dockerfile
ENV APP_ROOT=/app
WORKDIR $APP_ROOT/src  # 最终工作目录：/app/src
RUN pwd                # 输出：/app/src
```
⚠️ 注意：若引用未定义的变量（如 `WORKDIR $UNDEFINED`），会直接将变量名作为路径（如 `/$UNDEFINED`）。

##### （3）默认工作目录与最佳实践
- 若未显式设置 `WORKDIR`，默认工作目录是 `/`；
- 基础镜像（如 `ubuntu`/`alpine`）可能已设置 `WORKDIR`（如 `root` 家目录 `/root`），为避免未知目录的意外操作，**建议显式设置 `WORKDIR`**；
- 推荐路径：用语义化目录（如 `/app`/`/usr/src/app`），而非零散路径。

#### 4. 典型使用场景
```dockerfile
FROM node:20-alpine
# 设置工作目录（自动创建）
WORKDIR /app
# 相对路径：复制 package.json 到 /app/
COPY package*.json ./
# 相对路径：安装依赖（在 /app 下执行）
RUN npm install
# 相对路径：复制源码到 /app/
COPY . ./
# 运行命令（在 /app 下执行）
CMD ["node", "index.js"]
```
- 所有 `COPY`/`RUN`/`CMD` 的相对路径都基于 `/app`，无需写 `/app/package.json` 这类绝对路径，代码更简洁。

### 二、`ARG` 指令：构建阶段的参数
#### 1. 核心定义
`ARG` 用于**定义「构建阶段专用的参数」**，用户可在 `docker build` 时通过 `--build-arg` 传入值，用于动态定制镜像构建过程。
- 仅在「镜像构建阶段」生效，**不会持久化到最终镜像/容器中**（对比 `ENV` 是核心区别）；
- 可用于 `FROM`/`ENV`/`WORKDIR`/`RUN` 等指令，实现构建过程的动态配置。

#### 2. 基本语法
```dockerfile
# 基础形式：定义参数（无默认值）
ARG <name>
# 带默认值：未传入时使用默认值
ARG <name>=<default value>
# 批量定义（Dockerfile 1.4+ 支持）
ARG name1=value1 name2=value2
```

#### 3. 核心特性与规则
##### （1）作用域：从声明行开始生效
`ARG` 变量仅在「声明行之后的指令」中可用，声明前引用会返回空字符串（或使用默认值）：
```dockerfile
FROM busybox
# 引用未声明的 username → 空字符串，使用 fallback 值 some_user
USER ${username:-some_user}
ARG username          # 声明参数，作用域开始
USER $username        # 引用已声明的 username → 传入的值（如 what_user）
```
构建命令：
```bash
docker build --build-arg username=what_user .
```
- 第 2 行 `USER`：生效值为 `some_user`；
- 第 4 行 `USER`：生效值为 `what_user`。

##### （2）多阶段构建的继承规则
- 同一阶段内声明的 `ARG` 仅在该阶段生效；
- 子阶段会继承「父阶段声明的 `ARG`」，无关阶段无法访问；
- 若需在多个阶段使用同一 `ARG`，需在每个阶段重新声明（或基于共享基础阶段）。

##### （3）与 `ENV` 的优先级：`ENV` 覆盖 `ARG`
若 `ARG` 和 `ENV` 定义同名变量，`ENV` 的值会覆盖 `ARG`（即使 `ARG` 是构建时传入的）：
```dockerfile
FROM ubuntu
ARG VERSION=v1.0
ENV VERSION=v2.0  # 覆盖 ARG 的值
RUN echo $VERSION  # 输出：v2.0（而非构建时传入的 v1.0）
```
**实用技巧**：用 `ENV` 引用 `ARG`，实现「构建参数持久化到镜像」：
```dockerfile
FROM ubuntu
ARG VERSION=v1.0
# ENV 引用 ARG，未传入时用默认值 v1.0
ENV APP_VERSION=${VERSION:-v1.0}
RUN echo $APP_VERSION  # 构建时传入 --build-arg VERSION=v2.0 → 输出 v2.0
```
- `ARG VERSION` 仅在构建阶段生效；
- `ENV APP_VERSION` 会持久化到镜像中，容器运行时可访问。

##### （4）预定义 `ARG`（无需声明即可使用）
Docker 内置了一批预定义 `ARG`，无需在 Dockerfile 中声明即可通过 `--build-arg` 传入：

| 预定义 ARG       | 用途                  |
|------------------|-----------------------|
| `HTTP_PROXY`/`http_proxy` | 构建时 HTTP 代理      |
| `HTTPS_PROXY`/`https_proxy` | 构建时 HTTPS 代理    |
| `NO_PROXY`/`no_proxy`     | 构建时不代理的地址    |

示例：构建时通过代理下载依赖
```bash
docker build --build-arg HTTPS_PROXY=https://proxy.example.com .
```
⚠️ 注意：预定义 `ARG` 默认不会出现在 `docker history` 中（避免泄露代理密码），若显式声明 `ARG HTTPS_PROXY`，则会被记录到构建历史。

##### （5）BuildKit 专属的平台相关 ARG
使用 BuildKit 构建时（Docker 20.10+ 默认），会自动生成「平台相关的预定义 ARG」，用于多平台镜像构建：

| ARG 变量         | 说明                                  | 示例值          |
|------------------|---------------------------------------|-----------------|
| `TARGETPLATFORM` | 目标镜像的平台                        | `linux/amd64`   |
| `TARGETOS`       | 目标镜像的操作系统                    | `linux`/`windows` |
| `TARGETARCH`     | 目标镜像的架构                        | `amd64`/`arm64` |
| `BUILDPLATFORM`  | 构建节点的平台                        | `linux/arm64`   |

使用示例（根据架构选择不同依赖）：
```dockerfile
FROM alpine
# 显式声明以启用（全局 ARG 需重新声明才能在阶段内使用）
ARG TARGETARCH
# 不同架构安装不同版本的二进制包
RUN if [ "$TARGETARCH" = "amd64" ]; then \
      wget https://example.com/app-amd64; \
    else \
      wget https://example.com/app-arm64; \
    fi
```

#### 4. 关键注意事项
##### （1）禁止传递敏感信息
`ARG` 的值会出现在 `docker history`、构建日志中，**绝对不能用 `ARG` 传递密码、API 密钥等敏感信息**；
- 替代方案：使用 `RUN --mount=type=secret` 挂载密钥文件。

##### （2）对构建缓存的影响
`ARG` 变量会影响构建缓存：
- 若 `ARG` 的值变化，**首次使用该变量的指令及后续指令会触发缓存失效**；
- 预定义 `ARG`（如 `HTTP_PROXY`）默认不影响缓存，显式声明后才会影响。

示例：
```dockerfile
FROM ubuntu
ARG VERSION  # 声明不触发缓存失效
RUN echo $VERSION  # 若 VERSION 值变化，该 RUN 触发缓存失效
```

#### 5. 典型使用场景
##### （1）动态指定基础镜像版本
```dockerfile
# 定义构建参数，默认值为 20-alpine
ARG NODE_VERSION=20-alpine
# 引用 ARG 选择基础镜像
FROM node:$NODE_VERSION
WORKDIR /app
COPY . ./
CMD ["node", "index.js"]
```
构建时指定版本：
```bash
# 使用 node 18-alpine 构建
docker build --build-arg NODE_VERSION=18-alpine .
```

##### （2）动态设置构建环境
```dockerfile
FROM golang:1.22
ARG BUILD_ENV=production
WORKDIR /app
COPY . ./
# 根据构建参数设置编译标志
RUN if [ "$BUILD_ENV" = "dev" ]; then \
      go build -o app -race; \
    else \
      go build -o app -ldflags="-s -w"; \
    fi
CMD ["/app/app"]
```
构建开发版镜像：
```bash
docker build --build-arg BUILD_ENV=dev .
```

### 三、`WORKDIR` + `ARG` 协作示例
结合两个指令实现「动态工作目录」：
```dockerfile
# 定义构建参数，默认工作目录为 /app
ARG APP_DIR=/app
# 设置工作目录（引用 ARG）
WORKDIR $APP_DIR
# 复制文件到工作目录（相对路径）
COPY package.json ./
RUN npm install
# 构建时可自定义工作目录
# docker build --build-arg APP_DIR=/usr/src/app .
```

### 四、`ARG` vs `ENV` 核心区别（必记）
| 特性                | `ARG`                                  | `ENV`                              |
|---------------------|----------------------------------------|------------------------------------|
| 生效阶段            | 仅构建阶段（镜像构建完成后失效）       | 构建阶段 + 运行阶段（持久化到镜像） |
| 容器运行时可见      | ❌ 否                                  | ✅ 是（可通过 `env` 查看）          |
| 构建时传入方式      | `docker build --build-arg`             | 不支持（只能在 Dockerfile 中定义） |
| 优先级              | 低于 `ENV`（同名时被覆盖）             | 高于 `ARG`                         |
| 缓存影响            | 首次使用时触发缓存失效                 | 定义时触发缓存失效                 |
| 典型用途            | 动态定制构建过程（版本、环境）         | 设置容器运行时的环境变量           |

### 总结
#### 1. `WORKDIR` 关键点
- 为后续指令设置工作目录，相对路径基于它解析，不存在则自动创建；
- 多次使用时相对路径会叠加，支持解析 `ENV` 变量；
- 最佳实践：显式设置语义化路径（如 `/app`），避免默认目录的意外操作。

#### 2. `ARG` 关键点
- 构建阶段专用参数，不持久化到镜像，通过 `--build-arg` 传入；
- 作用域从声明行开始，`ENV` 会覆盖同名 `ARG`；
- 禁止传递敏感信息，预定义 `ARG` 可用于代理配置，BuildKit 专属 ARG 支持多平台构建。

#### 3. 协作要点
- `WORKDIR` 可引用 `ENV`，但不能直接引用 `ARG`（需通过 `ENV` 中转）；
- 两者结合可实现「动态构建路径」，提升镜像的灵活性。





