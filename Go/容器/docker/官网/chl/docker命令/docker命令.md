


### 一、核心入口（官方命令参考）
- 总入口：https://docs.docker.com/reference/
- CLI 命令大全：https://docs.docker.com/engine/reference/commandline/docker/
- 你之前问的两个命令直达页：
    - `docker build`：https://docs.docker.com/engine/reference/commandline/build/
    - `docker run`：https://docs.docker.com/engine/reference/commandline/run/

### 二、官网讲解的内容特点
1. **权威完整**：覆盖所有参数、选项、语法格式，与 Docker 最新版本同步。
2. **结构清晰**
    - 命令语法（Synopsis）
    - 所有参数（Flags）的详细说明（含默认值、适用场景）
    - 扩展说明（如上下文、缓存、网络、挂载等）
    - 完整示例（Example）
3. **版本标注**：明确哪些参数在哪个 Docker 版本新增/废弃。
4. **可检索**：支持按命令名、关键词快速定位。

### 三、快速使用方式
#### 1. 网页查询（推荐）
直接访问对应命令页，比如查 `docker run`：
https://docs.docker.com/engine/reference/commandline/run/

#### 2. 终端快速查（本地）
```bash
# 查看所有 docker 命令
docker help
# 查看单个命令详情（如 run）
docker run --help
# 查看 build 详情
docker build --help
```





### 一、Docker 基础命令（环境/信息）
这类命令用于查看 Docker 状态、版本、系统信息，是日常排查的基础。

| 命令 | 作用 | 示例 |
|------|------|------|
| `docker version` | 查看 Docker 客户端/服务端版本 | `docker version`（简洁版）<br>`docker version --format '{{.Server.Version}}'`（仅查服务端版本） |
| `docker info` | 查看 Docker 系统信息（镜像数、容器数、驱动等） | `docker info`（完整信息）<br>`docker info --format '{{.DockerRootDir}}'`（仅查镜像存储目录） |
| `docker --help` | 查看所有命令的帮助文档 | `docker --help`（全局帮助）<br>`docker run --help`（指定命令的帮助） |
| `docker system df` | 查看 Docker 磁盘使用情况（镜像/容器/卷占用） | `docker system df`（简洁版）<br>`docker system df -v`（详细版，含每个镜像/容器的占用） |
| `docker system prune` | 清理无用资源（停止的容器、悬空镜像、无用网络） | `docker system prune`（交互式清理）<br>`docker system prune -af`（强制清理，无需确认） |

### 二、镜像（Image）相关命令（核心）
镜像是容器的「模板」，这类命令用于管理镜像的拉取、构建、查看、删除等。

#### 1. 查看镜像
```bash
# 列出本地所有镜像
docker images
# 等价于 docker image ls
docker image ls

# 过滤镜像（如只看 ubuntu 相关）
docker images ubuntu

# 查看镜像的详细信息（含构建历史、配置）
docker inspect ubuntu:22.04

# 查看悬空镜像（无标签、无容器引用的镜像，标记为 <none>）
docker images -f "dangling=true"
```

#### 2. 拉取/推送镜像
```bash
# 从仓库拉取镜像（默认 Docker Hub）
docker pull ubuntu:22.04  # 指定版本
docker pull ubuntu        # 拉取 latest 标签（不推荐，版本不稳定）

# 推送镜像到仓库（需先登录）
docker login  # 登录 Docker Hub（输入用户名密码）
docker tag ubuntu:22.04 your-username/ubuntu:22.04  # 打标签（必须带用户名）
docker push your-username/ubuntu:22.04
```

#### 3. 构建镜像（从 Dockerfile）
```bash
# 基础用法：在当前目录构建，标签为 my-app:v1
docker build -t my-app:v1 .

# 指定 Dockerfile 路径（非当前目录）
docker build -t my-app:v1 -f /path/to/Dockerfile .

# 构建时传递构建参数（--build-arg）
docker build -t my-app:v1 --build-arg APP_PORT=8080 .

# 使用 Buildx 构建（多平台镜像，如同时构建 amd64/arm64）
docker buildx build -t my-app:v1 --platform linux/amd64,linux/arm64 .
```

#### 4. 删除/清理镜像
```bash
# 删除单个镜像（可指定 ID 或 名称:标签）
docker rmi ubuntu:22.04
# 等价于 docker image rm ubuntu:22.04

# 强制删除（即使有容器引用该镜像）
docker rmi -f ubuntu:22.04

# 删除所有悬空镜像
docker rmi $(docker images -f "dangling=true" -q)

# 删除所有未被容器使用的镜像
docker image prune -a  # -a 表示包括未被引用的镜像（非悬空）
```

### 三、容器（Container）相关命令（核心）
容器是镜像的「运行实例」，这类命令覆盖容器的创建、启动、停止、删除、进入等全生命周期。

#### 1. 查看容器
```bash
# 列出正在运行的容器
docker ps

# 列出所有容器（运行中 + 已停止）
docker ps -a

# 只显示容器 ID（常用于批量操作）
docker ps -aq

# 查看容器详细信息（IP、挂载、网络等）
docker inspect container-id/container-name
```

