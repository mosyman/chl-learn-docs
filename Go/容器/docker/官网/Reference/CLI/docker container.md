
你希望我系统整理并解释 `docker container run`（别名 `docker run`）这个核心命令，我会先明确它的核心定位，再按「高频常用参数 + 功能分类参数」的结构拆解，结合实战示例和使用场景，让你既能理解每个参数的作用，也知道在什么场景下该用什么参数。

### 一、`docker run` 核心定位
`docker run` 是 Docker 最核心的命令之一，作用是**从指定镜像创建并启动一个新容器**，包含三个关键动作：
1. 检查本地是否有指定镜像，没有则自动拉取（可通过 `--pull` 控制）；
2. 基于镜像创建容器（配置网络、存储、权限等）；
3. 启动容器并执行指定命令（默认执行镜像的 `ENTRYPOINT`/`CMD`）。

核心语法：`docker run [OPTIONS] IMAGE [COMMAND] [ARG...]`  
（注：`COMMAND` 和 `ARG` 用于覆盖镜像默认的启动命令，比如 `docker run nginx echo "hello"` 会执行 `echo` 而非启动 nginx）

### 二、高频核心参数（日常开发 80% 场景会用到）
按“功能 + 示例 + 关键说明”整理，优先级从高到低：

| 参数 | 简写 | 默认值 | 核心作用 | 实战示例 & 关键说明 |
|------|------|--------|----------|---------------------|
| `--name` | - | 随机生成 | 给容器分配自定义名称（便于管理，替代随机ID） | `docker run --name my-nginx nginx:alpine`<br>**说明**：<br>1. 名称唯一，重复会报错；<br>2. 后续可通过名称操作容器（如 `docker stop my-nginx`）。 |
| `--detach` | `-d` | 前台运行 | 后台运行容器（守护进程模式），返回容器ID | `docker run -d --name my-nginx nginx:alpine`<br>**说明**：<br>1. 不加 `-d` 会前台运行，终端关闭则容器停止；<br>2. 后台运行后，用 `docker logs 容器名` 查看输出。 |
| `--publish` | `-p` | 不暴露端口 | 将容器端口映射到宿主机端口（宿主机端口:容器端口） | **基础用法**：`docker run -d -p 8080:80 nginx:alpine`（宿主机8080 → 容器80）<br>**高级用法**：<br>- 指定宿主机IP：`-p 192.168.1.100:8080:80`；<br>- UDP端口：`-p 53:53/udp`；<br>- 端口范围：`-p 8080-8085:80-85`。<br>**说明**：<br>1. 容器内的端口需先在镜像中 `EXPOSE`（或用 `--expose` 临时暴露）；<br>2. 避免端口冲突（宿主机端口已被占用则报错）。 |
| `--publish-all` | `-P` | 关闭 | 自动将容器所有 `EXPOSE` 的端口映射到宿主机随机端口 | `docker run -d -P nginx:alpine`<br>**说明**：<br>1. 用 `docker ps` 查看随机映射的端口；<br>2. 适合临时测试，不适合生产（端口不固定）。 |
| `--volume` | `-v` | 不挂载 | 将宿主机目录/文件挂载到容器内（数据持久化） | **基础用法**：<br>`docker run -d -v /host/nginx.conf:/etc/nginx/nginx.conf nginx:alpine`<br>（宿主机文件 → 容器文件，覆盖容器内文件）<br>**目录挂载**：<br>`docker run -d -v /host/html:/usr/share/nginx/html nginx:alpine`<br>**说明**：<br>1. 格式：`宿主机路径:容器路径[:权限]`（如 `:ro` 只读）；<br>2. 容器内修改挂载目录的内容，宿主机同步更新；<br>3. 避免权限问题：可加 `:z`（如 `-v /host/html:/usr/share/nginx/html:z`）。 |
| `--env` | `-e` | 镜像默认 | 设置容器内的环境变量 | `docker run -d -e MYSQL_ROOT_PASSWORD=123456 -e MYSQL_DATABASE=test mysql:8`<br>**说明**：<br>1. 可多次使用 `-e` 设置多个变量；<br>2. 支持引用宿主机变量：`-e "PATH=$PATH"`；<br>3. 用 `docker exec 容器名 env` 查看容器内环境变量。 |
| `--env-file` | - | 无 | 从文件读取环境变量（批量设置） | `# 先创建 env.txt 文件`<br>`MYSQL_ROOT_PASSWORD=123456`<br>`MYSQL_DATABASE=test`<br>`# 运行容器`<br>`docker run -d --env-file env.txt mysql:8`<br>**说明**：<br>1. 文件每行格式：`KEY=VALUE`，空行/注释（#）会被忽略；<br>2. 优先级：`-e` > `--env-file` > 镜像默认。 |
| `--network` | - | 默认 `bridge` | 指定容器接入的网络 | `docker run -d --network my-app-net --name my-nginx nginx:alpine`<br>**说明**：<br>1. 接入自定义网络后，容器可通过名称访问同网络内的其他容器；<br>2. 支持特殊网络：`--network host`（共享宿主机网络）、`--network none`（无网络）。 |
| `--restart` | - | `no` | 容器退出后的重启策略（保障服务可用性） | **常用值**：<br>- `no`：不重启（默认）；<br>- `always`：总是重启（无论退出码）；<br>- `on-failure[:次数]`：非0退出码时重启（如 `on-failure:3` 最多重启3次）；<br>- `unless-stopped`：除非手动停止，否则一直重启。<br>**示例**：<br>`docker run -d --restart unless-stopped nginx:alpine`<br>**说明**：<br>1. `always`/`unless-stopped` 会在Docker重启后自动启动容器；<br>2. 生产环境推荐 `unless-stopped`。 |
| `--rm` | - | 关闭 | 容器停止后自动删除（包括匿名卷） | `docker run --rm -it ubuntu:22.04 bash`<br>**说明**：<br>1. 适合临时测试容器（避免残留无用容器）；<br>2. `-d` 模式也可使用，但需手动停止才会删除；<br>3. 不会删除命名卷（仅删除匿名卷）。 |
| `--interactive` | `-i` | 关闭 | 保持标准输入（STDIN）打开，即使未附加终端 | 通常和 `-t` 配合使用：<br>`docker run -it ubuntu:22.04 bash`<br>**说明**：<br>1. 不加 `-i` 无法在容器内输入命令；<br>2. 退出容器（`exit`）后，容器会停止（如需后台保留，加 `-d`）。 |
| `--tty` | `-t` | 关闭 | 分配伪终端（PTY），让容器有交互界面 | 同上，`-it` 是交互式容器的标配：<br>`docker run -it --rm centos:7 /bin/bash`<br>**说明**：<br>1. `-t` 提供终端模拟，`-i` 保证输入，缺一不可；<br>2. 退出终端用 `Ctrl+D` 或 `exit`。 |
| `--pull` | - | `missing` | 控制拉取镜像的策略 | **可选值**：<br>- `missing`：本地无镜像则拉取（默认）；<br>- `always`：总是拉取最新版本（覆盖本地）；<br>- `never`：仅用本地镜像，无则报错。<br>**示例**：<br>`docker run --pull always nginx:latest`（强制拉取最新nginx）。 |

### 三、功能分类参数（按场景整理）
#### 1. 资源限制类（控制CPU/内存/IO）
| 参数 | 作用 | 示例 | 说明 |
|------|------|------|------|
| `--cpus` | 限制容器使用的CPU核心数 | `--cpus 2`（最多用2核） | 支持小数：`--cpus 1.5`（1.5核） |
| `--memory`/`-m` | 限制容器内存 | `--memory 1G`（最多1GB内存） | 单位：b/k/m/g，超出会触发OOM |
| `--memory-swap` | 限制内存+交换分区 | `--memory-swap 2G` | 需配合 `--memory` 使用，`-1` 表示无限制 |
| `--blkio-weight` | 限制块设备IO权重 | `--blkio-weight 500`（默认500，范围10-1000） | 权重越高，IO优先级越高 |

#### 2. 网络配置类（进阶网络设置）
| 参数 | 作用 | 示例 | 说明 |
|------|------|------|------|
| `--ip` | 给容器分配固定IPv4地址 | `--ip 172.20.128.2` | 需先创建带 `--subnet` 的自定义网络 |
| `--dns` | 设置容器DNS服务器 | `--dns 8.8.8.8 --dns 114.114.114.114` | 覆盖容器默认DNS（默认继承宿主机） |
| `--hostname`/`-h` | 设置容器主机名 | `--hostname my-container` | 容器内 `hostname` 命令会返回该值 |
| `--add-host` | 添加主机名-IP映射 | `--add-host mysql-server:172.20.128.3` | 等价于修改容器内 `/etc/hosts` |
| `--network-alias` | 给容器加网络别名 | `--network-alias web --network-alias nginx` | 同网络内的容器可通过别名访问该容器 |

#### 3. 权限/安全类
| 参数 | 作用 | 示例 | 说明 |
|------|------|------|------|
| `--privileged` | 给容器扩展权限（接近宿主机root） | `--privileged` | 谨慎使用！容器可访问宿主机所有设备（如 `/dev`） |
| `--user`/`-u` | 指定容器运行的用户 | `--user 1000:1000`（UID:GID）或 `--user root` | 避免容器以root运行（提升安全性） |
| `--read-only` | 将容器根文件系统设为只读 | `--read-only` | 仅允许写入挂载的卷，提升安全性 |
| `--cap-add`/`--cap-drop` | 添加/移除Linux能力 | `--cap-add NET_ADMIN`（允许容器修改网络）<br>`--cap-drop ALL`（移除所有能力） | 精细化控制容器权限，替代 `--privileged` |

#### 4. 存储/挂载类（进阶）
| 参数 | 作用 | 示例 | 说明 |
|------|------|------|------|
| `--mount` | 更灵活的挂载方式（替代 `-v`） | `--mount type=bind,source=/host/html,target=/usr/share/nginx/html,readonly` | 支持更多类型（bind/volume/tmpfs），语义更清晰 |
| `--tmpfs` | 挂载临时文件系统（容器内，重启丢失） | `--tmpfs /tmp:size=100M` | 适合临时存储，速度快（内存级） |
| `--shm-size` | 设置 `/dev/shm` 大小 | `--shm-size 256M` | 默认64M，适合需要大共享内存的应用（如数据库） |

#### 5. 健康检查类
| 参数 | 作用 | 示例 | 说明 |
|------|------|------|------|
| `--health-cmd` | 定义健康检查命令 | `--health-cmd "curl -f http://localhost || exit 1"` | 检测容器是否正常运行 |
| `--health-interval` | 检查间隔 | `--health-interval 30s` | 每30秒检查一次 |
| `--health-timeout` | 检查超时时间 | `--health-timeout 5s` | 单次检查超过5秒则判定失败 |
| `--health-retries` | 失败重试次数 | `--health-retries 3` | 连续3次失败则标记容器为不健康 |

#### 6. 其他实用参数
| 参数 | 作用 | 示例 | 说明 |
|------|------|------|------|
| `--entrypoint` | 覆盖镜像默认的ENTRYPOINT | `docker run --entrypoint sh nginx:alpine` | 强制容器执行 `sh` 而非默认的nginx启动命令 |
| `--gpus` | 给容器分配GPU（需宿主机有GPU） | `--gpus all`（分配所有GPU）或 `--gpus "device=0,1"` | 用于AI/机器学习场景，需安装NVIDIA Docker运行时 |
| `--workdir`/`-w` | 设置容器内的工作目录 | `--workdir /app` | 等价于容器内执行 `cd /app` |
| `--label` | 给容器添加元数据标签 | `--label "env=prod" --label "owner=dev"` | 用于筛选容器：`docker ps --filter label=env=prod` |

### 四、典型场景实战示例
#### 场景1：启动一个带持久化的MySQL容器
```bash
docker run -d \
  --name my-mysql \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=123456 \
  -e MYSQL_DATABASE=my_db \
  -v /host/mysql/data:/var/lib/mysql \
  -v /host/mysql/conf:/etc/mysql/conf.d \
  --restart unless-stopped \
  mysql:8
```
**关键配置**：
- `-v` 挂载数据目录和配置目录，避免数据丢失；
- `--restart` 保证Docker重启后MySQL自动启动；
- `-e` 设置必要的环境变量。

#### 场景2：启动一个交互式临时容器（测试用）
```bash
docker run -it --rm \
  --name temp-ubuntu \
  --network my-app-net \
  ubuntu:22.04 \
  bash
```
**关键配置**：
- `-it` 交互式终端，`--rm` 退出后自动删除；
- 接入自定义网络 `my-app-net`，可测试网络内容器通信；
- 执行 `bash` 进入容器命令行。

