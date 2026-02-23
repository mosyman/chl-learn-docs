
从你的截图可以看到两个关键问题，它们是导致报错的核心原因：

---

### 1. 未定义 GOROOT（核心问题）
**原因**：
IDE（GoLand/IDEA）找不到 Go 的安装目录，无法定位 Go 编译器、标准库和工具链，因此无法解析 `import "fmt"` 等标准库包，导致代码报错。

**解决方法**：
1.  点击右侧的「设置 GOROOT」，或在 IDE 中打开设置：
    - GoLand：`Preferences → Go → GOROOT`
    - IDEA：`File → Project Structure → SDKs`
2.  选择你本地 Go 的安装路径：
    - Mac/Linux：默认路径通常是 `/usr/local/go` 或 `/usr/local/Cellar/go/...`（Homebrew 安装）
    - Windows：默认路径通常是 `C:\Go`
3.  确认后，IDE 会自动加载标准库，报错即可消失。

---

### 2. 未设置 GOPROXY（次要但影响依赖下载）
**原因**：
在国内网络环境下，直接访问 Go 官方模块仓库（proxy.golang.org）会被墙，导致 `go mod tidy` 等依赖下载命令失败。

**解决方法**：
1.  点击右侧的「在全局范围内设置 goproxy.cn」，或手动执行以下命令：
    ```bash
    go env -w GOPROXY=https://goproxy.cn,direct
    ```
2.  这会将 Go 模块代理设置为国内镜像，解决依赖下载慢/失败的问题。

---

### 3. 验证修复
完成以上两步后，在终端执行：
```bash
go version
go run .
```
如果能正常打印版本号和 `Hello, World!`，说明环境配置成功。

---

如果你不确定 Go 的安装路径，可以在终端执行 `go env GOROOT` 命令，它会直接告诉你正确的路径。需要我帮你一步步排查吗？








