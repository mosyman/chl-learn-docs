
## 目录
- [docker compose](#docker-compose)
- [docker compose up](#docker-compose-up)
- [docker compose run](#docker-compose-run)
- [docker compose exec](#docker-compose-exec)
- [docker compose stop](#docker-compose-stop)
- [docker compose start](#docker-compose-start)
- [docker compose down](#docker-compose-down)
- [docker compose logs](#docker-compose-logs)


## docker compose
   
### 一、先搞懂：docker compose 到底是什么？
`docker compose` 是 Docker 官方提供的**多容器应用编排工具**，核心作用是：
- 用一个 **YAML 文件（compose.yaml/docker-compose.yaml）** 定义多个容器的配置（镜像、端口、网络、卷、依赖关系等）；
- 用一条命令（如 `docker compose up`）一键启动/停止/管理整个应用集群，替代手动逐个运行 `docker run` 命令的繁琐操作。

简单说：它是多容器 Docker 应用的“一键启停+配置管理”工具，是开发、测试环境中管理多服务应用（如前端+后端+数据库）的标配。

### 二、核心选项深度拆解（对应你给出的 Options 表格）
下面逐个解释每个选项的**作用**、**使用场景**和**注意事项**，新手重点关注 `-f`、`-p`、`--profile` 这三个高频选项：

| 选项 | 核心作用 | 实际使用场景 | 关键说明 |
|------|----------|--------------|----------|
| `--all-resources` | 包含所有资源（即使服务未使用） | 排查/清理 Compose 管理的所有资源（如闲置网络、卷） | 日常用得少，主要用于运维排查 |
| `--ansi` | 控制 ANSI 颜色输出 | 脚本执行时禁用颜色（`--ansi never`），或强制显示（`--ansi always`） | 默认 `auto`：终端有颜色就显示，脚本/日志中自动禁用 |
| `--compatibility` | 兼容旧版 docker-compose（v1）语法 | 迁移旧版 `docker-compose.yml` 到新版 `compose.yaml` 时 | 新版 Compose（v2）已整合到 `docker compose`，旧版是 `docker-compose` 独立命令，此选项用于兼容旧配置 |
| `--dry-run` | 「试运行」模式，只输出执行步骤，不实际操作 | 验证命令是否正确（如 `up`/`pull`/`build` 前），避免误操作 | 不会创建容器/拉镜像/改配置，仅展示“会做什么”，新手调试必备 |
| `--env-file` | 指定自定义环境变量文件（默认读 `.env`） | 不同环境用不同变量（如开发/测试/生产） | 例：`docker compose --env-file .env.prod up` |
| `-f, --file` | 指定 Compose 配置文件（默认找 compose.yaml） | 1. 配置文件不在当前目录；2. 多文件组合配置；3. 从远程/OCI/ Git 加载配置 | 最常用选项之一，支持多个 `-f`（后文件覆盖前文件同字段） |
| `--parallel` | 控制并行操作的最大数量 | 限制镜像拉取/构建/容器启动的并发数（避免资源耗尽） | 默认 `-1`（无限制），例：`--parallel 2` 表示同时最多操作2个资源 |
| `--profile` | 启用指定的「配置文件」 | 启动可选服务（如开发环境需要 debug 服务，生产不需要） | 支持多个 `--profile`，也可通过 `COMPOSE_PROFILES` 环境变量设置 |
| `--progress` | 控制进度输出格式 | 脚本中输出 JSON（`--progress json`），或静默输出（`--progress quiet`） | 默认 `auto`，终端显示彩色进度，脚本显示简洁文本 |
| `--project-directory` | 指定项目工作目录（默认是第一个 `-f` 文件的目录） | 多文件组合时，统一所有文件的相对路径基准 | 避免路径混乱，例：`--project-directory /opt/myapp` |
| `-p, --project-name` | 指定项目名称（默认是目录名） | 区分同一主机上的多个相同应用（如测试/开发版） | 项目名会作为容器/网络/卷的前缀，例：`-p myapp-dev` |

### 三、示例场景解析（重点理解高频用法）
#### 场景1：`-f` 多文件组合（最常用）
核心逻辑：**后文件覆盖前文件的相同字段，新增字段追加**。
比如：
- `compose.yaml`（基础配置，所有环境通用）：定义 webapp 服务的基础镜像、端口；
- `compose.admin.yaml`（管理员配置，仅运维用）：覆盖 webapp 的 build 方式，新增环境变量。

执行命令：
```bash
docker compose -f compose.yaml -f compose.admin.yaml run backup_db
```
最终 webapp 服务的配置 = 基础配置 + 管理员配置的覆盖/追加。

还支持从远程加载配置：
```bash
# 从 Git 仓库加载
docker compose -f https://github.com/user/repo.git#main:compose.yaml up
# 从 OCI 镜像仓库加载
docker compose -f oci://registry.example.com/my-compose:v1 up
```

#### 场景2：`-p` 指定项目名（避免冲突）
假设你本地有两个相同的应用（开发版/测试版），默认用目录名做项目名会冲突，用 `-p` 区分：
```bash
# 启动开发版（前缀 myapp-dev）
docker compose -p myapp-dev up -d
# 启动测试版（前缀 myapp-test）
docker compose -p myapp-test up -d

# 查看开发版容器
docker compose -p myapp-dev ps
```
容器名会变成 `myapp-dev_web_1`、`myapp-test_web_1`，彻底避免冲突。

#### 场景3：`--profile` 启用可选服务
Compose 文件中可以给服务标记 `profile`，只有启用对应 profile 才会启动该服务：
```yaml
# compose.yaml
services:
  # 核心服务：无 profile，始终启动
  web:
    image: nginx
  # 可选服务：仅 dev profile 启用
  debug:
    image: busybox
    profiles: ["dev"]
  # 可选服务：仅 test profile 启用
  test:
    image: postgres
    profiles: ["test"]
```
执行命令：
```bash
# 只启动 web（无 profile）
docker compose up -d
# 启动 web + debug（启用 dev profile）
docker compose --profile dev up -d
# 启动 web + debug + test（启用多个 profile）
docker compose --profile dev --profile test up -d
```

#### 场景4：`--dry-run` 调试命令（新手必备）
执行前先看“会做什么”，避免误操作：
```bash
# 试运行 up 命令，查看会拉哪些镜像、创建哪些容器
docker compose --dry-run up -d
```
输出会标注 `DRY-RUN MODE`，告诉你每一步的操作，但不会实际执行，适合调试复杂配置。

#### 场景5：环境变量与选项的优先级
如果同时设置了“命令行选项”和“环境变量”，**命令行选项优先级更高**：
```bash
# 环境变量设置项目名
export COMPOSE_PROJECT_NAME=myapp-env
# 命令行设置另一个项目名，最终用命令行的 myapp-cli
docker compose -p myapp-cli ps
```

### 总结
1. `docker compose` 是多容器应用的编排工具，核心是用 YAML 文件统一管理容器配置；
2. 高频选项：`-f`（指定配置文件/多文件组合）、`-p`（指定项目名）、`--profile`（启用可选服务）、`--dry-run`（调试命令）；
3. 关键规则：多 `-f` 文件后覆盖前、命令行选项优先级高于环境变量、项目名会作为容器/网络前缀。

[目录](#目录)

## docker compose up

### 一、命令核心功能（通俗版）
`docker compose up` 是 Docker Compose 中最核心的命令，它的核心作用是**根据 `docker-compose.yml`（或 `compose.yml`）配置文件，一站式完成容器的「构建→创建→启动→日志关联」全流程**。
你可以把它理解为：给 Compose 下达“开工”指令，它会按配置文件的要求，把所有（或指定）服务的容器从无到有（或更新）地运行起来，并把所有容器的日志输出集中展示给你。

### 二、核心行为拆解
先理清这个命令默认的执行逻辑（不加任何参数时）：
1. **检查依赖**：先检查配置文件中定义的服务依赖（比如服务 A 依赖服务 B），如果依赖服务没运行，会先启动依赖。
2. **检查镜像**：如果服务对应的镜像本地没有，会先拉取（按 `--pull` 默认策略）；如果配置了 `build` 字段，会先构建镜像。
3. **容器处理**：
    - 如果容器不存在：直接创建并启动。
    - 如果容器已存在，但配置/镜像有变化：先停止旧容器，删除后重新创建并启动（保留挂载的卷）。
    - 如果容器已存在且配置无变化：直接启动（不重建）。
4. **日志聚合**：前台运行，把所有容器的日志输出整合到当前终端（像实时监控所有服务的控制台）。
5. **退出行为**：按 `Ctrl+C`（SIGINT）会停止所有容器并正常退出（退出码 0）；如果运行中出错（比如镜像拉取失败），退出码为 1。

### 三、关键参数详解（按使用频率+重要性排序）
下面把参数按“常用核心”“进阶配置”“特殊场景”分类，结合实际使用场景解释：

#### 1. 最常用的核心参数
| 参数 | 通俗解释             | 典型使用场景 |
|------|------------------|--------------|
| `-d` / `--detach` | 后台运行容器（“脱机”模式）   | 生产环境启动服务，不想占用终端，比如 `docker compose up -d` |
| `--build` | 启动前`强制构建/重新构建镜像`   | 代码/镜像配置改了，需要更新镜像后启动，比如 `docker compose up -d --build` |
| `--force-recreate` | 不管配置有没有变，`强制重建所有容器` | 想彻底重置容器（保留卷），比如 `docker compose up -d --force-recreate` |
| `--no-recreate` | 容器已存在就不重建（和上面相反） | 只想启动现有容器，避免误重建，比如 `docker compose up -d --no-recreate` |
| `--scale <服务名>=<数量>` | 扩展某个服务的实例数       | 多实例部署，比如 `docker compose up -d --scale web=3`（启动3个web容器） |
| `--remove-orphans` | 删除配置文件中已不存在的服务的容器 | 清理“孤儿”容器（比如之前删了某个服务配置），比如 `docker compose up -d --remove-orphans` |

#### 2. 进阶配置参数
| 参数 | 通俗解释 | 适用场景 |
|------|----------|----------|
| `--no-deps` | 只启动指定服务，不启动依赖服务 | 只想测试单个服务（比如依赖的数据库已手动启动），比如 `docker compose up -d web --no-deps` |
| `--exit-code-from <服务名>` | 以指定服务的退出码作为整个命令的退出码 | 集成测试（比如测试服务跑完退出，取它的结果），比如 `docker compose up --exit-code-from test` |
| `--abort-on-container-exit` | 只要有一个容器停止，就停所有容器 | 一次性任务（比如批量处理脚本），任务完成就全停 |
| `--pull <always/missing/never>` | 拉取镜像的策略 | 强制拉最新镜像：`--pull always`；本地没有才拉：`--pull missing`；从不拉：`--pull never` |
| `-V` / `--renew-anon-volumes` | 重建匿名卷（不保留旧数据） | 想清空匿名卷的临时数据，重置服务数据环境 |

#### 3. 日志/交互相关参数
| 参数 | 通俗解释 | 适用场景 |
|------|----------|----------|
| `--attach <服务名>` | 只查看指定服务的日志 | 服务太多，只想看核心服务（比如数据库）的日志，比如 `docker compose up --attach db` |
| `--no-attach <服务名>` | 不查看指定服务的日志 | 屏蔽日志刷屏的服务（比如日志量极大的web服务），比如 `docker compose up --no-attach web` |
| `--timestamps` | 日志中显示时间戳 | 排查问题时需要知道日志的时间点，比如 `docker compose up --timestamps` |
| `--no-log-prefix` | 日志不显示“服务名_容器ID”前缀 | 日志输出需要简洁，方便后续处理（比如重定向到文件） |

#### 4. 特殊场景参数
| 参数 | 通俗解释 | 适用场景 |
|------|----------|----------|
| `-w` / `--watch` | 监听源码文件变化，自动重建/重启容器 | 开发环境（改代码自动更新服务），比如 `docker compose up -d --watch` |
| `--wait` | 等待服务达到“运行/健康”状态才退出 | 自动化脚本（确保服务真的启动成功），比如 `docker compose up -d --wait` |
| `--wait-timeout <秒数>` | `--wait` 的超时时间 | 避免脚本无限等待，比如 `docker compose up -d --wait --wait-timeout 60`（等60秒超时） |
| `--no-start` | 只创建容器，不启动 | 只想预创建容器，后续手动启动（比如批量配置） |

### 四、典型使用示例
结合实际场景，给你几个最常用的命令组合：
1. **生产环境启动所有服务（后台运行，强制构建新镜像）**
   ```bash
   docker compose up -d --build
   ```
2. **开发环境启动web服务（只启动web，不启动依赖，后台运行，监听代码变化）**
   ```bash
   docker compose up -d web --no-deps --watch
   ```
3. **重置所有容器（强制重建，清理孤儿容器，重建匿名卷）**
   ```bash
   docker compose up -d --force-recreate --remove-orphans -V
   ```
4. **启动服务并等待健康状态（超时60秒）**
   ```bash
   docker compose up -d --wait --wait-timeout 60
   ```
5. **前台运行测试服务，测试完成后停所有容器，并取测试服务的退出码**
   ```bash
   docker compose up --exit-code-from test --abort-on-container-exit
   ```

### 总结
1. `docker compose up` 是 Compose 启动服务的核心命令，默认前台运行、聚合日志、自动处理依赖和容器重建。
2. 核心参数中，`-d`（后台）、`--build`（构建镜像）、`--force-recreate`（强制重建）是日常使用频率最高的，需重点掌握。
3. 参数有互斥性（比如 `--no-recreate` 和 `--force-recreate` 不能一起用），使用时需注意避免冲突，优先按场景选择组合参数。

掌握这些内容后，你就能根据开发/生产不同场景，灵活控制 Compose 服务的启动流程了。

[目录](#目录)




## docker compose run

### 一、核心概念：`docker compose run` 是什么？
`docker compose run` 是 Docker Compose 中用于**为指定服务临时运行一次性命令**的工具。简单来说：
- 它会基于 `docker-compose.yml` 中定义的服务配置（如镜像、卷挂载、环境变量、网络等），创建一个**新的临时容器**；
- 执行你指定的命令，命令执行完成后容器默认会保留（除非加 `--rm`）；
- 它和 `docker compose up` 的核心区别：`up` 是启动服务并执行服务配置里的默认命令，而 `run` 是临时覆盖默认命令，运行一次性任务。

### 二、核心特性（新手必懂）
#### 1. 继承服务配置，但覆盖默认命令
`run` 启动的容器会完全继承 `docker-compose.yml` 中对应服务的配置（卷、环境变量、网络等），但你指定的命令会覆盖服务原本的 `command` 配置。

**示例**：
假设 `docker-compose.yml` 中 `web` 服务的配置是：
```yaml
services:
  web:
    image: python:3.10
    command: python app.py  # 服务默认启动命令
    volumes:
      - ./app:/app  # 卷挂载
    environment:
      - DEBUG=true
```
执行 `docker compose run web bash`：
- 会创建一个基于 `web` 服务配置的新容器；
- 不执行 `python app.py`，而是执行 `bash`（进入容器交互终端）；
- 卷 `./app:/app`、环境变量 `DEBUG=true` 都会生效。

#### 2. 默认不映射端口（避免冲突）
`run` 命令**不会自动映射**服务配置中 `ports` 定义的端口（比如 `80:80`），这是为了避免和已运行的容器端口冲突。

- 如果需要映射服务配置的所有端口：用 `--service-ports`（简写 `-P`）
  ```bash
  docker compose run --service-ports web python manage.py runserver
  ```
- 如果需要手动映射指定端口：用 `--publish`（简写 `-p`），格式和 `docker run -p` 一致
  ```bash
  # 映射 8080（主机）:80（容器）、127.0.0.1:2021:21
  docker compose run --publish 8080:80 -p 127.0.0.1:2021:21 web bash
  ```

#### 3. 自动启动依赖服务（可禁用）
如果服务配置了 `depends_on`（依赖其他服务，比如 `web` 依赖 `db`），`run` 会先检查依赖服务是否运行，若未运行则自动启动。

- 禁用依赖启动：加 `--no-deps`
  ```bash
  # 只启动 web 容器执行命令，不启动依赖的 db 等服务
  docker compose run --no-deps web python manage.py shell
  ```

#### 4. 临时容器自动清理（`--rm`）
`run` 创建的容器默认会保留（即使命令执行完成），如果只是运行一次性任务（如数据库迁移），建议加 `--rm`，命令执行后自动删除容器，避免残留。

**示例**：
```bash
# 执行数据库升级，完成后删除容器
docker compose run --rm web python manage.py db upgrade
```

### 三、常用选项详解（按使用频率排序）
| 选项 | 核心作用 | 新手示例 |
|------|----------|----------|
| `--rm` | 命令执行后自动删除容器 | `docker compose run --rm web python app.py` |
| `--no-deps` | 不启动依赖服务 | `docker compose run --no-deps web bash` |
| `--service-ports`/`-P` | 映射服务配置的所有端口 | `docker compose run -P web python runserver` |
| `--publish`/`-p` | 手动映射指定端口 | `docker compose run -p 8080:80 web bash` |
| `-e`/`--env` | 临时设置环境变量（覆盖配置） | `docker compose run -e DEBUG=false web bash` |
| `--user`/`-u` | 指定容器运行的用户（避免权限问题） | `docker compose run -u 1000:1000 web bash` |
| `--entrypoint` | 覆盖镜像的入口命令 | `docker compose run --entrypoint sh web` |
| `-d`/`--detach` | 后台运行容器（不占用终端） | `docker compose run -d web python long_task.py` |

### 四、典型使用场景
1. **进入服务容器调试**：
   ```bash
   docker compose run --rm web bash  # 进入 web 容器的 bash 终端
   ```
2. **执行一次性任务（如数据库操作）**：
   ```bash
   # 进入 db 服务的 PostgreSQL 交互终端
   docker compose run --rm db psql -h db -U postgres
   # 执行 Django 数据库迁移
   docker compose run --rm web python manage.py migrate
   ```
3. **临时覆盖环境变量测试**：
   ```bash
   # 临时关闭 DEBUG 模式运行脚本
   docker compose run --rm -e DEBUG=false web python test.py
   ```
4. **手动映射端口测试服务**：
   ```bash
   # 临时映射 8888 端口，测试 web 服务
   docker compose run --rm -p 8888:80 web python app.py
   ```

### 总结
1. `docker compose run` 是**临时运行一次性命令**的工具，基于服务配置创建新容器，覆盖默认命令；
2. 核心特性：默认不映射端口、自动启动依赖服务、容器默认保留（建议加 `--rm` 清理）；
3. 高频选项：`--rm`（自动删容器）、`--no-deps`（禁用依赖）、`-p`/`-P`（端口映射）、`-e`（环境变量）。

这个命令的核心价值是：在不修改 `docker-compose.yml`、不影响已运行服务的前提下，快速基于现有服务配置执行临时任务，是日常调试、运维的高频工具。

[目录](#目录)


## docker compose exec

### 一、核心概念：`docker compose exec` 是什么？
`docker compose exec` 是 Docker Compose 中用于**在已经运行的容器内执行命令**的工具。简单来说：
- 它的核心前提是：目标服务的容器**必须已经处于运行状态**（比如通过 `docker compose up` 启动）；
- 它直接进入现有容器执行命令，**不会创建新容器**；
- 它等价于 `docker exec` 命令，但针对 Compose 服务做了封装，无需手动指定容器 ID/名称，只需指定服务名即可。

### 二、核心特性（新手必懂）
#### 1. 仅作用于「已运行」的容器
这是 `exec` 和 `run` 最核心的区别：
- `exec`：必须先有运行中的容器（比如 `docker compose up -d web` 启动了 `web` 容器），才能执行命令；
- `run`：不管服务是否运行，都会创建**新容器**执行命令。

**示例**：
如果 `web` 服务还没启动，执行 `docker compose exec web bash` 会直接报错：
```console
ERROR: No container found for web_1
```
必须先启动服务：
```bash
docker compose up -d web  # 启动 web 服务（创建并运行容器）
docker compose exec web bash  # 进入已运行的 web 容器执行 bash
```

#### 2. 默认分配交互式终端（TTY）
`docker compose exec` 会**默认开启交互式模式（-i）和 TTY 终端（-t）**，这意味着你可以直接进入容器的交互界面（比如 `bash`/`sh`），和在本地终端操作一样。

对比原生 `docker exec`：
- `docker compose exec web bash`（默认带 -it）
- 等价于 `docker exec -it <web容器ID> bash`（原生命令必须手动加 -it）

如果想禁用 TTY（比如在脚本中执行非交互式命令），可以加 `--no-tty`（简写 `-T`）：
```bash
# 脚本中执行命令，无需交互终端
docker compose exec -T web python manage.py migrate
```

#### 3. 支持多副本服务（--index）
如果你的服务通过 `deploy.replicas` 配置了多个副本（比如 3 个 `web` 容器），可以用 `--index` 指定具体操作第几个副本：
```bash
# 操作第 2 个 web 副本容器（索引从 1 开始）
docker compose exec --index 2 web bash
```

### 三、常用选项详解（按使用频率排序）
| 选项 | 核心作用 | 新手示例 |
|------|----------|----------|
| `-T`/`--no-tty` | 禁用 TTY 终端（适合脚本） | `docker compose exec -T web python test.py` |
| `-e`/`--env` | 临时设置环境变量 | `docker compose exec -e DEBUG=true web bash` |
| `-u`/`--user` | 指定执行命令的用户（解决权限问题） | `docker compose exec -u root web apt update` |
| `-w`/`--workdir` | 指定命令执行的工作目录 | `docker compose exec -w /app web python app.py` |
| `-d`/`--detach` | 后台执行命令（不占用终端） | `docker compose exec -d web nohup python long_task.py &` |
| `--index` | 指定多副本服务的容器索引 | `docker compose exec --index 2 web bash` |
| `--privileged` | 给命令赋予扩展权限（慎用） | `docker compose exec --privileged web mount /dev/sda1 /mnt` |

### 四、典型使用场景
1. **进入运行中的容器调试**（最常用）：
   ```bash
   # 进入 web 容器的 bash 终端，实时查看日志/修改文件
   docker compose exec web bash
   # 进入数据库容器，执行交互式 SQL
   docker compose exec db mysql -u root -p
   ```
2. **在运行中的容器执行一次性命令**：
   ```bash
   # 查看 web 容器的进程
   docker compose exec web ps aux
   # 查看容器内的日志文件
   docker compose exec web cat /var/log/app.log
   # 给运行中的容器安装依赖（临时调试）
   docker compose exec -u root web apt install vim
   ```
3. **脚本中批量操作运行中的容器**：
   ```bash
   # 脚本中执行数据库备份，禁用 TTY 避免交互
   docker compose exec -T db mysqldump -u root -p123456 mydb > backup.sql
   ```
4. **操作多副本服务的指定容器**：
   ```bash
   # 查看第 3 个 web 副本的资源使用情况
   docker compose exec --index 3 web top
   ```

### 五、`exec` vs `run` 核心区别（新手必记）
| 维度 | `docker compose exec` | `docker compose run` |
|------|-----------------------|----------------------|
| 容器状态 | 作用于**已运行**的容器 | 创建**新容器**执行命令 |
| 端口映射 | 继承容器已有的端口映射 | 默认不映射端口（需手动指定） |
| 依赖服务 | 不启动依赖（容器已运行） | 默认启动依赖服务（可加 --no-deps 禁用） |
| 容器生命周期 | 命令执行后，容器仍运行 | 命令执行后，容器默认保留（加 --rm 可删除） |
| 适用场景 | 调试运行中的服务、临时操作现有容器 | 执行一次性任务、测试服务配置、不影响现有容器 |

### 总结
1. `docker compose exec` 是**操作已运行容器**的工具，直接进入现有容器执行命令，不创建新容器；
2. 核心特性：默认带交互式终端（适合手动调试），支持多副本容器指定，需服务先启动；
3. 核心区别：`exec` 操作「已有运行容器」，`run` 创建「新临时容器」，根据是否要保留现有容器状态选择使用。

[目录](#目录)


## docker compose stop
## docker compose start
### 一、核心概念：`stop` + `start` 是什么？
这两个命令是 Docker Compose 中用于**管理容器「停止-启动」生命周期**的核心工具，核心特点是：
- `docker compose stop`：停止**已运行**的容器，但**不会删除容器**（容器的文件、数据、配置都保留）；
- `docker compose start`：重新启动**已停止**的容器（复用之前的容器实例，而非创建新容器）；
- 两者是「可逆操作」，区别于 `down`（停止并删除容器）和 `up`（创建并启动容器）。

---

## 1. `docker compose stop` 详细解析
### 作用
优雅停止指定服务（或所有服务）的运行中容器，容器会进入「Exited」状态，但容器本身、挂载的卷、网络配置都不会被删除。

### 基本用法
```bash
# 停止所有服务的容器
docker compose stop

# 停止指定服务的容器（比如只停 web 服务）
docker compose stop web

# 停止多个指定服务的容器
docker compose stop web db
```

### 核心选项：`-t/--timeout`
指定容器的「优雅关闭超时时间」（单位：秒）。Docker 会先向容器发送 `SIGTERM` 信号（让应用优雅退出，比如保存数据、关闭连接），如果超时容器仍未停止，会发送 `SIGKILL` 信号强制杀死容器。

**示例**：
```bash
# 给 web 容器 30 秒优雅关闭时间，超时则强制停止
docker compose stop -t 30 web
```
- 默认超时时间：Compose 未显式指定时，继承 Docker 引擎的默认值（通常是 10 秒）；
- 新手建议：如果你的应用关闭需要更多时间（比如数据库、后端服务），手动指定 `-t` 避免强制杀死导致数据损坏。

### 执行效果
```console
# 执行 stop 后查看容器状态
$ docker compose ps
NAME                COMMAND                  SERVICE             STATUS              PORTS
myapp-web-1         "python app.py"          web                 exited (0)          
myapp-db-1          "docker-entrypoint.s…"   db                  exited (0)          
```
- 状态显示 `exited (0)`：容器正常停止（退出码 0）；
- 容器 ID、名称、挂载的卷都不变，只是停止运行。

---

## 2. `docker compose start` 详细解析
### 作用
重新启动通过 `stop` 停止的「已存在但未运行」的容器，复用原容器的所有配置（卷、网络、环境变量等），不会创建新容器。

### 基本用法
```bash
# 启动所有已停止的服务容器
docker compose start

# 启动指定服务的容器（比如只启动 db 服务）
docker compose start db

# 启动多个指定服务的容器
docker compose start web db
```

### 核心选项
| 选项 | 作用 | 新手示例 |
|------|------|----------|
| `--wait` | 等待服务达到「运行中/健康」状态（隐含后台运行 `-d`） | `docker compose start --wait web` |
| `--wait-timeout` | 指定 `--wait` 的最大等待超时时间（单位：秒） | `docker compose start --wait --wait-timeout 60 web` |

#### 关键：`--wait` 选项的使用场景
如果你的服务配置了「健康检查」（`healthcheck`），`--wait` 会等待服务满足健康状态后再返回，避免启动后服务还没就绪就执行后续操作。

**示例**（假设 `db` 服务配置了健康检查）：
```bash
# 启动 db 服务，并等待其达到健康状态（最多等 60 秒）
docker compose start --wait --wait-timeout 60 db
echo "数据库已就绪，可以执行迁移了"
```

### 执行效果
```console
# 执行 start 后查看容器状态
$ docker compose start web
[+] Running 1/1
 ✔ Container myapp-web-1  Started                                                                                                                               0.5s

$ docker compose ps
NAME                COMMAND                  SERVICE             STATUS              PORTS
myapp-web-1         "python app.py"          web                 running             0.0.0.0:8000->8000/tcp
myapp-db-1          "docker-entrypoint.s…"   db                  exited (0)          
```
- 容器复用原实例（名称还是 `myapp-web-1`），只是状态从 `exited` 变为 `running`；
- 端口、卷挂载等配置和停止前完全一致。

---

### 二、核心特性与新手注意事项
#### 1. `stop/start` vs `down/up` 核心区别（新手必记）
这是最容易混淆的点，用表格清晰对比：

| 维度 | `stop` + `start` | `down` + `up` |
|------|------------------|---------------|
| 容器实例 | 复用原有容器（不删除） | 删除旧容器，创建新容器 |
| 数据保留 | 容器内数据、卷挂载完全保留 | 卷挂载的数据保留（除非加 `-v`），但容器实例被删除 |
| 网络/端口 | 网络配置不变，启动后端口自动恢复 | 重新创建网络，端口重新映射 |
| 适用场景 | 临时停止服务（比如服务器重启、临时维护），快速恢复 | 修改服务配置后重启（需重建容器）、彻底清理环境 |

**新手示例**：
- 场景 1：临时关机维护服务器 → 用 `docker compose stop`，开机后 `docker compose start`（10 秒恢复服务，数据不丢）；
- 场景 2：修改了 `docker-compose.yml` 中 web 服务的环境变量 → 用 `docker compose down` + `docker compose up -d`（重建容器使配置生效）。

#### 2. `start` 无法启动「未创建」的容器
`docker compose start` 仅能启动「已停止但存在」的容器，如果容器从未通过 `up`/`run` 创建过，执行 `start` 会报错：
```console
$ docker compose start web
ERROR: No containers to start for service "web"
```
解决：先执行 `docker compose up -d web` 创建并启动容器，之后才能用 `start/stop` 管理。

#### 3. `stop` 不会影响容器的「重启策略」
如果你的服务配置了 `restart: always`（容器退出后自动重启），执行 `docker compose stop` 后，容器不会自动重启（`stop` 是「手动停止」，会覆盖重启策略）；只有容器意外退出（比如崩溃），重启策略才会生效。

---

### 三、典型使用场景
#### 1. 临时维护服务器（最常用）
```bash
# 维护前停止所有服务
docker compose stop

# 执行维护操作（比如升级系统、清理磁盘）

# 维护完成后启动所有服务，等待健康状态
docker compose start --wait
```

#### 2. 单独停止/启动某个服务（不影响其他服务）
```bash
# 临时停止数据库服务（比如备份数据）
docker compose stop db

# 备份数据库文件
docker compose exec db mysqldump -u root -p mydb > backup.sql

# 启动数据库服务
docker compose start db
```

#### 3. 脚本中确保服务启动并就绪
```bash
#!/bin/bash
# 启动 web 服务，并等待其健康（最多等 60 秒）
docker compose start --wait --wait-timeout 60 web

# 服务就绪后执行后续操作
curl http://localhost:8000/health
```

---

### 总结
1. `docker compose stop`：**优雅停止运行中容器**，保留容器实例和所有数据，可通过 `-t` 指定优雅关闭超时；
2. `docker compose start`：**重启已停止的容器**，复用原容器配置，`--wait` 可等待服务达到健康状态；
3. 核心区别：`stop/start` 是「复用容器的可逆操作」，`down/up` 是「重建容器的不可逆操作」，根据是否需要保留容器实例选择使用。

这两个命令是日常维护中「临时启停服务」的首选，记住「stop 停容器不删容器，start 启容器不复建」的核心逻辑即可。

[目录](#目录)


## docker compose ps

### 一、核心概念：`docker compose ps` 是什么？
`docker compose ps` 是 Docker Compose 中用于**查看当前 Compose 项目下所有容器状态**的核心命令，相当于 Compose 版的 `docker ps`。它能清晰展示每个容器的名称、所属服务、运行状态、端口映射、创建时间等关键信息，是日常排查容器问题、确认服务运行状态的首选工具。

### 二、基础用法与默认输出解析
#### 1. 基础命令
```bash
# 查看当前 Compose 项目下「运行中」的容器（默认只显示运行态）
docker compose ps

# 查看指定服务的容器（比如只看 db 服务）
docker compose ps db

# 查看多个指定服务的容器
docker compose ps web db
```

#### 2. 默认输出字段详解
以官方示例为例，先看懂每一列的含义：
```console
NAME            IMAGE     COMMAND           SERVICE    CREATED         STATUS          PORTS
example-foo-1   alpine    "/entrypoint.…"   foo        4 seconds ago   Up 2 seconds    0.0.0.0:8080->80/tcp
```
| 字段 | 含义 | 新手解读 |
|------|------|----------|
| `NAME` | 容器名称 | Compose 自动生成，格式：`项目名-服务名-副本索引`（比如 `example-foo-1` 是 example 项目的 foo 服务第 1 个副本） |
| `IMAGE` | 容器使用的镜像 | 比如 `alpine`、`mysql:8.0` |
| `COMMAND` | 容器启动命令 | 显示核心命令（超长会截断，加 `--no-trunc` 可看完整） |
| `SERVICE` | 所属服务名 | 对应 `docker-compose.yml` 里的 `services` 下的名称（比如 `foo`） |
| `CREATED` | 容器创建时间 | 比如 `4 seconds ago`（4 秒前）、`2 hours ago`（2 小时前） |
| `STATUS` | 容器状态 | 核心字段！常见值：`Up 2 seconds`（运行中）、`exited (0)`（正常退出）、`exited (1)`（异常退出）、`restarting`（重启中） |
| `PORTS` | 端口映射 | 格式：`主机IP:主机端口->容器端口/协议`（比如 `0.0.0.0:8080->80/tcp` 表示主机所有IP的8080端口映射到容器80端口，TCP协议）；无映射则为空 |

### 三、核心选项详解（按使用频率排序）
#### 1. `-a/--all`：显示所有容器（包括已停止的）
默认只显示「运行中」的容器，加 `-a` 能看到所有状态的容器（包括 `docker compose stop` 停止的、`docker compose run` 创建的临时容器），是排查「容器是否存在」的关键选项。

**示例**：
```bash
docker compose ps -a
```
输出示例：
```console
NAME            IMAGE     COMMAND           SERVICE    CREATED         STATUS          PORTS
example-foo-1   alpine    "/entrypoint.…"   foo        4 seconds ago   Up 2 seconds    0.0.0.0:8080->80/tcp
example-bar-1   alpine    "/entrypoint.…"   bar        4 seconds ago   exited (0)      # 已停止的容器
```

#### 2. `--status`：按状态筛选容器
指定状态过滤，只显示符合条件的容器，支持的状态值：`paused`（暂停）、`restarting`（重启中）、`removing`（删除中）、`running`（运行中）、`dead`（僵死）、`created`（已创建未启动）、`exited`（已退出）。

**示例**：
```bash
# 只看运行中的容器（等价于默认不带参数）
docker compose ps --status=running

# 只看已退出的容器（排查停止的服务）
docker compose ps --status=exited

# 只看重启中的容器（排查服务崩溃重启问题）
docker compose ps --status=restarting
```

#### 3. `--filter`：过滤容器（目前仅支持 status 过滤）
`--filter` 是更通用的过滤参数，目前仅支持 `status=<状态>`（和 `--status` 等价），未来可能支持更多过滤维度。

**示例**：
```bash
# 等价于 --status=exited
docker compose ps --filter status=exited
```

#### 4. `--format`：定制输出格式
支持自定义输出格式，满足不同场景（比如脚本解析、简洁展示），核心取值：
| 格式值 | 作用 | 适用场景 |
|--------|------|----------|
| `table`（默认） | 表格格式（带表头） | 手动查看，直观 |
| `json` | JSON 格式输出 | 脚本解析（比如用 `jq` 处理） |
| `table TEMPLATE`/`TEMPLATE` | 自定义 Go 模板 | 按需展示指定字段 |

**示例1：JSON 格式输出（脚本解析）**
```bash
# 输出 JSON 格式，方便后续处理
docker compose ps --format json

# 结合 jq 工具美化 JSON（需先安装 jq）
docker compose ps --format json | jq .
```
输出会包含容器完整信息（ID、名称、状态、端口映射等），适合脚本批量处理。

**示例2：只显示容器名称和状态（自定义模板）**
```bash
# 自定义模板，只展示 Name 和 State 字段
docker compose ps --format "table {{.Name}}\t{{.State}}"
```
输出示例：
```console
NAME            STATE
example-foo-1   running
example-bar-1   exited (0)
```

#### 5. `--no-trunc`：不截断超长输出
默认输出中，`COMMAND` 等字段超长会用 `…` 截断，加 `--no-trunc` 可看完整内容。

**示例**：
```bash
docker compose ps --no-trunc
```
输出中 `COMMAND` 字段会显示完整的启动命令，而非 `/entrypoint.…`。

#### 6. `-q/--quiet`：只显示容器 ID
仅输出容器的短 ID，适合和其他命令结合（比如批量停止/删除容器）。

**示例**：
```bash
# 只显示容器 ID
docker compose ps -q

# 结合 docker stop 批量停止当前项目的所有容器
docker stop $(docker compose ps -q)
```

#### 7. `--services`：只显示服务名
仅列出 Compose 项目中的服务名称，而非容器信息。

**示例**：
```bash
docker compose ps --services
```
输出示例：
```console
foo
bar
db
```

#### 8. `--orphans`：包含孤立服务容器
「孤立服务」指不在当前 `docker-compose.yml` 中定义，但属于该项目的容器（比如之前删除了服务配置，但容器还在）。默认 `--orphans=true` 会显示，设为 `--orphans=false` 则隐藏。

### 四、典型使用场景（新手必会）
#### 1. 日常巡检：确认服务是否正常运行
```bash
# 查看所有运行中的服务
docker compose ps

# 关键看 STATUS 列：
# - Up X seconds/minutes：正常运行
# - exited (非0)：异常退出（比如 exited (1) 大概率是服务启动失败）
# - restarting：服务频繁重启（需排查日志）
```

#### 2. 排查端口映射问题
```bash
# 查看端口映射是否正确（比如确认 8080 是否映射到容器 80）
docker compose ps
# 看 PORTS 列：0.0.0.0:8080->80/tcp 表示映射正确
```

#### 3. 脚本批量处理容器
```bash
#!/bin/bash
# 1. 筛选出已退出的容器 ID
EXITED_IDS=$(docker compose ps --status=exited -q)

# 2. 如果有已退出的容器，删除它们
if [ -n "$EXITED_IDS" ]; then
  docker rm $EXITED_IDS
  echo "已删除所有已退出的容器"
else
  echo "无已退出的容器"
fi
```

#### 4. 查看容器完整启动命令（排查启动参数问题）
```bash
# 不截断输出，查看完整的启动命令
docker compose ps --no-trunc
# 对比 COMMAND 字段和预期的启动参数是否一致
```

### 五、新手常见问题
#### 1. 执行 `docker compose ps` 无输出？
- 原因：当前目录没有 `docker-compose.yml`，或没有运行中的容器；
- 解决：切换到 Compose 项目目录，或加 `-a` 查看所有容器（包括已停止的）。

#### 2. 容器 STATUS 显示 `exited (1)`？
- 原因：服务启动失败（比如配置错误、依赖缺失）；
- 解决：用 `docker compose logs <服务名>` 查看日志，排查启动失败原因。

#### 3. 容器 STATUS 显示 `restarting`？
- 原因：服务启动后立即崩溃，Compose 按重启策略反复重启；
- 解决：`docker compose logs <服务名>` 查看崩溃日志，修复后 `docker compose restart <服务名>`。

### 总结
1. `docker compose ps` 是 Compose 项目的「容器状态总览工具」，默认显示运行中容器的名称、状态、端口等核心信息；
2. 核心选项：`-a`（显示所有容器）、`--status`（按状态筛选）、`--format`（定制输出）、`-q`（只显示容器ID）；
3. 核心用途：日常巡检服务状态、排查端口/启动问题、脚本批量处理容器，重点关注 `STATUS` 和 `PORTS` 列。

记住：`docker compose ps` 是「看状态」的第一步，发现异常后，再用 `docker compose logs` 查日志，这是排查 Compose 服务问题的标准流程。

[目录](#目录)


## docker compose down

### 一、命令核心作用
`docker compose down` 是 Docker Compose 中用于**彻底清理**由 `docker compose up` 启动的容器环境的核心命令，
它的核心行为是：先停止运行中的容器，再删除这些容器、相关的网络（默认规则），可选删除卷和镜像。

简单类比：如果把 `docker compose up` 看作“启动一套完整的应用服务（比如前端+后端+数据库）”，那 `docker compose down` 就是“把这套服务彻底关停并清理掉”，而不是简单暂停（`docker compose stop` 只是暂停容器，容器还在）。

### 二、默认行为（不加任何参数）
执行 `docker compose down` 时，默认只会删除以下内容（**重点：默认不删卷、不删镜像**）：
1. **容器**：Compose 文件中定义的所有服务对应的容器（比如 `docker-compose.yml` 里的 `web`、`db` 服务容器）；
2. **网络**：
   - Compose 文件中 `networks` 段定义的自定义网络；
   - 如果服务使用了默认网络（没手动指定网络时 Compose 自动创建的网络），这个默认网络也会被删；
3. **不会删的内容（默认）**：
   - 外部网络/外部卷（在 Compose 文件中标记为 `external: true` 的网络/卷）；
   - 匿名卷（比如容器内未指定持久化路径的临时数据卷）；
   - 所有镜像（不管是本地构建的还是拉取的）；
   - “孤儿容器”（比如之前 Compose 文件里有，但现在删掉的服务对应的容器）。

### 三、各参数详解（Options）
| 参数 | 作用 | 实用示例 | 注意事项 |
|------|------|----------|----------|
| `--remove-orphans` | 删除“孤儿容器”（Compose 文件中已不存在的服务对应的容器） | `docker compose down --remove-orphans` | 比如你之前 Compose 文件有 `redis` 服务，后来删掉了这部分配置，执行此命令会把残留的 `redis` 容器删掉 |
| `--rmi <取值>` | 删除服务使用的镜像，取值有两种：<br>- `local`：只删“本地镜像”（没有自定义标签的镜像，比如 Compose 构建的镜像）；<br>- `all`：删所有相关镜像（包括拉取的官方镜像如 `mysql:8.0`） | `docker compose down --rmi local`（推荐）<br>`docker compose down --rmi all`（慎用） | `--rmi all` 会删除拉取的镜像，下次 `up` 要重新下载，耗时较长 |
| `-t/--timeout <秒数>` | 指定容器关停的超时时间（默认10秒） | `docker compose down -t 30` | 给容器足够的时间优雅关闭（比如保存数据、释放资源），超时后会强制杀死容器 |
| `-v/--volumes` | 删除两类卷：<br>1. Compose 文件中 `volumes` 段定义的**命名卷**；<br>2. 绑定到容器的**匿名卷** | `docker compose down -v` | **高危操作**：会删除卷中的所有数据（比如数据库数据），执行前务必确认数据已备份 |

### 四、典型使用场景示例
#### 场景1：常规清理（只删容器和网络，保留数据卷）
```bash
# 最常用的方式，清理容器和网络，保留卷（比如数据库数据）
docker compose down
```

#### 场景2：彻底清理（包括数据卷，谨慎使用）
```bash
# 停止并删除容器、网络，同时删除命名卷和匿名卷
docker compose down -v
```

#### 场景3：清理容器+网络+本地构建的镜像
```bash
# 清理容器、网络，同时删除Compose构建的本地镜像（不删拉取的镜像）
docker compose down --rmi local
```

#### 场景4：优雅关停+清理孤儿容器
```bash
# 给容器30秒优雅关停时间，同时删除孤儿容器
docker compose down -t 30 --remove-orphans
```

### 五、关键对比（避免和其他命令混淆）
| 命令 | 核心区别 |
|------|----------|
| `docker compose down` | 停止容器 + 删除容器/网络（可选删卷/镜像），彻底清理环境 |
| `docker compose stop` | 只停止容器，容器、网络、卷都保留，可通过 `start` 恢复 |
| `docker compose rm` | 只删除已停止的容器，不停止运行中的容器，也不删网络/卷 |

### 总结
1. `docker compose down` 是 Docker Compose 环境的“彻底清理命令”，核心是**停容器+删容器+删网络**，默认不删卷和镜像；
2. 核心参数：`-v` 删卷（高危）、`--rmi local` 删本地构建镜像、`--remove-orphans` 删孤儿容器；
3. 区别于 `stop`（只暂停）和 `rm`（只删已停容器），`down` 是“一站式清理”，适合重新部署、环境重置场景。

[目录](#目录)

## docker compose logs

### 一、命令核心作用
`docker compose logs` 是 Docker Compose 中专门用来**查看一个或多个服务容器日志**的核心命令，它能聚合展示 Compose 管理的所有（或指定）服务的日志输出，是排查容器运行问题、查看服务运行状态的最常用工具之一。

简单类比：如果把容器比作运行中的“小程序”，这个命令就相当于“查看小程序的运行日志/控制台输出”，能看到容器里程序的打印信息、报错内容、运行状态等。

### 二、默认行为（不加任何参数）
执行 `docker compose logs` 时，默认会：
1. 输出**所有**在 Compose 文件中定义的服务容器的日志；
2. 日志会带上**前缀**（默认格式：`服务名_容器ID片段`），方便区分不同服务的日志；
3. 输出**全部历史日志**（从容器启动到执行命令时的所有日志），输出完成后命令就结束（不会持续监听）；
4. 日志带颜色（不同服务的日志用不同颜色区分），无时间戳。

### 三、各参数详解（Options）
| 参数 | 作用 | 实用示例 | 关键说明 |
|------|------|----------|----------|
| `-f/--follow` | 实时跟踪（监听）日志输出（类似 Linux 的 `tail -f`） | `docker compose logs -f` | 执行后不会退出，会持续打印新产生的日志，按 `Ctrl+C` 停止监听 |
| `--index` | 指定查看服务多副本（replicas）中某个容器的日志 | `docker compose logs --index 2 web` | 比如 `web` 服务启动了3个副本（容器），`--index 2` 只看第2个容器的日志（索引从1开始） |
| `--no-color` | 关闭日志的彩色输出，只显示黑白文本 | `docker compose logs --no-color` | 适合将日志重定向到文件（如 `docker compose logs --no-color > app.log`），避免文件中混入颜色控制符 |
| `--no-log-prefix` | 不显示日志前缀（默认前缀是 `服务名_容器ID`） | `docker compose logs --no-log-prefix web` | 日志只保留原始内容，去掉 `web_1 |` 这类前缀，适合纯文本分析 |
| `--since` | 只显示指定时间**之后**的日志，支持两种格式：<br>1. 绝对时间：`2026-02-28T10:00:00`（ISO格式）、`2026-02-28 10:00:00`；<br>2. 相对时间：`10m`（10分钟前）、`1h`（1小时前）、`2d`（2天前） | `docker compose logs --since 1h web`（查看web服务近1小时日志）<br>`docker compose logs --since 2026-02-28T09:00:00 db` | 快速过滤近期日志，避免翻找大量历史内容 |
| `-n/--tail` | 只显示每个容器日志的**最后N行** | `docker compose logs -n 100`（显示所有服务最后100行日志）<br>`docker compose logs -n 50 web`（显示web服务最后50行日志） | 默认值是 `all`，即显示全部日志 |
| `-t/--timestamps` | 在每条日志前添加**精确时间戳**（格式：`YYYY-MM-DDTHH:MM:SS.sssZ`） | `docker compose logs -t web` | 排查时间相关问题（比如某个时间点的报错）时非常有用 |
| `--until` | 只显示指定时间**之前**的日志，格式和 `--since` 一致 | `docker compose logs --until 30m web`（查看web服务30分钟前及更早的日志）<br>`docker compose logs --since 1h --until 30m web`（查看web服务1小时前到30分钟前的日志） | 可和 `--since` 组合使用，精准截取某段时间的日志 |

### 四、典型使用场景示例
#### 场景1：查看单个服务的全部日志
```bash
# 查看db服务（比如数据库）的所有日志
docker compose logs db
```

#### 场景2：实时监听多个服务的日志（带时间戳）
```bash
# 实时查看web和db服务的日志，且每条日志带时间戳
docker compose logs -f -t web db
```

#### 场景3：截取指定时间段的日志并保存到文件
```bash
# 查看web服务2026-02-28 09:00到10:00的日志，去掉颜色和前缀，保存到文件
docker compose logs --since 2026-02-28T09:00:00 --until 2026-02-28T10:00:00 --no-color --no-log-prefix web > web_09_10.log
```

#### 场景4：查看多副本服务中某个容器的最新日志
```bash
# 查看web服务第3个副本容器的最后50行日志，带时间戳
docker compose logs --index 3 -n 50 -t web
```

#### 场景5：查看近30分钟的所有服务日志
```bash
# 查看所有服务近30分钟的日志，只显示最后200行
docker compose logs --since 30m -n 200
```

### 五、实用小技巧
1. **日志过长时分页查看**：结合 Linux 的 `less` 命令，方便翻页：
   ```bash
   docker compose logs -t web | less
   ```
2. **过滤关键字日志**：结合 `grep` 查找报错（比如找包含 `error` 的日志）：
   ```bash
   docker compose logs -t web | grep -i error  # -i 忽略大小写
   ```
3. **区分日志来源**：默认的彩色前缀（如 `web_1 |`、`db_1 |`）能快速区分不同服务/容器的日志，建议非文件输出时保留颜色。

### 总结
1. `docker compose logs` 是查看 Compose 服务日志的核心命令，支持单/多服务、实时/历史日志查看；
2. 核心参数：`-f` 实时监听、`-t` 显示时间戳、`--since/--until` 过滤时间段、`-n` 限制日志行数；
3. 实用组合：`-f -t` 实时监控带时间戳、`--since --until` 截取指定时段日志、结合 `grep/less` 增强日志分析能力。

[目录](#目录)
