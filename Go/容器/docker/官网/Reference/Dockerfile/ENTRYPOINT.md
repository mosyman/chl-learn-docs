

### 一、`ENTRYPOINT` 指令的核心定义
`ENTRYPOINT`（入口点）是 Dockerfile 中用于**定义容器启动时「固定执行的核心程序/命令」** 的指令，它的核心目标是让容器像一个「可执行程序」一样运行——你可以为它设置固定的主程序，仅通过 `CMD` 或运行时参数传递可变参数。

简单来说：`ENTRYPOINT` 定死「容器要执行的主程序」，`CMD` 提供「主程序的默认参数」，运行时传入的参数会覆盖 `CMD` 的默认值。

### 二、`ENTRYPOINT` 的两种语法形式
`ENTRYPOINT` 有两种写法，行为差异极大，**exec 形式是官方推荐的首选**。

#### 1. Exec 形式（推荐）
```dockerfile
ENTRYPOINT ["可执行文件", "参数1", "参数2"]
```
- 语法：JSON 数组格式，必须用双引号，直接调用可执行文件（不经过 `/bin/sh -c`）；
- 核心特性：
    - 容器启动时，该可执行文件会成为容器的 `PID 1` 进程（接收 Unix 信号，如 `SIGTERM`）；
    - `docker run` 传入的参数会**追加**到 `ENTRYPOINT` 之后，覆盖 `CMD` 的默认参数；
    - 支持与 `CMD` 配合，`CMD` 作为 `ENTRYPOINT` 的默认参数。

**示例**：
```dockerfile
# 固定执行 top -b，CMD 提供默认参数 -c
ENTRYPOINT ["top", "-b"]
CMD ["-c"]
```
- 运行容器 `docker run 镜像名` → 执行 `top -b -c`；
- 运行容器 `docker run 镜像名 -H` → 执行 `top -b -H`（`-H` 覆盖 `CMD` 的 `-c`）。

#### 2. Shell 形式（不推荐）
```dockerfile
ENTRYPOINT command param1 param2
```
- 语法：普通字符串格式，底层会调用 `/bin/sh -c` 执行命令；
- 核心问题：
    - 主程序不是 `PID 1`（`PID 1` 是 `sh`），无法接收 `docker stop` 发送的 `SIGTERM` 信号（容器停止时会被强制杀死，而非优雅退出）；
    - 会忽略 `CMD` 和 `docker run` 传入的所有参数；
    - 仅适合简单场景，或需要解析环境变量的场景（但可通过 exec 形式+脚本替代）。

**反例（踩坑）**：
```dockerfile
# Shell 形式，top 不是 PID 1，无法接收 SIGTERM
ENTRYPOINT top -b
CMD ["-c"]  # 该参数会被忽略！
```
- 运行容器 `docker run 镜像名 -H` → 仍执行 `top -b`，`-H` 参数无效；
- `docker stop` 会等待超时后强制杀死容器（而非优雅停止 `top`）。

**Shell 形式的优化**：若必须用 Shell 形式，需加 `exec` 让主程序成为 `PID 1`：
```dockerfile
# 加 exec 后，top 成为 PID 1，能接收信号
ENTRYPOINT exec top -b
```

### 三、`ENTRYPOINT` 与 `CMD` 的协作规则（核心）
`ENTRYPOINT` 和 `CMD` 是Docker容器启动命令的「黄金搭档」，它们的组合规则决定了容器最终执行的命令。Docker 官方总结的核心规则如下：

| 组合场景                | 无 ENTRYPOINT              | ENTRYPOINT exec_entry p1_entry（Shell 形式） | ENTRYPOINT ["exec_entry", "p1_entry"]（Exec 形式） |
|-------------------------|----------------------------|---------------------------------------------|---------------------------------------------------|
| 无 CMD                  | 报错（不允许）             | /bin/sh -c exec_entry p1_entry              | exec_entry p1_entry                                |
| CMD ["exec_cmd", "p1_cmd"] | exec_cmd p1_cmd            | /bin/sh -c exec_entry p1_entry              | exec_entry p1_entry exec_cmd p1_cmd                |
| CMD exec_cmd p1_cmd     | /bin/sh -c exec_cmd p1_cmd | /bin/sh -c exec_entry p1_entry              | exec_entry p1_entry /bin/sh -c exec_cmd p1_cmd     |

#### 关键结论：
1. **Exec 形式 ENTRYPOINT + Exec 形式 CMD**：这是最推荐的组合——`ENTRYPOINT` 定主程序，`CMD` 提供默认参数，运行时参数覆盖 `CMD`；
2. **Shell 形式 ENTRYPOINT**：完全忽略 `CMD` 和运行时参数，仅执行自身命令；
3. **无 ENTRYPOINT 只有 CMD**：容器启动时执行 `CMD` 命令，运行时参数覆盖 `CMD`。

