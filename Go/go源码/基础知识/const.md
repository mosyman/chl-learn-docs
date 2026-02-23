

你想了解 Go 语言中 `const`（常量）的相关概念和用法，我会从基础到应用，用通俗易懂的方式给你讲清楚。

### 一、const 是什么？
`const` 是 Go 语言中用于定义**常量**的关键字，常量指的是程序运行期间值**不会也不能被修改**的量（比如数学中的 π、一周的天数 7 等）。

和变量（`var` 定义）的核心区别：
- 变量：运行时可修改，存储在内存中；
- 常量：编译时确定值，运行时不可修改，通常直接嵌入到代码中，更节省内存。

### 二、const 的基本用法
#### 1. 定义单个常量
```go
package main

import "fmt"

func main() {
    // 定义字符串常量
    const name = "Go语言"
    // 定义数值常量（无需指定类型，Go会自动推导）
    const pi = 3.1415926
    // 显式指定类型的常量
    const age int = 100

    fmt.Println(name) // 输出：Go语言
    fmt.Println(pi)   // 输出：3.1415926
    fmt.Println(age)  // 输出：100

    // 尝试修改常量（会直接编译报错）
    // age = 200 // 错误：cannot assign to age (untyped int constant 100)
}
```

#### 2. 批量定义常量
可以用一个 `const` 块批量定义多个常量，简化代码：
```go
package main

import "fmt"

func main() {
    const (
        Monday = 1
        Tuesday = 2
        Wednesday = 3
        // 后续常量如果省略值，会继承上一个常量的值
        Thursday // 等同于 Thursday = 3
        Friday   // 等同于 Friday = 3
    )

    fmt.Println(Monday)    // 1
    fmt.Println(Thursday)  // 3
}
```

#### 3. 特殊：iota 常量生成器
`iota` 是 Go 内置的常量计数器，在 `const` 块中，第一个 `iota` 为 0，后续每新增一行常量声明，`iota` 自动加 1，非常适合定义枚举类常量：
```go
package main

import "fmt"

func main() {
    const (
        // 第一行 iota = 0
        Jan = iota 
        // 第二行 iota = 1
        Feb       
        // 第三行 iota = 2
        Mar       
        // 第四行 iota = 3
        Apr       
    )

    fmt.Println(Jan) // 0
    fmt.Println(Feb) // 1
    fmt.Println(Mar) // 2

    // 进阶：结合表达式使用
    const (
        // 1 << 0 = 1
        _ = 1 << (10 * iota) 
        // 1 << 10 = 1024
        KB                   
        // 1 << 20 = 1048576
        MB                   
        // 1 << 30 = 1073741824
        GB                   
    )
    fmt.Println(KB) // 1024
    fmt.Println(MB) // 1048576
}
```

### 三、const 的重要特性
1. **类型限制**：常量的类型可以是布尔型、字符串型、数值型（整数、浮点数、复数），不支持切片、映射（map）、结构体等复杂类型；
2. **编译期确定**：常量的值必须在编译时就能确定，不能用运行时才知道的值（比如用户输入、函数返回值）赋值；
   ```go
   // 错误示例：runtime.NumCPU() 是运行时获取的CPU核心数，编译时无法确定
   // const cpuNum = runtime.NumCPU() 
   ```
3. **无存储地址**：常量不像变量那样占用内存地址，无法用 `&` 取地址；
4. **隐式转换**：不同类型的常量可以直接运算（比如 `1 + 3.14` 会自动转换为浮点数），而变量不行。

### 总结
1. `const` 用于定义**运行时不可修改**的常量，值在编译期确定，比变量更高效；
2. 批量定义常量时，省略值会继承上一个值，结合 `iota` 可快速定义枚举类常量；
3. 常量仅支持基本类型（布尔、字符串、数值），不支持复杂类型，也不能用运行时动态值赋值。







