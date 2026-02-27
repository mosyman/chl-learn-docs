


### 一、Version 顶级元素：已废弃的兼容性字段
#### 1. 核心定义与状态
`version` 是 Compose 文件中**历史用于声明「Compose 规范版本」** 的顶级字段，但目前已被标记为「过时（obsolete）」—— 仅为了向后兼容保留，无实际功能作用。

#### 2. 关键特性（必记）
##### （1）仅为「信息性字段」，无校验作用
- 无论你设置 `version: "3.8"` 还是 `version: "2.4"`，Compose 都会**忽略该值**，始终使用「最新的 Compose 规范」来解析和验证整个 Compose 文件；
- 若使用该字段，Compose 会输出「版本字段已废弃」的警告（但不会影响文件执行）。

##### （2）字段不匹配的警告逻辑
- 若 Compose 文件中使用了「更新版本规范才支持的字段」（比如新版的 `profiles`/`secrets`），即使你设置了旧的 `version`，Compose 不会因为版本不匹配报错；
- 但如果 Compose 无法识别某些字段（比如拼写错误、字段属于更新版规范），会输出「未知字段」的警告，而非基于 `version` 校验。

#### 3. 历史用途与现在的替代方案
- **历史作用**：早期 Compose 版本（如 Compose V1）中，`version` 决定了 Compose 使用的解析器版本（不同版本支持不同字段）；
- **现在的处理**：Compose V2 及以上版本统一使用最新规范，无需声明 `version`，建议直接删除该字段以避免警告。

**示例（不推荐的写法）**：
```yml
# 会触发 "version is obsolete" 警告
version: "3.9"  # 无实际作用
services:
  web:
    image: nginx
```

**推荐写法**：
```yml
# 直接删除 version 字段
services:
  web:
    image: nginx
```

### 二、Name 顶级元素：定义 Compose 项目名称
#### 1. 核心定义
`name` 是 Compose 文件中**用于声明「默认项目名称」** 的顶级字段，作用是统一标识当前 Compose 管理的所有容器、网络、卷等资源的前缀。

- 若未显式设置 `name`，Compose 会使用「Compose 文件所在目录的名称」作为默认项目名；
- 项目名会作为资源命名的前缀（如容器名：`myapp_web_1`，网络名：`myapp_default`），避免多项目资源冲突。

#### 2. 基本语法
```yml
# 顶级字段，与 services/volumes/networks 同级
name: <project-name>
```
- `<project-name>`：字符串，建议使用小写字母、数字、连字符（`-`），避免特殊字符。

#### 3. 核心特性与规则
##### （1）项目名的优先级（从高到低）
Compose 会按以下顺序确定最终项目名（优先级高的覆盖低的）：
1. 运行 `docker compose` 时通过 `--project-name`/`-p` 参数显式指定（最高优先级）；
2. Compose 文件中 `name` 顶级字段定义的值；
3. 环境变量 `COMPOSE_PROJECT_NAME`；
4. Compose 文件所在目录的名称（默认）。

**示例**：
```bash
# 覆盖 Compose 文件中的 name 字段，项目名改为 "myapp-dev"
docker compose --project-name myapp-dev up
```

##### （2）插值与环境变量：`COMPOSE_PROJECT_NAME`
无论项目名通过哪种方式确定，Compose 都会将其暴露为环境变量 `COMPOSE_PROJECT_NAME`，可在 Compose 文件中通过「插值语法 `${COMPOSE_PROJECT_NAME}`」引用。

**完整示例**：
```yml
# 定义项目名
name: myapp

services:
  foo:
    image: busybox
    # 引用项目名环境变量
    command: echo "I'm running in project ${COMPOSE_PROJECT_NAME}"
  bar:
    image: nginx
    # 容器名包含项目名（也可手动拼接）
    container_name: ${COMPOSE_PROJECT_NAME}_web
```
运行结果：
```bash
$ docker compose up
# foo 容器输出：I'm running in project myapp
# bar 容器名：myapp_web
```

##### （3）多 Compose 文件的项目名规则
若使用 `-f` 指定多个 Compose 文件：
- 仅「第一个文件」中的 `name` 字段生效；
- 若多个文件都定义 `name`，仅第一个会被使用，其余被忽略；
- 建议在「基础文件」中统一定义 `name`，其他文件（如 `compose.override.yml`）不重复定义。

#### 4. 典型使用场景
##### （1）统一项目资源命名
避免不同项目的资源冲突（比如两个项目都有 `web` 服务，项目名不同则容器名分别为 `myapp_web_1` 和 `yourapp_web_1`）：
```yml
name: ecommerce
services:
  web:
    image: nginx
  db:
    image: mysql
    volumes:
      - ecommerce-data:/var/lib/mysql
volumes:
  ecommerce-data:
```
- 容器名：`ecommerce_web_1`、`ecommerce_db_1`；
- 卷名：`ecommerce_ecommerce-data`；
- 网络名：`ecommerce_default`。

##### （2）动态引用项目名配置资源
比如将项目名作为日志目录的一部分，方便区分不同项目的日志：
```yml
name: payment-service
services:
  api:
    image: my-api:latest
    volumes:
      - ./logs/${COMPOSE_PROJECT_NAME}:/var/log/api
```
- 日志目录：`./logs/payment-service`，不同项目的日志会自动隔离。

### 三、关键注意事项
1. **`version` 字段的处理**：建议直接删除 `version` 字段，避免 Compose 输出警告，Compose 会自动使用最新规范；
2. **`name` 字段的命名规范**：项目名避免使用大写字母、空格、特殊符号（如 `$`/`&`），否则可能导致资源命名失败；
3. **插值语法的生效范围**：`${COMPOSE_PROJECT_NAME}` 可在 Compose 文件的任意支持插值的字段中使用（如 `image`/`command`/`container_name`/`volumes` 等）；
4. **项目名修改的影响**：修改项目名后，Compose 会将其视为「新项目」，会创建新的容器/网络/卷（旧项目资源不会自动删除，需手动清理）。

### 总结
#### 1. Version 顶级元素关键点
- 已废弃，仅为向后兼容保留，无实际校验/解析作用；
- Compose 始终使用最新规范解析文件，建议直接删除该字段；
- 若文件包含新版字段，仅会输出「未知字段」警告，与 `version` 无关。

#### 2. Name 顶级元素关键点
- 定义 Compose 项目的默认名称，作为所有资源的命名前缀；
- 项目名优先级：`--project-name` 参数 > `name` 字段 > 环境变量 > 目录名；
- 项目名会暴露为 `COMPOSE_PROJECT_NAME` 环境变量，支持插值引用；
- 核心作用：避免多项目资源冲突，统一管理容器/网络/卷等资源。

#### 3. 最佳实践
- 显式设置 `name` 字段，避免依赖目录名（目录重命名会导致项目名变化）；
- 删除无用的 `version` 字段，保持 Compose 文件简洁；
- 运行 Compose 时通过 `--project-name` 区分开发/测试/生产环境（如 `myapp-dev`/`myapp-test`/`myapp-prod`）。



