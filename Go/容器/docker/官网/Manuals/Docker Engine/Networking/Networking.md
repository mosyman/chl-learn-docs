
# Docker 网络概览 

---

# 1. 什么是容器网络（Container Networking）
容器网络 = 让容器之间、容器与外部服务之间**能够互相通信**的能力。

- 默认开启，容器默认可以发起外部连接
- 容器只看到：IP、网关、路由、DNS 等网络信息
- 容器不知道自己是 Docker 容器，也不知道对方是不是容器

---

# 2. 默认网络：default bridge
Linux 上 Docker 第一次启动时，会自动创建一个内置网络：
**默认桥接网络（default bridge）**

- 不指定 `--network` 的容器，都会连到这个 bridge
- 容器可访问外网（通过主机的 NAT/masquerade）
- 容器之间只能用 **IP 互相访问**，**不能用容器名访问**
- 默认桥接网络**不推荐在生产使用**

示例（默认网络访问外网）：
```bash
docker run --rm -ti busybox ping -c1 docker.com
```

---

# 3. 用户自定义网络（User-defined networks）
## 为什么要用自定义网络
- 容器之间**可以用容器名互相通信**（DNS 自动解析）
- 网络隔离：不同项目/组的容器分开
- 更安全、更易管理

## 创建自定义桥接网络
```bash
docker network create -d bridge my-net
```

运行容器并指定网络：
```bash
docker run --network=my-net -it busybox
```

---

# 4. Docker 网络驱动（核心）
Docker 提供多种网络驱动，决定容器的网络模型：

| 驱动 | 作用 |
|------|------|
| **bridge** | 默认驱动，单机桥接网络 |
| **host** | 容器共享主机网络，无隔离 |
| **none** | 完全禁用网络，完全隔离 |
| **overlay** | 跨主机/集群网络（Swarm） |
| **ipvlan** | 容器直连外部 VLAN |
| **macvlan** | 容器拥有独立 MAC，像物理机一样在局域网 |

---

# 5. 一个容器连多个网络
容器可以像插多根网线一样，**同时连多个 Docker 网络**。

典型场景：
- 前端容器：连外网 bridge + 内网 `--internal` 网络
- 后端容器：只连内网，不暴露外网

## 网关优先级 gw-priority
容器连多个网络时，Docker 自动选默认网关。
可以手动指定优先级：
```bash
docker run --network name=gwnet,gw-priority=1 ...
```
- 优先级越高，越优先成为默认网关
- 默认优先级是 0



你想要理解这段关于 Docker ipvlan 网络中默认网关和网关优先级配置的英文说明，我会用通俗易懂的方式拆解这段内容，并解释其中的核心概念和实际应用场景。

### 一、逐段核心释义
#### 1. 基础发包逻辑（第一段核心）
> When sending packets, if the destination is an address in a directly connected network, packets are sent to that network. Otherwise, packets are sent to a default gateway for routing to their destination.

**翻译 & 解读**：
当容器发送数据包时：
- 如果目标地址属于「直连网络」（容器直接接入的网络，比如容器所在的 ipvlan 子网），数据包会直接发往该网络；
- 如果目标地址不属于直连网络（比如外网地址、其他非直连的内网网段），数据包会先发送到「默认网关」，由网关负责将数据包路由到最终目的地。

**通俗比喻**：你在公司（容器）寄快递（数据包），如果收件人在公司大楼里（直连网络），你直接送过去；如果收件人在外地（非直连网络），你先交给公司前台（默认网关），由前台转发。

#### 2. ipvlan 网关的特殊要求（第二段核心）
> In the example above, the ipvlan network's gateway must be the default gateway.The default gateway is selected by Docker, and may change whenever a container's network connections change.

**翻译 & 解读**：
- 在上文的示例中，ipvlan 网络的网关**必须**配置为容器的默认网关（这是 ipvlan 网络的一个特性要求）；
- 默认网关默认由 Docker 自动选择，而且只要容器的网络连接发生变化（比如新增/删除网络、重启网络），这个默认网关就可能会变。

**关键提醒**：Docker 自动选网关的机制可能导致网络不稳定（比如网关变了，容器访问外网就会出问题），所以需要手动指定固定网关。

#### 3. 解决方案：网关优先级（第三段核心）
> To make Docker choose a specific default gateway when creating the container or connecting a new network, set a gateway priority. See option gw-priority for the docker run and docker network connect commands.

**翻译 & 解读**：
- 如果你想让 Docker 在创建容器/为容器连接新网络时，**固定选择某个指定的默认网关**，可以通过设置「网关优先级（gw-priority）」来实现；
- 具体用法可以参考 `docker run` 和 `docker network connect` 命令的 `gw-priority` 参数说明。

### 二、核心概念补充
| 术语 | 通俗解释 |
|------|----------|
| 直连网络 (directly connected network) | 容器直接接入的子网，容器与该网段内的地址通信无需经过网关 |
| 默认网关 (default gateway) | 容器访问非直连网络时的「转发中转站」，是数据包的默认出口 |
| ipvlan | Docker 的一种网络驱动，特点是容器与宿主机共享同一个物理网卡的 IP 网段，需要网关和宿主机/网络环境匹配 |
| gw-priority | Docker 提供的网关优先级参数，数值越小优先级越高（比如 gw-priority=1 比 gw-priority=10 优先），Docker 会优先选择优先级高的网关作为默认网关 |

