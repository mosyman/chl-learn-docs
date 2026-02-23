


![const.png](img/const.png)


`maxInt` 读不到，是因为它在源码里是一个**局部常量**，只存在于 `Scan()` 函数的编译期，调试器在运行时看不到它的符号。

### 1. 源码位置与定义
在 `bufio.Scanner.Scan()` 函数内部，`maxInt` 是这样定义的：

```go
func (s *Scanner) Scan() bool {
    // ...
    const maxInt = int(^uint(0) >> 1)
    // ...
    if len(s.buf) >= s.maxTokenSize || len(s.buf) > maxInt/2 {
        s.setErr(ErrTooLong)
        return false
    }
    // ...
}
```

- `maxInt` 是一个**函数内的局部常量**，不是包级变量，也不是结构体字段。
- 它的值是 `int` 类型的最大值，由 `^uint(0) >> 1` 计算得出，这个值在编译时就已经确定，不会作为运行时符号存在。

### 2. 调试器为什么看不到它
- <span style="color: #ff6600; font-size: 16px; font-weight: bold;">编译期常量</span>：Go 编译器会把 `maxInt` 直接内联到代码中，比如 `maxInt/2` 会被替换成一个具体的数字（如 64 位系统上是 4611686018427387903），调试器在运行时找不到 `maxInt` 这个符号名。
- **局部作用域**：它只在 `Scan()` 函数内部可见，调试器的变量列表默认展示的是当前栈帧的变量，而<span style="color: #ff6600; font-size: 16px; font-weight: bold;">这个常量在运行时已经被“吃掉”了，没有对应的内存位置</span>。

### 3. 如何在调试时确认它的值
你可以在调试器的“表达式求值”窗口直接计算它的值，而不是试图读取 `maxInt` 这个符号：

```go
// 在调试器的 Watch 或 Evaluate 中输入：
int(^uint(0) >> 1)
```

在 64 位系统上，这个表达式会返回：

```text
9223372036854775807
```

这就是 `maxInt` 代表的 `int` 最大值。

---

如果你需要，我可以帮你写一段小代码，在运行时打印出这个 `maxInt` 的值，方便你验证。需要吗？