#### 场景3：启动一个资源受限的Nginx容器
```bash
docker run -d \
  --name limited-nginx \
  -p 8080:80 \
  --cpus 1 \
  --memory 512M \
  --memory-swap 1G \
  --restart on-failure:3 \
  nginx:alpine
```
**关键配置**：
- 限制1核CPU、512M内存（+512M交换分区）；
- 非0退出码时最多重启3次，避免无限重启。

### 五、核心注意事项
1. **容器命名**：名称唯一，若需重复使用同名容器，先执行 `docker rm 容器名`；
2. **端口映射**：宿主机端口需未被占用，生产环境建议固定端口（避免 `-P` 随机端口）；
3. **数据持久化**：务必用 `-v`/`--mount` 挂载关键目录，否则容器删除后数据丢失；
4. **权限控制**：避免滥用 `--privileged`，优先用 `--cap-add`/`--cap-drop` 精细化控制权限；
5. **重启策略**：生产环境优先用 `unless-stopped`，避免 `always` 导致无用容器一直重启；
6. **镜像拉取**：`--pull always` 会拉取最新镜像，可能导致版本不一致，生产环境建议指定镜像标签（如 `nginx:1.25-alpine`）。

### 总结
1. **核心逻辑**：`docker run` = 拉取镜像 + 创建容器 + 启动容器，`OPTIONS` 是配置容器的核心；
2. **高频参数**：`-d`（后台）、`--name`（命名）、`-p`（端口）、`-v`（挂载）、`-e`（环境变量）、`--network`（网络）、`--restart`（重启策略）是日常开发的核心；
3. **使用原则**：
    - 测试场景：用 `--rm` 避免残留容器；
    - 生产场景：指定镜像标签、配置持久化、设置重启策略、限制资源；
    - 安全场景：避免root运行（`--user`）、只读文件系统（`--read-only`）、精细化权限。

掌握这些参数和场景后，你可以灵活应对绝大多数容器启动的需求，兼顾易用性、安全性和稳定性。



## `--name`：为容器指定自定义名称
##### 功能定位
`--name` 是 `docker run` 最常用的参数之一，用于给容器分配**唯一的自定义名称**（替代 Docker 自动生成的随机名称，如 `vibrant_cannon`），核心价值是「易识别、易操作、支持DNS解析」。

##### 核心特性
| 特性 | 说明 |
|------|------|
| 唯一性 | 同一 Docker 节点上容器名称必须唯一，重复创建会报错 |
| 可操作性 | 后续可通过名称直接操作容器（如 `docker stop test`），无需记随机容器ID |
| DNS解析 | 接入自定义桥接网络的容器，可通过名称互相访问（Docker 内置DNS解析） |

##### 实战示例
###### 示例1：创建命名容器（后台运行）
```bash
# 创建名为 test 的 nginx 容器，后台运行
docker run --name test -d nginx:alpine

# 查看容器（NAME 列显示 test）
docker ps
# 输出：CONTAINER ID   IMAGE          COMMAND                  STATUS        PORTS     NAMES
#       4bed76d3ad42   nginx:alpine   "/docker-entrypoint.…"   Up 10 seconds  80/tcp    test
```

###### 示例2：通过名称操作容器
```bash
# 停止容器
docker stop test

# 删除容器
docker rm test
```

###### 示例3：自定义网络中通过名称DNS访问
```bash
# 1. 创建自定义网络 mynet
docker network create mynet

# 2. 启动命名容器并接入该网络
docker run --name test --net mynet -d nginx:alpine

# 3. 同网络内的容器可通过名称 ping 通（DNS解析）
docker run --net mynet busybox:latest ping test
# 输出：PING test (172.18.0.2): 56 data bytes
#       64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.073 ms
```

##### 关键注意事项
- 名称仅支持字母、数字、下划线、连字符、点，且不能以点开头；
- 若需复用名称，需先删除旧容器（`docker rm 旧名称`）；
- 默认 `bridge` 网络中，容器名称无法被DNS解析（仅自定义网络支持）。

---

## `--cidfile`：将容器ID写入指定文件
##### 功能定位
`--cidfile` 用于让 Docker 将新创建容器的**唯一ID** 写入指定的文件中，核心价值是「自动化场景下快速获取容器ID」（类似程序的PID文件）。

##### 核心特性
| 特性 | 说明 |
|------|------|
| 文件创建规则 | Docker 会尝试创建指定文件，若文件已存在则直接报错（避免覆盖） |
| 文件生命周期 | `docker run` 执行完成后，Docker 会关闭该文件（文件内容保留容器ID） |
| 自动化适配 | 适合脚本/自动化工具读取容器ID，无需解析 `docker ps` 输出 |

##### 实战示例
```bash
# 创建容器并将ID写入 /tmp/docker_test.cid 文件
docker run --cidfile /tmp/docker_test.cid ubuntu echo "test"

# 查看文件内容（即容器ID）
cat /tmp/docker_test.cid
# 输出：f9f874980e9d7f09936e89999876a5e87f8a7b8c9d0e1f2a3b4c5d6e7f8a9b0c
```

##### 典型使用场景
- 自动化部署脚本：通过读取CID文件快速定位刚创建的容器；
- 日志/监控系统：基于CID文件关联容器与日志、监控数据；
- 批量操作：批量读取CID文件，对多个容器执行统一操作。

##### 关键注意事项
- 指定的文件路径需有写入权限（否则Docker报错）；
- 文件仅存储容器ID，无其他内容，可直接作为参数传入 `docker` 命令（如 `docker stop $(cat /tmp/docker_test.cid)`）。

---

## `--pid`：设置容器的PID命名空间
##### 功能定位
`--pid` 用于控制容器的**进程命名空间（PID Namespace）**，决定容器能否看到宿主机/其他容器的进程，核心价值是「进程隔离/共享」，满足调试、监控等特殊场景需求。

##### 核心参数值
| 参数值 | 作用 | 适用场景 |
|--------|------|----------|
| 默认值（空） | 容器拥有独立的PID命名空间，仅能看到自身进程 | 常规场景（进程隔离） |
| `host` | 共享宿主机的PID命名空间，容器内可看到宿主机所有进程 | 宿主机进程监控/调试 |
| `container:<容器名/ID>` | 共享指定容器的PID命名空间，可看到该容器的所有进程 | 容器内进程调试（如strace） |

##### 实战示例
###### 示例1：共享宿主机PID命名空间（运行htop监控宿主机）
```bash
# 1. 启动容器，共享宿主机PID命名空间，交互式运行
docker run --rm -it --pid=host alpine

# 2. 容器内安装htop（进程监控工具）
/ # apk add --quiet htop

# 3. 运行htop（可看到宿主机所有进程，而非容器内进程）
/ # htop
```

###### 示例2：共享其他容器的PID命名空间（调试容器进程）
```bash
# 1. 启动目标容器（nginx）
docker run --rm --name my-nginx -d nginx:alpine

# 2. 启动调试容器，共享my-nginx的PID命名空间
# --cap-add SYS_PTRACE：授予调试权限
# --security-opt seccomp=unconfined：关闭seccomp限制（调试需要）
docker run --rm -it --pid=container:my-nginx \
  --cap-add SYS_PTRACE \
  --security-opt seccomp=unconfined \
  alpine

# 3. 容器内安装strace（进程追踪工具）
/ # apk add strace

# 4. 追踪my-nginx容器的1号进程（nginx主进程）
/ # strace -p 1
# 输出：strace: Process 1 attached
```

##### 关键注意事项
- `--pid=host` 会打破容器进程隔离，谨慎在生产环境使用；
- 调试其他容器时，需添加 `--cap-add SYS_PTRACE` 和 `--security-opt seccomp=unconfined` 才能正常使用strace等调试工具；
- PID命名空间共享是单向的：调试容器能看到目标容器进程，目标容器看不到调试容器进程。

### 二、核心场景总结
| 参数 | 核心场景 | 核心价值 |
|------|----------|----------|
| `--name` | 1. 日常容器管理（易识别/操作）<br>2. 自定义网络内容器通信 | 替代随机ID，支持DNS解析 |
| `--cidfile` | 1. 自动化脚本获取容器ID<br>2. 日志/监控系统关联容器 | 避免解析docker ps输出，提升自动化效率 |
| `--pid` | 1. 宿主机进程监控（--pid=host）<br>2



## `--tmpfs`：挂载临时文件系统（内存级）
#### 1. 核心定义
`--tmpfs` 用于在容器内创建**临时文件系统（tmpfs）**，该文件系统存储在宿主机内存（或swap）中，而非磁盘，容器停止/删除后数据会**完全丢失**。
- 本质：Linux 原生的 `tmpfs` 挂载，Docker 只是封装了对应的系统调用；
- 核心特点：读写速度极快（内存级）、数据不持久、无磁盘IO消耗。

#### 2. 语法与参数
**基本语法**：`--tmpfs <容器内路径>:<挂载选项>`
- 容器内路径：必须是绝对路径（如 `/run`），不存在则自动创建；
- 挂载选项：和 Linux `mount -t tmpfs -o` 的选项完全一致，常用选项如下：

| 选项 | 作用 |
|------|------|
| `rw` | 读写权限（默认） |
| `ro` | 只读权限 |
| `noexec` | 禁止在该目录执行可执行文件（提升安全性） |
| `nosuid` | 禁止SetUID/SetGID权限（提升安全性） |
| `size` | 限制 tmpfs 大小（如 `size=65536k` 即64MB），超出会报错 |

#### 3. 实战示例
```bash
# 启动容器，在 /run 目录挂载 tmpfs，限制大小64MB，禁止执行/SetUID
docker run -d --tmpfs /run:rw,noexec,nosuid,size=65536k my_image
```
**关键说明**：
- 容器内 `/run` 目录的所有数据都存储在内存中，读写速度远快于磁盘；
- 若容器内写入数据超过 `size=65536k`，会触发“磁盘空间不足”错误；
- 容器停止后，`/run` 内的所有数据会被清空，宿主机无任何残留。

#### 4. 适用场景
- 临时存储高频读写的临时文件（如进程PID文件、缓存数据）；
- 敏感数据存储（避免数据落盘，降低泄露风险）；
- 需极致IO性能的场景（如临时计算、批量处理）。

#### 5. 关键注意事项
- `tmpfs` 仅存在于容器生命周期内，**绝对不能用于持久化数据**；
- 挂载选项需用逗号分隔，无空格（如 `rw,noexec` 而非 `rw, noexec`）；
- 若不指定 `size`，默认受宿主机内存限制（可能占用大量内存）；
- `tmpfs` 不支持跨容器共享，仅当前容器可访问。



## `-v/--volume`：绑定挂载（宿主机目录/文件 → 容器）
#### 1. 核心定义
`-v`（全称 `--volume`）是 Docker 最常用的挂载方式，核心作用是**将宿主机的目录/文件绑定到容器内指定路径**，实现「宿主机 ↔ 容器」的数据双向同步，是数据持久化、文件共享的核心手段。

#### 2. 核心语法
**基础格式**：`-v <宿主机路径>:<容器内路径>:<权限（可选）>`
- 宿主机路径：支持绝对路径（如 `/root/data`）、相对路径（Docker 23+，如 `./content`）；
- 容器内路径：必须是绝对路径；
- 权限（可选）：`ro`（只读）/`rw`（读写，默认）。

#### 3. 实战示例拆解
##### 示例1：挂载当前目录到容器（绝对路径）
```bash
# 将宿主机当前目录（$(pwd)）挂载到容器同路径，设置为工作目录，执行pwd
docker run -v $(pwd):$(pwd) -w $(pwd) -i -t ubuntu pwd
```
**执行逻辑**：
- `-v $(pwd):$(pwd)`：宿主机当前目录 ↔ 容器内当前目录（双向同步）；
- `-w $(pwd)`：将容器的工作目录设为该路径；
- `pwd`：容器内执行 `pwd`，输出和宿主机 `pwd` 相同的路径。

##### 示例2：挂载相对路径（Docker 23+ 新特性）
```bash
# 将宿主机当前目录下的 content 目录，挂载到容器 /content 路径
docker run -v ./content:/content -w /content -i -t ubuntu pwd
```
**关键说明**：
- `./content` 是宿主机相对路径（相对于执行 `docker run` 的目录）；
- 容器内 `pwd` 输出 `/content`，宿主机 `./content` 内的文件会同步到容器 `/content`。

##### 示例3：宿主机目录不存在时自动创建
```bash
# 挂载宿主机 /doesnt/exist 到容器 /foo（宿主机该目录不存在）
docker run -v /doesnt/exist:/foo -w /foo -i -t ubuntu bash
```
**关键特性**：
- Docker 会自动在宿主机创建 `/doesnt/exist` 目录（权限为当前执行用户）；
- 容器内 `/foo` 目录会同步宿主机 `/doesnt/exist` 的内容（初始为空）；
- 容器内写入 `/foo` 的数据，会持久化到宿主机 `/doesnt/exist` 目录。