### 四、核心使用场景与示例
#### 场景 1：让容器成为「可执行工具」（基础场景）
目标：容器启动时固定执行 `nginx`，可通过参数修改启动方式。
```dockerfile
FROM nginx:alpine
# Exec 形式：固定执行 nginx，默认前台运行
ENTRYPOINT ["nginx"]
# 默认参数：-g "daemon off;"（前台运行，避免容器退出）
CMD ["-g", "daemon off;"]
```
- 正常运行：`docker run -d 镜像名` → `nginx -g "daemon off;"`；
- 调试运行：`docker run 镜像名 -t` → `nginx -t`（检查配置，覆盖默认参数）。

#### 场景 2：自定义启动脚本（复杂场景）
目标：容器启动前做初始化（如权限配置），再执行主程序，且保证主程序接收信号。
步骤 1：编写启动脚本 `entrypoint.sh`（需加 `exec` 让主程序成为 PID 1）：
```bash
#!/bin/sh
set -e

# 初始化操作：设置目录权限
chown -R www-data:www-data /var/www/html

# 执行主程序（$@ 接收所有传入的参数）
exec nginx -g "daemon off;" "$@"
```
步骤 2：Dockerfile 中配置：
```dockerfile
FROM nginx:alpine
COPY entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/entrypoint.sh
# Exec 形式调用脚本
ENTRYPOINT ["entrypoint.sh"]
# 无默认参数，也可加 CMD 提供
```
- 运行容器：`docker run 镜像名` → 先执行初始化，再运行 `nginx`；
- `docker stop` 能优雅停止 `nginx`（因为 `exec` 让 `nginx` 成为 PID 1）。

#### 场景 3：覆盖 ENTRYPOINT（运行时修改）
若需临时修改容器的入口点，可使用 `docker run --entrypoint` 标志（仅能指定可执行文件，无 Shell 解析）：
```bash
# 临时将 nginx 容器的入口点改为 bash
docker run --entrypoint bash -it 镜像名
```

### 五、关键注意事项
1. **唯一性**：一个 Dockerfile 中仅最后一个 `ENTRYPOINT` 生效（前面的会被覆盖）；
2. **信号处理**：只有 Exec 形式的 `ENTRYPOINT` 能让主程序成为 PID 1，接收 `SIGTERM` 信号（`docker stop` 优雅停止）；
3. **环境变量解析**：Exec 形式不会自动解析环境变量（如 `ENTRYPOINT ["echo", "$HOME"]` 会输出 `$HOME` 而非实际路径），若需解析变量，可通过 Shell 脚本或 `ENTRYPOINT ["/bin/sh", "-c", "echo $HOME"]` 实现；
4. **基础镜像继承**：若基础镜像已定义 `ENTRYPOINT`，当前 Dockerfile 会重置 `CMD` 为空，需重新定义 `CMD`。

### 六、Shell 形式 vs Exec 形式 对比（避坑重点）
| 特性                | Shell 形式（ENTRYPOINT command） | Exec 形式（ENTRYPOINT ["exec"]） |
|---------------------|----------------------------------|----------------------------------|
| PID 1 进程          | /bin/sh -c                       | 自定义可执行文件                 |
| 接收 SIGTERM 信号   | ❌ 否（容器强制停止）            | ✅ 是（优雅停止）                |
| 支持 CMD 参数       | ❌ 忽略                          | ✅ 支持（覆盖/追加）             |
| 运行时参数覆盖      | ❌ 忽略                          | ✅ 支持                          |
| 环境变量解析        | ✅ 自动解析                      | ❌ 需手动处理（脚本/Shell）      |
| 官方推荐            | ❌ 不推荐                        | ✅ 推荐                          |

### 总结
1. `ENTRYPOINT` 用于定义容器启动的「核心可执行程序」，让容器像独立工具一样运行，优先使用 **Exec 形式**；
2. Exec 形式的 `ENTRYPOINT` 配合 `CMD` 是最佳实践：`ENTRYPOINT` 定主程序，`CMD` 提供默认参数，运行时参数可覆盖 `CMD`；
3. Shell 形式的 `ENTRYPOINT` 会导致主程序非 PID 1，无法接收信号，仅在必须解析环境变量且无其他方案时使用（需加 `exec` 优化）；
4. 核心规则：`docker run` 传入的参数会追加到 Exec 形式 `ENTRYPOINT` 之后，覆盖 `CMD` 的默认值；Shell 形式 `ENTRYPOINT` 忽略所有外部参数。

