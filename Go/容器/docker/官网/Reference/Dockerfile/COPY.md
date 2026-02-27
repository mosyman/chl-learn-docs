

### 一、`COPY` 指令的核心定义
`COPY` 是 Dockerfile 中**最基础、最推荐的文件复制指令**，核心作用是将文件/目录从「构建上下文」「其他构建阶段」或「外部镜像」复制到当前镜像的文件系统中。
- 功能聚焦：仅做「纯文件复制」，无自动解压、远程URL下载等额外功能（对比 `ADD` 更简洁、可预测）；
- 设计目标：Docker 官方明确推荐优先使用 `COPY`（除非需要 `ADD` 的解压/远程资源功能），因为其行为更简单、易维护。

### 二、`COPY` 的基本语法
`COPY` 有两种语法形式（后者用于路径含空格的场景）：
```dockerfile
# 基础形式（推荐，无空格路径）
COPY [OPTIONS] <src> ... <dest>
# JSON数组形式（路径含空格时必须用）
COPY [OPTIONS] ["<src>", ... "<dest>"]
```
- `<src>`：源路径（可多个，支持通配符），仅能是「构建上下文内的文件/目录」「其他构建阶段的路径」或「外部镜像的路径」；
- `<dest>`：镜像内的目标路径（绝对/相对路径）；
- `[OPTIONS]`：可选扩展参数（如 `--from`/`--chmod`/`--exclude` 等）。

#### 核心语法规则
1. 若指定多个 `<src>`（或通配符匹配多个文件），`<dest>` 必须是目录（以 `/` 结尾），否则会报错；
2. 目标路径 `<dest>` 不存在时，Docker 会自动创建（包括所有父目录）；
3. 路径解析关键规则：
    - 绝对路径：以 `/` 开头，直接映射到镜像根目录（如 `/usr/src/app`）；
    - 相对路径：基于 `WORKDIR` 指令的当前工作目录；
    - 末尾 `/` 决定最终形态：
        - `COPY test.txt /abs` → 生成 `/abs` 文件（覆盖同名目录）；
        - `COPY test.txt /abs/` → 生成 `/abs/test.txt` 文件（放入目录）。

### 三、`COPY` 处理不同源的核心逻辑
#### 1. 源：构建上下文内的文件/目录（最常用）
`<src>` 为构建上下文（本地）内的文件/目录，是 `COPY` 最基础的使用场景。

##### （1）本地文件的处理
- 复制文件及其元数据（权限、时间戳）到目标路径；
- 若目标是目录（`/` 结尾），文件会被放到该目录下；
- 若目标是文件（无 `/` 结尾），会覆盖同名目标文件；
- 若目标是目录但同名文件已存在，直接报错（避免意外覆盖）。

**示例**：
```dockerfile
# 构建上下文有 ./config.yml，复制到镜像 /etc/app/ 目录
COPY config.yml /etc/app/
# 最终镜像内路径：/etc/app/config.yml
```

##### （2）本地目录的处理
- 仅复制目录**内容**（不含目录本身），子目录会递归复制；
- 若目标目录已有文件，冲突时以源文件为准（文件级覆盖）；
- 若试图将目录复制到同名文件上，直接报错。

**示例**：
```dockerfile
# 构建上下文有 ./src/ 目录（含 a.txt/b.txt）
COPY src/ /app/
# 镜像内生成 /app/a.txt、/app/b.txt（无 /app/src 目录）
```

##### （3）通配符匹配（Go的filepath.Match规则）
支持 `*`（任意字符）、`?`（单个字符）、`[]`（字符集），特殊字符需转义：
```dockerfile
# 复制所有 .png 文件到 /images/
COPY *.png /images/
# 复制 index.js/index.ts（? 匹配单个字符）
COPY index.?s /dest/
# 复制 arr[0].txt（[] 是特殊字符，需转义为 [[]]）
COPY arr[[]0].txt /dest/
```

#### 2. 源：其他构建阶段/外部镜像（`--from` 核心参数）
`COPY --from` 是多阶段构建的核心能力，允许从「命名构建阶段」「外部镜像」复制文件，突破构建上下文的限制。

##### （1）从命名构建阶段复制（多阶段构建）
```dockerfile
# 阶段1：构建可执行文件
FROM golang:1.22 AS build
WORKDIR /app
COPY . .
RUN go build -o /myapp ./main.go

# 阶段2：仅复制构建产物，减小镜像体积
FROM alpine:3.20
# 从 build 阶段复制 /myapp 到当前镜像的 /usr/bin/
COPY --from=build /myapp /usr/bin/
CMD ["/usr/bin/myapp"]
```

##### （2）从外部镜像复制
```dockerfile
# 从官方 nginx 镜像复制配置文件到当前镜像
COPY --from=nginx:latest /etc/nginx/nginx.conf /my-nginx.conf
```

##### 关键规则
- `--from` 的源路径始终从「目标阶段/镜像的根目录」开始解析；
- 可简写为数字（如 `--from=0` 表示第一个 `FROM` 阶段），但推荐用命名阶段（更易读）。

### 四、`COPY` 的所有关键可选参数
`COPY` 的扩展参数是其核心能力的补充，以下是每个参数的详细说明和示例：

