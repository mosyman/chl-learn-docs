
### 一、核心概念：Docker Compose 网络到底是什么？
简单来说，Docker Compose 的网络就是**容器之间的“通信桥梁”**：
- 默认情况下，Compose 会为你的应用创建一个默认网络，所有服务的容器都会加入这个网络，并且可以通过**服务名**互相访问（比如 `app` 服务可以直接 ping 通 `db` 服务）。
- 你也可以通过顶层的 `networks` 配置自定义网络，实现服务之间的“隔离”或“精准通信”（比如让前端服务只能访问应用服务，应用服务能访问数据库服务，前端不能直接访问数据库）。

### 二、核心用法拆解
#### 1. 基础用法：自定义网络并让服务接入
核心逻辑是：先在顶层 `networks` 里定义网络，再在 `services` 里通过 `networks` 字段让服务“加入”这个网络。
```yml
services:
  frontend: # 前端服务
    image: example/webapp
    networks: # 让前端服务接入两个网络
      - front-tier
      - back-tier

networks: # 定义两个自定义网络
  front-tier: # 前端网络（仅声明，用默认配置）
  back-tier: # 后端网络（仅声明，用默认配置）
```
**效果**：`frontend` 容器既在 `front-tier` 网络，也在 `back-tier` 网络，能和这两个网络里的所有容器通信。

#### 2. 进阶用法：配置网络的驱动/参数
你可以给自定义网络指定“驱动”（比如 `bridge`、`overlay`），或传递驱动参数，实现更精细的控制：
```yml
services:
  proxy: # 代理服务，只在 frontend 网络
    build: ./proxy
    networks:
      - frontend
  app: # 应用服务，跨 frontend 和 backend 网络（作为中间层）
    build: ./app
    networks:
      - frontend
      - backend
  db: # 数据库服务，只在 backend 网络
    image: postgres:18
    networks:
      - backend

networks:
  frontend: # 前端网络，用 bridge 驱动并指定参数
    driver: bridge
    driver_opts:
      # 绑定 IPv4 到 127.0.0.1，限制外部访问
      com.docker.network.bridge.host_binding_ipv4: "127.0.0.1"
  backend: # 后端网络，用自定义驱动
    driver: custom-driver
```
**关键效果**：`proxy` 和 `db` 无法直接通信（不在同一个网络），只有 `app` 能同时和两者通信，实现了“网络隔离”。

#### 3. 默认网络的特性与自定义
如果你的 Compose 文件没显式定义网络，Compose 会自动创建一个名为 `default` 的网络，所有服务默认加入这个网络：
```yml
# 极简写法（等价于下面带 default 网络的写法）
services:
  some-service:
    image: foo

# 等价的完整写法
services:
  some-service:
    image: foo
    networks:
      default: {} # 接入默认网络
networks:
  default: {} # 声明默认网络（用默认配置）
```
你也可以自定义 `default` 网络的配置（比如改名字、加驱动参数）：
```yml
networks:
  default: 
    name: my-app-default # 自定义默认网络的名字
    driver_opts: # 给驱动传参数
      com.docker.network.bridge.host_binding_ipv4: 127.0.0.1
```

### 三、关键配置项（Attributes）详解
每个配置项我会用“作用 + 示例 + 注意点”的形式说明：

| 配置项       | 作用                                                                 | 示例 & 注意点                                                                                                               |
|--------------|----------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| `attachable` | 允许“独立容器”（非 Compose 启动的容器）接入这个网络                  | 仅适用于 `overlay` 等集群驱动，示例：<br>`networks:`<br>`  mynet1:`<br>`    driver: overlay`<br>`    attachable: true`                    |
| `driver`     | 指定网络驱动（Docker 内置：bridge/host/overlay/macvlan，也可自定义）  | 默认是 `bridge`，示例：<br>`networks:`<br>`  db-data:`<br>`    driver: bridge`                                                    |
| `driver_opts`| 给网络驱动传自定义参数（参数格式由驱动决定）                         | 示例：给 bridge 驱动绑定 IPv4，<br>`driver_opts:`<br>`  com.docker.network.bridge.host_binding_ipv4: "127.0.0.1"`                 |
| `enable_ipv4`/`enable_ipv6` | 开启/关闭 IPv4/IPv6 地址分配 | 需同时配置 IPv6 时，要确保 Docker 守护进程开启了 IPv6，示例：<br>`networks:`<br>`  ip6net:`<br>`    enable_ipv4: false`<br>`    enable_ipv6: true` |
| `external`   | 声明网络是“外部已存在的”（Compose 不会创建/删除这个网络）            | 注意：设置 `external: true` 后，除了 `name` 外其他配置项都无效，示例：<br>`networks:`<br>`  outside:`<br>`    external: true`                    |
| `ipam`       | 自定义 IP 地址管理（比如指定子网、网关、IP 范围）                    | `ipam` 是“IP Address Management”的缩写，示例见文档，核心是配置 `subnet`/`gateway` 等                                                    |
| `internal`   | 让网络“隔离外部”（容器只能内网通信，不能访问宿主机/外网）            | 设置 `internal: true` 后，网络没有出口，容器无法访问外网                                                                                  |
| `labels`     | 给网络加“元数据标签”（用于分类、备注）                               | 推荐用反向 DNS 格式（如 `com.example.description`）避免冲突，支持数组/字典格式                                                                |
| `name`       | 自定义网络的名字（默认 Compose 会加“项目名前缀”，设置后用原值）      | 可配合 `external` 使用，示例：<br>`networks:`<br>`  network1:`<br>`    name: my-app-net`                                            |

### 四、实战场景举例
#### 场景1：实现服务隔离（前端→应用→数据库，前端不直连数据库）
```yml
services:
  frontend:
    image: nginx:alpine
    networks:
      - front-net
  app:
    image: node:20-alpine
    networks:
      - front-net
      - back-net
  db:
    image: mysql:8
    networks:
      - back-net
    environment:
      MYSQL_ROOT_PASSWORD: 123456

networks:
  front-net: # 前端网络
  back-net: # 后端网络（隔离数据库）
```
**效果**：`frontend` 只能访问 `app`，`app` 能访问 `db`，`frontend` 无法直接访问 `db`，提升安全性。

#### 场景2：使用外部已存在的网络
假设你已经用 `docker network create my-external-net` 创建了一个网络，想让 Compose 服务接入：
```yml
services:
  app:
    image: python:3.12
    networks:
      - my-external-net

networks:
  my-external-net:
    external: true # 声明是外部网络
    name: my-external-net # 指定外部网络的名字
```

### 总结
1. **核心逻辑**：Docker Compose 网络是容器通信的桥梁，默认创建 `default` 网络，也可自定义网络实现服务隔离；
2. **使用步骤**：先在顶层 `networks` 定义网络，再在 `services` 里通过 `networks` 让服务接入；
3. **关键配置**：`external` 用于接入外部网络，`driver` 指定网络驱动，`internal` 实现内网隔离，`ipam` 自定义 IP 配置。

通过这些配置，你可以灵活控制容器之间的通信范围和方式，既保证服务互通，又能通过网络隔离提升应用的安全性。