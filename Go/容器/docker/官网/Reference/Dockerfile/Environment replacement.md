

### 一、核心定义与基础写法
环境变量替换指的是：Dockerfile 中通过 `ENV` 声明的变量，或构建时传入的变量，可以在指定指令中被解析替换成实际值。

#### 1. 基础变量写法（两种等效）
Dockerfile 中引用环境变量有两种格式，效果完全一样：
- 简洁版：`$variable_name`（比如 `$FOO`）；
- 花括号版：`${variable_name}`（比如 `${FOO}`）。

**花括号版的核心优势**：解决变量名与后续字符拼接的问题，比如 `${foo}_bar` 能明确识别变量是 `foo`，而 `$foo_bar` 会被识别为变量 `foo_bar`（而非 `foo` + `_bar`）。

#### 2. 基础示例（直观理解）
```dockerfile
FROM busybox
ENV FOO=/bar          # 声明变量 FOO，值为 /bar
WORKDIR ${FOO}        # 解析为 WORKDIR /bar
ADD . $FOO            # 解析为 ADD . /bar
COPY \$FOO /quux      # 转义后，解析为 COPY $FOO /quux（保留 $FOO 字面量）
```

### 二、环境变量的 bash 风格修饰符
Dockerfile 支持部分 bash 风格的变量修饰符，用于对变量值做动态处理，分为「基础修饰符（全版本支持）」和「高级修饰符（预发布版本支持）」两类。

#### 1. 基础修饰符（所有版本支持）
| 语法                  | 作用说明                                                                 |
|-----------------------|--------------------------------------------------------------------------|
| `${variable:-word}`   | 变量 `variable` 已设置 → 用变量值；未设置 → 用 `word` 作为默认值         |
| `${variable:+word}`   | 变量 `variable` 已设置 → 用 `word`；未设置 → 结果为空字符串              |

**示例**：
```dockerfile
# 未声明 BAR 变量
ENV FOO=hello
WORKDIR ${FOO:-world}  # FOO 已设置 → WORKDIR hello
WORKDIR ${BAR:-world}  # BAR 未设置 → WORKDIR world
WORKDIR ${FOO:+world}  # FOO 已设置 → WORKDIR world
WORKDIR ${BAR:+world}  # BAR 未设置 → WORKDIR （空）
```

#### 2. 高级修饰符（预发布版本支持）
需先在 Dockerfile 顶部声明特殊语法指令：`# syntax=docker/dockerfile-upstream:master`，支持字符串截取、替换类修饰符：

| 语法                  | 作用说明                                                                 | 示例（str=foobarbaz） | 结果   |
|-----------------------|--------------------------------------------------------------------------|-----------------------|--------|
| `${variable#pattern}` | 从字符串开头，移除 `pattern` 的**最短匹配**（glob 模式）                  | `${str#f*b}`          | arbaz  |
| `${variable##pattern}`| 从字符串开头，移除 `pattern` 的**最长匹配**（glob 模式）                  | `${str##f*b}`         | az     |
| `${variable%pattern}` | 从字符串末尾，移除 `pattern` 的**最短匹配**（glob 模式）                  | `${string%b*}`        | foobar |
| `${variable%%pattern}`| 从字符串末尾，移除 `pattern` 的**最长匹配**（glob 模式）                  | `${string%%b*}`       | foo    |
| `${variable/pattern/replacement}` | 替换 `pattern` 的**第一次匹配**为 `replacement` | `${string/ba/fo}`     | fooforbaz |
| `${variable//pattern/replacement}` | 替换 `pattern` 的**所有匹配**为 `replacement` | `${string//ba/fo}`    | fooforfoz |

**关键补充**：
- `pattern` 是 glob 通配符：`?` 匹配单个字符，`*` 匹配任意字符（包括0个）；
- 匹配字面量 `?`/`*` 需转义：`\?`、`\*`。

### 三、变量转义（保留字面量 $）
如果不想让 Docker 解析变量，而是保留 `$variable` 字面量，需要在 `$` 前加反斜杠 `\`：
- `\$foo` → 解析为 `$foo`（字面量）；
- `\${foo}` → 解析为 `${foo}`（字面量）。

**示例**：
```dockerfile
COPY \$FOO /quux  # 最终执行 COPY $FOO /quux（$FOO 不会被替换）
COPY \${FOO} /quux # 最终执行 COPY ${FOO} /quux
```

### 四、支持环境变量替换的指令
环境变量替换的处理主体分两类，规则不同：

#### 1. 由 Docker 构建器（builder）处理的指令（核心支持）
这些指令的变量替换由 Docker 直接解析，无需依赖 shell：
- `ADD`、`COPY`、`ENV`、`EXPOSE`、`FROM`、`LABEL`
- `STOPSIGNAL`、`USER`、`VOLUME`、`WORKDIR`
- `ONBUILD`（仅当结合上述指令时，比如 `ONBUILD WORKDIR $FOO`）

#### 2. 由 shell 处理的指令（间接支持）
`RUN`、`CMD`、`ENTRYPOINT` 也能使用环境变量，但替换由**命令 shell**（如 `/bin/sh`）处理，而非 Docker 构建器：
- 「shell 形式」指令（如 `RUN echo $FOO`）：自动调用 shell，支持变量替换；
- 「exec 形式」指令（如 `RUN ["echo", "$FOO"]`）：不自动调用 shell，`$FOO` 会被当作字面量，需手动指定 shell：
  ```dockerfile
  # 正确：手动调用 shell 处理变量
  RUN ["/bin/sh", "-c", "echo $FOO"]
  # 错误：exec 形式无 shell，$FOO 不会被替换
  RUN ["echo", "$FOO"]
  ```

### 五、变量替换的执行逻辑（关键坑点）
#### 1. 单指令内变量值固定
同一指令中，变量替换会使用**指令开始解析时**的变量值，即使指令内修改了变量，也不会立即生效：
```dockerfile
ENV abc=hello
ENV abc=bye def=$abc  # def 取 abc 初始值 hello，而非新值 bye
ENV ghi=$abc          # ghi 取 abc 新值 bye
```
最终结果：
- `def = hello`（指令执行时 abc 还是 hello）；
- `ghi = bye`（下一条指令，abc 已更新为 bye）。

#### 2. 变量生效范围
修改变量后，新值仅对**后续指令**生效，当前指令内的替换仍用旧值（如上例）。

### 总结
1. 环境变量引用有 `$var` 和 `${var}` 两种写法，花括号版解决变量名拼接问题；
2. 支持 bash 风格修饰符：基础版（`:-`/`:+`）全版本支持，高级版（截取/替换）需指定预发布语法；
3. 转义 `\$var` 可保留字面量，避免 Docker 解析；
4. 变量替换主体分两类：构建器直接处理（ADD/WORKDIR 等）、shell 处理（RUN/CMD 等，exec 形式需手动指定 shell）；
5. 核心坑点：单指令内变量值固定，修改后仅对后续指令生效。