#### 4. 核心特性
| 特性 | 说明 |
|------|------|
| 双向同步 | 宿主机/容器任意一方修改挂载目录的文件，另一方立即同步； |
| 数据持久化 | 容器停止/删除后，宿主机目录的数据仍保留； |
| 自动创建目录 | 宿主机挂载路径不存在时，Docker 自动创建（仅目录，文件不会自动创建）； |
| 权限控制 | 可通过 `:ro` 限制容器只读（如 `-v ./data:/data:ro`）； |

#### 5. 适用场景
- 数据持久化（如数据库数据目录、应用配置文件）；
- 宿主机与容器共享文件（如代码目录、日志目录）；
- 批量部署（将宿主机配置文件挂载到容器，统一管理）。

#### 6. 关键注意事项
- 相对路径仅支持 Docker 23 及以上版本，低版本需用绝对路径（如 `$(pwd)/content`）；
- 若挂载宿主机文件（而非目录），文件必须提前存在（Docker 不会自动创建文件）；
- 权限问题：容器内进程的UID/GID可能和宿主机不一致，导致权限拒绝（可通过 `:z` 或 `:Z` 解决，如 `-v ./data:/data:z`）；
- 避免挂载宿主机敏感目录（如 `/etc`、`/root`），防止容器篡改宿主机文件。

### 三、`--tmpfs` vs `-v` 核心差异对比
| 维度 | `--tmpfs` | `-v/--volume` |
|------|-----------|---------------|
| 存储位置 | 宿主机内存（/swap） | 宿主机磁盘 |
| 数据持久化 | 容器停止/删除后丢失 | 永久保留在宿主机 |
| IO性能 | 内存级，极快 | 依赖磁盘性能，较慢 |
| 跨容器共享 | 不支持 | 支持（多个容器挂载同一宿主机目录） |
| 大小限制 | 可通过 `size` 限制，默认受内存限制 | 受宿主机磁盘空间限制 |
| 适用场景 | 临时高频读写、敏感数据、无持久化需求 | 数据持久化、文件共享、配置挂载 |

### 四、典型使用场景总结
#### 1. 用 `--tmpfs` 的场景
- 容器内需要临时存储PID文件（如 `/run` 目录）；
- 应用需要临时缓存数据，且无需持久化；
- 测试环境中临时存储大量临时文件，避免占用磁盘。

#### 2. 用 `-v` 的场景
- 部署MySQL容器，挂载宿主机 `/data/mysql` 到容器 `/var/lib/mysql`（数据持久化）；
- 开发环境中，挂载本地代码目录到容器，修改代码后容器实时生效；
- 挂载宿主机 `/etc/nginx/conf.d` 到容器，统一管理Nginx配置。

### 总结
1. **`--tmpfs`**：内存级临时挂载，速度快但数据不持久，适合临时高频读写场景；
2. **`-v`**：磁盘级绑定挂载，数据持久化且支持双向同步，是数据共享/持久化的核心方式；
3. 核心选择原则：
   - 无需持久化 → `--tmpfs`；
   - 需要持久化/跨容器共享 → `-v`；
   - 敏感数据临时使用 → `--tmpfs`（避免落盘）；
   - 配置/代码共享 → `-v`（支持修改同步）。



## `--read-only` 只读文件系统
#### 1. 功能定位
`--read-only` 是 `docker run` 的核心安全参数，作用是将容器的**根文件系统（/）挂载为只读模式**——默认情况下容器可任意写入根文件系统（如修改系统文件、写入临时数据），启用该参数后，容器无法在根文件系统的任何位置写入数据，仅能向「显式挂载的卷（`-v`/`--mount`）」中写入，核心价值是：
- 提升容器安全性：防止恶意程序/误操作篡改容器系统文件；
- 控制数据写入路径：强制所有写入操作仅发生在指定卷中，便于数据管理；
- 符合“不可变基础设施”理念：容器镜像本身只读，仅卷可写。

#### 2. 核心逻辑
`--read-only` + `-v` 是“组合拳”：
- `--read-only`：锁死根文件系统，禁止所有默认写入；
- `-v`：显式声明“可写目录”，仅这些目录允许容器写入数据；
- 若仅用 `--read-only` 不挂载卷，容器几乎无法执行任何写入操作（包括创建临时文件）。

### 二、Linux 系统下的实战示例与解析
#### 示例1：基础只读容器 + 可写卷
```bash
# 核心命令：只读根文件系统，仅 /icanwrite 目录可写
docker run --read-only -v /icanwrite busybox touch /icanwrite/here
```
**执行逻辑拆解**：
1. `--read-only`：容器根目录（/）设为只读，无法在 `/tmp`、`/var` 等默认目录创建文件；
2. `-v /icanwrite`：创建一个匿名卷并挂载到容器 `/icanwrite` 目录（匿名卷默认可写）；
3. `touch /icanwrite/here`：在可写卷目录中创建文件，操作成功；
4. 若尝试执行 `touch /tmp/here`：会报错 `touch: /tmp/here: Read-only file system`（根文件系统只读）。

#### 示例2：挂载Docker套接字实现容器管理宿主机Docker
```bash
docker run -t -i \
  -v /var/run/docker.sock:/var/run/docker.sock \  # 挂载Docker守护进程套接字
  -v /path/to/static-docker-binary:/usr/bin/docker \  # 挂载静态Docker二进制文件
  busybox sh
```
**核心用途与风险**：
- 作用：容器内可执行 `docker ps`、`docker run` 等命令，直接管理宿主机的Docker守护进程（相当于给容器“宿主机Docker管理员权限”）；
- 结合 `--read-only` 优化：
  ```bash
  # 更安全的配置：只读根文件系统，仅必要目录可写
  docker run -t -i --read-only \
    -v /var/run/docker.sock:/var/run/docker.sock:ro \  # 套接字设为只读（进一步限制）
    -v /path/to/static-docker-binary:/usr/bin/docker:ro \
    -v /tmp:/tmp  # 仅临时目录可写，满足Docker命令运行需求
    busybox sh
  ```
- 风险提示：该配置赋予容器极高权限，生产环境需严格限制，仅在必要场景使用。

### 三、Windows 系统下卷挂载的特殊规则（重点）
Windows容器的卷/绑定挂载逻辑与Linux差异极大，核心规则和示例如下：

#### 1. 基础语法
Windows下挂载路径需使用**Windows风格路径**（反斜杠 `\` 或正斜杠 `/`），支持两种格式：
- 格式1：`宿主机路径:容器内路径`（如 `c:\foo:c:\dest`）；
- 格式2：`宿主机路径:容器内驱动器`（如 `c:\foo:d:`，将宿主机 `c:\foo` 挂载为容器 `d:` 驱动器）。

**成功示例**：
```powershell
# 示例1：挂载 c:\foo 到容器 c:\dest，读取文件
PS C:\> docker run -v c:\foo:c:\dest microsoft/nanoserver cmd /s /c type c:\dest\somefile.txt
Contents of file

# 示例2：挂载 c:\foo 到容器 d: 驱动器，读取文件
PS C:\> docker run -v c:\foo:d: microsoft/nanoserver cmd /s /c type d:\somefile.txt
Contents of file
```

#### 2. 关键限制（失败场景）
Windows容器挂载**必须满足以下条件**，否则会失败，以下是典型错误示例及原因：

| 错误命令 | 失败原因 |
|----------|----------|
| `docker run -v z:\foo:c:\dest ...` | 不支持挂载网络驱动器（`z:` 是映射的远程共享目录） |
| `docker run -v \\uncpath\to\directory:c:\dest ...` | 不支持UNC路径（`\\remotemachine\share` 这类网络共享路径） |
| `docker run -v c:\foo\somefile.txt:c:\dest ...` | 不支持挂载单个文件，仅支持目录挂载 |
| `docker run -v c:\foo:c: ...` | 容器内 `c:` 驱动器是根文件系统，不允许直接挂载覆盖 |
| `docker run -v c:\foo:c:\existing-directory-with-contents ...` | 容器内目标目录若已存在且非空，不允许挂载（仅支持空目录/不存在的目录/其他驱动器） |

#### 3. 核心规则总结
- 挂载源：仅支持**本地目录**（不支持网络驱动器、UNC路径、单个文件）；
- 挂载目标：
   - 可以是容器内**不存在的目录**（Docker自动创建）；
   - 可以是容器内**空目录**；
   - 可以是容器内**非C盘的驱动器**（如 `d:`、`e:`）；
   - 禁止覆盖容器 `c:` 驱动器，禁止挂载到非空目录。

### 四、关键注意事项（跨平台通用）
#### 1. Linux 只读容器的优化配置
- 必要时挂载临时目录：部分应用需要写入 `/tmp` 目录，需显式挂载卷：
  ```bash
  docker run --read-only -v /tmp busybox  # 匿名卷挂载 /tmp，允许写入
  ```
- 卷权限精细化：对非必要的挂载卷设为只读（`ro`），降低风险：
  ```bash
  # 配置文件只读，数据目录可写
  docker run --read-only \
    -v ./config:/etc/app:ro \
    -v ./data:/var/lib/app \
    my-app
  ```

#### 2. Windows 挂载的额外注意事项
- 路径分隔符：PowerShell中推荐用反斜杠 `\`，CMD中可混用 `\` 和 `/`；
- 权限问题：Windows容器挂载的目录权限继承宿主机权限，需确保Docker服务账户有访问权限；
- 驱动器冲突：容器内默认只有 `c:` 驱动器，挂载时建议使用 `d:`/`e:` 等未占用的驱动器名。

#### 3. 生产环境最佳实践
- 优先使用命名卷而非绑定挂载：命名卷（`docker volume create`）管理更规范，跨平台兼容性更好；
- 只读容器必挂载必要卷：至少挂载应用数据目录、临时目录（`/tmp`），避免应用因无法写入崩溃；
- 限制卷权限：非写入目录一律设为 `ro`，最小化权限暴露。

### 总结
1. **`--read-only` 核心**：锁死容器根文件系统，仅 `-v` 挂载的卷可写，提升安全性，需配合卷挂载使用；
2. **Linux 挂载要点**：支持文件/目录挂载，可通过 `ro`/`rw` 控制权限，挂载Docker套接字需注意权限风险；
3. **Windows 挂载核心规则**：
   - 仅支持本地目录挂载，不支持网络路径/单个文件；
   - 目标路径仅允许空目录/不存在的目录/非C盘驱动器；
   - 路径需用Windows风格，禁止覆盖容器C盘。



## `--mount` 

### 一、`--mount` 核心定位与优势
#### 1. 核心作用
`--mount` 是 Docker 推荐的挂载参数，用于在容器内挂载**卷（volume）**、**宿主机目录（bind mount）**、`tmpfs` 临时文件系统，功能完全覆盖 `-v/--volume`，且语法更清晰、语义更明确。

#### 2. 核心优势（对比 `-v`）
| 维度 | `--mount` | `-v/--volume` |
|------|-----------|---------------|
| 语法结构 | 键值对形式（`key=value`），语义清晰，易读易维护 | 单字符串拼接（`src:dst:opt`），参数多了易混淆 |
| 功能扩展性 | 支持更多细分参数（如 `readonly`、`consistency`），未来会持续新增 | 仅支持基础参数，扩展性差 |
| 错误提示 | 参数错误时提示更明确 | 参数错误可能静默失败或报错模糊 |
| 官方态度 | 推荐使用，是未来的主流方式 | 无废弃计划，但不推荐新场景使用 |

### 二、`--mount` 核心语法
#### 1. 基础格式
```bash
--mount type=<类型>,key1=value1,key2=value2,...
```
- **必选参数**：`type`（挂载类型），取值为 `volume`（命名卷/匿名卷）、`bind`（宿主机目录绑定）、`tmpfs`（临时文件系统）；
- **可选参数**：根据 `type` 不同，支持的键值对不同，核心通用参数如下：

| 挂载类型 | 核心参数 | 说明 |
|----------|----------|------|
| `volume` | `target`（或 `dst`） | 容器内的挂载路径（绝对路径） |
|          | `source`（或 `src`） | 卷名称（不指定则创建匿名卷） |
|          | `readonly`（或 `ro`） | 设为 `true` 则卷只读（默认 `false`） |
| `bind`   | `target`（或 `dst`） | 容器内的挂载路径（绝对路径） |
|          | `source`（或 `src`） | 宿主机的目录/文件路径（绝对路径，Docker 23+ 支持相对路径） |
|          | `readonly`（或 `ro`） | 设为 `true` 则绑定挂载只读（默认 `false`） |
| `tmpfs`  | `target`（或 `dst`） | 容器内的挂载路径（绝对路径） |
|          | `size` | tmpfs 大小限制（如 `size=64M`） |
|          | `tmpfs-mode` | tmpfs 权限（如 `tmpfs-mode=1777`） |

> 注：`target`/`dst`、`source`/`src` 是等价的，可互换使用（推荐用 `target`/`source`，语义更清晰）。

#### 2. 语法规则
- 参数之间用**逗号分隔**，无空格（如 `type=bind,src=/data,dst=/data`）；
- 布尔型参数（如 `readonly`）：设为 `true`/`false`，或简写为 `readonly`（等价于 `readonly=true`）；
- 路径参数：`source`（宿主机路径）/`target`（容器路径）必须是绝对路径（`tmpfs` 同理）。

### 三、实战示例与解析
#### 示例1：挂载卷（volume）+ 只读容器
```bash
docker run --read-only --mount type=volume,target=/icanwrite busybox touch /icanwrite/here
```
**执行逻辑拆解**：
1. `--read-only`：容器根文件系统设为只读，禁止默认写入；
2. `--mount type=volume,target=/icanwrite`：
   - `type=volume`：创建匿名卷（未指定 `source`，Docker 自动生成卷名）；
   - `target=/icanwrite`：将该匿名卷挂载到容器 `/icanwrite` 目录；
3. `touch /icanwrite/here`：在可写的卷目录中创建文件，操作成功（根文件系统只读不影响卷）；
4. 等价的 `-v` 命令：`docker run --read-only -v /icanwrite busybox touch /icanwrite/here`。

**扩展：命名卷挂载**
```bash
# 先创建命名卷
docker volume create my-volume