### 三、实际应用示例（帮你理解怎么用）
比如你想创建一个 ipvlan 网络的容器，并指定 192.168.1.1 为默认网关（优先级设为 1，最高），命令如下：
```bash
# 1. 先创建 ipvlan 网络（指定网关）
docker network create -d ipvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \  # eth0 是宿主机的物理网卡
  my-ipvlan-network

# 2. 创建容器时指定网关优先级，固定默认网关
docker run -itd \
  --network my-ipvlan-network \
  --gw-priority 1 \  # 设置网关优先级为1（最高）
  --name my-container \
  ubuntu:latest
```

如果是给已有的容器连接新网络并指定网关优先级：
```bash
docker network connect \
  --gw-priority 1 \
  my-ipvlan-network \
  my-existing-container
```

### 总结
1. 容器发包遵循「直连网络直送，非直连网络走默认网关」的逻辑，ipvlan 网络要求其网关必须是容器默认网关；
2. Docker 自动选的默认网关可能随网络变化而改变，导致网络不稳定；
3. 可通过 `docker run`/`docker network connect` 的 `gw-priority` 参数指定网关优先级，固定默认网关（数值越小优先级越高）。








---



# 6. 端口发布（Published ports）
## 默认情况
- 同网络内：容器互相访问所有端口
- 外部/其他网络：**无法访问**容器端口

## 暴露端口给外部
使用 `-p` 或 `--publish`：
```bash
-p 主机端口:容器端口
```

示例：
```bash
docker run -p 8080:80 nginx
```

---

# 7. IP 地址与主机名
## 自动分配 IP
- 每个网络都会给容器分配一个 IP
- 来自该网络的子网网段
- Docker  daemon 负责子网划分与 IP 管理

## 指定静态 IP
```bash
docker run --ip 172.18.0.10 --network my-net ...
```

## 主机名 hostname
- 默认主机名 = 容器 ID
- 可指定：`--hostname mycontainer`

## 网络别名（--alias）
给容器在某个网络里起别名：
```bash
docker network connect --alias db my-net mycontainer
```

---

# 8. 子网分配（非常重要）
## 手动指定子网
```bash
docker network create \
  --subnet 192.168.100.0/24 \
  --gateway 192.168.100.1 \
  my-net
```

## 自动子网（默认地址池）
Docker 默认从这些网段自动分配：
- `172.17.0.0/16`
- `172.18.0.0/16`
- `172.19.0.0/16`
- `172.20.0.0/14`
- `192.168.0.0/16` 等

可以在 `/etc/docker/daemon.json` 自定义：
```json
{
  "default-address-pools": [
    {"base":"172.17.0.0/16","size":24}
  ]
}
```
- `base`：基础网段
- `size`：每个网络使用的掩码长度

---

# 9. IPv6
启用 IPv6：
```bash
docker network create --ipv6 --subnet 2001:db8::/64 my-net
```

---

# 10. DNS 服务
## 默认规则
- 容器继承主机 `/etc/resolv.conf`
- **自定义网络**：使用 Docker **内置 DNS**
    - 支持容器名解析
    - 外部 DNS 转发给主机 DNS

## 自定义 DNS 参数
| 参数 | 作用 |
|------|------|
| `--dns` | 指定 DNS 服务器 |
| `--dns-search` | DNS 搜索域 |
| `--dns-opt` | DNS 选项 |
| `--hostname` | 主机名 |

示例：
```bash
docker run --dns 8.8.8.8 --dns-search example.com ...
```

## 自定义 hosts
```bash
--add-host host.docker.internal:host-gateway
```

---

# 11. Container 网络模式（共享网络栈）
使用 `--network container:<容器名/ID>`
让新容器**完全共享另一个容器的网络**。

## 特点
- 共享 IP、端口、端口映射
- 本地回环 `127.0.0.1` 互通
- **不能再用**：`--hostname`、`--dns`、`-p`、`--expose` 等

示例：
```bash
# 启动 redis，只绑定 127.0.0.1
docker run -d --name redis redis --bind 127.0.0.1

# 另一个容器共享它的网络，直接 127.0.0.1 连接
docker run --rm -it --network container:redis redis redis-cli -h 127.0.0.1
```

---

# 12. 网络常用命令速查
```bash
# 创建网络
docker network create -d bridge my-net

# 查看所有网络
docker network ls

# 查看网络详情
docker network inspect my-net

# 运行容器时指定网络
docker run --network my-net ...

# 给运行中的容器加网络
docker network connect my-net container_name

# 断开容器与网络
docker network disconnect my-net container_name

# 删除未使用的网络
docker network prune

# 删除指定网络
docker network rm my-net
```

---

# 13. 极简总结（最核心）
- **bridge**：单机默认，隔离，推荐自定义使用
- **host**：无隔离，性能最高，直接用主机网络
- **overlay**：跨主机集群通信
- **macvlan/ipvlan**：容器像物理机一样在局域网
- **自定义网络**：支持容器名互通，生产必用
- **container:xxx**：共享另一个容器网络

---

如果你需要，我可以继续给你整理：
- **Docker 所有网络模式对比（面试版）**
- **Docker 网络排障大全（ping、telnet、nslookup、iptables）**
- **Compose 网络最佳实践**











