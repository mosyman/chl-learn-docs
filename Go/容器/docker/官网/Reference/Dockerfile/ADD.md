

### 一、`ADD` 指令的核心定义
`ADD` 是 Dockerfile 中用于<span style="color: #ff6600; font-size: 16px; font-weight: bold;">将文件/目录/远程资源复制到镜像文件系统</span> 的指令，功能比 `COPY` 更丰富：
- 支持从**构建上下文**（本地）、**远程URL**、**Git仓库** 复制资源；
- 自动解压本地压缩包（tar/gzip/bzip2/xz 格式）；
- 支持通配符匹配、权限设置、路径解析等高级特性。

简单来说，`ADD` 是「增强版的文件复制工具」，覆盖了本地/远程资源的复制+解压场景。

### 二、`ADD` 的基本语法
`ADD` 有两种语法形式（后者用于路径含空格的场景）：
```dockerfile
# 基础形式（推荐，无空格路径）
ADD [OPTIONS] <src> ... <dest>
# JSON数组形式（路径含空格时必须用）
ADD [OPTIONS] ["<src>", ... "<dest>"]
```
- `<src>`：源路径/资源地址（可多个，支持通配符）；
- `<dest>`：镜像内的目标路径（绝对/相对路径）；
- `[OPTIONS]`：可选参数（如 `--chmod`/`--unpack`/`--exclude` 等）。

#### 核心语法规则
1. 若指定多个 `<src>`（或通配符匹配多个文件），`<dest>` 必须是目录（以 `/` 结尾）；
2. 目标路径 `<dest>` 不存在时，Docker 会自动创建（包括所有父目录）；
3. 路径解析：
    - 绝对路径：以 `/` 开头，直接映射到镜像根目录（如 `/usr/src/app`）；
    - 相对路径：基于 `WORKDIR` 指令的当前工作目录；
    - 末尾 `/` 有特殊意义：`ADD test.txt /abs/` 会创建 `/abs/test.txt`，而 `ADD test.txt /abs` 会创建 `/abs` 文件（覆盖同名目录）。

### 三、`ADD` 处理不同源的核心逻辑
`ADD` 会根据 `<src>` 的类型（本地/远程URL/Git仓库）做不同处理，这是它最核心的特性：

#### 1. 源：本地文件/目录（构建上下文内）
这是最常用的场景，`<src>` 为构建上下文内的文件/目录（不能超出上下文范围，`../` 会被自动忽略）。

##### （1）本地文件的处理
- 复制文件及其元数据（权限、时间戳）到目标路径；
- 若目标路径是目录（以 `/` 结尾），文件会被放到该目录下；
- 若目标路径是文件（无 `/` 结尾），会覆盖目标文件（若存在）；
- 若目标是目录但同名文件已存在，直接报错。

**示例**：
```dockerfile
# 构建上下文有 ./app/config.yml，复制到镜像 /etc/app/ 目录
ADD app/config.yml /etc/app/
# 等价于：镜像内生成 /etc/app/config.yml
```

##### （2）本地目录的处理
- 仅复制目录**内容**（不含目录本身），子目录会递归复制；
- 若目标目录已有文件，冲突时以源文件为准（文件级覆盖）；
- 若试图将目录复制到同名文件上，直接报错。

**示例**：
```dockerfile
# 构建上下文有 ./src/ 目录（含 a.txt/b.txt），复制到镜像 /app/
ADD src/ /app/
# 镜像内生成 /app/a.txt、/app/b.txt（无 /app/src 目录）
```

##### （3）通配符匹配（Go的filepath.Match规则）
支持 `*`（任意字符）、`?`（单个字符）、`[]`（字符集）等通配符，需注意特殊字符转义：
```dockerfile
# 复制所有 .png 文件到 /images/
ADD *.png /images/
# 复制 index.js/index.ts（? 匹配单个字符）
ADD index.?s /dest/
# 复制 arr[0].txt（[] 是特殊字符，需转义为 [[]]）
ADD arr[[]0].txt /dest/
```

##### （4）本地压缩包的自动解压
本地 tar 压缩包（tar/gzip/bzip2/xz 格式）会被**自动解压**（等同于 `tar -x`），判断依据是文件内容（而非后缀名）：
```dockerfile
# 本地 app.tar.gz 会被解压到 /app/ 目录（仅复制内容，无压缩包文件）
ADD app.tar.gz /app/
# 若想禁用解压，用 --unpack=false
ADD --unpack=false app.tar.gz /app/
```

#### 2. 源：远程URL（HTTP/HTTPS）
`<src>` 为远程URL时，`ADD` 会下载资源并复制到目标路径，核心规则：
- 远程压缩包**默认不解压**（需手动加 `--unpack=true` 解压）；
- 目标路径以 `/` 结尾：文件名从URL路径推断（如 `ADD https://example.com/foo.tar.gz /download/` → `/download/foo.tar.gz`）；
- 目标路径无 `/` 结尾：直接作为文件名（如 `ADD https://example.com/foo /bar` → `/bar`）；
- 权限：下载的文件默认权限为 600；
- 不支持认证：URL需要登录时，需改用 `RUN wget/curl`。