# 挂载命名卷到容器
docker run --mount type=volume,source=my-volume,target=/data busybox
```
- `source=my-volume`：指定使用已创建的命名卷 `my-volume`；
- 等价的 `-v` 命令：`docker run -v my-volume:/data busybox`。

#### 示例2：绑定挂载（bind）宿主机目录
```bash
docker run -t -i --mount type=bind,src=/data,dst=/data busybox sh
```
**执行逻辑拆解**：
1. `type=bind`：声明为绑定挂载（宿主机目录 ↔ 容器目录）；
2. `src=/data`：宿主机的 `/data` 目录（必须存在，否则报错）；
3. `dst=/data`：容器内的 `/data` 目录（不存在则自动创建）；
4. 宿主机 `/data` 和容器 `/data` 双向同步数据，默认可读写；
5. 等价的 `-v` 命令：`docker run -t -i -v /data:/data busybox sh`。

**扩展1：只读绑定挂载**
```bash
# 宿主机目录只读挂载到容器
docker run --mount type=bind,src=/data,dst=/data,readonly=true busybox
# 等价简写（省略=true）
docker run --mount type=bind,src=/data,dst=/data,readonly busybox
# 等价 -v 命令：docker run -v /data:/data:ro busybox
```

**扩展2：相对路径绑定挂载（Docker 23+）**
```bash
# 宿主机当前目录下的 content 目录 → 容器 /content
docker run --mount type=bind,src=./content,dst=/content busybox
# 等价 -v 命令：docker run -v ./content:/content busybox
```

#### 示例3：挂载 tmpfs 临时文件系统
```bash
# 挂载 tmpfs 到 /run，限制大小64MB，权限1777
docker run --mount type=tmpfs,target=/run,size=64M,tmpfs-mode=1777 busybox
# 等价的 --tmpfs 命令：docker run --tmpfs /run:size=65536k,mode=1777 busybox
```
- `size=64M`：限制 tmpfs 最大占用64MB内存；
- `tmpfs-mode=1777`：设置 tmpfs 目录权限为 1777（所有用户可读写执行）；
- 数据仅存在于内存，容器停止后丢失。

### 四、`--mount` vs `-v` 核心对比（重点）
为了让你更清晰地选择，以下是核心场景的等价命令对比：

| 场景 | `--mount` 命令 | `-v` 命令 |
|------|---------------|-----------|
| 匿名卷挂载 | `--mount type=volume,target=/data` | `-v /data` |
| 命名卷挂载 | `--mount type=volume,source=my-vol,target=/data` | `-v my-vol:/data` |
| 宿主机目录绑定 | `--mount type=bind,src=/host/data,dst=/data` | `-v /host/data:/data` |
| 只读绑定挂载 | `--mount type=bind,src=/host/data,dst=/data,readonly` | `-v /host/data:/data:ro` |
| tmpfs 挂载 | `--mount type=tmpfs,target=/run,size=64M` | `--tmpfs /run:size=65536k` |

### 五、关键注意事项
#### 1. 语法易错点
- 参数之间**不能加空格**：错误写法 `--mount type=bind, src=/data, dst=/data`（逗号后有空格），正确写法 `--mount type=bind,src=/data,dst=/data`；
- 布尔参数格式：`readonly` 等价于 `readonly=true`，但不能写 `readonly=1`（会报错）；
- 路径必须是绝对路径：`--mount type=bind,src=./data,dst=/data` 仅 Docker 23+ 支持，低版本需用绝对路径 `src=$(pwd)/data`。

#### 2. 功能差异点
- `--mount` 支持 `consistency` 参数（仅 macOS/Windows）：控制挂载的一致性（`consistent`/`cached`/`delegated`），优化跨平台性能；
- `--mount` 挂载绑定目录时，若宿主机目录不存在，**不会自动创建**（`-v` 会自动创建），需手动创建或检查；
- `--mount` 挂载命名卷时，若卷不存在，Docker 会自动创建（和 `-v` 一致）。

#### 3. 生产环境最佳实践
- 新脚本/新场景优先用 `--mount`：语法清晰，便于维护和扩展；
- 批量挂载时分组注释：复杂挂载命令建议分行注释，提升可读性：
  ```bash
  docker run \
    --mount type=volume,source=app-data,target=/var/lib/app \  # 应用数据卷
    --mount type=bind,src=/etc/app/config.yaml,dst=/etc/app/config.yaml,readonly \  # 配置文件只读
    --mount type=tmpfs,target=/tmp,size=32M \  # 临时目录
    my-app
  ```
- 优先使用命名卷而非匿名卷：命名卷（`source=my-vol`）便于管理，匿名卷易丢失。

### 总结
1. **`--mount` 核心价值**：语法清晰、语义明确、功能扩展强，是 Docker 推荐的挂载方式，完全覆盖 `-v`/`--tmpfs` 功能；
2. **核心语法**：`--mount type=<类型>,key=value`，按类型分为 `volume`（卷）、`bind`（绑定目录）、`tmpfs`（临时文件系统）；
3. **关键对比**：
   - 和 `-v` 功能等价，但 `--mount` 更易读、易维护；
   - `--mount` 挂载绑定目录时不会自动创建宿主机目录，需手动检查；
4. **使用原则**：新场景用 `--mount`，老脚本可保留 `-v`，批量/复杂挂载优先用 `--mount` 提升可维护性。


---

你想让我详细解释 Docker 中 `-p`、`--expose` 和 `-P` 这几个端口相关参数的用法和区别，我会从基础概念到实际示例，把这些内容讲得通俗易懂。

### 核心概念先理清
在解释具体命令前，先明确两个核心概念：
- **暴露（expose）**：仅声明容器内应用监听的端口，是“告知”Docker 容器用了哪个端口，不做实际的端口映射，外部无法访问。
- **发布（publish）**：把容器的端口映射到宿主机的端口，是“打通”宿主机和容器的网络通道，外部可以通过宿主机端口访问容器内应用。

---

## `-p` (--publish)：发布端口（映射宿主机-容器端口）
这是最常用的端口操作，核心是**将容器内的端口映射到宿主机的指定端口/IP**，让外部能通过宿主机访问容器应用。

#### 语法格式
`-p [宿主机IP]:[宿主机端口]:[容器端口]/[协议]`
各部分说明：
- `宿主机IP`：可选，不填默认绑定宿主机所有网卡（`0.0.0.0`），填 `127.0.0.1` 则仅本机可访问；
- `宿主机端口`：必填，宿主机上要对外开放的端口；
- `容器端口`：必填，容器内应用实际监听的端口；
- `协议`：可选，支持 `tcp`（默认）、`udp`、`sctp`。

#### 示例解析：`docker run -p 127.0.0.1:80:8080/tcp nginx:alpine`
- 容器内：Nginx 应用监听 `8080` 端口（TCP 协议）；
- 宿主机：仅本机（`127.0.0.1`）的 `80` 端口会映射到容器的 `8080` 端口；
- 访问效果：只有在宿主机上访问 `http://127.0.0.1:80`，才能打到容器内的 Nginx 服务；其他机器无法访问（因为绑定了 `127.0.0.1`）。

#### 常见变体
- `docker run -p 80:80 nginx:alpine`：不指定 IP 和协议，默认绑定宿主机所有网卡（`0.0.0.0`），TCP 协议，宿主机 80 端口映射容器 80 端口（外部机器可通过宿主机 IP:80 访问）；
- `docker run -p 8080:80/udp nginx:alpine`：指定 UDP 协议，宿主机 8080（UDP）映射容器 80（UDP）。

#### 重要注意点
如果不指定宿主机 IP（如 `-p 80:80`），Docker 会默认把端口绑定到宿主机的 `0.0.0.0`（所有网卡），这意味着**外网能通过宿主机的公网 IP 访问这个端口**，即使你用 UFW（Linux 防火墙）屏蔽了该端口也无效——因为 Docker 会直接修改系统的 iptables 规则，绕过 UFW 限制。

---

## `--expose`：暴露端口（仅声明，不映射）
`--expose` 仅用于**声明容器内应用使用的端口**，不会把这个端口映射到宿主机，也不会对外开放，核心作用是“文档化”（告诉使用者容器用了哪个端口），或配合 `-P` 使用。

#### 语法格式
`--expose [容器端口]/[协议]`

#### 示例解析：`docker run --expose 80 nginx:alpine`
- 仅声明容器内暴露 80 端口（默认 TCP），但宿主机没有任何端口映射到容器 80 端口；
- 访问效果：外部（包括宿主机）无法通过端口访问容器内的 80 端口，只能在容器内部或 Docker 网络内的其他容器访问。

#### 补充：Dockerfile 中的 `EXPOSE`
和 `--expose` 作用一致，只是在构建镜像时声明端口（如 `EXPOSE 80`），运行容器时无需再写 `--expose 80` 也能识别。

---

### 3. `-P` (--publish-all)：发布所有暴露的端口（随机映射）
`-P` 是“批量发布”，核心是**将容器中所有“暴露的端口”（Dockerfile 的 `EXPOSE` 或 `--expose` 声明的端口）随机映射到宿主机的随机端口**。

#### 示例解析：`docker run -P nginx:alpine`
- Nginx 的官方镜像 Dockerfile 中声明了 `EXPOSE 80`（暴露 80 端口）；
- 运行 `-P` 后，Docker 会随机选一个宿主机端口（比如 32768），映射到容器的 80 端口；
- 查看映射关系：用 `docker ps` 或 `docker port <容器ID>` 可看到具体的宿主机随机端口。

#### 关键规则
- `-P` 只映射“显式暴露”的端口（Dockerfile `EXPOSE` 或 `--expose` 声明的），未声明的端口不会映射；
- 宿主机的随机端口范围由系统文件 `/proc/sys/net/ipv4/ip_local_port_range` 定义（通常是 32768-61000）；
- 随机端口默认绑定宿主机所有网卡（`0.0.0.0`）。

---

### 三者核心区别对比
| 命令/参数 | 核心作用 | 宿主机端口 | 外部可访问？ | 典型场景 |
|----------|----------|------------|--------------|----------|
| `-p 宿主机IP:宿端口:容器端口` | 精准映射宿主机-容器端口 | 自定义指定 | 是（取决于宿主机IP） | 生产环境固定端口访问（如 Nginx 80 端口） |
| `--expose 容器端口` | 仅声明容器端口（无映射） | 无 | 否 | 文档化容器端口，或配合 `-P` 使用 |
| `-P` | 批量映射所有暴露的端口 | 随机分配 | 是（绑定 0.0.0.0） | 测试环境快速验证，无需记固定端口 |

### 总结
1. **`-p` 是精准映射**：手动指定宿主机 IP/端口 映射容器端口，是生产中最常用的端口发布方式，注意不填 IP 会绑定所有网卡（外网可访问）；
2. **`--expose` 仅声明不映射**：仅告知 Docker 容器用了哪个端口，无实际网络打通效果，需配合 `-P` 才会被映射；
3. **`-P` 是批量随机映射**：自动把所有暴露的端口映射到宿主机随机端口，适合测试场景快速验证。



