
你想全面掌握 `docker run` 命令的用法，包括核心功能、参数含义、使用场景和最佳实践，我会从基础到进阶，帮你系统梳理这个命令的所有关键知识点，让你能灵活应对不同的容器运行需求。

### 一、`docker run` 核心概念
`docker run` 是 Docker 中**创建并启动容器**的核心命令（如果指定的镜像本地不存在，会自动从仓库拉取）。简单来说，它的作用是基于指定的镜像，创建一个可运行的容器实例，并执行指定的命令。

#### 基本语法
```bash
docker run [OPTIONS] IMAGE[:TAG|@DIGEST] [COMMAND] [ARG...]
```
- `IMAGE`：指定要运行的镜像（必填），可带标签（如 `nginx:1.25`）或摘要。
- `COMMAND`：容器启动后要执行的命令（可选，覆盖镜像默认命令）。
- `ARG`：传递给启动命令的参数。
- `OPTIONS`：核心配置参数（下文重点讲解）。

### 二、核心参数详解（按使用场景分类）
#### 1. 基础运行参数（最常用）
| 参数 | 缩写 | 作用 | 示例 |
|------|------|------|------|
| `--name` | - | 给容器指定自定义名称（避免随机名称） | `docker run --name my-nginx nginx` |
| `--rm` | - | 容器停止后**自动删除**（适合临时测试容器） | `docker run --rm nginx` |
| `-it` | - | 组合参数：`-i`（交互模式）+`-t`（分配伪终端），用于交互式容器 | `docker run -it ubuntu bash` |
| `-d` | - | 后台运行容器（守护进程模式） | `docker run -d --name my-nginx nginx` |

#### 2. 网络配置参数
| 参数 | 缩写 | 作用 | 示例 |
|------|------|------|------|
| `-p` | - | 端口映射（主机端口:容器端口），可多次映射 | `docker run -d -p 8080:80 nginx` |
| `-P` | - | 随机映射容器暴露的所有端口（大写P） | `docker run -d -P nginx` |
| `--network` | - | 指定容器所属网络 | `docker run -d --network my-network nginx` |
| `--hostname` | `-h` | 设置容器主机名 | `docker run -h nginx-host nginx` |
| `--add-host` | - | 添加主机名映射（类似/etc/hosts） | `docker run --add-host db:192.168.1.100 nginx` |

#### 3. 数据持久化参数
| 参数 | 缩写 | 作用 | 示例 |
|------|------|------|------|
| `-v` | `--volume` | 挂载数据卷/目录（主机路径:容器路径[:权限]） | `docker run -d -v /host/html:/usr/share/nginx/html nginx` |
| `--mount` | - | 更灵活的挂载方式（推荐生产环境） | `docker run -d --mount type=bind,source=/host/html,target=/usr/share/nginx/html nginx` |
| `-v` | - | 挂载命名卷（自动创建） | `docker run -d -v nginx-data:/usr/share/nginx/html nginx` |

#### 4. 资源限制参数（生产环境必备）
| 参数 | 作用 | 示例 |
|------|------|------|
| `--memory`/`-m` | 限制内存（如1g、512m） | `docker run -d -m 1g nginx` |
| `--cpus` | 限制CPU核心数（如0.5、2） | `docker run -d --cpus 2 nginx` |
| `--cpu-shares` | 设置CPU权重（相对值，默认1024） | `docker run -d --cpu-shares 2048 nginx` |
| `--restart` | 容器退出后的重启策略 | `docker run -d --restart always nginx` |

#### 5. 权限与环境参数
| 参数 | 缩写 | 作用 | 示例 |
|------|------|------|------|
| `-e` | `--env` | 设置环境变量，可多次使用 | `docker run -d -e MYSQL_ROOT_PASSWORD=123456 mysql` |
| `--env-file` | - | 从文件加载环境变量 | `docker run -d --env-file .env nginx` |
| `--user` | `-u` | 指定运行容器的用户（UID/GID） | `docker run -u 1000:1000 nginx` |
| `--privileged` | - | 赋予容器特权（慎用，接近主机权限） | `docker run --privileged -d nginx` |

