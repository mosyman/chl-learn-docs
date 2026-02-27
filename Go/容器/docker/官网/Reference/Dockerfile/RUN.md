

### 一、`RUN` 指令的核心定义
`RUN` 是 Dockerfile 中最核心的指令之一，核心作用是：
1. 在**当前镜像层之上执行指定命令**；
2. 命令执行完成后，将“执行结果 + 镜像层变更”打包成**新的镜像层**；
3. 这个新层会作为后续 Dockerfile 指令的基础。

简单来说：`RUN` 是用来**在构建镜像时执行操作**（比如安装软件、创建目录、编译代码）的指令，每一个 `RUN` 都会生成一个独立的镜像层（Docker 镜像的分层特性）。

### 二、`RUN` 的两种语法形式
`RUN` 支持 Shell 形式和 Exec 形式，核心差异和规则完全复用「Shell and exec form」的通用逻辑，具体如下：

#### 1. Shell 形式（最常用）
##### 语法格式
```dockerfile
RUN [OPTIONS] <command> ...
```
##### 核心特点
- 自动调用系统默认 Shell（Linux：`/bin/sh -c`；Windows：`cmd /S /C`）；
- 支持 Shell 所有特性：环境变量替换、管道（`|`）、重定向（`>`）、通配符（`*`）；
- 支持多行命令：用 `\` 换行续接，或用 Here-doc（`<<EOF`）拆分多行，可读性更好。

##### 示例
```dockerfile
# 单行简单命令
RUN apt-get update && apt-get install -y curl

# 换行续接（\）
RUN apt-get update && \
    apt-get install -y curl \
    git \
    vim