## `--pull` 

### 核心作用先明确
`--pull` 是 `docker run` 命令的参数，核心作用是**控制 Docker 在创建并运行容器前，是否以及如何拉取指定的镜像**。它本质上是给镜像拉取行为设定了“规则”，避免不必要的网络请求或意外使用旧镜像。


### `--pull` 的三种取值详解
| 取值 | 中文理解 | 核心行为 | 适用场景 |
|------|----------|----------|----------|
| `missing` (默认) | 仅缺失时拉取 | 1. 先检查本地镜像缓存是否有指定镜像；<br>2. 有 → 直接用本地镜像创建容器，不联网；<br>3. 无 → 自动从镜像仓库拉取，再创建容器。 | 绝大多数日常场景：<br>- 想复用本地已有的镜像，减少网络消耗；<br>- 本地镜像不存在时，自动补全，不影响使用。 |
| `never` | 永不拉取 | 1. 仅检查本地镜像缓存，完全不联网；<br>2. 有 → 用本地镜像创建容器；<br>3. 无 → 直接报错，容器创建失败。 | 离线环境/严格管控场景：<br>- 服务器无外网，只能用本地镜像；<br>- 防止误拉取外部镜像，保证容器仅使用本地验证过的镜像。 |
| `always` | 总是拉取 | 1. 无论本地是否有该镜像，都会先联网从仓库拉取最新版本；<br>2. 拉取完成后，用最新拉取的镜像创建容器（会覆盖本地同名旧镜像）。 | 需确保使用最新镜像的场景：<br>- 生产环境，必须用最新版本的镜像，避免本地旧镜像导致的问题；<br>- 团队协作，确保所有人使用同一版本的最新镜像。 |

---

### 实际示例解析
#### 1. 默认行为（`--pull=missing`，可省略不写）
```console
# 假设本地没有 nginx:alpine 镜像
$ docker run nginx:alpine
# 执行结果：Docker 发现本地缺失镜像，自动从仓库拉取，拉取完成后启动容器

# 再次执行同一命令（本地已有镜像）
$ docker run nginx:alpine
# 执行结果：直接用本地镜像启动容器，无网络请求，启动速度快
```

#### 2. `--pull=never`（永不拉取）
```console
# 本地没有 hello-world 镜像时执行
$ docker run --pull=never hello-world
# 执行结果：直接报错，容器创建失败
docker: Error response from daemon: No such image: hello-world:latest.

# 本地有 hello-world 镜像时执行
$ docker run --pull=never hello-world
# 执行结果：正常启动容器，完全不联网
```
> 关键：`never` 模式下，Docker 完全“断网”处理镜像，只认本地缓存，缺失即报错。

#### 3. `--pull=always`（总是拉取）
```console
# 本地已有旧版本的 nginx:alpine 镜像
$ docker run --pull=always nginx:alpine
# 执行结果：
# 1. Docker 先输出 "Pulling nginx:alpine..."，联网拉取最新版本；
# 2. 拉取完成后，覆盖本地旧镜像，用最新镜像启动容器。
```
> 注意：`always` 会覆盖本地同名镜像，如果你的本地镜像是自己构建的、未推送到仓库的版本，使用该参数会丢失本地镜像，需谨慎！

---

### 补充关键细节
1. **镜像缓存的判定**：Docker 是按“镜像名+标签”（如 `nginx:alpine`）判断本地是否有镜像的，如果标签是 `latest`，`always` 会拉取最新的 `latest` 版本，`missing` 则会复用本地已有的 `latest` 版本（哪怕仓库已有更新）。
2. **报错逻辑**：只有 `never` 会因为本地镜像缺失直接报错；`missing` 和 `always` 仅在“拉取失败”（如网络问题、镜像不存在）时才会报错。
3. **与 `docker pull` 的关系**：`--pull=always` 等价于先手动执行 `docker pull <镜像名>`，再执行 `docker run <镜像名>`；`--pull=missing` 等价于“有则不用拉，无则自动 `docker pull`”。

### 总结
1. `--pull` 核心控制容器创建前的镜像拉取规则，默认 `missing` 兼顾本地复用和自动补全；
2. `never` 适合离线/管控场景，仅用本地镜像，缺失即报错；
3. `always` 适合需强制使用最新镜像的场景，会联网拉取并覆盖本地旧镜像（注意本地自定义镜像的覆盖风险）。



## `-e`/`--env`/`--env-file` 

### 核心作用先明确
`-e`/`--env`/`--env-file` 是 `docker run` 命令中用来**给容器设置环境变量**的参数，既可以直接定义变量值，也可以从文件批量加载，还能覆盖镜像 Dockerfile 中定义的环境变量，是容器运行时配置的核心方式之一。

---

### 一、基础用法：`-e` / `--env`（直接设置环境变量）
`-e` 和 `--env` 是完全等价的（`-e` 是 `--env` 的简写），核心是**在命令行直接给容器设置单个/多个环境变量**，有两种使用形式：

#### 形式1：显式指定变量值（`VAR=value`）
直接定义变量名和对应的值，容器内会直接使用这个值，和本地环境无关。
```console
# 示例：设置两个环境变量并验证
$ docker run --env VAR1=value1 --env VAR2=value2 ubuntu env | grep VAR
VAR1=value1  # 容器内 VAR1 的值是 value1
VAR2=value2  # 容器内 VAR2 的值是 value2
```
- 命令解析：`ubuntu env` 是在 ubuntu 容器中执行 `env` 命令（打印所有环境变量），`| grep VAR` 只筛选包含 VAR 的行；
- 核心特点：变量值是“硬编码”在命令里的，不受本地环境影响，适合固定值的配置（如数据库地址、端口）。

#### 形式2：仅指定变量名（`VAR`）
不写等号和值，Docker 会**读取你本地终端已导出（export）的该变量值**，并传递给容器；如果本地没导出这个变量，容器内该变量会被设为“未定义（unset）”。
```console
# 第一步：在本地终端导出变量
$ export VAR1=value1
$ export VAR2=value2

# 第二步：仅指定变量名，传递给容器
$ docker run --env VAR1 --env VAR2 ubuntu env | grep VAR
VAR1=value1  # 容器内继承本地 VAR1 的值
VAR2=value2  # 容器内继承本地 VAR2 的值

# 测试：本地未导出的变量
$ docker run --env VAR3 ubuntu env | grep VAR3
# 无输出 → 容器内 VAR3 未定义
```
- 核心特点：变量值“动态继承”本地环境，适合需要和宿主机环境联动的场景（如传递宿主机的用户名、路径）。

---

### 二、批量设置：`--env-file`（从文件加载环境变量）
当需要设置大量环境变量时，逐个写 `-e` 会很繁琐，`--env-file` 可以**从指定文件中批量加载环境变量**，文件格式有明确规则。

#### 1. 环境文件的语法规则
| 语法 | 说明 | 示例 |
|------|------|------|
| `VAR=value` | 显式定义变量值（和 `-e VAR=value` 等价） | `DB_PORT=3306` |
| `VAR` | 继承本地环境的变量值（和 `-e VAR` 等价） | `USER`（继承本地 USER 变量） |
| `# 注释` | 以 `#` 开头的行是注释，会被忽略 | `# 这是数据库配置` |
| `# 出现在行中` | `#` 不在行首时，会被当作变量值的一部分 | `PASSWORD=123#456` → 变量值是 `123#456` |

#### 2. 实际示例
```console
# 第一步：创建环境文件 env.list
$ cat env.list
# 这是注释行，会被忽略
VAR1=value1          # 显式定义 VAR1
VAR2=value2          # 显式定义 VAR2
USER                 # 继承本地 USER 变量
PASSWORD=abc#123     # 值包含 #，不会被当作注释

# 第二步：加载文件并验证
$ docker run --env-file env.list ubuntu env | grep -E 'VAR|USER|PASSWORD'
VAR1=value1
VAR2=value2
USER=jonzeolla  # 继承本地 USER 的值（假设本地 USER 是 jonzeolla）
PASSWORD=abc#123
```

---

### 三、混合使用：`-e`/`--env` + `--env-file`
可以同时使用命令行参数和环境文件，**命令行的 `-e` 优先级更高**（会覆盖文件中同名的变量）：
```console
# env.list 内容：VAR1=oldvalue, VAR2=value2
$ docker run --env-file env.list --env VAR1=newvalue ubuntu env | grep VAR1
VAR1=newvalue  # 命令行的 VAR1 覆盖了文件中的值
```

---

### 四、关键补充细节
1. **覆盖 Dockerfile 中的变量**：如果镜像的 Dockerfile 中用 `ENV` 定义了环境变量（如 `ENV VAR1=default`），运行容器时用 `-e VAR1=custom` 会覆盖这个默认值；
2. **特殊字符处理**：如果变量值包含空格、特殊符号（如 `$`、`&`），需要用引号包裹（如 `-e "VAR=hello world"`）；
3. **数组变量不支持**：文档明确说明，这些参数仅支持“非数组”的简单环境变量，无法传递数组类型的变量；
4. **文件路径**：`--env-file` 后的文件路径可以是相对路径（如 `./env.list`）或绝对路径（如 `/root/env.list`），文件不存在会直接报错。

---

### 总结
1. **`-e/--env`**：适合少量环境变量，支持 `VAR=value`（固定值）和 `VAR`（继承本地值）两种形式，命令行优先级最高；
2. **`--env-file`**：适合大量环境变量，通过文件批量加载，语法支持注释、继承本地值，管理更整洁；
3. **核心规则**：命令行 `-e` 会覆盖 `--env-file` 和 Dockerfile 中的同名变量，`#` 仅在行首时为注释，行中则为变量值的一部分。


##  `--network` 

### 核心作用先明确
`--network` 是 `docker run` 命令中用于**指定容器启动时要连接的网络**的核心参数，它决定了容器的网络环境：比如容器能和哪些其他容器通信、使用什么 IP 地址、是否支持容器名解析等。同时，还能通过扩展语法为不同网络配置专属参数（如静态 IP、MAC 地址）。

---

### 一、基础用法：容器连接单个网络
#### 1. 核心逻辑
Docker 的网络是“隔离且可复用”的：
- 先通过 `docker network create` 创建自定义网络（如 `my-net`）；
- 再通过 `--network=网络名` 让容器启动时接入该网络；
- 同一网络内的容器可通过 **IP 地址** 或 **容器名** 通信（用户自定义网络支持容器名 DNS 解析，默认桥接网络仅支持 IP）。

#### 2. 基础示例
```console
# 第一步：创建名为 my-net 的自定义桥接网络（默认驱动是 bridge）
$ docker network create my-net

# 第二步：启动 busybox 容器并接入 my-net 网络（-itd 是交互+后台运行）
$ docker run -itd --network=my-net busybox
```

#### 3. 关键补充：默认网络 vs 自定义网络
| 网络类型 | 通信方式 | 容器名解析 | 适用场景 |
|----------|----------|------------|----------|
| 默认 bridge 网络（docker0） | 仅支持 IP 通信 | ❌ 不支持 | 临时测试，无需容器名解析 |
| 用户自定义网络（如 my-net） | IP/容器名均可 | ✅ 支持 | 生产环境，容器间需通过名称通信（如微服务调用） |

---

### 二、进阶用法1：为容器分配静态 IP
默认情况下，容器接入网络后会自动获取动态 IP，若需固定 IP（静态 IP），需满足两个条件：
1. 创建网络时**显式指定子网段**（`--subnet`）；
2. 启动容器时用 `--ip`（IPv4）/`--ip6`（IPv6）指定静态 IP（需在子网段内）。

#### 示例：分配静态 IPv4
```console
# 第一步：创建带子网段的网络（192.0.2.0/24 是自定义子网，范围 192.0.2.1-192.0.2.254）
$ docker network create --subnet 192.0.2.0/24 my-net

# 第二步：启动容器并指定静态 IP（192.0.2.69 需在子网段内）
$ docker run -itd --network=my-net --ip=192.0.2.69 busybox

# 验证：查看容器 IP
$ docker exec <容器ID> ip addr
# 输出中会看到 eth0 网卡的 IP 是 192.0.2.69
```

> 注意：静态 IP 必须属于网络的子网段，否则容器启动失败；IPv6 配置需先开启 Docker 的 IPv6 支持，再指定 `--subnet`（如 `2001:db8::/64`）和 `--ip6`。

---

### 三、进阶用法2：容器连接多个网络
一个容器可以接入多个网络（类似一台机器有多个网卡），只需**重复 `--network` 参数**即可。

