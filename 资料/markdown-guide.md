
## Contents

<details open>
<summary>目录</summary>

- [Contents](#contents)
- [常用术语](#常用术语)
- [markdown常用语法](#markdown常用语法)
- [details标签](#details标签)
    - [一、基础概念与语法](#一基础概念与语法)
    - [二、实用示例](#二实用示例)
    - [三、使用场景](#三使用场景)
    - [总结](#总结)

</details>

## 常用术语
> 用**纯标准 Markdown 格式**，可直接复制到 `.md` 文件中使用，无特殊格式、无多余样式。

> 原本英文翻译插入中文翻译+详细解释代码（用**纯标准 Markdown 格式**，可直接复制到 `.md` 文件中使用)


> 转换为中英文翻译,示例如下：
> /**
> * A WebSocket session abstraction. Allows sending messages over a WebSocket
> * connection and closing it.
> * <p>WebSocket 会话的抽象接口。支持通过 WebSocket 连接发送消息，也支持关闭该连接。
> * <p>Finish the initialization of this context's bean factory,
> * initializing all remaining singleton beans.
> * <p>完成此上下文的 Bean 工厂初始化，初始化所有剩余的单例 Bean。*/

**[⬆ back to top](#contents)**

## markdown常用语法

### 1. 设置字体颜色
> <span style="color: red;">这段文字是红色的</span>
<span style="color: #0066cc;">这段文字是蓝色的（十六进制）</span>
<span style="color: rgb(0, 128, 0);">这段文字是绿色的（RGB）</span>

### 2. 设置字体大小
> <span style="font-size: 12px;">小号字体（12px）</span>
<span style="font-size: 20px;">大号字体（20px）</span>
<span style="font-size: 1.5em;">相对大小（1.5em）</span>

### 3. 同时设置颜色和大小
> <span style="color: #ff6600; font-size: 16px; font-weight: bold;">大号橙色加粗文字</span>


**[⬆ back to top](#contents)**

## details标签

你想了解 Markdown 中 `<details>` 和 `</details>` 标签的含义、用法和实际场景，这是 HTML 原生标签（非标准 Markdown 语法，但绝大多数 Markdown 渲染器都支持），核心作用是**创建可折叠/展开的内容块**，能让文档更简洁、层次更清晰。

**[⬆ back to top](#contents)**

### 一、基础概念与语法
#### 1. 核心作用
`<details>` 是 HTML5 新增的「详情披露元素」，搭配 `<summary>` 标签使用，实现：
- 默认折叠内容，只显示标题（`<summary>` 里的内容）；
- 点击标题可展开/收起完整内容；
- 无需 JavaScript，纯 HTML 原生实现，Markdown 中可直接嵌入使用。

#### 2. 基础语法
```markdown
<details>
  <summary>点击展开/收起（这里是折叠标题）</summary>
  
  这里是折叠的内容，可以写：
  - 多行文本
  - 列表
  - 代码块
  - 其他 Markdown 内容
</details>
```

#### 3. 关键说明
- `<summary>`：**必须嵌套在 `<details>` 内**，定义折叠块的标题（点击区域），省略的话，渲染器会显示默认标题（如「Details」）；
- 内容分隔：`<summary>` 后空一行，再写折叠内容，排版更清晰；
- 兼容性：GitHub、GitLab、VS Code、Typora 等主流工具都支持，纯静态 Markdown 编辑器可能不兼容（需预览模式）。

**[⬆ back to top](#contents)**


### 二、实用示例
#### 1. 基础折叠块
```markdown
<details>
  <summary>Linux curl 常用参数</summary>
  
  | 参数 | 作用 |
  |------|------|
  | -v   | 显示详细请求过程 |
  | -O   | 按原文件名保存文件 |
  | -L   | 跟随重定向 |
</details>
```
**渲染效果**：
<details>
  <summary>Linux curl 常用参数</summary>

| 参数 | 作用 |
  |------|------|
| -v   | 显示详细请求过程 |
| -O   | 按原文件名保存文件 |
| -L   | 跟随重定向 |
</details>

#### 2. 默认展开的折叠块
给 `<details>` 加 `open` 属性，默认展开内容：
```markdown
<details open>
  <summary>curl -v 输出解析（默认展开）</summary>
  
  - * 开头：curl 调试日志
  - > 开头：客户端请求头
  - < 开头：服务器响应头
</details>
```
**渲染效果**：
<details open>
  <summary>curl -v 输出解析（默认展开）</summary>

- * 开头：curl 调试日志
- > 开头：客户端请求头
- < 开头：服务器响应头
</details>

#### 3. 嵌套折叠块
支持多层嵌套，适合复杂文档的层级管理：
```markdown
<details>
  <summary>curl 高级用法</summary>
  
  <details>
    <summary>1. POST 请求（JSON 格式）</summary>

    ```bash
    curl -X POST -H "Content-Type: application/json" -d '{"name":"test"}' https://api.example.com
    ```
  </details>
  
  <details>
    <summary>2. 断点续传下载</summary>

    ```bash
    curl -C - -O https://example.com/bigfile.iso
    ```
  </details>
</details>
```
**渲染效果**：
<details>
  <summary>curl 高级用法</summary>

  <details>
    <summary>1. POST 请求（JSON 格式）</summary>

    ```bash
    curl -X POST -H "Content-Type: application/json" -d '{"name":"test"}' https://api.example.com
    ```
  </details>

  <details>
    <summary>2. 断点续传下载</summary>

    ```bash
    curl -C - -O https://example.com/bigfile.iso
    ```
  </details>
</details>

**[⬆ back to top](#contents)**


### 三、使用场景
1. **文档精简**：把次要信息（如详细参数、代码示例、日志解析）折叠，只展示核心内容，避免页面过长；
2. **教程/笔记**：分步骤折叠，比如「基础用法」「进阶用法」「实战示例」，读者按需展开；
3. **代码说明**：折叠冗长的代码块或日志内容，只显示关键代码片段，提升可读性；
4. **FAQ 文档**：每个问题作为 `<summary>`，答案折叠在内部，点击展开查看。

**[⬆ back to top](#contents)**


### 总结
1. `<details>` 是 HTML 标签（非标准 Markdown），核心功能是创建**可折叠内容块**，搭配 `<summary>` 定义标题；
2. 常用属性：`open`（默认展开），支持嵌套、兼容主流 Markdown 渲染器；
3. 核心价值：精简文档结构，让读者聚焦核心内容，按需查看细节。


**[⬆ back to top](#contents)**



