#### 6. 进阶参数
| 参数 | 作用 | 示例 |
|------|------|------|
| `--log-driver` | 指定日志驱动（如json-file、syslog） | `docker run -d --log-driver json-file nginx` |
| `--healthcheck` | 设置健康检查 | `docker run -d --healthcheck "curl -f http://localhost || exit 1" nginx` |
| `--entrypoint` | 覆盖镜像的默认入口命令 | `docker run --entrypoint bash nginx` |
| `--platform` | 指定容器运行的平台（跨架构） | `docker run --platform linux/arm64 nginx` |

### 三、完整使用示例
#### 示例1：基础交互式运行（临时测试）
```bash
# 启动ubuntu容器，进入bash交互模式，退出后自动删除容器
docker run --rm -it ubuntu bash
# 进入容器后可执行命令，比如：
# apt update && apt install -y curl
# 执行完后输入 exit 退出，容器会自动删除
```

#### 示例2：后台运行Web服务（端口映射+重启策略）
```bash
# 启动nginx容器，后台运行，映射8080端口，设置自动重启，指定自定义名称
docker run \
  -d \
  --name my-nginx \
  -p 8080:80 \
  --restart unless-stopped \  # 除非手动停止，否则一直重启
  -v /host/nginx/conf:/etc/nginx/conf.d \  # 挂载配置目录
  -v /host/nginx/html:/usr/share/nginx/html \  # 挂载网页目录
  -e TZ=Asia/Shanghai \  # 设置时区
  nginx:1.25
```

#### 示例3：运行数据库容器（环境变量+数据卷）
```bash
# 启动MySQL 8.0容器，设置密码，挂载数据卷，限制资源
docker run \
  -d \
  --name my-mysql \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=MyPass123! \
  -e MYSQL_DATABASE=my_db \
  -v mysql-data:/var/lib/mysql \  # 命名卷持久化数据
  --memory 2g \  # 限制2G内存
  --cpus 1 \  # 限制1个CPU核心
  --restart on-failure:3 \  # 失败时重启，最多3次
  mysql:8.0
```

#### 示例4：覆盖默认命令运行容器
```bash
# 启动nginx容器，但不运行默认的nginx服务，而是执行ps命令查看进程
docker run --rm nginx ps aux
```

### 四、常见问题与最佳实践
#### 1. 容器启动后立即退出？
- 原因：容器没有前台运行的进程（后台运行的命令会导致容器退出）。
- 解决：
    - 交互式容器加 `-it` 参数。
    - 后台容器确保启动命令是前台运行（如 `nginx -g "daemon off;"`）。

#### 2. 端口映射失败（端口被占用）？
- 查看主机端口占用：`netstat -tulpn | grep 8080`。
- 更换映射端口：`-p 8081:80`。

#### 3. 数据卷挂载权限问题？
- 检查主机目录权限，确保容器用户有访问权限。
- 临时解决：挂载时指定权限 `--mount type=bind,source=/host/path,target=/container/path,readonly`（只读）。

#### 4. 生产环境最佳实践
- 必加 `--name`：方便管理容器。
- 必加 `--restart`：保证容器异常退出后自动重启。
- 必加资源限制：`--memory`、`--cpus`，避免容器占用过多主机资源。
- 使用数据卷/挂载：避免容器删除后数据丢失。
- 避免 `--privileged`：除非特殊需求，否则不要赋予特权。

### 总结
1. `docker run` 核心是**基于镜像创建并启动容器**，基础语法为 `docker run [OPTIONS] 镜像 [命令]`，`-d`（后台）、`-it`（交互）、`-p`（端口）、`-v`（挂载）是最常用参数。
2. 生产环境需重点关注：`--name`（命名）、`--restart`（重启策略）、资源限制（`--memory`/`--cpus`）、数据持久化（`-v`/`--mount`）。
3. 临时测试容器推荐加 `--rm`，避免产生无用容器；交互操作需加 `-it`，后台服务需加 `-d`。

### 关键点回顾
- 运行模式：`-d` 后台运行（守护进程），`-it` 交互式运行（适合调试）。
- 数据持久化：`-v` 是简化版挂载，`--mount` 是更灵活的生产级挂载方式。
- 容器可靠性：`--restart` 策略决定容器异常后的行为，生产环境常用 `unless-stopped` 或 `on-failure:N`。