# Here-doc 形式（推荐，无续接符，更易读）
RUN <<EOF
apt-get update
apt-get install -y curl git vim
rm -rf /var/lib/apt/lists/*  # 清理缓存，减小镜像体积
EOF
```

#### 2. Exec 形式
##### 语法格式
```dockerfile
RUN [OPTIONS] [ "<command>", "<param1>", "<param2>", ... ]
```
##### 核心特点
- 不自动调用 Shell，直接执行指定的可执行文件；
- 必须用 JSON 数组格式：双引号（`"`）包裹元素，单引号无效；
- 如需 Shell 特性（如变量替换），需手动调用 Shell（`sh -c`）；
- Windows 路径需转义反斜杠（`\` → `\\`）。

##### 示例
```dockerfile
# 基础示例：直接执行命令（无 Shell 解析）
RUN [ "mkdir", "-p", "/app/logs" ]

# 手动调用 Shell 解析变量/管道
RUN [ "sh", "-c", "echo $HOME > /app/home.txt && ls -l /app" ]

# Windows 示例（转义反斜杠）
RUN [ "cmd", "/c", "mkdir", "C:\\app\\logs" ]
```

#### 两种形式对比（关键避坑）
| 场景                          | Shell 形式                  | Exec 形式                          |
|-------------------------------|-----------------------------|------------------------------------|
| 变量替换（`$HOME`）           | ✅ 自动支持                  | ❌ 需手动加 `sh -c`                |
| 管道/重定向                   | ✅ 自动支持                  | ❌ 需手动加 `sh -c`                |
| 避免 Shell 作为父进程         | ❌ 无法避免                  | ✅ 直接执行程序，无额外进程        |
| 可读性（多行命令）            | ✅ Here-doc 更友好           | ❌ 多行需写在一个字符串里，易乱    |

### 三、`RUN` 支持的可选参数（OPTIONS）
`RUN` 提供了多个扩展参数，用于定制命令的执行环境，需注意不同参数要求的 Dockerfile 最低版本：

| 参数                | 最低 Dockerfile 版本 | 核心作用                                                                 |
|---------------------|----------------------|--------------------------------------------------------------------------|
| `--device`          | 1.14-labs            | 挂载宿主机设备到构建环境（如 `--device=/dev/sda:/dev/sda`），用于需要访问硬件的构建场景 |
| `--mount`           | 1.2                  | 构建时挂载临时目录/文件（如挂载缓存、秘密文件），避免写入镜像层，提升构建效率 |
| `--network`         | 1.3                  | 指定构建时的网络模式（如 `--network=host`/`--network=none`），控制构建阶段的网络访问 |
| `--security`        | 1.20                 | 覆盖构建时的安全策略（如禁用 seccomp、AppArmor），用于特殊权限的构建操作 |

#### 常用参数示例（`--mount`）
`--mount` 是最实用的扩展参数，比如挂载 apt 缓存，加速重复构建：
```dockerfile
# 挂载 apt 缓存目录，避免每次构建都重新下载
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y curl
```

### 四、`RUN` 指令的缓存机制（核心优化点）
Docker 构建镜像时会对 `RUN` 指令做**缓存优化**，这是提升构建速度的关键，但也容易引发“缓存失效/不生效”的问题，规则如下：

#### 1. 缓存的默认行为
- Docker 会缓存每个 `RUN` 指令的执行结果（镜像层）；
- 后续构建时，若 `RUN` 指令的内容、上下文（如基础镜像、前置指令）未变，会直接复用缓存层，跳过命令执行；
- 示例：`RUN apt-get dist-upgrade -y` 只要指令内容不变，每次构建都会复用缓存，不会重新执行升级。

#### 2. 手动失效缓存
如需强制跳过缓存、重新执行 `RUN` 指令，可在构建时加 `--no-cache` 标志：
```bash
docker build --no-cache -t myimage:latest .
```

#### 3. 自动失效缓存（关键规则）
`ADD` 和 `COPY` 指令会**触发 `RUN` 缓存失效**：
- 若 `ADD`/`COPY` 复制的文件内容发生变化（比如代码修改），则**所有在其之后的 `RUN` 指令缓存都会失效**；
- 示例：
  ```dockerfile
  FROM ubuntu:22.04
  # 缓存有效：只要基础镜像不变，每次构建都复用
  RUN apt-get update && apt-get install -y curl

  # 关键：若 app.py 内容变化，后续 RUN 缓存全部失效
  COPY app.py /app/

  # 缓存失效：app.py 变了，这个 RUN 会重新执行
  RUN python /app/app.py --build
  ```

#### 4. 缓存最佳实践
- 合并高频不变的 `RUN` 命令：减少镜像层数，且缓存命中率更高（比如 `apt-get update && apt-get install` 写在一行）；
- 清理缓存文件：在同一个 `RUN` 中执行安装 + 清理（如 `rm -rf /var/lib/apt/lists/*`），避免残留文件写入镜像层；
- 利用 `--mount` 挂载缓存：如 apt、npm、pip 缓存，加速重复构建。

### 五、`RUN` 指令的核心注意事项
1. **减少镜像层数**：尽量合并多个 `RUN` 命令（用 `&&` 连接），避免生成过多镜像层（Docker 镜像层数有上限）；
   ```dockerfile
   # 不好：3 个 RUN，生成 3 层
   RUN apt-get update
   RUN apt-get install -y curl
   RUN rm -rf /var/lib/apt/lists/*

   # 好：1 个 RUN，生成 1 层
   RUN apt-get update && \
       apt-get install -y curl && \
       rm -rf /var/lib/apt/lists/*
   ```
2. **避免无用操作**：不在 `RUN` 中执行临时文件写入、无关软件安装，减小镜像体积；
3. **Exec 形式的引号**：必须用双引号，且每个参数单独作为数组元素（比如 `RUN ["echo hello"]` 错误，应写 `RUN ["echo", "hello"]`）；
4. **Shell 形式的换行**：用 `\` 续接时，`\` 前必须加空格（比如 `apt-get install -y curl\` 错误，应写 `apt-get install -y curl \`）。

### 总结
1. `RUN` 指令用于在构建镜像时执行命令，生成新的镜像层，是 Dockerfile 构建操作的核心；
2. 支持 Shell 形式（常用，自动调用 Shell，支持多行/Here-doc）和 Exec 形式（无 Shell，需手动调用 `sh -c` 解析变量）；
3. 扩展参数（如 `--mount`/`--network`）可定制构建环境，提升构建效率；
4. 缓存机制：默认复用缓存，`--no-cache` 手动失效，`ADD`/`COPY` 会自动触发后续 `RUN` 缓存失效；
5. 最佳实践：合并命令减少层数、清理缓存文件、利用 `--mount` 挂载缓存。




## `RUN --device` 核心定义与前置条件
#### 1. 核心作用
`RUN --device` 是 Docker BuildKit 提供的进阶参数，核心作用是：**在 `RUN` 指令执行的构建阶段中，向构建环境挂载/暴露指定的 CDI 设备**（如 NVIDIA GPU、FPGA、专用加速卡等），让构建步骤能直接使用宿主机的硬件设备。

简单来说：传统 `RUN` 指令只能利用 CPU/内存，而 `RUN --device` 允许你在构建镜像时调用 GPU、专用芯片等硬件，比如在构建阶段直接用 GPU 编译模型、执行推理、加速编译等。

#### 2. 关键前置条件（必须满足）
这个特性目前还不是稳定版功能，使用前需满足以下条件：

| 条件 | 具体要求 |
|------|----------|
| Dockerfile 语法 | 必须指定实验性语法：`# syntax=docker/dockerfile:1-labs` |
| BuildKit 版本 | 需 BuildKit 0.20.0 及以上（Docker 24.0+ 通常自带） |
| 权限配置 | 需开启 `device` 权限：<br>1. 启动 buildkitd 时加 `--allow-insecure-entitlement device` 标志；<br>2. 构建命令加 `--allow device` 标志（如 `docker buildx build --allow device ...`）；<br>3. 或在 buildkitd 配置文件中永久开启该权限 |
| 设备支持 | 宿主机需安装对应设备的 CDI 规范文件（如 NVIDIA GPU 需 NVIDIA CDI 插件） |

### 二、CDI 是什么？（理解 `--device` 的基础）
`CDI` 全称 **Container Device Interface**（容器设备接口），是一套标准化的规范，用于定义“如何向容器/构建环境暴露硬件设备”，核心作用是：
1. 统一设备的命名、挂载方式（比如 NVIDIA GPU 统一命名为 `nvidia.com/gpu`）；
2. 自动注入设备所需的环境变量、挂载点、权限等（无需手动配置）；
3. 支持按“厂商/类型/类”筛选设备。

BuildKit 通过读取宿主机上的 CDI 规范文件（YAML 格式），识别可用设备，`RUN --device` 就是基于这套规范来请求设备。

### 三、`RUN --device` 语法规则
#### 1. 基础语法
```dockerfile
RUN --device=<设备标识符>[,<required>] <命令>
```
- `<设备标识符>`：核心参数，格式灵活，支持多种匹配规则；
- `<required>`（可选）：指定 `required` 表示“必须找到该设备，否则构建失败”（默认就是 required，可省略）。

#### 2. 设备标识符的匹配规则（核心）
设备标识符支持灵活的匹配模式，适配不同的设备选择需求：

| 匹配模式 | 示例 | 含义 |
|----------|------|------|
| 厂商/类型 | `vendor1.com/device` | 请求该厂商下的第一个可用设备 |
| 指定设备名 | `vendor1.com/device=foo` | 请求该厂商下名为 `foo` 的具体设备 |
| 通配符 | `vendor1.com/device=*` | 请求该厂商下的所有设备 |
| 按类匹配 | `class1` | 请求标注为 `class1` 类的所有设备（通过 CDI 注释 `org.mobyproject.buildkit.device.class` 定义） |
| 通用硬件 | `nvidia.com/gpu=all` | 请求所有 NVIDIA GPU（最常用的场景） |

#### 3. CDI 规范文件示例解析（理解设备定义）
文档中的 CDI 规范文件示例（YAML）是核心参考，拆解如下：
```yaml
cdiVersion: "0.6.0"          # CDI 规范版本
kind: "vendor1.com/device"    # 设备类型（厂商/设备类型）
devices:
  - name: foo                 # 设备名
    containerEdits:           # 注入到构建环境的配置
      env: [FOO=injected]     # 自动注入环境变量
  - name: bar
    annotations:              # 设备注释（用于分类）
      org.mobyproject.buildkit.device.class: class1
    containerEdits:
      env: [BAR=injected]
  - name: baz
    annotations:
      org.mobyproject.buildkit.device.class: class1
    containerEdits:
      env: [BAZ=injected]
  - name: qux
    annotations:
      org.mobyproject.buildkit.device.class: class2
    containerEdits:
      env: [QUX=injected]
annotations:
  # 自动允许使用该厂商的所有设备（无需额外权限）
  org.mobyproject.buildkit.device.autoallow: true
```
- `containerEdits`：BuildKit 会自动将这里的配置（环境变量、挂载点等）注入到 `RUN` 构建阶段；
- `annotations`：用于设备分类，支持按类请求（如 `--device=class1`）；
- `autoallow`：自动允许使用该设备，无需手动配置权限（简化操作）。

### 四、实战示例：CUDA 加速 LLaMA 模型推理（核心场景）
文档中的示例是 `RUN --device` 最典型的应用：**在构建镜像时直接用 NVIDIA GPU 执行 LLaMA 模型推理**，拆解如下：

#### 1. 完整 Dockerfile 解析
```dockerfile
# 必须指定实验性语法，否则 --device 无效
# syntax=docker/dockerfile:1-labs

# 阶段1：下载 LLaMA 模型文件（无硬件依赖）
FROM scratch AS model
ADD https://huggingface.co/.../Llama-3.2-1B-Instruct-Q4_K_M.gguf /model.gguf

# 阶段2：准备推理提示词（无硬件依赖）
FROM scratch AS prompt
COPY <<EOF prompt.txt
Q: 生成10个人口最多国家的1900/2024人口数据（JSON格式）
A:
[
    {
EOF

# 阶段3：核心构建阶段（使用 GPU 执行推理）
# 基于带 CUDA 的 llama.cpp 镜像
FROM ghcr.io/ggml-org/llama.cpp:full-cuda-b5124
# --device=nvidia.com/gpu=all：请求所有 NVIDIA GPU
# --mount：从前面的阶段挂载模型和提示词文件
RUN --device=nvidia.com/gpu=all \
    --mount=from=model,target=/models \
    --mount=from=prompt,target=/tmp \
    # 执行 GPU 加速的模型推理
    ./llama-cli -m /models/model.gguf -no-cnv -ngl 99 -f /tmp/prompt.txt
```

#### 2. 核心亮点
- **构建阶段用 GPU**：传统做法是“构建镜像后，运行容器时才用 GPU”，而这里直接在 `RUN` 构建阶段调用 GPU 执行推理，推理结果会被打包进镜像；
- **多阶段构建 + 设备挂载**：结合 `--mount` 从其他阶段挂载模型/提示词，避免把大文件写入镜像层；
- **简化 GPU 配置**：通过 CDI 自动注入 NVIDIA GPU 的环境变量、驱动挂载点，无需手动配置 `LD_LIBRARY_PATH` 等。

#### 3. 构建命令（需带权限）
执行构建时，必须加 `--allow device` 开启权限：
```bash
docker buildx build \
  --allow device \          # 允许使用 device 权限
  --tag llama-inference:latest \
  .
```

### 五、使用注意事项（避坑要点）
1. **仅适用于构建阶段**：`RUN --device` 是给 `RUN` 指令的构建阶段用的，容器运行时的设备挂载仍需用 `docker run --gpus` 或 Kubernetes 的 `resources.limits`；
2. **镜像体积问题**：如果在 `RUN` 中执行 GPU 推理/编译，结果文件（如推理输出）会被写入镜像层，需注意清理无用文件；
3. **权限安全**：`--allow-insecure-entitlement device` 是不安全的权限，生产环境需严格控制，仅给可信的构建任务使用；
4. **CDI 规范版本**：注释匹配（如 `class1`）需要 CDI 规范 0.6.0 及以上，需确保宿主机 CDI 插件版本匹配；
5. **设备可用性**：构建前需确认宿主机的硬件设备已正确配置（如 NVIDIA GPU 需安装驱动、nvidia-container-toolkit、CDI 插件）。

### 六、适用场景
`RUN --device` 主要用于“构建阶段需要硬件加速”的场景，比如：
1. AI/大模型：构建阶段用 GPU 编译模型、执行推理、量化模型；
2. 编译加速：用 GPU/FPGA 加速代码编译（如 CUDA 程序编译）；
3. 专用硬件：构建阶段调用专用芯片（如加密卡、AI 加速卡）处理数据；
4. 测试验证：构建阶段用 GPU 测试程序的硬件兼容性。

### 总结
1. `RUN --device` 是 Docker BuildKit 的实验性特性，允许在 `RUN` 构建阶段调用 CDI 规范的硬件设备（如 NVIDIA GPU）；
2. 使用前需满足：指定 1-labs 语法、BuildKit 0.20+、开启 device 权限、配置 CDI 规范；
3. 设备标识符支持灵活匹配（厂商/名称/类），核心场景是构建阶段的硬件加速；
4. 典型应用：AI 模型推理/编译、CUDA 程序编译、专用硬件数据处理等。



## `RUN --mount` 
#### 1. 核心作用
`RUN --mount` 是 Docker BuildKit 提供的核心进阶参数，
核心作用是：<span style="color: #ff6600; font-size: 15px; font-weight: bold;">在 `RUN` 指令执行的构建阶段中，临时挂载文件系统/资源（而非将资源写入镜像层）</span>，构建完成后挂载自动解除。

#### 2. 核心价值（解决传统构建的痛点）
| 传统构建问题 | `RUN --mount` 解决方案 |
|--------------|-----------------------|
| 镜像体积大（缓存文件、密钥写入镜像） | 临时挂载缓存/密钥，构建后自动清理，不写入镜像层 |
| 构建速度慢（重复下载依赖包） | 持久化包管理缓存（apt/pip/go mod），重复构建无需重新下载 |
| 安全风险（密钥/私钥硬编码或复制到镜像） | 挂载秘密文件/SSH 密钥，仅构建阶段可用，不泄露到最终镜像 |
| 多阶段构建文件传递繁琐 | 直接挂载其他构建阶段的文件，无需 `COPY`（避免中间层） |

### 二、基础语法
```dockerfile
RUN --mount=[type=<TYPE>][,option=<value>[,option=<value>]...] <命令>
```
- `type`：挂载类型（必填，默认 `bind`），支持 `bind`/`cache`/`tmpfs`/`secret`/`ssh`；
- `option`：挂载选项（可选，不同类型支持不同选项），多个选项用逗号分隔（无空格）。

### 三、各挂载类型详解（按使用频率排序）
#### 1. `type=cache`：持久化缓存（最常用）
##### 核心作用
挂载一个**持久化的缓存目录**，用于存储包管理器（apt/pip/npm/go mod）、编译器的缓存文件，重复构建时直接复用缓存，大幅提升构建速度。

##### 关键特性
- 缓存目录跨构建会话持久化（不会因 `docker build` 结束而消失）；
- 缓存内容不会写入镜像层，不增加镜像体积；
- 支持并发控制（`sharing` 选项），避免多构建进程同时读写缓存冲突。

##### 核心选项
| 选项 | 说明 |
|------|------|
| `target`（必填） | 缓存目录在构建容器内的挂载路径（如 `/var/cache/apt`） |
| `id`（可选） | 缓存唯一标识，默认等于 `target`（用于区分不同缓存） |
| `sharing`（可选） | 并发策略：<br>- `shared`（默认）：多构建进程共享缓存<br>- `private`：每个构建进程创建独立缓存<br>- `locked`：多进程排队使用缓存（避免冲突） |
| `ro`/`readonly`（可选） | 设为只读（默认可写） |
| `mode`/`uid`/`gid`（可选） | 缓存目录的文件权限、所属用户/组（默认 0755、root/root） |

##### 实战示例
###### 示例1：缓存 Go 编译缓存
```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.21
# 挂载 go build 缓存目录，重复构建无需重新编译依赖
RUN --mount=type=cache,target=/root/.cache/go-build \
  go build -o /app/myapp main.go
```

###### 示例2：缓存 apt 包（解决并行冲突）
apt 需独占缓存，因此用 `sharing=locked`：
```dockerfile
# syntax=docker/dockerfile:1
FROM ubuntu:22.04
# 禁用 apt 自动清理下载的包文件
RUN rm -f /etc/apt/apt.conf.d/docker-clean; \
    echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache

# 挂载 apt 缓存目录，锁定避免并行冲突
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y gcc
```

#### 2. `type=bind`：绑定挂载（默认类型）
##### 核心作用
将**构建上下文、其他构建阶段、镜像**中的文件/目录，绑定挂载到当前构建容器内（默认只读），用于在构建阶段访问外部文件，无需 `COPY`（避免写入镜像层）。

##### 核心选项
| 选项 | 说明 |
|------|------|
| `target`（必填） | 挂载路径（如 `/app/code`） |
| `source`（可选） | 源路径（相对于 `from` 的根目录，默认根目录） |
| `from`（可选） | 源来源：<br>- 构建上下文（默认）：本地代码目录<br>- 构建阶段名：如 `builder`（多阶段构建）<br>- 镜像名：如 `ubuntu:22.04` |
| `rw`/`readwrite`（可选） | 设为可写（默认只读，写入数据会被丢弃） |

##### 实战示例（多阶段构建挂载）
```dockerfile
# syntax=docker/dockerfile:1
# 阶段1：编译代码
FROM golang:1.21 AS builder
WORKDIR /go/src/app
COPY . .
RUN go build -o /go/bin/myapp .

# 阶段2：运行（直接挂载阶段1的产物，无需 COPY）
FROM alpine:3.18
RUN --mount=type=bind,from=builder,source=/go/bin/myapp,target=/usr/local/bin/myapp \
  myapp --version  # 直接执行挂载的二进制文件
```

#### 3. `type=secret`：挂载秘密文件（安全）
##### 核心作用
将**敏感信息（密钥、令牌、配置文件）** 临时挂载到构建容器内，仅构建阶段可用，**不会写入镜像层**，解决传统构建中“密钥泄露”的问题。

##### 关键特性
- 支持挂载为文件或环境变量；
- 秘密文件由构建命令传入（`docker buildx build --secret`），不暴露在 Dockerfile 中；
- 可选“必填校验”（`required=true`），确保秘密文件存在。

##### 核心选项
| 选项 | 说明 |
|------|------|
| `id`（必填） | 秘密文件的唯一标识（与构建命令的 `--secret id=` 对应） |
| `target`（可选） | 秘密文件的挂载路径（默认 `/run/secrets/<id>`） |
| `env`（可选） | 将秘密文件内容挂载为环境变量（如 `env=API_KEY`） |
| `required`（可选） | 设为 `true` 时，秘密文件不存在则构建失败（默认 `false`） |
| `mode`/`uid`/`gid`（可选） | 秘密文件的权限（默认 0400，仅所有者可读） |

##### 实战示例
###### 示例1：挂载 AWS 凭证（文件形式）
```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12
RUN pip install awscli
# 挂载 AWS 凭证文件到 /root/.aws/credentials
RUN --mount=type=secret,id=aws,target=/root/.aws/credentials \
  aws s3 cp s3://my-bucket/data.txt /app/
```
构建命令（传入本地凭证文件）：
```bash
docker buildx build --secret id=aws,src=$HOME/.aws/credentials .
```

###### 示例2：挂载为环境变量
```dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.18
# 将 id=API_KEY 的秘密挂载为环境变量 API_KEY
RUN --mount=type=secret,id=API_KEY,env=API_KEY \
    curl -H "Authorization: Bearer $API_KEY" https://api.example.com/data
```
构建命令（使用本地环境变量作为秘密）：
```bash
docker buildx build --secret id=API_KEY .  # 自动读取本地 API_KEY 环境变量
```

#### 4. `type=ssh`：挂载 SSH 密钥（安全）
##### 核心作用
将**SSH 密钥/SSH Agent 套接字** 挂载到构建容器内，用于构建阶段访问私有 Git 仓库、SSH 服务器，无需将私钥复制到镜像。

##### 关键特性
- 支持带密码的 SSH 密钥（通过 SSH Agent）；
- 仅构建阶段可用，不泄露私钥；
- 兼容主流 Git 平台（GitHub/GitLab/GitLab）。

##### 核心选项
| 选项 | 说明 |
|------|------|
| `id`（可选） | SSH 密钥/Agent 的标识（默认 `default`） |
| `target`（可选） | SSH Agent 套接字的挂载路径（默认 `/run/buildkit/ssh_agent.${N}`） |
| `required`（可选） | 密钥不存在则构建失败（默认 `false`） |

##### 实战示例（访问 GitLab 私有仓库）
```dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.18
# 安装 SSH 客户端，添加 GitLab 到已知主机
RUN apk add --no-cache openssh-client && \
    mkdir -p -m 0700 ~/.ssh && \
    ssh-keyscan gitlab.com >> ~/.ssh/known_hosts

# 挂载 SSH Agent 套接字，测试 GitLab 连接
RUN --mount=type=ssh \
  ssh -q -T git@gitlab.com 2>&1 | tee /gitlab_welcome.txt
```
构建前准备（启动 SSH Agent 并添加密钥）：
```bash
eval $(ssh-agent)  # 启动 SSH Agent
ssh-add ~/.ssh/id_rsa  # 添加私钥（支持密码）
docker buildx build --ssh default=$SSH_AUTH_SOCK .  # 传入 Agent 套接字
```

#### 5. `type=tmpfs`：临时文件系统
##### 核心作用
在构建容器内挂载 `tmpfs`（内存文件系统），用于存储临时数据，**构建结束后数据完全消失**，不写入镜像层，也不占用磁盘空间。

##### 核心选项
| 选项 | 说明 |
|------|------|
| `target`（必填） | `tmpfs` 的挂载路径 |
| `size`（可选） | `tmpfs` 的最大大小（如 `size=1G`） |

##### 实战示例
```dockerfile
# syntax=docker/dockerfile:1
FROM ubuntu:22.04
# 挂载 1GB 大小的 tmpfs 到 /tmp/data，用于临时处理大文件
RUN --mount=type=tmpfs,target=/tmp/data,size=1G \
  dd if=/dev/zero of=/tmp/data/large_file bs=1M count=500 && \
  sha256sum /tmp/data/large_file
```

### 四、关键使用规则与避坑要点
1. **语法要求**：
    - 多个 `--mount` 可叠加（如同时挂载 cache + secret）；
    - 选项之间用逗号分隔，**无空格**（如 `--mount=type=cache,target=/var/cache/apt,sharing=locked`）；
    - 需使用 BuildKit 构建（Docker 20.0+ 默认启用，旧版本需加 `DOCKER_BUILDKIT=1`）。

2. **缓存规则**：
    - `--mount` 本身不会触发 `RUN` 缓存失效；
    - `cache` 类型的挂载内容不影响缓存（即使缓存内容变化，`RUN` 仍复用缓存）。

3. **安全要点**：
    - `secret`/`ssh` 类型的挂载内容仅当前 `RUN` 指令可用，其他指令无法访问；
    - 避免将 `secret`/`ssh` 的挂载路径设置为镜像中已存在的目录（防止覆盖）；
    - 生产环境建议设置 `required=true`，避免秘密文件缺失导致构建失败/不安全。

4. **兼容性**：
    - 需指定 Dockerfile 语法为 `docker/dockerfile:1` 及以上；
    - `env` 选项（secret 类型）需 Dockerfile v1.10.0+（对应 `docker/dockerfile:1.4`+）。

### 五、总结
1. `RUN --mount` 是 Docker BuildKit 提升构建效率和安全性的核心特性，支持 5 种挂载类型，核心价值是“临时挂载、不写入镜像层”；
2. **高频场景**：
    - `cache`：加速包管理器/编译器缓存（apt/pip/go mod）；
    - `secret`：安全挂载密钥/令牌，避免泄露；
    - `ssh`：访问私有 Git 仓库，不暴露私钥；
    - `bind`：多阶段构建文件传递，减少镜像层；
3. **核心优势**：提升构建速度、减小镜像体积、增强构建安全性；
4. **避坑关键**：注意选项格式（无空格）、安全选项（`required=true`）、并发控制（`sharing=locked`）。









