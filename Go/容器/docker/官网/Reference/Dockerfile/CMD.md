
### 一、`CMD` 指令的核心定义
`CMD` 是 Dockerfile 中的核心指令之一，它的唯一作用是**为基于该镜像启动的容器设置「默认要执行的命令」**。
- 这个命令**不会在镜像构建阶段（`docker build`）执行**，只在容器运行阶段（`docker run`）生效。
- 如果用户运行容器时手动指定了命令（比如 `docker run 镜像名 自定义命令`），则会覆盖 `CMD` 设置的默认命令。

### 二、`CMD` 的三种写法（格式）
Docker 提供了三种合法的 `CMD` 写法，核心分为「exec 格式」和「shell 格式」两类，各自有不同的使用场景：

#### 1. Exec 格式（推荐）
这是 Docker 官方推荐的写法，也是最规范的格式，使用 JSON 数组形式，必须用双引号（`"`），不能用单引号。
- **写法 1**：`CMD ["可执行文件", "参数1", "参数2"]`
  作用：直接指定容器启动时要运行的可执行文件和参数。
  示例：
  ```dockerfile
  # 启动容器时执行 `echo "Hello Docker"`
  CMD ["echo", "Hello Docker"]
  ```
- **写法 2**：`CMD ["参数1", "参数2"]`
  作用：仅作为 `ENTRYPOINT` 指令的「默认参数」，必须和 `ENTRYPOINT` 配合使用（且两者都要使用 exec 格式）。
  示例：
  ```dockerfile
  # ENTRYPOINT 定死执行 echo，CMD 提供默认参数
  ENTRYPOINT ["echo"]
  CMD ["Hello", "Docker"]
  # 运行容器时，默认执行 `echo Hello Docker`；如果手动传参 `docker run 镜像名 Hi`，则执行 `echo Hi`
  ```

#### 2. Shell 格式
写法：`CMD 命令 参数1 参数2`
作用：底层会调用 `/bin/sh -c` 来执行命令，相当于间接执行（会创建 shell 进程）。
示例：
```dockerfile
# 等价于 CMD ["/bin/sh", "-c", "echo Hello Docker"]
CMD echo Hello Docker
```
⚠️ 注意：shell 格式不适合和 `ENTRYPOINT` 配合传递参数，也可能导致信号处理问题（比如 `docker stop` 无法正确终止进程），因此优先用 exec 格式。

### 三、`CMD` 的关键规则
1. **唯一性**：一个 Dockerfile 中可以写多个 `CMD`，但只有**最后一个**会生效（前面的都会被覆盖）。
   反例（错误示范）：
   ```dockerfile
   CMD ["echo", "第一个CMD"]
   CMD ["echo", "第二个CMD"] # 只有这行生效
   ```
2. **可覆盖性**：用户运行容器时传入的命令会覆盖 `CMD` 的默认值。
   示例：
   假设 Dockerfile 中有 `CMD ["echo", "默认内容"]`，运行容器时执行：
   ```bash
   docker run 镜像名 echo "自定义内容"
   ```
   最终容器会执行 `echo "自定义内容"`，而非默认的 `echo "默认内容"`。
3. **与 `ENTRYPOINT` 的配合**：
    - 若想让容器「固定执行某个程序，仅允许修改参数」，用 `ENTRYPOINT + CMD`（exec 格式）。
    - 若 `ENTRYPOINT` 是 shell 格式，`CMD` 的参数会失效（不推荐这么用）。

### 四、`CMD` vs `RUN`（核心区别）
这是新手最容易混淆的两个指令，核心差异如下：

| 特性         | `CMD`                          | `RUN`                          |
|--------------|--------------------------------|--------------------------------|
| 执行阶段     | 容器运行阶段（`docker run`）   | 镜像构建阶段（`docker build`） |
| 执行目的     | 设置容器启动的默认命令         | 执行命令并将结果提交到镜像层   |
| 执行次数     | 仅容器启动时执行一次           | 构建时执行（写几次就执行几次） |
| 能否覆盖     | 可以被 `docker run` 参数覆盖   | 构建后固化到镜像，无法覆盖     |

示例对比：
```dockerfile
# RUN：构建时执行，会在镜像中创建 test.txt 文件
RUN echo "构建时内容" > /test.txt

# CMD：运行容器时执行，仅输出内容（不会修改镜像）
CMD cat /test.txt
```

### 总结
1. `CMD` 是为容器设置**运行时默认命令**，仅在 `docker run` 阶段生效，构建阶段不执行。
2. `CMD` 有 exec（推荐）和 shell 两种格式，exec 格式适合和 `ENTRYPOINT` 配合传递参数。
3. 一个 Dockerfile 中仅最后一个 `CMD` 生效，且其默认值可被 `docker run` 的自定义命令覆盖。
4. 区分 `CMD` 和 `RUN`：`RUN` 是「构建时执行并固化结果」，`CMD` 是「运行时指定默认行为」。