| 参数                | 最低Dockerfile版本 | 核心作用                                                                 | 示例                                  |
|---------------------|--------------------|--------------------------------------------------------------------------|---------------------------------------|
| `--from`            | -                  | 指定源为构建阶段/外部镜像/命名上下文                                     | `COPY --from=build /myapp /usr/bin/`  |
| `--chmod`           | 1.2                | 设置目标文件/目录的权限（替代 `RUN chmod`，减少镜像层）                   | `COPY --chmod=755 app.sh /app/`       |
|                     | 1.14               | 支持符号权限（如 `+x`/`u=rwX`），`X` 表示仅目录/已有可执行权限的文件加执行权 | `COPY --chmod=u=rwX,go=rX . /app/`    |
| `--chown`           | -                  | 设置目标文件的属主（用户:组），默认 UID/GID 为 0（root）                  | `COPY --chown=1000:1000 data /app/`   |
| `--link`            | 1.4                | 独立复制层，提升缓存复用率（修改前序层不失效），不跟随目标路径的软链接     | `COPY --link src/ /app/`              |
| `--parents`         | 1.20               | 保留源文件的父目录结构（类似 `cp --parents`）                            | `COPY --parents x/a.txt /dest/` → `/dest/x/a.txt` |
| `--exclude`         | 1.19               | 排除匹配的文件/目录（支持多轮排除、通配符）                              | `COPY --exclude=*.log --exclude=node_modules . /app/` |

#### 重点参数深度解析
##### （1）`--chmod`：权限设置最佳实践
传统方式需要先复制再改权限（多一层 `RUN`），`--chmod` 可一步完成：
```dockerfile
# 传统方式（2层）
COPY app.sh /app/
RUN chmod 755 /app/app.sh

# 优化方式（1层）
COPY --chmod=755 app.sh /app/
```
符号权限示例（灵活控制文件/目录权限）：
```dockerfile
# 目录设为 755，文件设为 644，保留已有可执行文件的权限
COPY --chmod=u=rwX,go=rX . /app/
```

##### （2）`--parents`：保留目录结构
解决「复制多文件时保留原目录层级」的问题：
```dockerfile
# 构建上下文：x/a.txt、y/b.txt
# 不使用 --parents → /dest/a.txt、/dest/b.txt（覆盖同名文件）
COPY x/a.txt y/b.txt /dest/

# 使用 --parents → /dest/x/a.txt、/dest/y/b.txt（保留层级）
COPY --parents x/a.txt y/b.txt /dest/

# 仅保留部分层级（./ 后的目录）
COPY --parents x/./y/*.txt /dest/
# 构建上下文：x/y/c.txt → 镜像内：/dest/y/c.txt
```

##### （3）`--exclude`：排除不需要的文件
避免复制日志、依赖目录等无用文件，减小镜像体积：
```dockerfile
# 复制所有 .js 文件，排除 test.js 和 .log 文件
COPY --exclude=test.js --exclude=*.log *.js /app/

# 排除 node_modules 目录（前端项目常用）
COPY --exclude=node_modules . /app/
```

##### （4）`--link`：提升构建缓存效率
核心优势：复制的文件作为独立层，修改前序层不会让该层失效，尤其适合多阶段构建：
```dockerfile
# 推荐：使用 --link 提升缓存复用
COPY --link ./dist /app/dist
```
⚠️ 注意：`--link` 不跟随目标路径的软链接，目标路径最终仅为目录（无软链接）。

### 五、`COPY` vs `ADD`（核心区别与最佳实践）
| 特性                | `COPY`                                  | `ADD`                                  |
|---------------------|----------------------------------------|----------------------------------------|
| 核心功能            | 纯文件复制（本地/多阶段/外部镜像）      | 复制 + 本地压缩包解压 + 远程URL/Git下载 |
| 远程资源            | 不支持（仅 `--from` 从镜像复制）        | 支持URL/Git仓库                        |
| 压缩包处理          | 仅复制，不解压                          | 自动解压本地tar/gzip/bzip2/xz包        |
| 官方推荐            | 优先使用（行为可预测）                  | 仅在需要解压/远程资源时使用            |
| 适用场景            | 绝大多数文件复制场景                    | 解压本地压缩包、拉取远程资源            |

**最佳实践总结**：
- 90% 的场景用 `COPY`（如复制代码、配置文件）；
- 仅当需要「解压本地压缩包」或「拉取远程URL/Git仓库」时，才用 `ADD`；
- 避免用 `ADD` 下载远程文件（优先用 `RUN wget/curl`，可控制权限/清理缓存）。

### 六、实操示例（完整Dockerfile）
```dockerfile
# 声明使用BuildKit（支持所有COPY参数）
# syntax=docker/dockerfile:1
FROM node:20-alpine AS build
WORKDIR /app
# 复制package.json并安装依赖（利用缓存）
COPY package*.json ./
RUN npm install
# 复制源码（排除node_modules，设置权限）
COPY --exclude=node_modules --chmod=644 . ./
RUN npm run build

# 最终镜像（轻量）
FROM nginx:alpine
# 从build阶段复制构建产物，保留目录结构
COPY --from=build --parents /app/dist/ /usr/share/nginx/html/
# 复制自定义配置，设置属主
COPY --chown=nginx:nginx nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 总结
1. `COPY` 是 Dockerfile 中**最推荐的文件复制指令**，行为简单可预测，优先于 `ADD` 使用；
2. 核心语法规则：多源时目标必须是目录（/结尾），末尾/决定文件/目录形态，路径不能超出构建上下文；
3. 关键参数：`--from`（多阶段/外部镜像）、`--chmod`（权限）、`--exclude`（排除文件）、`--parents`（保留目录结构）是高频使用的扩展功能；
4. 与 `ADD` 的核心区别：`COPY` 仅做纯复制，`ADD` 额外支持解压本地压缩包、拉取远程资源；
5. 最佳实践：利用 `COPY` 的参数（如 `--chmod`/`--exclude`）减少镜像层，提升构建效率。


