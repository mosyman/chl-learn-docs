

## 目录
- [docker network](#docker-network)
- [docker network connect](#docker-network-connect)
- [docker network create](#docker-network-create)


## docker network
### 一、`docker network` 核心定位
`docker network` 是 Docker 提供的**命令行工具**，专门用于管理 Docker 网络（包括创建、查看、连接、删除等操作）。它是“手动管理”Docker 网络的核心方式，和你之前了解的 Compose 配置网络（声明式）是互补的：
- Compose 是“写配置文件，一键创建/管理网络”（适合多服务场景）；
- `docker network` 是“直接敲命令，即时操作网络”（适合调试、单容器/临时网络场景）。

### 二、核心子命令详解（附示例+使用场景）
每个子命令我会按“作用 + 基本语法 + 实战示例 + 关键说明”的结构讲解，确保你能直接上手使用。

#### 1. `docker network ls`：列出所有 Docker 网络
**作用**：查看当前 Docker 主机上已创建的所有网络（包括默认网络、自定义网络）。
**基本语法**：
```bash
docker network ls [OPTIONS]
```
**常用选项**：
- `-q`：只输出网络 ID（方便批量操作）；
- `--filter`：过滤结果（比如 `--filter driver=bridge` 只看 bridge 驱动的网络）。

**实战示例**：
```bash
# 查看所有网络（默认输出：ID、名称、驱动、作用域）
docker network ls

# 只看 bridge 驱动的网络
docker network ls --filter driver=bridge

# 只输出网络 ID
docker network ls -q
```
**输出示例**：
```
NETWORK ID     NAME      DRIVER    SCOPE
abc123456789   bridge    bridge    local
def098765432   host      host      local
ghi789012345   none      null      local
jkl123098765   my-net    bridge    local
```
**关键说明**：
- Docker 默认会创建 3 个网络：`bridge`（默认容器网络）、`host`（容器共享宿主机网络）、`none`（容器无网络）；
- 你自定义的网络会出现在列表末尾（比如上面的 `my-net`）。

#### 2. `docker network create`：创建自定义网络
**作用**：手动创建一个自定义网络（替代 Compose 的 `networks` 声明）。
**基本语法**：
```bash
docker network create [OPTIONS] NETWORK_NAME
```
**常用选项**：
- `--driver`（简写 `-d`）：指定网络驱动（默认 `bridge`）；
- `--subnet`：指定子网（比如 `172.20.0.0/16`）；
- `--gateway`：指定网关（比如 `172.20.0.1`）；
- `--name`：指定网络名称（也可以直接写在最后）。

**实战示例**：
```bash
# 创建默认 bridge 驱动的网络，命名为 my-app-net
docker network create my-app-net

# 创建带自定义子网和网关的 bridge 网络
docker network create --driver bridge --subnet 172.20.0.0/16 --gateway 172.20.0.1 my-custom-net
```
**关键说明**：
- 自定义网络的容器之间可以通过**容器名/别名**互相访问（默认 `bridge` 网络需要用 `--link` 才可以），这是自定义网络的核心优势；
- 驱动可选值：`bridge`（单机）、`overlay`（集群）、`host`、`macvlan` 等。

#### 3. `docker network connect`：将容器连接到网络
**作用**：把一个已运行的容器接入指定网络（容器可以同时接入多个网络）。
**基本语法**：
```bash
docker network connect [OPTIONS] NETWORK_NAME CONTAINER_NAME/ID
```
**实战示例**：
```bash
# 先启动一个 nginx 容器（默认接入 bridge 网络）
docker run -d --name my-nginx nginx:alpine

# 将 my-nginx 容器接入自定义网络 my-app-net
docker network connect my-app-net my-nginx
```
**关键说明**：
- 容器接入新网络后，会获得该网络的 IP 地址，能和该网络内的其他容器通信；
- 也可以在启动容器时直接指定网络：`docker run -d --name my-nginx --network my-app-net nginx:alpine`。

#### 4. `docker network disconnect`：将容器从网络断开
**作用**：移除容器与指定网络的连接（不会删除容器，也不会删除网络）。
**基本语法**：
```bash
docker network disconnect [OPTIONS] NETWORK_NAME CONTAINER_NAME/ID
```
**实战示例**：
```bash
# 把 my-nginx 容器从 my-app-net 网络断开
docker network disconnect my-app-net my-nginx
```
**关键说明**：
- 如果容器只接入了这一个网络，断开后容器仍有 `lo` 回环网络，但无法和其他容器/外网通信；
- 不能断开容器的默认网络（比如 `bridge`），除非容器接入了其他网络。

#### 5. `docker network inspect`：查看网络详细信息
**作用**：查看指定网络的完整配置（包括子网、网关、接入的容器等）。
**基本语法**：
```bash
docker network inspect [OPTIONS] NETWORK_NAME/ID
```
**实战示例**：
```bash
# 查看 my-app-net 网络的详细信息（JSON 格式）
docker network inspect my-app-net

# 只查看该网络下的容器名称（结合 jq 工具，需提前安装）
docker network inspect my-app-net | jq '.[0].Containers | keys'
```
**输出核心信息**：
```json
[
    {
        "Name": "my-app-net",
        "Driver": "bridge",
        "IPAM": {
            "Config": [{"Subnet": "172.18.0.0/16", "Gateway": "172.18.0.1"}]
        },
        "Containers": {
            "容器ID": {
                "Name": "my-nginx",
                "IPv4Address": "172.18.0.2/16"
            }
        }
    }
]
```
**关键说明**：
- 从 `Containers` 字段可以清晰看到哪些容器接入了这个网络，以及它们的 IP 地址；
- 调试容器通信问题时，这个命令是核心工具（比如检查容器是否在同一网络）。

#### 6. `docker network rm`：删除指定网络
**作用**：删除一个或多个未被使用的网络（如果网络里还有容器，会删除失败）。
**基本语法**：
```bash
docker network rm NETWORK_NAME/ID [NETWORK_NAME/ID ...]
```
**实战示例**：
```bash
# 删除单个网络
docker network rm my-app-net

# 批量删除多个网络（用空格分隔）
docker network rm net1 net2 net3
```
**关键说明**：
- 必须先断开所有容器与该网络的连接，才能删除；
- 无法删除 Docker 默认的 `bridge`、`host`、`none` 网络。

#### 7. `docker network prune`：清理所有未使用的网络
**作用**：一键删除所有“没有容器接入”的网络（适合清理垃圾网络，释放资源）。
**基本语法**：
```bash
docker network prune [OPTIONS]
```
**常用选项**：
- `-f`（`--force`）：跳过确认提示，直接删除；
- `--filter`：按条件过滤（比如 `--filter until=24h` 删除 24 小时内未使用的网络）。

**实战示例**：
```bash
# 交互式清理（会提示确认）
docker network prune

# 强制清理（无提示）
docker network prune -f
```
**关键说明**：
- 不会删除默认的 `bridge`、`host`、`none` 网络；
- 执行前建议先用 `docker network ls` 确认要清理的网络，避免误删。

### 三、实战场景：用 `docker network` 调试容器通信问题
假设你启动了两个容器，但它们无法互相访问，排查步骤：
1. 用 `docker network ls` 查看所有网络；
2. 用 `docker network inspect bridge`（或自定义网络）查看两个容器是否在同一网络；
3. 如果不在同一网络，用 `docker network connect` 将它们接入同一个网络；
4. 验证：在其中一个容器内 ping 另一个容器的名称/IP，确认通信正常。

示例操作：
```bash
# 启动两个容器（默认接入 bridge 网络）
docker run -d --name c1 nginx:alpine
docker run -d --name c2 busybox:latest sleep 3600

# 检查 c1 和 c2 是否在同一网络（bridge）
docker network inspect bridge | grep -E "c1|c2"

# 在 c2 容器内 ping c1（验证通信）
docker exec -it c2 ping c1
```

### 总结
1. **核心定位**：`docker network` 是 Docker 网络的命令行管理工具，用于手动创建、查看、连接、删除网络；
2. **高频子命令**：`ls`（查看）、`create`（创建）、`inspect`（调试）、`connect`（连接容器）、`prune`（清理）是日常使用最多的；
3. **核心原则**：容器必须在同一网络才能通信，自定义网络支持容器名解析（比默认 `bridge` 网络更友好）。

掌握这些命令后，你既能手动管理网络，也能快速排查容器通信问题，和 Compose 的声明式网络配置配合使用，能覆盖绝大多数 Docker 网络场景。


[目录](#目录)

## docker network connect
### 一、`docker network connect` 核心定位
这个命令的核心作用是：**将一个已运行的容器接入指定的 Docker 网络**（容器也可以在启动时通过 `docker run --network` 直接接入网络）。
- 容器接入网络后，就能和该网络内的其他容器通信（通过容器名/IP/别名）；
- 一个容器可以同时接入多个网络（比如既在 `front-net` 也在 `back-net`）；
- 命令的核心语法：`docker network connect [参数] 网络名 容器名/容器ID`。

### 二、核心参数（Options）详解
每个参数我会按“作用 + 示例 + 关键说明”的结构讲解，重点覆盖常用参数：

| 参数 | 默认值 | 作用 | 实战示例 + 关键说明 |
|------|--------|------|---------------------|
| `--alias` | 无 | 给容器在指定网络内添加**自定义别名**（网络内的其他容器可以通过这个别名访问它） | **示例**：<br>`docker network connect --alias db --alias mysql my-net container2`<br>**说明**：<br>1. `container2` 在 `my-net` 网络内有两个别名：`db`、`mysql`；<br>2. 同一网络内的其他容器可以 `ping db` 或 `ping mysql` 访问 `container2`；<br>3. 别名只在当前网络生效，容器在其他网络的别名不受影响。 |
| `--driver-opt` | 无 | 给容器的网络接口设置驱动级别的参数（比如 sysctl 网络参数） | **示例**：<br>`docker network connect --driver-opt="com.docker.network.endpoint.sysctls=net.ipv4.conf.IFNAME.log_martians=1" my-net container2`<br>**说明**：<br>1. `IFNAME` 会自动替换为容器的网络接口名（比如 `eth1`）；<br>2. 主要用于调整网络内核参数（如禁用转发、开启日志）；<br>3. 仅部分网络驱动支持（如 `bridge`），且参数有严格限制（仅 `net.ipv4.`/`net.ipv6.` 开头的 sysctl）。 |
| `--gw-priority` | 无 | 设置容器默认网关的优先级（数值越高，优先级越高；支持正负值） | **示例**：<br>`docker network connect --gw-priority 10 my-net container2`<br>**说明**：<br>1. 仅适用于容器接入多个网络、有多个网关的场景；<br>2. 优先级高的网关会成为容器的默认路由（比如访问外网时优先走这个网关）；<br>3. 日常开发中很少用到，主要用于复杂网络路由场景。 |
| `--ip` | `<nil>` | 给容器在指定网络内分配**固定 IPv4 地址**（需在网络的子网范围内） | **示例**：<br># 先创建带子网的网络<br>`docker network create --subnet 172.20.0.0/16 my-net`<br># 给容器分配固定 IP<br>`docker network connect --ip 172.20.128.2 my-net container2`<br>**说明**：<br>1. 必须先创建带 `--subnet` 的网络，才能指定固定 IP；<br>2. 指定的 IP 必须在网络的子网范围内，且未被其他容器占用；<br>3. 容器重启后，若该 IP 仍可用，会自动复用；若被占用，容器启动失败。 |
| `--ip6` | `<nil>` | 给容器在指定网络内分配**固定 IPv6 地址**（需网络开启 IPv6） | **示例**：<br># 先创建开启 IPv6 的网络<br>`docker network create --subnet 2001:db8::/64 --enable-ipv6 my-ipv6-net`<br># 分配 IPv6 地址<br>`docker network connect --ip6 2001:db8::33 my-ipv6-net container2`<br>**说明**：<br>1. 网络必须先通过 `--enable-ipv6` 开启 IPv6，并指定 IPv6 子网；<br>2. 用法和 `--ip` 完全一致，只是针对 IPv6 地址。 |
| `--link` | 无 | （遗留参数）将当前容器和另一个容器建立“链接”，并给对方设置别名 | **示例**：<br>`docker network connect --link container1:c1 my-net container2`<br>**说明**：<br>1. 作用是让 `container2` 能通过别名 `c1` 访问 `container1`；<br>2. 这是 Docker 早期的容器通信方式，现在推荐用 `--alias` 替代；<br>3. 不建议在生产环境使用，仅用于兼容旧系统。 |
| `--link-local-ip` | 无 | 给容器添加**链路本地地址**（仅在当前物理链路生效的 IP，比如 IPv4 的 169.254.0.0/16 段） | **示例**：<br>`docker network connect --link-local-ip 169.254.1.2 my-net container2`<br>**说明**：<br>1. 链路本地地址无需手动配置，通常用于自动寻址；<br>2. 日常开发中极少用到，主要用于特殊网络场景（如无 DHCP 服务器的环境）。 |

### 三、关键示例与实战场景
#### 场景1：给运行中的容器接入网络（基础用法）
```bash
# 1. 创建自定义网络
docker network create my-app-net

# 2. 启动一个容器（默认接入 bridge 网络）
docker run -d --name my-nginx nginx:alpine

# 3. 将容器接入 my-app-net 网络
docker network connect my-app-net my-nginx

# 4. 验证：查看 my-app-net 网络的详情，确认容器已接入
docker network inspect my-app-net
```
**效果**：`my-nginx` 现在同时在 `bridge` 和 `my-app-net` 两个网络中，能和这两个网络的容器通信。

#### 场景2：给容器分配固定 IP（避免 IP 漂移）
```bash
# 1. 创建带子网和 IP 范围的网络（IP 范围外的地址用于固定分配）
docker network create --subnet 172.20.0.0/16 --ip-range 172.20.240.0/20 my-net

# 2. 给容器分配 IP（选在 IP 范围外，避免被自动分配）
docker network connect --ip 172.20.128.2 my-net container2

# 3. 验证容器的 IP
docker exec -it container2 ip addr
```
**关键说明**：
- `--ip-range 172.20.240.0/20` 表示 Docker 自动分配 IP 时，只从这个段选；
- 我们指定的 `172.20.128.2` 不在这个段内，因此不会被其他容器占用，保证容器重启后能复用该 IP。

#### 场景3：给容器添加网络别名（方便访问）
```bash
# 1. 创建网络
docker network create db-net

# 2. 启动 mysql 容器
docker run -d --name mysql-server mysql:8

# 3. 将容器接入 db-net，并添加别名 db、mysql
docker network connect --alias db --alias mysql db-net mysql-server

# 4. 启动另一个容器，测试别名访问
docker run -it --rm --network db-net busybox ping db
# 输出：PING db (172.18.0.2): 56 data bytes → 访问成功
```
**效果**：在 `db-net` 网络内，任何容器都可以通过 `db` 或 `mysql` 访问 `mysql-server`，无需记容器名/IP。

### 四、重要注意事项
1. **容器状态要求**：只能连接**运行中的容器**（如果容器已停止，需先 `docker start` 启动，再执行 `connect`）；
2. **IP 冲突问题**：指定 `--ip` 时，若 IP 已被占用，命令会失败；容器重启时，若原 IP 被占用，容器启动失败；
3. **网络类型兼容**：容器可以接入不同类型的网络（如同时接入 `bridge` 和 `overlay` 网络），无需类型一致；
4. **别名作用域**：`--alias` 只在当前网络生效，容器在其他网络的别名不受影响；
5. **断开网络**：若要移除容器与网络的连接，使用 `docker network disconnect 网络名 容器名`。

### 总结
1. **核心功能**：`docker network connect` 用于给运行中的容器接入指定网络，支持多网络接入；
2. **高频参数**：`--ip`（固定 IP）、`--alias`（网络别名）是日常开发中最常用的参数；
3. **关键技巧**：指定固定 IP 时，建议创建网络时用 `--ip-range` 划分自动分配段，固定 IP 选在段外，避免冲突。

掌握这些用法后，你可以灵活控制容器的网络接入方式，解决容器通信、固定 IP、别名访问等常见场景问题。

[目录](#目录)

## docker network create

### 一、`docker network create` 核心定位
这个命令是 Docker 中**手动创建自定义网络**的核心工具，解决了默认 `bridge` 网络（`docker0`）的局限性（无 DNS 解析、隔离性差）。
- 核心语法：`docker network create [OPTIONS] 网络名称`；
- 默认驱动：如果不指定 `-d/--driver`，会自动创建 `bridge` 类型网络；
- 核心价值：
    1. 实现容器网络隔离（不同自定义网络的容器默认无法通信）；
    2. 自定义网络内置 DNS 解析（容器可通过名称/别名访问）；
    3. 支持跨主机网络（`overlay` 驱动，需 Swarm 模式）；
    4. 可精细配置 IP 段、网关、MTU 等网络参数。

### 二、核心参数（Options）详解
按“常用程度 + 重要性”排序，每个参数包含「作用 + 示例 + 关键说明」，重点覆盖日常开发和生产场景：

| 参数 | 默认值 | 作用 | 实战示例 & 关键说明 |
|---|----|----|---------------------|
| `-d/--driver` | `bridge` | 指定网络驱动（决定网络类型和能力） | **核心驱动类型**：<br>1. `bridge`：单机隔离网络（默认，最常用）<br>2. `overlay`：跨主机网络（需 Swarm 模式）<br>3. `host`/`macvlan`/`none`：特殊场景驱动<br>**示例**：<br>`docker network create -d overlay my-overlay-net`（创建跨主机网络）<br>**说明**：<br>- `bridge` 适用于单机多容器隔离；<br>- `overlay` 适用于 Swarm 集群跨主机通信；<br>- 第三方驱动需提前安装。 |
| `--subnet` | 自动分配 | 指定网络的子网（CIDR 格式），决定容器 IP 范围 | **示例**：<br>`docker network create --subnet 172.28.0.0/16 my-net`<br>**说明**：<br>1. 必须是合法的 CIDR 格式（如 `/16`/`/24`）；<br>2. `bridge` 网络仅支持 1 个子网，`overlay` 可支持多个；<br>3. 子网不能和宿主机/其他网络重叠，否则创建失败；<br>4. 指定子网后，可配合 `--ip` 给容器分配固定 IP。 |
| `--gateway` | 自动分配 | 指定子网的网关 IP（需在子网范围内） | **示例**：<br>`docker network create --subnet 172.28.0.0/16 --gateway 172.28.5.254 my-net`<br>**说明**：<br>1. 网关是容器访问外网/其他子网的出口；<br>2. 多个子网（如 overlay）可指定多个网关；<br>3. 网关 IP 不能被容器占用。 |
| `--ip-range` | 全部子网 | 限定容器 IP 的分配范围（从子网中划分更小的段） | **示例**：<br>`docker network create --subnet 172.28.0.0/16 --ip-range 172.28.5.0/24 my-net`<br>**说明**：<br>1. `--ip-range` 必须是 `--subnet` 的子集；<br>2. 作用：预留部分 IP 用于手动分配（如固定 IP），避免自动分配冲突；<br>3. 示例中 Docker 仅从 `172.28.5.0/24` 自动分配 IP，`172.28.0.0/16` 其他段可手动指定。 |
| `--internal` | `false` | 启用“内部网络”模式，禁止容器访问外网/其他网络 | **示例**：<br>`docker network create --internal my-internal-net`<br>**说明**：<br>1. 内部网络的容器仅能互相通信，无法访问外网/宿主机；<br>2. 无默认路由，防火墙会丢弃外部流量；<br>3. 适用于需严格隔离的场景（如数据库集群）。 |
| `--attachable` | `false` | 允许“手动启动的容器”接入 Swarm 范围的网络 | **示例**：<br>`docker network create -d overlay --scope swarm --attachable my-swarm-net`<br>**说明**：<br>1. 仅适用于 `overlay`/Swarm 网络；<br>2. 默认 Swarm 网络仅允许 Swarm 服务接入，`--attachable` 允许普通容器接入；<br>3. 注意安全：非管理节点接入可能带来风险。 |
| `--ipv6` | `false` | 启用 IPv6 地址分配 | **示例**：<br>`docker network create --ipv6 --subnet 2001:db8::/64 my-ipv6-net`<br>**说明**：<br>1. 需先确保 Docker 守护进程开启 IPv6（修改 `daemon.json`）；<br>2. 启用后需指定 IPv6 子网；<br>3. 配合 `--ip6` 可给容器分配固定 IPv6 地址。 |
| `--ipv4` | `true` | 启用/禁用 IPv4 地址分配 | **示例**：<br>`docker network create --ipv4 false --ipv6 --subnet 2001:db8::/64 my-ipv6-only-net`<br>**说明**：<br>1. 禁用 IPv4 后，容器仅能使用 IPv6 通信；<br>2. 需和 `--ipv6` 配合使用，适用于纯 IPv6 环境。 |
| `-o/--opt` | 无 | 给网络驱动传递自定义参数（驱动专属） | **bridge 驱动常用参数**：<br>| 参数键 | 作用 | 示例 |
|--------|------|------|
| `com.docker.network.bridge.name` | 自定义 Linux 网桥名称 | `-o com.docker.network.bridge.name=my-bridge` |
| `com.docker.network.bridge.enable_icc` | 启用/禁用容器间通信 | `-o com.docker.network.bridge.enable_icc=false`（禁止容器互访） |
| `com.docker.network.driver.mtu` | 设置网络 MTU（最大传输单元） | `-o com.docker.network.driver.mtu=9216` |
| `com.docker.network.bridge.host_binding_ipv4` | 容器端口映射的默认 IP | `-o com.docker.network.bridge.host_binding_ipv4=172.19.0.1` |
**示例**：<br>`docker network create -o com.docker.network.driver.mtu=9216 my-net` |
| `--ingress` | `false` | 创建 Swarm 集群的“路由网格”网络 | **示例**：<br>`docker network create -d overlay --ingress my-ingress-net`<br>**说明**：<br>1. 仅适用于 `overlay` 驱动，且 Swarm 模式已启用；<br>2. 整个集群仅能创建 1 个 ingress 网络；<br>3. 用于 Swarm 服务的端口映射（路由网格）；<br>4. 不能和 `--attachable` 同时使用。 |
| `--scope` | 自动 | 控制网络的作用域（`local`/`swarm`） | **示例**：<br>`docker network create -d bridge --scope swarm --attachable my-swarm-bridge`<br>**说明**：<br>1. `local`：仅当前节点可见（默认）；<br>2. `swarm`：整个 Swarm 集群可见；<br>3. 非 `overlay` 驱动（如 `bridge`）设置 `--scope swarm` 后，可用于 Swarm 服务。 |
| `--aux-address` | 无 | 给网络驱动预留辅助 IP（如路由/交换机 IP） | **示例**：<br>`docker network create --subnet 192.168.10.0/25 --aux-address "my-router=192.168.10.5" my-net`<br>**说明**：<br>1. 辅助 IP 不会分配给容器；<br>2. 用于网络驱动内部使用（如 macvlan/overlay 路由）；<br>3. 可指定多个（用空格分隔）。 |
| `--label` | 无 | 给网络添加元数据标签（用于分类/筛选） | **示例**：<br>`docker network create --label "env=prod" --label "owner=dev-team" my-net`<br>**说明**：<br>1. 支持键值对格式，推荐反向 DNS 命名（如 `com.example.env=prod`）；<br>2. 可通过 `docker network ls --filter label=env=prod` 筛选网络。 |
| `--config-from`/`--config-only` | 无 | 基于已有网络复制配置/创建“仅配置”网络 | **示例**：<br>`# 创建仅配置网络`<br>`docker network create --config-only --subnet 192.168.100.0/24 mv-config`<br>`# 基于配置创建 swarm 网络`<br>`docker network create -d macvlan --scope swarm --config-from mv-config my-swarm-net`<br>**说明**：<br>1. `--config-only` 仅保存网络配置，不创建实际网络；<br>2. 适用于 Swarm 集群中不同节点的网络配置差异化（如 macvlan 不同子网）。 |

### 三、场景化实战示例
#### 场景1：创建基础自定义 bridge 网络（最常用）
需求：创建一个单机隔离网络，支持容器名解析，替代默认 `bridge`。
```bash
# 创建自定义 bridge 网络
docker network create my-app-net

# 启动容器接入该网络
docker run -d --name app --network my-app-net nginx:alpine
docker run -d --name db --network my-app-net mysql:8

# 验证：app 容器可通过容器名访问 db
docker exec -it app ping db
```
**核心优势**：容器可通过名称访问，无需记 IP；不同自定义网络的容器默认隔离。

#### 场景2：创建带固定 IP 段的网络（避免 IP 冲突）
需求：指定子网和 IP 分配范围，预留部分 IP 给固定分配。
```bash
# 创建网络：子网 172.28.0.0/16，仅从 172.28.5.0/24 自动分配 IP
docker network create \
  --driver bridge \
  --subnet 172.28.0.0/16 \
  --ip-range 172.28.5.0/24 \
  --gateway 172.28.5.254 \
  my-fixed-ip-net

# 给容器分配固定 IP（选在自动分配范围外）
docker network connect --ip 172.28.128.2 my-fixed-ip-net app
```
**关键说明**：固定 IP 选在 `--ip-range` 外，避免被自动分配占用。

#### 场景3：创建跨主机 overlay 网络（Swarm 集群）
需求：Swarm 集群中跨节点容器通信，支持 2 个子网。
```bash
# 1. 先启用 Swarm 模式（主节点）
docker swarm init

# 2. 创建 overlay 网络，支持 2 个子网，允许手动容器接入
docker network create \
  -d overlay \
  --scope swarm \
  --attachable \
  --subnet 192.168.10.0/25 \
  --subnet 192.168.20.0/25 \
  --gateway 192.168.10.100 \
  --gateway 192.168.20.100 \
  my-overlay-net
```
**注意事项**：
- overlay 网络建议用 `/24` 或 `/25` 子网（最多 256 个 IP），避免 Swarm 限制；
- 如需更多 IP，可创建多个小 overlay 网络，而非扩大子网。

#### 场景4：创建内部隔离网络（禁止访问外网）
需求：数据库容器仅能和应用容器通信，无法访问外网。
```bash
# 创建内部网络
docker network create --internal my-internal-db-net

# 启动数据库容器接入内部网络
docker run -d --name db --network my-internal-db-net mysql:8

# 应用容器同时接入内部网络和外网网络
docker network create my-external-net
docker run -d --name app --network my-external-net nginx:alpine
docker network connect my-internal-db-net app

# 验证：db 无法 ping 外网（如 8.8.8.8），但 app 可访问 db 和外网
docker exec -it db ping 8.8.8.8  # 失败
docker exec -it app ping db      # 成功
docker exec -it app ping 8.8.8.8 # 成功
```

#### 场景5：自定义 bridge 驱动参数（调整网络行为）
需求：修改 bridge 网络的 MTU，禁止容器间通信。
```bash
docker network create \
  -o com.docker.network.driver.mtu=9216 \
  -o com.docker.network.bridge.enable_icc=false \
  my-custom-bridge
```
**说明**：
- `enable_icc=false`：禁止该网络内容器互相通信（仅能通过宿主机端口映射访问）；
- `mtu=9216`：适配大文件传输场景，减少分片。

### 四、关键注意事项
1. **网络名称唯一性**：同一 Docker 节点上网络名称必须唯一，Docker 不保证自动检测冲突，需手动避免；
2. **子网重叠问题**：创建网络时，子网不能和宿主机网卡、其他网络子网重叠，否则创建失败；
3. **overlay 网络限制**：
    - 推荐子网大小为 `/24`（256 个 IP），超过会触发 Swarm 已知问题；
    - 如需更多 IP，创建多个小 overlay 网络，而非扩大子网；
4. **ingress 网络限制**：
    - 整个 Swarm 集群仅能有 1 个 ingress 网络；
    - 不能删除正在被 Swarm 服务使用的 ingress 网络；
5. **内部网络访问**：内部网络容器无法访问外网，但宿主机可直接访问容器 IP；
6. **IPv6 配置**：启用 IPv6 前需修改 Docker 守护进程配置（`/etc/docker/daemon.json`），重启 Docker 生效：
   ```json
   {
     "ipv6": true,
     "fixed-cidr-v6": "2001:db8:1::/64"
   }
   ```

### 总结
1. **核心功能**：`docker network create` 用于创建自定义网络，解决默认 `bridge` 网络的隔离性和解析问题；
2. **高频参数**：`-d`（驱动）、`--subnet`（子网）、`--ip-range`（IP 范围）、`--internal`（隔离）是日常最常用的参数；
3. **场景选择**：
    - 单机隔离用 `bridge` 驱动；
    - 跨主机通信（Swarm）用 `overlay` 驱动；
    - 严格隔离用 `--internal`；
    - 固定 IP 需先指定 `--subnet` 和 `--ip-range`。

掌握这些用法后，你可以根据业务需求灵活配置容器网络，兼顾隔离性、可访问性和安全性。