#### 基础示例：连接两个网络
```console
# 第一步：创建两个不同子网的网络
$ docker network create --subnet 192.0.2.0/24 my-net1
$ docker network create --subnet 192.0.3.0/24 my-net2

# 第二步：启动容器，同时接入 my-net1 和 my-net2
$ docker run -itd --network=my-net1 --network=my-net2 busybox

# 验证：容器会有两个网卡（eth0 对应 my-net1，eth1 对应 my-net2）
$ docker exec <容器ID> ip addr
```

---

### 四、高级用法：`--network` 扩展语法（多网络专属配置）
当容器连接多个网络时，若需为每个网络配置不同参数（如不同 IP、MAC 地址），需使用**扩展语法**：
```
--network=name=网络名,参数1=值1,参数2=值2
```

#### 1. 扩展语法支持的核心参数
| 参数 | 等价顶层参数 | 说明 | 示例 |
|------|--------------|------|------|
| `name` | - | 必选，指定网络名称 | `name=my-net1` |
| `ip` | `--ip` | 该网络下的静态 IPv4 | `ip=192.0.2.42` |
| `ip6` | `--ip6` | 该网络下的静态 IPv6 | `ip6=2001:db8::33` |
| `mac-address` | `--mac-address` | 该网络下的 MAC 地址 | `mac-address=92:d0:c6:0a:29:33` |
| `alias` | `--network-alias` | 该网络内的容器别名（DNS 解析用） | `alias=web-server` |
| `driver-opt` | `docker network connect --driver-opt` | 网络驱动选项（如配置 sysctl） | `driver-opt=com.docker.network.endpoint.sysctls=...` |

#### 2. 扩展语法示例：多网络配置不同静态 IP
```console
# 先创建两个子网网络（同前）
$ docker network create --subnet 192.0.2.0/24 my-net1
$ docker network create --subnet 192.0.3.0/24 my-net2

# 启动容器，为每个网络配置专属 IP
$ docker run -itd \
  --network=name=my-net1,ip=192.0.2.42 \
  --network=name=my-net2,ip=192.0.3.42 \
  busybox

# 验证：容器在 my-net1 中 IP 是 192.0.2.42，my-net2 中是 192.0.3.42
```

#### 3. 高级示例：配置网络接口的 sysctl 参数
`driver-opt` 可配置网络接口的内核参数（sysctl），需使用标签 `com.docker.network.endpoint.sysctls`，且用 `IFNAME` 代指容器内的网卡名（Docker 会自动替换为实际网卡名，如 eth0）。

```console
# 创建带子网的网络
$ docker network create --subnet 192.0.2.0/24 my-net

# 启动容器，配置 sysctl + 静态 IP（注意引号转义，适配 shell 语法）
$ docker run -itd \
  --network=name=my-net,\
  "driver-opt=com.docker.network.endpoint.sysctls=net.ipv4.conf.IFNAME.log_martians=1,net.ipv4.conf.IFNAME.forwarding=0",\
  ip=192.0.2.42 \
  busybox
```
- 效果：容器在 my-net 中的网卡（如 eth0）会被设置：
    - `net.ipv4.conf.eth0.log_martians=1`（记录非法源地址的数据包）
    - `net.ipv4.conf.eth0.forwarding=0`（关闭该接口的 IP 转发）
- 注意：网络驱动可能限制部分 sysctl 参数的修改，以保护网络稳定性。

---

### 五、补充命令：容器运行中调整网络
- **连接现有网络**：`docker network connect 网络名 容器ID`（给运行中的容器新增网络）；
- **断开网络**：`docker network disconnect 网络名 容器ID`（移除容器的某个网络）。

### 总结
1. `--network` 核心是指定容器启动时接入的网络，自定义网络支持容器名 DNS 解析（默认桥接网络仅支持 IP）；
2. 配置静态 IP 需先为网络指定子网段，再用 `--ip`/`--ip6` 赋值；
3. 容器可接入多个网络，重复 `--network` 即可；多网络专属配置需用扩展语法（`name=网络名,参数=值`）；
4. `driver-opt` 可配置网络接口的 sysctl 参数，但受驱动限制，需谨慎使用。



## `--volumes-from` 

### 核心作用先明确
`--volumes-from` 是 `docker run` 命令中用于**“继承/挂载其他容器已定义的卷（volumes）”** 的参数，核心价值是实现**容器间的数据共享**——让新容器直接复用已有容器的卷（包括数据卷 Volume 和绑定挂载 Bind Mount），无需重新配置挂载路径。

---

### 一、基础用法：挂载其他容器的所有卷
#### 1. 核心逻辑
- 前提：已有容器（称为“源容器”）定义了卷（比如通过 `-v`/`--volume` 挂载的目录/数据卷）；
- 作用：新容器通过 `--volumes-from 源容器ID/名称`，会**完整挂载源容器的所有卷**，相当于和源容器共享这些目录/数据；
- 特点：可重复使用 `--volumes-from`，挂载多个源容器的卷。

#### 2. 基础示例解析
```console
$ docker run --volumes-from 777f7dc92da7 --volumes-from ba8c0c54f0f2:ro -i -t ubuntu pwd
```
- `777f7dc92da7`：第一个源容器的 ID，新容器会以**源容器的默认模式**（只读/读写）挂载该容器的所有卷；
- `ba8c0c54f0f2:ro`：第二个源容器的 ID，后缀 `:ro` 强制将该容器的卷以**只读模式**挂载到新容器（即使源容器是读写模式）；
- `-i -t ubuntu pwd`：启动交互式 ubuntu 容器并执行 `pwd` 命令，核心是演示卷挂载。

#### 3. 完整实操示例（理解数据共享）
```console
# 步骤1：创建源容器，挂载一个数据卷到 /data 目录
$ docker run -v /data --name source-container ubuntu touch /data/test.txt
# 解释：-v /data 会创建匿名数据卷并挂载到容器 /data，touch 命令在该目录创建 test.txt

# 步骤2：通过 --volumes-from 启动新容器，挂载源容器的卷
$ docker run --volumes-from source-container --name new-container ubuntu ls /data
# 输出：test.txt → 新容器能看到源容器 /data 目录下的文件，实现数据共享

# 步骤3：验证读写模式（默认和源容器一致，源容器是 rw 读写）
$ docker run --volumes-from source-container ubuntu touch /data/new.txt
# 执行后，源容器的 /data 目录也会出现 new.txt → 双向读写
```

---

### 二、关键扩展：读写模式控制（`:ro`/`:rw`）
`--volumes-from` 支持通过后缀指定卷的挂载模式，覆盖源容器的默认模式：

| 后缀 | 含义 | 说明 |
|------|------|------|
| 无后缀（默认） | 继承源容器模式 | 源容器是 `rw`（读写）则新容器也是 `rw`，源容器是 `ro` 则新容器也是 `ro` |
| `:ro` | 只读模式 | 强制新容器对该卷只有读取权限，无法修改/新增文件 |
| `:rw` | 读写模式 | 强制新容器对该卷有读写权限（需源容器卷本身支持读写） |

#### 示例：只读挂载
```console
# 启动新容器，以只读模式挂载源容器的卷
$ docker run --volumes-from source-container:ro ubuntu touch /data/error.txt
# 输出：touch: cannot touch '/data/error.txt': Read-only file system → 只读模式无法写入
```

---

### 三、高级细节：SELinux 标签（`:z`/`:Z`）
这是针对启用了 SELinux（Linux 安全增强模块）的系统（如 CentOS、RHEL）的特殊配置，用于解决 SELinux 阻止容器访问挂载卷的权限问题。

#### 核心背景
SELinux 会给文件/目录打“安全标签”，默认情况下，宿主机或其他容器的卷挂载到新容器后，SELinux 可能因标签不匹配禁止容器访问。`:z`/`:Z` 让 Docker 自动重新标记卷的 SELinux 标签，解决权限问题。

#### 标签含义与区别
| 标签 | 含义 | 适用场景 |
|------|------|----------|
| `:z` | 共享标签（shared） | 多个容器**共享同一卷**（比如多个容器挂载同一个宿主机目录），Docker 会将卷标签设为“共享”，所有容器都能读写 |
| `:Z` | 私有标签（private） | 仅当前容器使用该卷，Docker 会将卷标签设为“私有”，其他容器无法访问该卷 |

#### 用法示例（结合 `--volumes-from`）
```console
# 场景：SELinux 系统中，让新容器共享源容器的卷，且允许多容器访问
$ docker run --volumes-from source-container:z ubuntu ls /data

# 场景：SELinux 系统中，让新容器独占源容器的卷（私有标签）
$ docker run --volumes-from source-container:Z ubuntu ls /data
```

> 注意：
> 1. 仅在启用 SELinux 的 Linux 系统中需要使用，Ubuntu/Debian（默认关闭 SELinux）无需关注；
> 2. `:z`/`:Z` 可和 `:ro`/`:rw` 组合使用（如 `:ro,z`），顺序不影响。

---

### 四、补充关键注意事项
1. **源容器无需运行**：即使源容器已停止（`exited`），只要未被删除，`--volumes-from` 仍能挂载其卷；
2. **卷的生命周期**：通过 `--volumes-from` 挂载的卷，其生命周期独立于容器——只有删除最后一个引用该卷的容器，且执行 `docker volume prune` 才会清理；
3. **权限继承**：卷的文件权限（如属主、属组）会原样继承，若需修改，需在容器内手动调整。

### 总结
1. `--volumes-from` 核心是复用其他容器的卷，实现容器间数据共享，可重复指定多个源容器；
2. 后缀 `:ro`/`:rw` 可控制卷的读写模式，默认继承源容器的模式；
3. SELinux 系统中，`:z`（共享标签）/`:Z`（私有标签）用于解决卷的访问权限问题，非 SELinux 系统无需使用。


## `-d/--detach`（后台运行模式）
### 核心作用先明确
`-d`（`--detach` 的简写）是 `docker run` 中最常用的参数之一，核心作用是**让容器在“后台（detached）”模式下运行**，不再占用当前的终端窗口，容器会作为一个后台进程在宿主机中运行，这也是生产环境中启动容器的标准方式。

---

### 一、基础用法与核心规则
#### 1. 基本使用方式
```console
# 前台运行（默认，不加 -d）：容器占用当前终端，关闭终端则容器停止
$ docker run -p 80:80 nginx:alpine

# 后台运行（加 -d）：容器在后台运行，终端立即返回容器ID，可继续执行其他命令
$ docker run -d -p 80:80 nginx:alpine
# 输出示例：7f929e87e633e82198f5e5158e9f09f0f1e9f8e7f6d5c4b3a2s1d0f9e8d7c6b5a
```

#### 2. 核心生命周期规则
- **容器退出条件**：后台运行的容器，会在其**根进程（PID 1）退出时立即停止**（这是 Docker 容器的核心设计：容器的生命周期和根进程绑定）；
- **--rm 组合使用**：如果同时指定 `-d --rm`，容器退出（根进程结束）或 Docker 守护进程退出时，容器会被**自动删除**（适合临时运行的后台容器，避免残留无用容器）；
- **默认行为（不加 --rm）**：容器退出后不会自动删除，需手动执行 `docker rm <容器ID>` 清理。

---

### 二、最常见的误区：后台容器启动“服务”却立即停止
这是新手使用 `-d` 时最容易踩的坑，核心原因是**错误地将“启动服务的命令”作为容器的根进程**，而这类命令执行后会立即返回（根进程退出），导致容器随之停止。

#### 错误示例解析
```console
# 错误用法：试图启动 nginx 服务，容器却立即退出
$ docker run -d -p 80:80 my_image service nginx start
```
- 问题根源：`service nginx start` 是“启动服务的命令”——它的作用是让 nginx 以**后台守护进程**的方式运行，执行完这个命令后，`service nginx start` 本身（容器的根进程）就会立即退出；
- 容器行为：根进程退出 → Docker 认为容器任务完成 → 后台运行的容器直接停止，导致 nginx 服务虽然启动了，但容器已退出，无法访问。

#### 正确做法：让服务进程成为容器的根进程
容器的根进程必须是**前台运行的服务进程**，这样根进程不退出，容器就会一直运行。以 nginx 为例：
```console
# 正确用法：让 nginx 以前台模式运行（作为容器根进程）
$ docker run -d -p 80:80 my_image nginx -g 'daemon off;'
```
- 关键参数：`nginx -g 'daemon off;'` 是 nginx 的前台运行参数——强制 nginx 不进入后台，而是作为前台进程运行；
- 容器行为：nginx 进程成为容器的根进程，只要 nginx 不崩溃，容器就会一直后台运行，可正常访问 80 端口。

> 通用规则：所有后台运行的容器，其启动命令必须是**前台运行的进程**（如 `nginx -g 'daemon off;'`、`redis-server --daemonize no`、`java -jar app.jar` 等），而非“启动后台服务的命令”。

