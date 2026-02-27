

## 目录
- [Examples](#Examples)
- [Attributes](#Attributes)



## Examples

### 一、Services 核心概念：容器的抽象与统一管理
#### 1. 服务的本质
`services` 是 Compose 文件中**最核心的顶级元素**，它是对「应用组件」的抽象定义：
- 一个「服务」对应一组**完全相同的容器**（可按需扩缩容），比如 `web` 服务可启动 3 个 nginx 容器，所有容器使用相同的镜像、配置、运行参数；
- 服务是 Compose 管理资源的基本单位，可独立缩放、更新、替换，不影响其他服务（比如单独扩缩 `db` 服务，不影响 `web` 服务）；
- 服务的底层是 Docker 容器，但 Compose 为服务提供了更高层的抽象（比如依赖管理、网络互通、批量操作）。

#### 2. 核心规则
- Compose 文件**必须包含 `services` 顶级元素**（否则无效），它是一个「键值对映射」：
    - 键：服务名（字符串，如 `web`/`db`/`proxy`），建议使用小写字母、数字、连字符（`-`），唯一标识服务；
    - 值：服务定义（嵌套的配置项），决定该服务容器的运行方式、镜像来源、资源约束等。
- 所有同服务的容器会被 Compose 统一管理：比如 `docker compose up web` 启动 `web` 服务的所有容器，`docker compose scale web=3` 扩缩容到 3 个容器。

### 二、Services 基本语法与结构
`services` 是顶级字段，与 `name`/`volumes`/`networks` 同级，基本结构如下：
```yaml
# 顶级字段
services:
  # 服务名（自定义，唯一）
  <service-name>:
    # 服务配置项（镜像、端口、环境变量、构建规则等）
    image: <image-name>  # 必选（除非用 build 构建镜像）
    [其他配置项: 值]
  # 第二个服务
  <another-service-name>:
    [配置项]
```

#### 核心配置项分类（基础必懂）
| 配置项       | 作用                                                                 | 是否必选 |
|--------------|----------------------------------------------------------------------|----------|
| `image`      | 指定服务使用的 Docker 镜像（如 `nginx:latest`/`postgres:18`）        | 是（或用 `build` 替代） |
| `build`      | 定义镜像构建规则（替代 `image`，从 Dockerfile 构建镜像）             | 否       |
| `ports`      | 端口映射（宿主机端口:容器端口）                                      | 否       |
| `environment`| 设置容器内的环境变量                                                | 否       |
| `volumes`    | 数据卷挂载（宿主机/卷 → 容器目录）                                   | 否       |
| `depends_on` | 服务依赖（启动顺序：先启动依赖服务，再启动当前服务）                 | 否       |
| `deploy`     | 部署约束（扩缩容、资源限制、调度策略等）                             | 否       |

### 三、核心配置项详解（从基础到进阶）
#### 1. 基础配置：`image`/`ports`/`environment`
这是最常用的基础配置，用于快速定义服务的镜像、端口、环境变量。

##### （1）`image`：指定镜像（核心必选）
- 语法：`image: <镜像名>:<标签>`，支持官方镜像、私有镜像、本地镜像；
- 示例：
  ```yaml
  services:
    db:
      image: postgres:18  # 使用 postgres 18 版本镜像
  ```

##### （2）`ports`：端口映射
- 作用：将容器端口暴露到宿主机，实现外部访问；
- 语法：
  ```yaml
  ports:
    - "宿主机端口:容器端口"  # 推荐：固定端口
    - "容器端口"            # 宿主机随机分配端口
  ```
- 示例：
  ```yaml
  services:
    web:
      image: nginx:latest
      ports:
        - "8080:80"  # 宿主机 8080 端口映射到容器 80 端口
  ```

##### （3）`environment`：环境变量
- 作用：设置容器内的环境变量，用于配置应用（比如数据库用户名、密码）；
- 语法（两种形式，效果一致）：
  ```yaml
  # 形式 1：键值对（推荐，易读）
  environment:
    POSTGRES_USER: example
    POSTGRES_DB: exampledb
  # 形式 2：数组（适合值含特殊字符）
  environment:
    - POSTGRES_USER=example
    - POSTGRES_DB=exampledb
  ```

#### 2. 进阶配置：`volumes`/`depends_on`
用于数据持久化、服务依赖管理，解决「配置挂载」「启动顺序」问题。

##### （1）`volumes`：数据卷挂载
- 作用：将宿主机文件/目录、Docker 卷挂载到容器内，实现数据持久化或配置注入；
- 语法（推荐「长格式」，语义更清晰）：
  ```yaml
  volumes:
    - type: bind        # 类型：bind（宿主机绑定）/ volume（Docker 卷）/ tmpfs（临时文件）
      source: ./proxy/nginx.conf  # 宿主机源路径
      target: /etc/nginx/conf.d/default.conf  # 容器内目标路径
      read_only: true   # 可选：设为只读（避免容器修改配置）
  ```
- 示例（挂载 Nginx 配置文件）：
  ```yaml
  services:
    proxy:
      image: nginx
      volumes:
        - type: bind
          source: ./proxy/nginx.conf
          target: /etc/nginx/conf.d/default.conf
          read_only: true
  ```

##### （2）`depends_on`：服务依赖
- 作用：定义服务启动顺序（比如先启动 `backend`，再启动 `proxy`），解决「依赖服务未启动导致当前服务失败」的问题；
- 注意：仅保证「启动顺序」，不保证「依赖服务完全就绪」（比如 `db` 服务启动了但数据库未初始化完成，`web` 服务仍可能连接失败，需自己处理健康检查）；
- 示例：
  ```yaml
  services:
    proxy:
      depends_on:
        - backend  # 先启动 backend，再启动 proxy
    backend:
      image: my-backend:latest
  ```

#### 3. 特殊配置：`build`/`deploy`
这两个配置是可选的，分别用于「构建镜像」和「部署约束」，属于 Compose 规范的扩展能力。

##### （1）`build`：从 Dockerfile 构建镜像
- 作用：替代 `image`，指定「从本地 Dockerfile 构建镜像」，而非使用现成镜像；
- 核心子配置：
    - `context`：构建上下文路径（Dockerfile 所在目录，必填）；
    - `target`：多阶段构建的目标阶段（可选，比如只构建 `builder` 阶段）；
- 示例：
  ```yaml
  services:
    backend:
      build:
        context: backend  # 构建上下文：当前目录下的 backend 文件夹
        target: builder   # 仅构建 Dockerfile 中 AS builder 的阶段
  ```

##### （2）`deploy`：部署约束
- 作用：定义服务的部署策略（扩缩容数量、资源限制、调度规则等），适用于 Swarm/Kubernetes 等编排平台；
- 特性：属于「可选规范」，若 Compose 不支持（比如单机 Compose），会忽略该配置，不影响文件有效性；
- 示例（简单资源限制）：
  ```yaml
  services:
    web:
      image: nginx
      deploy:
        replicas: 3  # 启动 3 个容器
        resources:
          limits:
            cpus: '0.5'  # 每个容器最多用 0.5 核 CPU
            memory: 512M # 每个容器最多用 512MB 内存
  ```

### 四、示例解析：从基础到进阶
#### 1. 基础示例（核心配置演示）
```yaml
services:
  # web 服务：Nginx 镜像，端口映射
  web:
    image: nginx:latest
    ports:
      - "8080:80"  # 宿主机 8080 → 容器 80

  # db 服务：PostgreSQL 镜像，环境变量配置
  db:
    image: postgres:18
    environment:
      POSTGRES_USER: example    # 数据库用户名
      POSTGRES_DB: exampledb    # 默认数据库名
```
- 核心逻辑：启动两个独立服务，`web` 提供 HTTP 服务（外部可通过 8080 访问），`db` 提供 PostgreSQL 数据库（仅内网可访问）；
- 运行命令：`docker compose up` → 自动创建网络，启动 `web`/`db` 容器，网络互通（`web` 可通过 `db` 主机名访问数据库）。

#### 2. 进阶示例（构建+挂载+依赖）
```yaml
services:
  # proxy 服务：Nginx 反向代理，依赖 backend 服务
  proxy:
    image: nginx
    volumes:
      - type: bind
        source: ./proxy/nginx.conf  # 挂载本地配置文件
        target: /etc/nginx/conf.d/default.conf
        read_only: true  # 配置文件只读
    ports:
      - 80:80  # 宿主机 80 → 容器 80
    depends_on:
      - backend  # 先启动 backend，再启动 proxy

  # backend 服务：从本地 Dockerfile 构建镜像
  backend:
    build:
      context: backend  # 构建上下文：backend 目录
      target: builder   # 多阶段构建：仅构建 builder 阶段
```
- 核心逻辑：
    1. `backend` 服务：从 `backend` 目录的 Dockerfile 构建镜像（仅构建 `builder` 阶段）；
    2. `proxy` 服务：挂载本地 Nginx 配置文件（反向代理到 `backend`），依赖 `backend` 确保启动顺序；
    3. 外部访问：宿主机 80 端口 → proxy 容器 → 转发到 backend 服务。

### 总结
#### 1. Services 核心关键点
- `services` 是 Compose 文件的核心，每个服务对应一组相同的容器，可独立管理；
- 服务定义的核心是「镜像来源」（`image` 或 `build`）+「运行配置」（端口、环境变量、挂载等）；
- 基础配置（`image`/`ports`/`environment`）满足大部分单机场景，进阶配置（`volumes`/`depends_on`/`build`）解决复杂场景。

#### 2. 最佳实践
- 服务名使用语义化命名（如 `web`/`db`/`proxy`），便于识别；
- 优先使用 `image` 而非 `build`（复用现成镜像，提升构建速度），仅需自定义镜像时用 `build`；
- 挂载配置文件时使用 `read_only: true`，避免容器意外修改配置；
- `depends_on` 仅用于启动顺序，需自行处理「服务就绪检查」（比如用 `healthcheck`）。

#### 3. 核心规则回顾
- Compose 文件必须包含 `services` 顶级元素；
- `build` 和 `image` 二选一（同时写时，`build` 构建的镜像会覆盖 `image` 命名）；
- `deploy` 是可选配置，单机 Compose 忽略，Swarm 等平台生效；
- 同服务的容器共享网络、配置，可通过 `docker compose scale` 扩缩容。


[↑目录](#目录)

## Attributes

### 1. `annotations`（注解/元数据）
#### 翻译
`annotations` 用于为容器定义**注解（元数据）**。它既可以使用**映射（map）** 格式，也可以使用**数组（array）** 格式。

#### 核心解释
- **作用**：给容器添加**自定义的键值对元数据**，这些数据不会影响容器的运行逻辑，主要用于：
    - 标记容器的用途（如 `owner: dev-team`）
    - 记录版本信息（如 `build-version: v1.2.3`）
    - 供运维工具/平台识别和筛选容器
- **格式规则**：
    1. **Map 格式（推荐）**：键值对清晰，是最常用的写法
       ```yml
       annotations:
         com.example.foo: bar       # 键：com.example.foo，值：bar
         com.company.owner: alice   # 键：com.company.owner，值：alice
       ```
    2. **Array 格式**：用等号分隔键值，适合简单场景
       ```yml
       annotations:
         - com.example.foo=bar      # 等价于 com.example.foo: bar
         - env=production           # 等价于 env: production
       ```
- **注意事项**：
    - 注解的键通常建议加**域名前缀**（如 `com.example.`），避免和其他工具的元数据冲突；
    - 这些注解会被 Docker 引擎存储，但不会暴露到容器内部（容器内无法直接读取）。

### 2. `attach`（日志挂载/附加）
#### 翻译
当定义 `attach` 并将其设置为 `false` 时，Compose 不会主动收集该服务的日志，除非你**显式地主动请求**（比如执行 `docker compose logs <服务名>`）。
服务的默认配置是 `attach: true`。

#### 核心解释
- **作用**：控制 Compose 是否**实时附加（挂载）** 服务的日志输出到当前终端。
- **关键场景**：
    - 默认 `attach: true`：执行 `docker compose up` 时，终端会实时打印该服务的日志（stdout/stderr）；
    - 设置 `attach: false`：执行 `docker compose up` 时，该服务的日志不会出现在终端，终端会更简洁，适合日志量很大的服务（如数据库、中间件）。
- **示例用法**：
  ```yml
  services:
    mysql:
      image: mysql:8.0
      attach: false  # 不实时打印 mysql 日志，避免终端被刷屏
      environment:
        MYSQL_ROOT_PASSWORD: 123456
    web:
      image: nginx
      attach: true   # 实时打印 nginx 日志（默认值，可省略）
  ```
- **如何主动获取日志**：
  即使设置 `attach: false`，也可以通过以下命令查看日志：
  ```bash
  docker compose logs mysql  # 查看 mysql 服务的日志
  docker compose logs -f mysql  # 实时跟踪 mysql 日志（-f = follow）
  ```
- **注意事项**：
    - `attach` 只控制**终端是否实时显示日志**，不会影响日志的存储（Docker 仍会保存日志，直到被清理）；
    - 若执行 `docker compose up -d`（后台运行），`attach` 配置会被忽略（因为后台运行本身就不会在终端输出日志）。

### 3. `build`（构建配置）
#### 翻译
`build` 用于指定**从源代码构建容器镜像**的配置，其具体规则遵循 [Compose Build Specification（Compose 构建规范）](https://docs.docker.com/compose/compose-file/build/)。

#### 核心解释
- **作用**：替代 `image` 字段（或和 `image` 配合），告诉 Compose：「不要直接拉取现成镜像，而是从本地源码构建镜像」。
- **基础用法**（最简版）：
  只指定构建上下文（源码目录），Compose 会自动找该目录下的 `Dockerfile` 进行构建：
  ```yml
  services:
    myapp:
      build: ./app  # 构建上下文：当前目录下的 app 文件夹（里面要有 Dockerfile）
  ```
- **进阶用法**（指定 Dockerfile 路径/镜像名称）：
  ```yml
  services:
    myapp:
      build:
        context: ./app        # 构建上下文（必选）：源码根目录
        dockerfile: Dockerfile.prod  # 可选：指定非默认的 Dockerfile 名称/路径
        args:                 # 可选：传递构建参数（对应 Dockerfile 里的 ARG）
          BUILD_ENV: production
      image: mycompany/myapp:v1.0  # 可选：指定构建后的镜像名称+标签
  ```
- **关键概念**：
    - **构建上下文（context）**：Docker 构建镜像时能访问的文件目录（比如 `./app`），Dockerfile 里的 `COPY`/`ADD` 只能复制该目录下的文件；
    - **dockerfile**：默认是上下文目录下的 `Dockerfile`，若文件名/路径不同，需显式指定；
    - **和 image 的关系**：若只写 `build`，Compose 会生成随机镜像名；若配合 `image`，构建后的镜像会被打上指定标签。
- **注意事项**：
    - 构建前确保上下文目录下有合法的 Dockerfile；
    - 执行 `docker compose up` 时，若镜像不存在或源码/Dockerfile 有修改，Compose 会自动重新构建；
    - 也可以单独执行 `docker compose build` 只构建镜像，不启动服务。

---

### 总结
1. `annotations`：给容器加**自定义元数据**，不影响运行，支持 map/array 两种格式，建议加域名前缀避免冲突；
2. `attach`：控制日志是否**实时输出到终端**，默认 `true`，设置 `false` 可简化终端输出，仍可通过 `logs` 命令查看日志；
3. `build`：指定**从本地源码构建镜像**的配置，核心是 `context`（构建上下文），可配合 `dockerfile`/`args`/`image` 实现自定义构建。


你希望我详细解释 Docker Compose 中 `blkio_config` 这个配置项的所有参数含义、使用场景和具体规则，我会用通俗易懂的语言拆解每个字段，结合示例说明其作用和实际效果。

### 4. `blkio_config` 
`blkio_config` 是 Docker Compose 中专门用于**限制容器块设备 I/O 资源**的配置项（blkio = block I/O，即块设备输入输出），作用是管控容器对磁盘（如 `/dev/sda`、`/dev/sdb`）的读写速度、读写操作频率，以及容器之间的 I/O 带宽分配比例。
简单说：它能防止某个容器霸占磁盘 I/O 资源，导致其他容器（或主机）磁盘读写卡顿，是容器资源隔离的重要配置。

### 二、完整配置参数拆解（结合示例）
先回顾示例结构，再逐字段解析：
```yml
services:
  foo:
    image: busybox
    blkio_config:
       weight: 300                # 全局I/O权重
       weight_device:             # 按设备细分权重
         - path: /dev/sda
           weight: 400
       device_read_bps:           # 按设备限制读字节/秒
         - path: /dev/sdb
           rate: '12mb'
       device_read_iops:          # 按设备限制读操作/秒
         - path: /dev/sdb
           rate: 120
       device_write_bps:          # 按设备限制写字节/秒
         - path: /dev/sdb
           rate: '1024k'
       device_write_iops:         # 按设备限制写操作/秒
         - path: /dev/sdb
           rate: 30
```

#### 1. `weight`：全局 I/O 带宽权重
- **作用**：设置当前容器的磁盘 I/O 带宽占比（相对其他容器），是「全局层面」的资源分配比例，不针对具体设备。
- **取值范围**：整数 10 ~ 1000，默认值 500。
- **核心逻辑**：
  - 权重越高，容器能抢占的磁盘 I/O 带宽越多；
  - 权重是「相对值」而非「绝对值」，比如：
    - 容器 A `weight: 600`，容器 B `weight: 300` → A 能获得的 I/O 带宽约是 B 的 2 倍；
    - 若只有一个容器，无论 `weight` 设为 10 还是 1000，都能使用全部 I/O 带宽（无竞争时权重不生效）。
- **示例**：配置中 `weight: 300` 表示该容器的全局 I/O 权重低于默认值（500），在多容器竞争 I/O 时，会分配到更少的带宽。

#### 2. `weight_device`：按设备细分 I/O 权重
- **作用**：在 `weight` 全局权重基础上，**针对具体磁盘设备**微调 I/O 权重（优先级高于全局 `weight`）。
- **配置结构**：数组形式，每个元素包含两个必填字段：
  - `path`：磁盘设备的符号路径（如 `/dev/sda`、`/dev/sdb`，对应主机的物理磁盘）；
  - `weight`：该设备的权重，取值 10 ~ 1000。
- **示例**：配置中 `weight_device` 针对 `/dev/sda` 设 `weight: 400` → 该容器对 `/dev/sda` 的 I/O 权重是 400（高于全局的 300），对其他设备则沿用全局 300。

#### 3. `device_read_bps` / `device_write_bps`：按设备限制读写字节/秒
- **作用**：限制容器对指定设备的「读写速度（字节/秒）」，是**绝对值限制**（不管其他容器，该容器的读写速度不会超过此值）。
- **字段说明**：
- 
  | 字段 | 含义 | 示例 |
  |------|------|------|
  | `path` | 目标磁盘设备路径（如 `/dev/sdb`） | `/dev/sdb` |
  | `rate` | 速度限制，支持两种格式：<br>1. 纯整数：表示字节数（如 `1024` = 1KB）；<br>2. 字符串：带单位（`k`=KB、`m`=MB、`g`=GB，大小写均可，如 `12mb`、`1024k`） | `'12mb'`（12MB/秒）、`'1024k'`（1MB/秒） |
- **示例**：
  - `device_read_bps` 中 `/dev/sdb` 的 `rate: '12mb'` → 该容器从 `/dev/sdb` 读取数据的速度最多 12MB/秒；
  - `device_write_bps` 中 `/dev/sdb` 的 `rate: '1024k'` → 该容器向 `/dev/sdb` 写入数据的速度最多 1MB/秒。

#### 4. `device_read_iops` / `device_write_iops`：按设备限制读写操作/秒
- **作用**：限制容器对指定设备的「读写操作次数（IOPS）」，IOPS 是衡量磁盘每秒能处理的读写操作数量（比如机械硬盘 IOPS 约 100~200，固态硬盘可达数万）。
- **字段说明**：
- 
  | 字段 | 含义 | 示例 |
  |------|------|------|
  | `path` | 目标磁盘设备路径 | `/dev/sdb` |
  | `rate` | 整数，代表每秒允许的最大读写操作次数（无单位） | 120（读操作/秒）、30（写操作/秒） |
- **核心区别**：
  - `bps` 限制「速度（字节）」，关注「传输多少数据」；
  - `iops` 限制「操作次数」，关注「执行多少次读写」；
  - 比如：一次读操作可能读取 4KB 数据，`device_read_iops: 120` 意味着每秒最多读 120 次 → 最大读速度约 120×4KB = 480KB/秒（若未设 `device_read_bps`）。
- **示例**：
  - `device_read_iops: 120` → 该容器对 `/dev/sdb` 的读操作最多 120 次/秒；
  - `device_write_iops: 30` → 该容器对 `/dev/sdb` 的写操作最多 30 次/秒。

### 三、关键使用注意事项
1. **生效范围**：`blkio_config` 仅对 Linux 主机生效（Docker for Windows/Mac 因底层虚拟化，部分参数可能不生效）；
2. **单位大小写**：`rate` 中的单位（如 `mb`/`MB`、`k`/`K`）不区分大小写，`12mb` 和 `12MB` 效果一致；
3. **优先级**：`device_read/write_bps/iops`（绝对值限制）优先级 > `weight_device`（设备权重） > `weight`（全局权重）；
4. **设备路径**：需确保配置的 `path`（如 `/dev/sdb`）在主机上真实存在，否则配置无效；
5. **IOPS 与 bps 的关系**：若同时设置 `device_read_bps` 和 `device_read_iops`，以「先触达的限制」为准（比如 IOPS 到上限，即使 bps 没到，也会停止读写）。

### 总结
`blkio_config` 是 Docker 容器磁盘 I/O 资源管控的核心配置，核心要点：
1. `weight`/`weight_device` 是**相对权重**，用于多容器间的 I/O 带宽分配；
2. `device_read/write_bps` 是**读写速度限制**（字节/秒），`device_read/write_iops` 是**读写操作次数限制**（次/秒），均为绝对值限制；
3. 配置时需结合磁盘设备的实际路径，且仅在 Linux 主机上生效。

通过合理配置 `blkio_config`，可以避免单个容器耗尽磁盘 I/O 资源，保障多容器环境的稳定性。





### 5. `command`
`command` 的核心作用是**覆盖容器镜像本身默认的启动命令**（也就是 Dockerfile 里 `CMD` 指令定义的命令），让你在不修改镜像的前提下，自定义容器启动时执行的操作。

#### 1. 基本用法
| 取值类型       | 效果说明                                                                 | 示例                          |
|----------------|--------------------------------------------------------------------------|-------------------------------|
| 字符串         | 直接覆盖默认命令，注意 shell 特性（如环境变量）需要显式调用 shell        | `command: bundle exec thin -p 3000` |
| `null`         | 不覆盖，使用镜像自带的默认命令（Dockerfile 的 `CMD`）                   | `command: null`               |
| 空列表 `[]`/空字符串 `''` | 忽略镜像默认命令，容器启动后不执行任何命令                             | `command: []` 或 `command: ''` |

#### 2. 关键注意事项（新手必看）
- **与 Dockerfile `CMD` 的区别**：`command` 不会自动使用镜像中 `SHELL` 指令定义的 shell 环境。如果你的命令依赖 shell 特性（比如环境变量展开 `$HOSTNAME`、管道 `|`、重定向 `>` 等），必须显式调用 shell：
  ```yaml
  # 正确：用 /bin/sh -c 包裹，才能解析 $$HOSTNAME（$$ 是 Compose 转义 $）
  command: /bin/sh -c 'echo "hello $$HOSTNAME"'
  # 错误：直接写会把 echo、hello、$HOSTNAME 当作三个独立参数，无法解析环境变量
  command: echo "hello $HOSTNAME"
  ```
- **列表形式（推荐）**：和 Dockerfile 的 `exec` 格式一致，更清晰，避免 shell 解析问题：
  ```yaml
  # 列表形式（exec 格式），等价于 "bundle exec thin -p 3000"
  command: ["bundle", "exec", "thin", "-p", "3000"]
  ```

---

### 二、`configs` 字段详解
`configs` 是 Docker Compose 中用于**给服务挂载配置文件**的功能，核心优势是：无需重建镜像，就能动态调整容器内的配置（比如配置文件、密钥、规则等），且只有显式授权的服务才能访问对应的配置。

#### 1. 两种语法对比
##### （1）短语法（简洁，适合基础场景）
- 仅指定配置名称，Docker 会自动将配置文件挂载到容器的 `/<配置名>`（Linux）或 `C:\<配置名>`（Windows）路径。
- 示例：
  ```yaml
  services:
    redis:
      image: redis:latest
      configs:
        - my_config       # 给 redis 服务授权访问 my_config 配置
        - my_other_config # 授权访问 my_other_config 配置
  configs:
    my_config:
      file: ./my_config.txt # 配置内容来自本地文件 ./my_config.txt
    my_other_config:
      external: true        # 配置是“外部的”（已在 Docker 平台提前创建）
  ```
  - 效果：`my_config.txt` 的内容会被挂载到 redis 容器内的 `/my_config` 文件，`my_other_config` 会挂载到 `/my_other_config`。
  - 注意：如果 `external: true` 的配置不存在，启动服务会直接失败。

##### （2）长语法（精细控制，适合定制化场景）
- 支持自定义挂载路径、文件权限、所属用户/组，字段说明：
 
- | 字段    | 作用                                                                 |
  |---------|----------------------------------------------------------------------|
  | `source`| 对应 top-level `configs` 里的配置名称（平台中存在的配置名）            |
  | `target`| 配置文件在容器内的挂载路径（自定义，比如 `/redis_config`）            |
  | `uid`/`gid` | 容器内挂载文件的所属用户/组 ID（数字，比如 103）                     |
  | `mode`  | 挂载文件的权限（八进制，默认 0444，仅可读；支持 0440 等，不可写）    |
- 示例：
  ```yaml
  services:
    redis:
      image: redis:latest
      configs:
        - source: my_config       # 关联平台中的 my_config 配置
          target: /redis_config   # 挂载到容器内的 /redis_config 路径
          uid: "103"              # 文件所属用户 ID 为 103
          gid: "103"              # 文件所属组 ID 为 103
          mode: 0440              # 权限：所有者和组可读，其他不可读
  configs:
    my_config:
      external: true
    my_other_config:
      external: true # redis 服务未授权，无法访问这个配置
  ```

#### 2. 核心特点
- 权限控制：只有显式在 `services.xxx.configs` 里列出的配置，服务才能访问；
- 无需改镜像：配置内容变更后，重启服务即可生效，不用重新构建镜像；
- 跨平台：支持 Linux/Windows 容器，挂载路径自动适配系统。

---

### 三、`container_name` 字段详解
`container_name` 用于**自定义容器的名称**，替代 Docker Compose 自动生成的默认名称（默认格式：`项目名_服务名_序号`，比如 `myapp_redis_1`）。

#### 1. 基本用法
```yaml
services:
  web:
    image: nginx:latest
    container_name: my-web-container # 自定义容器名为 my-web-container
```

#### 2. 关键限制（新手必避坑）
- **无法扩缩容**：如果指定了 `container_name`，该服务只能启动 1 个容器（`docker compose up --scale web=2` 会直接报错），因为容器名称必须唯一；
- **命名规则**：必须符合正则 `[a-zA-Z0-9][a-zA-Z0-9_.-]+`，即：
  - 首字符只能是字母/数字；
  - 后续字符可以是字母、数字、下划线 `_`、点 `.`、短横线 `-`；
  - 不能包含空格、特殊符号（如 `@`、`#` 等）。

---

### 总结
1. **command**：覆盖容器默认启动命令，依赖 shell 特性需显式调用 `/bin/sh -c`，列表形式更稳定；
2. **configs**：给服务挂载配置文件，短语法简洁、长语法可定制权限/路径，无需重建镜像；
3. **container_name**：自定义容器名，但会限制服务扩缩容，命名需符合固定正则规则。




### `volumes` 
`volumes` 是 Compose 中**最核心的存储配置字段**，用于给容器挂载存储卷（可以是主机路径、命名卷、临时内存卷等），实现容器和主机/其他容器之间的数据共享、持久化。简单来说，就是把「主机的文件/目录」或「Docker 管理的卷」映射到「容器内的指定路径」。

#### 1. 核心概念：4 种挂载类型
| 挂载类型 | 说明                                                                 | 适用场景                     |
|----------|----------------------------------------------------------------------|------------------------------|
| `volume` | Docker 管理的命名卷（数据存在 Docker 专属目录，如 `/var/lib/docker/volumes`） | 容器间共享数据、数据持久化   |
| `bind`   | 主机路径绑定挂载（直接映射主机的文件/目录到容器）                     | 开发时代码热更新、挂载配置文件 |
| `tmpfs`  | 临时内存卷（数据只存在容器内存，容器停止即消失）                     | 临时缓存、敏感数据（不落地） |
| `npipe`  | 命名管道挂载（仅 Windows 平台）                                      | Windows 容器间进程通信       |

#### 2. 两种语法形式（短语法 vs 长语法）
##### （1）短语法（简洁，适合基础场景）
用**冒号分隔**的字符串表示，格式：`[主机路径/卷名]:[容器路径]:[访问模式]`
- 各部分说明：
  - 第一部分：`VOLUME` → 主机路径（bind 挂载）或命名卷名（volume 挂载）；
  - 第二部分：`CONTAINER_PATH` → 容器内的挂载路径；
  - 第三部分（可选）：`ACCESS_MODE` → 访问模式（`rw` 读写/`ro` 只读，默认 `rw`），还支持 SELinux 选项 `z`（共享）/`Z`（私有）。

- 示例：
  ```yaml
  services:
    web:
      image: nginx
      volumes:
        # 1. bind 挂载：主机 ./html 目录 → 容器 /usr/share/nginx/html（读写）
        - ./html:/usr/share/nginx/html
        # 2. bind 挂载：主机 /var/log/nginx → 容器 /var/log/nginx（只读）
        - /var/log/nginx:/var/log/nginx:ro
        # 3. 命名卷：db-data 卷 → 容器 /data（读写，需 top-level 定义）
        - db-data:/data
        # 4. SELinux 共享：主机 ./conf → 容器 /etc/nginx/conf.d（共享标签）
        - ./conf:/etc/nginx/conf.d:z

  # top-level 定义命名卷（供多个服务复用）
  volumes:
    db-data: # Docker 自动创建和管理这个卷
  ```

- 短语法关键注意事项：
  1. 相对路径（如 `./html`）仅支持**本地容器运行时**（比如 Docker Desktop），部署到远程平台会报错，且相对路径必须以 `.` 或 `..` 开头；
  2. bind 挂载时，如果主机路径不存在，Compose 会**自动创建该目录**（为了兼容旧版），如果不想创建，需用长语法；
  3. SELinux 选项（`z`/`Z`）仅在开启 SELinux 的系统（如 CentOS）生效，其他系统会忽略。

##### （2）长语法（精细控制，适合定制化场景）
通过键值对配置更多细节，弥补短语法的不足，核心字段如下：

| 一级字段       | 子字段                | 作用说明                                                                 |
|----------------|-----------------------|--------------------------------------------------------------------------|
| `type`         | -                     | 挂载类型：`volume`/`bind`/`tmpfs`/`npipe` 等                             |
| `source`       | -                     | 挂载源：bind 挂载填主机路径，volume 挂载填卷名，tmpfs 无需填             |
| `target`       | -                     | 容器内的挂载路径（必填）                                                 |
| `read_only`    | -                     | 是否只读：`true`/`false`（替代短语法的 `ro`/`rw`）                       |
| `bind`         | `propagation`         | bind 挂载的传播模式（如 `rprivate`/`shared`，平台相关）                  |
|                | `create_host_path`    | 是否自动创建主机路径：`true`（默认）/`false`（解决短语法自动创建的问题）  |
|                | `selinux`             | SELinux 选项：`z`/`Z`                                                    |
| `volume`       | `nocopy`              | 创建卷时是否禁用从容器复制数据：`true`/`false`（默认复制）               |
|                | `subpath`             | 挂载卷内的子路径（而非卷根目录）                                         |
| `tmpfs`        | `size`                | 临时卷大小（字节，如 `10485760` 或 `10m`）                              |
|                | `mode`                | 临时卷文件权限（八进制，如 `0755`）                                      |

- 长语法示例（覆盖核心场景）：
  ```yaml
  services:
    backend:
      image: example/backend
      volumes:
        # 1. 命名卷挂载（定制化）
        - type: volume
          source: db-data       # 关联 top-level 的 db-data 卷
          target: /data         # 容器内路径
          read_only: false      # 读写
          volume:
            nocopy: true        # 不复制容器内原有数据到卷
            subpath: sub        # 只挂载卷内的 sub 子目录
        
        # 2. bind 挂载（禁止自动创建主机路径）
        - type: bind
          source: /var/run/postgres/postgres.sock
          target: /var/run/postgres/postgres.sock
          read_only: true
          bind:
            create_host_path: false # 主机路径不存在则报错，不自动创建
        
        # 3. 临时内存卷（大小 100MB，权限 0700）
        - type: tmpfs
          target: /tmp/cache
          tmpfs:
            size: 100000000     # 100MB
            mode: 0700          # 仅所有者可读可写可执行
  volumes:
    db-data:
  ```

#### 3. 核心使用原则
- 数据持久化/容器间共享 → 用 `volume` 类型（Docker 管理，跨平台更稳定）；
- 开发时挂载代码/配置 → 用 `bind` 类型（直接映射主机文件）；
- 临时缓存/敏感数据 → 用 `tmpfs` 类型（不落地，更安全）。

---

### 二、`volumes_from` 字段详解
`volumes_from` 是**批量挂载卷**的简化方式，直接挂载「另一个服务/容器」的所有卷，无需重复定义 `volumes`。

#### 1. 基本语法
格式：`[service_name/container:container_name][:ro/rw]`
- `service_name`：Compose 中其他服务的名称；
- `container:container_name`：非 Compose 管理的外部容器名称（需加 `container:` 前缀）；
- `ro/rw`：可选，指定访问模式（默认 `rw` 读写）。

#### 2. 示例
```yaml
services:
  web:
    image: nginx
    volumes_from:
      - backend          # 挂载 backend 服务的所有卷（读写）
      - db:ro           # 挂载 db 服务的所有卷（只读）
      - container:my-external-container # 挂载外部容器 my-external-container 的所有卷（读写）
      - container:redis-container:rw    # 挂载外部容器 redis-container 的所有卷（显式读写）

  backend:
    image: backend-app
    volumes:
      - ./code:/app
      - backend-data:/data

  db:
    image: mysql
    volumes:
      - db-data:/var/lib/mysql

volumes:
  backend-data:
  db-data:
```

#### 3. 注意事项
- `volumes_from` 仅挂载卷，不共享容器的其他配置（如环境变量、端口）；
- 如果挂载的目标服务/容器不存在，Compose 启动会失败；
- 该字段偏老旧，新版 Compose 更推荐用 `volumes` 定义命名卷实现共享（更清晰可控）。

---

### 三、`working_dir` 字段详解
`working_dir` 用于**覆盖容器镜像默认的工作目录**（即 Dockerfile 中 `WORKDIR` 指令定义的目录），容器启动后，命令（`command`/`CMD`）会在这个目录下执行。

#### 1. 基本用法
```yaml
services:
  app:
    image: python:3.10
    working_dir: /app  # 覆盖镜像默认的 WORKDIR（比如 /usr/src/app）
    command: python main.py # 该命令会在 /app 目录下执行（等价于 cd /app && python main.py）
```

#### 2. 核心作用
- 无需修改镜像（重新构建），即可调整容器的工作目录；
- 配合 `command` 使用，让命令在指定目录下执行（比如运行代码时，不用写绝对路径）。

#### 3. 示例对比
| 配置方式                | 执行效果                                                                 |
|-------------------------|--------------------------------------------------------------------------|
| 镜像 `WORKDIR` 为 `/usr/src/app`，无 `working_dir` | `command: python main.py` → 执行 `/usr/src/app/main.py`                  |
| 加 `working_dir: /app`  | `command: python main.py` → 执行 `/app/main.py`                          |

---

### 总结
1. **volumes**：核心存储挂载字段，短语法简洁适合基础场景，长语法支持定制化（如禁止创建主机路径、设置临时卷大小）；`volume` 类型适合持久化，`bind` 适合开发挂载，`tmpfs` 适合临时数据；
2. **volumes_from**：批量挂载其他服务/容器的所有卷，简化配置但可控性低，新版更推荐用命名卷共享；
3. **working_dir**：覆盖容器默认工作目录，让命令在指定目录下执行，无需修改镜像。




### `network_mode` 
`network_mode` 用于**设置容器的网络模式**，直接决定容器如何接入网络（是用主机网络、禁用网络，还是复用其他容器的网络）。它是一个“全局”网络配置，优先级高于 `networks` 字段——一旦设置了 `network_mode`，就**不能再使用 `networks` 字段**（Compose 会直接报错）。

#### 1. 核心取值及含义
| 取值格式               | 作用说明                                                                 | 适用场景                     |
|------------------------|--------------------------------------------------------------------------|------------------------------|
| `network_mode: none`   | 禁用容器所有网络功能（容器无法访问外网/其他容器）                        | 离线任务、纯本地计算的容器   |
| `network_mode: host`   | 容器直接使用主机的网络栈（共享主机 IP/端口，无网络隔离）                 | 需要高性能网络、直接占用主机端口的场景（如监控工具） |
| `network_mode: service:{name}` | 复用指定服务（{name} 是 Compose 中其他服务名）的网络命名空间 | 让容器和目标服务共享网络（比如 sidecar 容器） |
| `network_mode: container:{name/id}` | 复用指定容器（{name/id} 是容器名/ID）的网络命名空间 | 复用非 Compose 管理的容器网络 |

#### 2. 示例
```yaml
services:
  # 示例1：使用主机网络（容器端口直接映射到主机，无需 ports 映射）
  monitor:
    image: prom/prometheus
    network_mode: host # 容器用主机的 IP 和端口，比如容器内 9090 端口直接对应主机 9090

  # 示例2：禁用网络（容器完全离线）
  offline-job:
    image: alpine
    command: sh -c "echo '离线计算' && sleep 10"
    network_mode: none

  # 示例3：复用其他服务的网络（sidecar 容器和主服务共享网络）
  app:
    image: my-app
    ports:
      - 8080:8080

  app-sidecar:
    image: my-sidecar
    network_mode: service:app # 复用 app 服务的网络，sidecar 可直接访问 app 的 8080 端口
```

#### 3. 关键注意事项
- `network_mode` 和 `networks` 互斥：同时配置会触发 Compose 语法错误；
- `host` 模式的限制：仅支持 Linux 平台（Docker Desktop for Windows/macOS 中 `host` 模式不生效，因为是虚拟机环境）；
- `service:{name}` 模式：目标服务必须和当前服务在同一个 Compose 配置中，且先启动。

---

### `networks` 
`networks` 是 Compose 中**最常用的网络配置字段**，用于将服务容器连接到指定的自定义网络（需在 top-level `networks` 中定义），实现容器间的网络隔离、自定义 IP、别名访问等精细化控制。

#### 1. 基础用法：连接自定义网络
##### （1）隐式默认网络
如果不配置 `networks` 字段，Compose 会自动将服务连接到**默认网络**（`default`），以下两种配置等价：
```yaml
# 简化写法（隐式连接 default 网络）
services:
  some-service:
    image: foo

# 等价写法（显式连接 default 网络）
services:
  some-service:
    image: foo
    networks:
      default: {}

networks:
  default: {} # Compose 自动创建，可省略
```

##### （2）显式连接自定义网络
通过 `networks` 字段指定服务要连接的多个自定义网络，需在 top-level 定义这些网络：
```yaml
services:
  frontend:
    image: nginx
    networks:
      - front-tier # 连接 front-tier 网络
      - back-tier  # 连接 back-tier 网络

networks:
  front-tier: {} # 定义前端网络（默认 bridge 驱动）
  back-tier: {}  # 定义后端网络
```

#### 2. `networks` 的核心子属性（精细化配置）
以下子属性需针对每个网络单独配置（格式：`网络名: { 子属性 }`），覆盖不同的网络需求：

##### （1）`aliases`：网络别名（容器访问的备用主机名）
- 作用：给服务在指定网络上设置**备用主机名**，同网络的其他容器可通过「服务名」或「别名」访问该服务；
- 特点：别名是**网络级别的**（同一服务在不同网络可设不同别名），且多个容器/服务可共享同一别名（但解析结果不保证）。

- 示例：
  ```yaml
  services:
    backend:
      image: mysql
      networks:
        back-tier:
          aliases:
            - database # back-tier 网络中，可通过 database 访问 backend
        admin:
          aliases:
            - mysql    # admin 网络中，可通过 mysql 访问 backend

    frontend:
      image: nginx
      networks:
        - back-tier # 可通过 backend 或 database 访问后端

    monitoring:
      image: prometheus
      networks:
        - admin # 可通过 backend 或 mysql 访问后端

  networks:
    back-tier: {}
    admin: {}
  ```

##### （2）`interface_name`：指定容器内的网络接口名
- 作用：强制容器连接该网络时使用指定的接口名（如 `eth0`），保证接口命名的一致性；
- 示例：
  ```yaml
  services:
    backend:
      image: alpine
      command: ip link show # 查看网络接口
      networks:
        back-tier:
          interface_name: eth0 # 强制该网络的接口名为 eth0
  networks:
    back-tier: {}
  ```
  执行结果会显示 `eth0` 接口（而非默认随机命名）。

##### （3）`ipv4_address`/`ipv6_address`：静态 IP 地址
- 作用：给容器分配固定的 IPv4/IPv6 地址（需先在 top-level 网络中配置对应的子网）；
- 注意：网络必须配置 `ipam`（IP 地址管理），且静态 IP 需在子网范围内。

- 示例：
  ```yaml
  services:
    frontend:
      image: nginx
      networks:
        front-tier:
          ipv4_address: 172.16.238.10 # 固定 IPv4
          ipv6_address: 2001:3984:3989::10 # 固定 IPv6

  networks:
    front-tier:
      ipam: # IP 地址管理配置
        driver: default # 默认驱动
        config:
          - subnet: "172.16.238.0/24" # IPv4 子网
          - subnet: "2001:3984:3989::/64" # IPv6 子网
  ```

##### （4）`link_local_ips`：链路本地 IP
- 作用：指定链路本地 IP 列表（属于特殊子网，由运维手动管理，和平台架构相关）；
- 示例：
  ```yaml
  services:
    app:
      image: busybox
      command: top
      networks:
        app_net:
          link_local_ips:
            - 57.123.22.11
            - 57.123.22.13
  networks:
    app_net:
      driver: bridge
  ```

##### （5）`mac_address`：MAC 地址
- 作用：设置容器连接该网络时使用的 MAC 地址（网卡物理地址），用于网络身份识别；
- 示例：
  ```yaml
  services:
    app:
      image: alpine
      networks:
        app_net:
          mac_address: 02:42:ac:11:65:43 # 自定义 MAC 地址
  networks:
    app_net: {}
  ```

##### （6）`driver_opts`：驱动选项
- 作用：给网络驱动传递自定义键值对参数（参数依赖驱动类型，需参考驱动文档）；
- 示例：
  ```yaml
  services:
    app:
      networks:
        app_net:
          driver_opts:
            foo: "bar" # 自定义驱动参数
            baz: 1
  networks:
    app_net:
      driver: bridge # 桥接驱动
  ```

##### （7）`gw_priority`：默认网关优先级
- 作用：多个网络时，**优先级最高（数值大）** 的网络会被选为容器的默认网关；默认值为 0；
- 示例：
  ```yaml
  services:
    app:
      image: busybox
      command: top
      networks:
        app_net_1: {} # 优先级 0
        app_net_2:
          gw_priority: 1 # 优先级 1（最高），作为默认网关
        app_net_3: {} # 优先级 0
  networks:
    app_net_1: {}
    app_net_2: {}
    app_net_3: {}
  ```

##### （8）`priority`：网络连接顺序优先级
- 作用：指定 Compose 连接网络的**顺序**（数值大的先连接）；默认值为 0；
- 注意：
  - 不影响默认网关（网关由 `gw_priority` 控制）；
  - 不决定容器内接口名（如 `eth0`/`eth1`）的顺序；
  - 如果容器支持全局 `mac_address`，会应用到 `priority` 最高的网络。

- 示例：
  ```yaml
  services:
    app:
      image: busybox
      command: top
      networks:
        app_net_1:
          priority: 1000 # 最先连接
        app_net_2: {}    # 优先级 0（次之）
        app_net_3:
          priority: 100  # 最后连接
  networks:
    app_net_1: {}
    app_net_2: {}
    app_net_3: {}
  ```

---

### 总结
1. **network_mode**：全局网络模式配置，支持 `host`/`none`/`service:{name}` 等，和 `networks` 互斥，适合简单的网络复用/禁用场景；
2. **networks**：精细化网络配置，支持连接自定义网络，并可通过子属性设置别名、静态 IP、MAC 地址、网关优先级等；
3. 核心区别：`network_mode` 是“粗粒度”的网络模式选择，`networks` 是“细粒度”的网络连接和属性配置，二者不可同时使用。