#### 2. 创建/启动容器
```bash
# 最常用：创建并启动容器（run = create + start）
docker run [参数] 镜像名 [容器内执行的命令]

# 核心参数说明（必记）：
# -d：后台运行（守护进程）
# -p：端口映射（宿主端口:容器端口）
# -v：目录挂载（宿主目录:容器目录）
# --name：指定容器名称（不指定则随机生成）
# -e：设置环境变量
# -it：交互式运行（结合 bash/sh 使用，可进入容器）
# --rm：容器停止后自动删除（适合临时测试）

# 示例 1：后台运行 nginx，映射 8080 端口，命名为 my-nginx
docker run -d -p 8080:80 --name my-nginx nginx

# 示例 2：交互式运行 ubuntu，进入 bash，停止后自动删除
docker run -it --rm ubuntu bash

# 示例 3：挂载宿主目录到容器，设置环境变量
docker run -d -v /host/data:/container/data -e MYSQL_ROOT_PASSWORD=123456 --name my-mysql mysql:8.0
```

#### 3. 启动/停止/重启容器
```bash
# 启动已停止的容器
docker start container-id/container-name

# 停止运行中的容器（优雅停止，发送 SIGTERM 信号）
docker stop container-id/container-name

# 强制停止容器（发送 SIGKILL 信号，立即终止）
docker kill container-id/container-name

# 重启容器
docker restart container-id/container-name

# 批量停止所有运行中的容器
docker stop $(docker ps -q)
```

#### 4. 进入容器（执行命令/交互）
```bash
# 方式 1：进入容器的交互式终端（推荐，支持窗口调整）
docker exec -it container-id/container-name bash

# 方式 2：老版本用法（不推荐，窗口调整可能出问题）
docker attach container-id/container-name

# 示例：在容器外执行容器内的命令（无需进入）
docker exec my-nginx ls /usr/share/nginx/html  # 查看 nginx 静态文件目录
```

#### 5. 容器日志/资源监控
```bash
# 查看容器日志（实时输出）
docker logs -f container-id/container-name

# 查看最后 100 行日志
docker logs --tail 100 container-id/container-name

# 查看容器资源使用情况（CPU、内存、网络、磁盘）
docker stats  # 实时监控所有容器
docker stats container-id/container-name  # 监控指定容器
```

#### 6. 删除容器
```bash
# 删除单个已停止的容器
docker rm container-id/container-name

# 强制删除运行中的容器
docker rm -f container-id/container-name

# 批量删除所有已停止的容器
docker rm $(docker ps -aq)

# 批量删除所有容器（包括运行中）
docker rm -f $(docker ps -aq)
```

### 四、网络（Network）相关命令
Docker 网络用于容器间通信、容器与宿主通信，默认有 bridge（桥接）、host（宿主网络）、none（无网络）三种模式。

```bash
# 列出所有网络
docker network ls

# 创建自定义网络（推荐，容器间可通过名称通信）
docker network create my-network

# 将容器连接到自定义网络
docker network connect my-network my-nginx

# 断开容器与网络的连接
docker network disconnect my-network my-nginx

# 删除无用网络
docker network prune

# 删除指定网络（需先断开所有容器）
docker network rm my-network
```

### 五、数据卷（Volume）相关命令
数据卷是 Docker 持久化数据的核心方式（比直接挂载宿主目录更安全、易管理）。

```bash
# 列出所有数据卷
docker volume ls

# 创建数据卷
docker volume create my-volume

# 查看数据卷详细信息（存储路径等）
docker volume inspect my-volume

# 删除指定数据卷
docker volume rm my-volume

# 删除所有未被使用的数据卷
docker volume prune
```

### 六、Docker Compose 命令（批量管理容器）
Docker Compose 基于 `docker-compose.yml` 文件批量启动/停止/管理多个容器，是生产环境常用工具。

```bash
# 启动所有容器（后台运行）
docker compose up -d

# 停止所有容器
docker compose down

# 查看容器状态
docker compose ps

# 查看日志
docker compose logs -f

# 重建镜像并启动容器（适用于 Dockerfile 修改后）
docker compose up -d --build

# 执行容器内命令
docker compose exec web bash
```

### 总结
Docker 命令的核心逻辑可归纳为 3 个关键点：
1. **分层记忆**：先掌握「镜像-容器」核心命令，再扩展网络/数据卷/Compose，符合 Docker 从「模板」到「实例」再到「集群」的使用逻辑；
2. **参数优先级**：`docker run` 的参数（如 `-p`/`-v`/`-e`）是容器运行的关键，记住 `-it`（交互）、`-d`（后台）、`--rm`（临时）三个高频参数；
3. **批量操作技巧**：利用 `$(docker ps -aq)` 等命令批量处理容器/镜像，结合 `prune` 系列命令快速清理无用资源。

如果需要针对某个具体场景（如构建镜像、容器调试、持久化数据）的命令详解，可以告诉我，我会补充更针对性的示例和注意事项。