---

### 三、后台容器的交互方式
后台运行的容器不再监听启动它的终端，因此无法直接通过终端输入/输出，需通过以下方式交互：
1. **网络连接**：通过映射的端口（`-p`）访问容器内的服务（如浏览器访问 nginx 的 80 端口）；
2. **共享卷**：通过 `-v`/`--volumes-from` 挂载的卷，读写容器内的文件（如修改配置、查看日志）；
3. **docker 命令交互**：
    - 查看日志：`docker logs <容器ID>`（查看后台容器的输出）；
    - 进入容器：`docker exec -it <容器ID> bash`（进入后台容器的终端）；
    - 停止容器：`docker stop <容器ID>`（停止后台运行的容器）。

#### 示例：查看后台容器日志
```console
# 查看 nginx 后台容器的访问日志
$ docker logs -f <容器ID>  # -f 实时跟踪日志输出
```

---

### 四、补充关键细节
1. **前台 vs 后台的本质**：
    - 前台模式：容器的标准输入/输出/错误（STDIN/STDOUT/STDERR）绑定到当前终端；
    - 后台模式：容器的 IO 不再绑定终端，Docker 会将容器的输出重定向到日志系统（可通过 `docker logs` 查看）；
2. **容器命名**：后台运行的容器建议用 `--name` 指定名称（如 `docker run -d --name my-nginx -p 80:80 nginx`），方便后续通过名称管理（而非记容器ID）；
3. **资源限制**：`-d` 可和资源限制参数（如 `--memory`、`--cpus`）组合使用，限制后台容器的资源使用。

### 总结
1. `-d/--detach` 让容器后台运行，不占用终端，根进程退出则容器停止，`-d --rm` 可让容器退出后自动删除；
2. 后台容器的核心陷阱：不能用“启动后台服务的命令”（如 `service nginx start`）作为根进程，必须让服务前台运行（如 `nginx -g 'daemon off;'`）；
3. 后台容器的交互依赖网络、共享卷或 `docker logs/exec` 等命令，而非直接的终端输入输出。


## `-a/--attach` 

### 核心作用先明确
`-a/--attach`（`--attach` 是全称，`-a` 是简写）的核心作用是**显式指定 Docker 要将当前终端与容器的哪些标准流（STDIN/STDOUT/STDERR）绑定**，从而精准控制容器的输入/输出方向——比如只接收容器的错误输出、只给容器传输入数据，而不是默认绑定所有流。

首先先理清三个基础概念（新手必看）：

| 流名称 | 含义 | 通俗理解 |
|--------|------|----------|
| `STDIN` (0) | 标准输入 | 容器从哪里获取输入（比如终端键盘、管道数据） |
| `STDOUT` (1) | 标准输出 | 容器正常运行的输出内容（比如 `echo test` 的结果） |
| `STDERR` (2) | 标准错误 | 容器运行出错时的输出内容（比如命令不存在的报错） |

默认情况下，`docker run -it` 会绑定容器的 **STDIN + STDOUT + STDERR** 所有流；而 `-a` 可以精准选择绑定某一个/某几个流，实现更灵活的输入输出控制。

---

### 一、基础用法：指定绑定的流
#### 语法格式
```bash
docker run -a <流名称> [其他参数] <镜像> <命令>
```
支持的流名称：`stdin`（或 `0`）、`stdout`（或 `1`）、`stderr`（或 `2`），可重复使用 `-a` 指定多个流。

#### 基础示例：绑定 STDIN + STDOUT
```console
$ docker run -a stdin -a stdout -i -t ubuntu /bin/bash
```
- `-a stdin -a stdout`：显式绑定容器的标准输入和标准输出；
- `-i`：保持 STDIN 打开（即使没有附加），`-t`：分配伪终端；
- 效果：和 `docker run -it ubuntu /bin/bash` 等价，进入容器的交互式终端，能输入命令、看到正常输出。

---

### 二、典型场景示例：精准控制输入输出
#### 场景1：仅向容器传递输入（只绑定 STDIN）
需求：将宿主机的数据通过管道传给容器，只绑定容器的 STDIN，不关心输出。
```console
$ echo "test" | docker run -i -a stdin ubuntu cat -
```
- 拆解分析：
    1. `echo "test"`：宿主机输出字符串 "test"；
    2. `|`：管道，将前一个命令的 STDOUT 传给后一个命令的 STDIN；
    3. `-i`：保持容器 STDIN 打开（必须加，否则管道数据无法传入）；
    4. `-a stdin`：仅绑定容器的 STDIN，不绑定 STDOUT/STDERR；
    5. `ubuntu cat -`：在 ubuntu 容器中执行 `cat -`（`cat -` 表示从 STDIN 读取数据并输出）；
- 执行效果：虽然 `cat -` 会输出 "test"，但因为只绑定了 STDIN，终端不会显示输出（容器的 STDOUT 未绑定到当前终端），但数据确实传入了容器。

#### 场景2：仅接收容器的错误输出（只绑定 STDERR）
需求：只看容器的错误信息，正常输出不显示。
```console
$ docker run -a stderr ubuntu echo test
```
- 拆解分析：
    1. `-a stderr`：仅绑定容器的 STDERR，不绑定 STDIN/STDOUT；
    2. `ubuntu echo test`：执行 `echo test` 是正常命令，输出到 STDOUT；
- 执行效果：终端**无任何输出**（因为 STDOUT 未绑定），但如果执行错误命令（如 `ubuntu ech test`），终端会显示报错（因为报错输出到 STDERR，且 STDERR 已绑定）；
- 关键补充：容器的所有输出（STDOUT/STDERR）都会被 Docker 日志系统记录，即使终端没显示，也能通过 `docker logs <容器ID>` 查看。

#### 场景3：向容器传入文件数据（实用场景）
需求：将宿主机的文件内容传入容器，用于构建/处理，同时获取容器ID以便后续查日志。
```console
$ cat somefile | docker run -i -a stdin mybuilder dobuild
```
- 拆解分析：
    1. `cat somefile`：读取宿主机 `somefile` 文件的内容；
    2. `|`：将文件内容传给容器的 STDIN；
    3. `-i -a stdin`：保持 STDIN 打开并绑定，确保文件内容能传入；
    4. `mybuilder dobuild`：自定义镜像 `mybuilder` 执行 `dobuild` 命令（该命令从 STDIN 读取 `somefile` 内容进行构建）；
- 执行效果：终端会输出容器ID（Docker 默认在容器启动/结束时输出ID），文件内容传入容器完成构建，后续可通过 `docker logs <容器ID>` 查看构建日志。

---

### 三、关键补充：PID 1 进程的特殊行为（重要注意事项）
文档中特别强调：**容器内 PID 为 1 的进程（根进程）会被 Linux 特殊处理——它会忽略所有“默认动作的信号”**，比如：
- 按 `Ctrl+C` 发送的 `SIGINT` 信号；
- `docker stop` 发送的 `SIGTERM` 信号；
- 这些信号不会终止 PID 1 进程，除非进程本身编码了处理这些信号的逻辑。

#### 示例说明
```console
# 启动容器，PID 1 是 bash（交互式）
$ docker run -a stdin -a stdout -i -t ubuntu /bin/bash

# 在容器内执行：ps aux（查看进程）
root         1  0.0  0.0   4708  3800 pts/0    Ss   02:00   0:00 /bin/bash
```
此时按 `Ctrl+C`，bash 不会退出（因为是 PID 1），需手动输入 `exit` 或执行 `docker stop` 才能终止容器。

---

### 四、`-a` 与 `-i/-t` 的关系（新手易混点）
| 参数 | 作用 | 与 `-a` 的配合 |
|------|------|----------------|
| `-i` (--interactive) | 保持容器的 STDIN 打开，即使没有附加 | 若要通过管道/终端向容器传输入（`-a stdin`），必须加 `-i`，否则 STDIN 会立即关闭 |
| `-t` (--tty) | 为容器分配伪终端（模拟终端环境） | 仅在需要交互式终端（如 `bash`）时加，纯管道传数据（如 `cat file | docker run`）无需加 `-t` |
| `-a` | 指定绑定哪些流 | 可单独控制流的绑定，比如 `-a stderr` 无需加 `-i`（因为不需要输入） |

#### 正确搭配示例
- 交互式终端：`-a stdin -a stdout -i -t`（等价于 `-it`）；
- 管道传输入：`-a stdin -i`（无需 `-t`）；
- 只看错误输出：`-a stderr`（无需 `-i/-t`）。

### 总结
1. `-a/--attach` 用于**精准绑定容器的 STDIN/STDOUT/STDERR**，替代默认的“全绑定”，实现输入输出的精细化控制；
2. 核心场景：仅传输入（`-a stdin -i`）、仅看错误（`-a stderr`）、文件数据传入容器（`cat file | docker run -i -a stdin`）；
3. 关键注意：`-i` 是管道/终端输入的前提，容器 PID 1 进程忽略默认信号，需手动处理终止逻辑，所有输出都会记录到 Docker 日志（可通过 `docker logs` 查看）。



## `-i/--interactive`、`--init`、`-t/--tty` 这三个参数的用法，包括它们的核心作用、搭配场景、实际示例，以及背后的原理（比如伪终端、PID 1 进程、僵尸进程清理），我会从基础概念到实际案例，把这些知识点讲得清晰易懂。

### 先理清核心概念（新手必看）
在解释参数前，先明确两个基础概念，避免理解偏差：
- **STDIN**：容器的标准输入流，是容器接收外部输入的通道（比如键盘输入、管道数据）；
- **伪终端（pseudo-TTY/PTY）**：模拟物理终端的虚拟设备，让容器能像本地终端一样支持交互（如密码隐藏、光标移动、命令补全）。

---

## 一、`-i/--interactive`：保持 STDIN 打开（核心是“输入不中断”）
### 核心作用
`-i` 的唯一核心作用是**强制保持容器的 STDIN（标准输入）处于打开状态**，即使没有直接绑定到终端，也能接收来自管道、文件等的输入数据——如果不加 `-i`，容器的 STDIN 会立即关闭，无法传入任何输入。

### 关键特点
- 只负责“打开输入通道”，不提供终端交互功能（比如无法隐藏密码、没有命令行提示符）；
- 常和管道（`|`）配合，实现“向容器传输入”的场景；
- 可单独使用，也可和 `-t` 搭配实现完整的交互式终端。

### 实际示例
#### 示例1：基础用法（管道传输入）
```console
# echo 输出 "hello"，通过管道传给容器的 STDIN，容器内 cat 读取并输出
$ echo hello | docker run --rm -i busybox cat
hello
```
- 拆解：
    - `--rm`：容器退出后自动删除（避免残留）；
    - `-i`：保持容器 STDIN 打开，确保管道的 "hello" 能传入；
    - 如果去掉 `-i`：容器 STDIN 关闭，`cat` 无输入，终端无输出。

#### 示例2：多容器管道组合（核心场景）
`-i` 让容器可以作为“管道处理节点”，实现多容器协作处理数据：
```console
# 第一步：输出 "foo bar baz" → 第二步：awk 取第二个字段 "bar" → 第三步：rev 反转成 "rab"
$ docker run --rm -i busybox echo "foo bar baz" \
  | docker run --rm -i busybox awk '{ print $2 }' \
  | docker run --rm -i busybox rev
rab
```
- 关键：每个容器都加了 `-i`，确保 STDIN 打开，管道数据能依次传递。

#### 示例3：和 `-t` 搭配（交互式终端）
```console
# -i 保持输入打开，-t 分配伪终端，组合成完整的交互式终端
$ docker run -it debian
root@10a3e71492b0:/# factor 90  # 执行命令，有提示符、可交互
90: 2 3 3 5
root@10a3e71492b0:/# exit       # 退出容器
exit
```

---

## 二、`--init`：指定 init 进程作为容器 PID 1（核心是“清理僵尸进程”）
### 核心背景
Docker 容器的默认行为：容器启动后，**用户指定的命令会成为 PID 1 进程**（根进程），而 Linux 对 PID 1 进程有特殊规则：
1. PID 1 进程不会自动“收割（reap）”子进程的僵尸进程（子进程退出后残留的进程条目）；
2. PID 1 进程会忽略默认的信号（如 `SIGINT`/`SIGTERM`），除非手动处理。

### 核心作用
`--init` 会让 Docker 在容器内启动一个**轻量级 init 进程（默认是 tini）** 作为 PID 1，这个 init 进程会：
1. 自动收割容器内的僵尸进程（避免进程资源泄漏）；
2. 正确转发信号给用户进程（比如 `docker stop` 发送的 `SIGTERM` 能被用户进程接收）；
3. 作为父进程管理用户进程，保证容器退出时清理所有子进程。