**示例**：
```dockerfile
# 下载并解压远程压缩包到 /download/
ADD --unpack=true https://example.com/archive.tar.gz /download/
# 下载远程文件并命名为 /app/config.yml
ADD https://example.com/config /app/config.yml
```

#### 3. 源：Git仓库
`<src>` 为Git仓库地址（HTTP/SSH）时，`ADD` 会克隆仓库到目标路径，核心规则：
- 支持指定分支/标签/提交/子目录（通过URL片段 `#` 实现）；
- 默认忽略 `.git` 目录（加 `--keep-git-dir=true` 可保留）；
- SSH仓库需通过 `docker build --ssh` 传递密钥认证；
- 文件权限：普通文件 644，可执行文件 755，目录 755。

**示例**：
```dockerfile
# 克隆仓库到 /mydir/
ADD https://github.com/user/repo.git /mydir/
# 克隆 v0.14.1 标签的 docs 目录到 /buildkit-docs/
ADD git@github.com:moby/buildkit.git#v0.14.1:docs /buildkit-docs/
# 保留 .git 目录克隆
ADD --keep-git-dir=true https://github.com/moby/buildkit.git#v0.10.1 /buildkit
```

### 四、`ADD` 的关键可选参数
`ADD` 支持多个参数扩展功能，以下是最常用的：

| 参数                | 作用                                                                 | 示例                                  |
|---------------------|----------------------------------------------------------------------|---------------------------------------|
| `--chmod`           | 设置目标文件/目录的权限（替代 `RUN chmod`）                          | `ADD --chmod=755 app.sh /usr/bin/`    |
| `--chown`           | 设置目标文件/目录的属主（用户:组）                                   | `ADD --chown=1000:1000 data /app/`    |
| `--unpack`          | 控制是否解压tar包（本地默认true，远程默认false）                     | `ADD --unpack=true remote.tar.gz /dir`|
| `--exclude`         | 排除匹配的文件/目录（如忽略 node_modules）                          | `ADD --exclude=node_modules src/ /app/`|
| `--checksum`        | 验证远程资源的校验和（Git用commit SHA，HTTP用sha256）                | `ADD --checksum=sha256:xxx url /dir`  |
| `--keep-git-dir`    | 克隆Git仓库时保留 `.git` 目录（默认false）                           | `ADD --keep-git-dir=true git://...`   |

### 五、`ADD` vs `COPY`（核心区别）
这是新手必掌握的知识点，两者核心都是复制文件，但定位不同：

| 特性                | `ADD`                                  | `COPY`                                |
|---------------------|----------------------------------------|---------------------------------------|
| 核心功能            | 复制 + 解压 + 远程资源（URL/Git）      | 仅纯文件复制（本地构建上下文）        |
| 本地压缩包          | 自动解压（可通过 `--unpack=false` 禁用） | 仅复制，不解压                        |
| 远程资源            | 支持URL/Git仓库                        | 不支持（仅本地文件）                  |
| 通配符              | 支持                                   | 支持                                  |
| 推荐场景            | 需解压本地压缩包、拉取远程资源时       | 仅复制本地文件（Docker官方推荐优先用）|

**最佳实践**：
- 仅复制本地文件 → 用 `COPY`（功能单一，更易维护）；
- 需要解压本地压缩包、拉取远程URL/Git仓库 → 用 `ADD`；
- 避免用 `ADD` 下载远程文件（优先用 `RUN wget/curl`，可控制权限/重命名/清理缓存）。

### 六、实操示例（完整Dockerfile）
```dockerfile
# 声明使用BuildKit（支持ADD的高级参数）
# syntax=docker/dockerfile:1
FROM alpine:3.20

# 设置工作目录
WORKDIR /app

# 1. 复制本地文件（通配符），设置权限
ADD --chmod=755 scripts/*.sh /app/scripts/

# 2. 解压本地压缩包到指定目录
ADD app.tar.gz /app/

# 3. 下载远程文件并解压
ADD --unpack=true https://example.com/config.tar.gz /app/config/

# 4. 克隆Git仓库（仅docs目录）
ADD git@github.com:user/my-app.git#v1.0:docs /app/docs/

# 5. 复制文件（用COPY，仅本地复制场景）
COPY config.yml /app/
```

### 总结
1. `ADD` 是增强版文件复制指令，支持本地文件（含解压）、远程URL、Git仓库的复制；
2. 核心规则：多个源时目标必须是目录（/结尾），本地tar包自动解压，远程tar包默认不解压；
3. 路径解析：绝对路径基于根目录，相对路径基于 `WORKDIR`，末尾 `/` 决定是文件/目录；
4. 最佳实践：优先用 `COPY` 做本地文件复制，仅在需要解压/远程资源时用 `ADD`；
5. 关键参数：`--chmod`（权限）、`--unpack`（解压）、`--exclude`（排除）是最常用的扩展功能。