## `--restart` 参数（容器重启策略）的用法，
包括四种核心策略的区别、重试机制、延迟规则，以及实际使用场景和示例，我会从核心逻辑到细节规则，把这个参数讲得清晰易懂，帮你精准选择适合的重启策略。

### 核心作用先明确
`--restart` 是 `docker run` 中用于**定义容器退出后 Docker 守护进程（daemon）是否/如何自动重启容器**的核心参数，本质是给容器设置“故障自愈”规则，适用于需要长期稳定运行的服务（如数据库、Web 服务），避免容器意外退出后服务中断。

---

## 一、四种重启策略详解
Docker 支持四种重启策略，核心区别在于“重启触发条件”和“手动停止后的行为”，以下是详细对比和解析：

| 策略 | 中文理解 | 核心触发规则 | 手动停止容器后 | Docker 守护进程重启后 | 适用场景 |
|------|----------|--------------|----------------|----------------------|----------|
| `no`（默认） | 永不重启 | 容器退出后，Docker 完全不自动重启 | - | - | 临时测试容器（如一次性数据处理、调试），无需自动重启 |
| `on-failure[:max-retries]` | 失败时重启 | 仅当容器**非 0 退出码**（表示运行出错，如程序崩溃、配置错误）时才重启；<br>可选 `:max-retries` 限制最大重启次数（默认无限重试） | 手动停止后，容器不会自动重启 | Docker 重启后，不会重启该容器（仅针对“手动停止”的容器） | 业务服务（如 API 服务）：仅在程序出错时重启，正常退出（0 码）不重启；需限制重试次数避免无限循环 |
| `always` | 总是重启 | 无论容器以何种退出码（0/非 0）停止，都会自动重启；<br>即使是正常退出（如 `docker stop` 手动停止），Docker 守护进程重启后也会重启该容器 | 手动停止后，容器暂时不重启 | Docker 守护进程重启时，会重启该容器 | 核心基础设施服务（如 Redis、Nginx）：必须始终运行，哪怕正常退出也需重启 |
| `unless-stopped` | 除非停止否则重启 | 和 `always` 几乎一致（任何退出码都重启）；<br>但**手动停止的容器**，即使 Docker 守护进程重启，也不会再重启 | 手动停止后，容器永久不重启（除非手动 `docker start`） | Docker 重启后，不会重启“手动停止”的容器 | 生产环境核心服务：既保证故障自动重启，又允许手动停止后不再自动启动（比 `always` 更灵活） |

### 关键示例解析
#### 示例1：基础用法（`always` 策略）
```console
$ docker run --restart=always redis
```
- 效果：Redis 容器无论因何原因停止（崩溃、正常退出），Docker 都会自动重启；
- 手动停止：执行 `docker stop <容器ID>` 后，容器暂时停止；但如果重启 Docker 守护进程（如 `systemctl restart docker`），该容器会被重新启动。

#### 示例2：`on-failure` 带重试次数
```console
$ docker run --restart=on-failure:10 redis
```
- 效果：仅当 Redis 容器非 0 退出码时重启，且最多重试 10 次；
- 触发停止：如果容器连续 10 次启动后都出错退出，Docker 会停止重试，容器保持停止状态；
- 注意：`max-retries` 仅对 `on-failure` 策略有效，`always`/`unless-stopped` 不支持该参数。

---

## 二、重启的延迟规则（核心细节，避免服务器过载）
Docker 为了防止容器重启“刷屏”（短时间内无限重启），设计了**指数退避延迟机制**，核心规则如下：
1. **初始延迟**：第一次重启前等待 100 毫秒（ms）；
2. **指数增长**：每次重启失败后，延迟时间翻倍（100ms → 200ms → 400ms → 800ms → ...）；
3. **上限限制**：延迟时间最多增长到 1 分钟（60 秒），之后不再翻倍；
4. **延迟重置**：如果容器**成功运行至少 10 秒**后才退出，延迟会重置为初始的 100ms（表示容器曾正常运行，不是启动就崩溃）；
5. **停止触发**：执行 `docker stop` 或 `docker rm -f` 会立即终止重启流程，重置延迟。

### 示例：延迟机制实际表现
假设 Redis 容器启动后立即崩溃（退出码非 0），重启流程如下：
- 第1次重启：等待 100ms → 启动 → 崩溃；
- 第2次重启：等待 200ms → 启动 → 崩溃；
- 第3次重启：等待 400ms → 启动 → 崩溃；
- ...
- 延迟达到 60s 后，每次重启都等待 60s；
- 如果某次启动后，Redis 正常运行了 15 秒才崩溃，下一次重启延迟会重置为 100ms。

---

## 三、如何验证重启策略是否生效
### 1. 查看容器状态（`docker ps`）
- 容器正常运行：状态显示 `Up`；
- 容器正在重启：状态显示 `Restarting (退出码) 延迟时间 ago`；
- 示例：`Restarting (1) 5 seconds ago`（表示因退出码 1 失败，5 秒前尝试重启）。

### 2. 查看 Docker 事件日志（`docker events`）
实时监控容器的重启行为，能清晰看到重启触发的原因：
```console
# 实时输出 Docker 事件
$ docker events
# 当容器重启时，会输出类似日志：
2026-02-27T10:00:00+08:00 container restart <容器ID> (image=redis, restart_policy=always)
```

---

## 四、关键注意事项（避坑指南）
1. **`always` vs `unless-stopped` 核心区别**：
    - `always`：即使手动 `docker stop` 容器，重启 Docker 守护进程后，容器会被重新启动；
    - `unless-stopped`：手动 `docker stop` 容器后，哪怕重启 Docker 守护进程，容器也不会启动（需手动 `docker start`）；
    - 生产环境优先选 `unless-stopped`，避免手动停止的容器被 Docker 重启“意外复活”。

2. **`on-failure` 的退出码判断**：
    - 只有非 0 退出码才触发重启（如程序崩溃是 1，OOM 被杀死是 137）；
    - 正常退出（0 码，如 `docker stop`）不会触发重启。

3. **重启策略不影响手动操作**：
    - `docker stop`/`docker start`/`docker rm` 始终优先于重启策略；
    - 执行 `docker rm <容器ID>` 会彻底清除容器，包括其重启策略。

4. **不支持动态修改**：
    - 容器启动后，无法修改其重启策略；
    - 如需调整，需删除容器后重新启动（`docker rm <容器ID>` → `docker run --restart=新策略 ...`）。

### 总结
1. `--restart` 控制容器退出后 Docker 的自动重启行为，核心有 `no`/`on-failure[:次数]`/`always`/`unless-stopped` 四种策略；
2. `on-failure` 仅在容器出错时重启，可限制重试次数；`always` 无条件重启，`unless-stopped` 是更灵活的“总是重启”（手动停止后除外）；
3. 重启有指数退避延迟（100ms 起步，翻倍到 1 分钟上限），容器正常运行 10 秒后退出会重置延迟，避免服务器过载。


## `--rm` 

### 核心作用先明确
`--rm` 是 `docker run` 中用于**容器退出后自动清理资源**的参数，核心作用是：当容器运行结束（无论正常退出还是异常退出）时，Docker 会自动删除该容器的文件系统和相关的匿名卷，避免宿主机残留大量无用的退出容器和匿名卷，节省磁盘空间。

---

## 一、基础用法与核心逻辑
### 1. 默认行为（不加 `--rm`）
Docker 容器的文件系统是“持久化”的——即使容器退出（状态变为 `exited`），容器的文件系统、配置、数据仍会保留在宿主机上，你可以通过 `docker ps -a` 看到这些退出的容器，也能通过 `docker start <容器ID>` 重新启动它们，或通过 `docker exec` 进入容器查看最终状态（方便调试）。

### 2. `--rm` 的行为（加 `--rm`）
容器退出后，Docker 会立即执行以下操作：
- 删除容器本身（`docker ps -a` 中不再显示该容器）；
- 删除容器关联的**匿名卷**（无名称的卷）；
- 保留命名卷、绑定挂载（宿主机目录）的数据（核心！避免误删重要数据）。

### 3. 基础示例
```console
# 启动一个临时的 busybox 容器，执行 top 命令，退出后自动删除容器
$ docker run --rm busybox top
```
- 执行后：容器运行 `top` 命令，按 `Ctrl+C` 停止 `top` → 容器退出 → Docker 自动删除该容器，`docker ps -a` 中看不到这个容器。

---

## 二、关键细节：卷的清理规则（重点避坑）
`--rm` 对“卷”的清理逻辑是新手最容易混淆的点，核心规则是：**只删匿名卷，不删命名卷/绑定挂载**。

### 先理清卷的类型
| 卷类型 | 定义 | 示例 | `--rm` 是否删除 |
|--------|------|------|----------------|
| 匿名卷 | 无自定义名称，Docker 自动生成随机名称的卷 | `-v /foo`（仅指定容器内路径） | ✅ 是 |
| 命名卷 | 有自定义名称的卷（通过 `docker volume create` 或 `-v 卷名:路径` 创建） | `-v awesome:/bar`（`awesome` 是卷名） | ❌ 否 |
| 绑定挂载 | 直接挂载宿主机目录到容器 | `-v /host/path:/container/path` | ❌ 否（仅卸载挂载，宿主机目录数据保留） |

### 官方示例解析
```console
$ docker run --rm -v /foo -v awesome:/bar busybox top
```
- `-v /foo`：创建匿名卷，挂载到容器 `/foo` 目录 → 容器退出后，`/foo` 对应的匿名卷被删除；
- `-v awesome:/bar`：创建/使用命名卷 `awesome`，挂载到容器 `/bar` 目录 → 容器退出后，命名卷 `awesome` 保留，数据不丢失；
- 最终效果：容器被删除，匿名卷 `/foo` 被删除，命名卷 `awesome` 仍存在（可通过 `docker volume ls` 查看）。

### 补充：`--volumes-from` 继承的卷
如果容器通过 `--volumes-from` 继承了其他容器的卷，清理逻辑和原卷类型一致：
- 继承的是匿名卷 → `--rm` 会删除；
- 继承的是命名卷 → `--rm` 不删除。

---

## 三、适用场景与不适用场景
### 1. 适合用 `--rm` 的场景
- **临时一次性任务**：比如运行一个脚本处理数据、临时测试命令、打包文件等，执行完就不需要保留容器；
- **前台短进程**：比如 `docker run --rm ubuntu ls /`，执行完 `ls` 命令容器就退出，自动清理无残留；
- **避免磁盘占用**：批量运行临时容器时，防止宿主机堆积大量 `exited` 状态的容器。

### 2. 不适合用 `--rm` 的场景
- **长期运行的服务容器**：比如 Redis、MySQL、Nginx 等，需要容器持久化（即使重启/退出也能恢复）；
- **需要保留数据/日志的容器**：比如应用容器的日志、数据库数据存储在匿名卷中（如果加 `--rm`，退出后数据会被删除）；
- **需要调试的容器**：容器运行出错后，需要保留容器文件系统查看日志、配置等（加 `--rm` 会直接删除，无法调试）。

---

## 四、关键注意事项（避坑指南）
1. **`--rm` 与 `-d`（后台运行）的配合**：
    - 可以同时使用 `--rm -d`：后台运行的容器退出后（如根进程崩溃），会自动删除；
    - 注意：如果手动执行 `docker stop <容器ID>` 停止后台容器，容器也会被自动删除。

2. **`--rm` 与重启策略（`--restart`）不兼容**：
    - Docker 不允许同时指定 `--rm` 和 `--restart`（重启策略是让容器持续运行，`--rm` 是退出后删除，逻辑冲突）；
    - 执行 `docker run --rm --restart=always redis` 会直接报错：`conflict: --rm and --restart cannot be used together`。

3. **`--rm` 等价于 `docker rm -v`**：
    - 手动删除容器时，`docker rm -v <容器ID>` 会删除容器并清理关联的匿名卷，和 `--rm` 的清理逻辑完全一致；
    - 而单纯 `docker rm <容器ID>` 只会删除容器，不会删除匿名卷（需手动 `docker volume rm` 清理）。

4. **数据安全提醒**：
    - 不要在生产环境给存储重要数据的匿名卷容器加 `--rm`，否则容器退出后数据会永久丢失；
    - 重要数据建议使用**命名卷**或**绑定挂载**，即使加 `--rm` 也不会丢失。

---

### 总结
1. `--rm` 核心是容器退出后自动删除容器本身和关联的**匿名卷**，避免宿主机残留无用资源；
2. 卷清理规则：只删匿名卷，不删命名卷/绑定挂载（保护重要数据）；
3. 适用临时一次性任务，不适合长期服务/需保留数据的容器，且与 `--restart` 重启策略不兼容。

