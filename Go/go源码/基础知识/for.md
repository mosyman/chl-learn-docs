

你想全面掌握 Go 语言中 `for` 循环的所有用法，我会从基础语法到进阶特性，结合实战示例帮你系统梳理——Go 没有 `while`/`do-while` 循环，所有循环逻辑都靠 `for` 实现，这也是它和其他语言的核心区别之一。

### 一、核心基础：for 循环的本质
`for` 是 Go 中唯一的循环语句，支持**三种语法形式**，能覆盖所有循环场景（普通计数循环、条件循环、无限循环、遍历循环）。核心语法框架：
```go
for [初始化语句]; [条件表达式]; [后置语句] {
    // 循环体（条件为true时执行）
}
```
- 初始化语句：循环开始前执行一次（通常定义循环变量）；
- 条件表达式：每次循环前判断，`true` 执行循环体，`false` 退出；
- 后置语句：每次循环体执行完后执行（通常更新循环变量）；
- 注意：Go 不支持 `for (i=0; i<10; i++)` 这种带括号的写法，括号会编译报错。

### 二、for 循环的四种核心用法
#### 1. 标准计数循环（最常用，对应其他语言的 for-i 循环）
**语法**：完整的初始化+条件+后置语句，适合明确循环次数的场景。
```go
package main

import "fmt"

func main() {
    // 经典用法：循环10次（i从0到9）
    for i := 0; i < 10; i++ {
        fmt.Printf("i = %d\n", i)
    }

    // 进阶：初始化语句可定义多个变量，后置语句可执行多个操作
    for i, j := 0, 10; i < 5; i++, j-- {
        fmt.Printf("i=%d, j=%d\n", i, j) // 输出 i=0,j=10; i=1,j=9...i=4,j=6
    }
}
```
**关键说明**：
- 初始化语句定义的变量（如 `i`）作用域仅限循环体内部；
- 后置语句常用 `i++`/`i--`，也可以是 `i += 2` 等复合操作。

#### 2. 条件循环（替代 while 循环）
**语法**：省略初始化和后置语句，只保留条件表达式，对应其他语言的 `while(条件)`。
```go
func main() {
    num := 0
    // 条件循环：num < 5 时持续执行
    for num < 5 {
        fmt.Printf("num = %d\n", num) // 输出 0-4
        num++ // 手动更新变量，避免死循环
    }
}
```
**典型场景**：循环次数不明确，依赖动态条件（如用户输入、数据读取）。

#### 3. 无限循环（替代 do-while/死循环）
**语法**：省略所有语句（仅保留 `for` + `{}`），条件永远为 `true`，需手动用 `break` 退出。
```go
func main() {
    count := 0
    // 无限循环
    for {
        count++
        fmt.Printf("count = %d\n", count)
        // 手动退出条件
        if count >= 3 {
            break // 跳出循环
        }
    }
    // 输出：count=1; count=2; count=3
}
```
**扩展：模拟 do-while 循环**  
Go 没有原生 `do-while`，但可通过“无限循环+条件判断”实现（先执行循环体，再判断是否退出）：
```go
func main() {
    num := 0
    for {
        // 先执行循环体
        fmt.Printf("num = %d\n", num)
        num++
        // 后判断条件（模拟 do-while）
        if num >= 3 {
            break
        }
    }
}
```

#### 4. for-range 遍历（核心特性，遍历集合）
`for-range` 是 Go 特有的遍历语法，支持遍历**切片、数组、字符串、map、通道**，语法简洁且安全（不会越界）。

##### 4.1 遍历切片/数组
```go
func main() {
    nums := []int{10, 20, 30, 40}
    // index：索引，value：对应值（副本）
    for index, value := range nums {
        fmt.Printf("索引：%d，值：%d\n", index, value)
    }

    // 省略值：只遍历索引
    for i := range nums {
        fmt.Printf("索引：%d\n", i)
    }

    // 省略索引：用 _ 占位（Go 不允许变量声明后不使用）
    for _, v := range nums {
        fmt.Printf("值：%d\n", v)
    }
}
```

##### 4.2 遍历字符串（按字符，而非字节）
```go
func main() {
    str := "Go语言"
    // index：字节索引，value：字符的 Unicode 码点（rune 类型）
    for i, c := range str {
        fmt.Printf("字节索引：%d，字符：%c（Unicode：%d）\n", i, c, c)
    }
    // 输出：
    // 字节索引：0，字符：G（Unicode：71）
    // 字节索引：1，字符：o（Unicode：111）
    // 字节索引：2，字符：语（Unicode：35821）
    // 字节索引：5，字符：言（Unicode：35328）
}
```
**关键说明**：中文等非 ASCII 字符占多个字节，`for-range` 会自动按 Unicode 字符拆分，避免乱码。

##### 4.3 遍历 map（无序）
```go
func main() {
    userAge := map[string]int{
        "张三": 20,
        "李四": 25,
        "王五": 30,
    }
    // key：键，value：值（map 遍历顺序不固定）
    for key, value := range userAge {
        fmt.Printf("键：%s，值：%d\n", key, value)
    }

    // 只遍历键
    for k := range userAge {
        fmt.Printf("键：%s\n", k)
    }

    // 只遍历值
    for _, v := range userAge {
        fmt.Printf("值：%d\n", v)
    }
}
```

##### 4.4 遍历通道（channel）
```go
func main() {
    ch := make(chan int, 3)
    ch <- 1
    ch <- 2
    ch <- 3
    close(ch) // 关闭通道，否则遍历会阻塞

    // 遍历通道：取出所有元素，通道关闭后自动退出
    for v := range ch {
        fmt.Printf("通道值：%d\n", v) // 输出 1、2、3
    }
}
```

### 三、循环控制关键字（break/continue/goto/label）
#### 1. break：跳出当前循环
```go
func main() {
    for i := 0; i < 10; i++ {
        if i == 5 {
            break // 跳出整个循环，后续i=5~9不再执行
        }
        fmt.Println(i) // 输出 0-4
    }
}
```

#### 2. continue：跳过当前循环，进入下一次
```go
func main() {
    for i := 0; i < 5; i++ {
        if i == 2 {
            continue // 跳过i=2，直接执行i++
        }
        fmt.Println(i) // 输出 0、1、3、4
    }
}
```

#### 3. 标签（label）+ break/continue：跳出多层循环
Go 支持给循环加标签，用于多层循环中精准控制退出/跳过的层级（标签名大写，是通用规范）：
```go
func main() {
    // 定义标签（对应外层循环）
    OUTER:
    for i := 0; i < 3; i++ {
        for j := 0; j < 3; j++ {
            if i == 1 && j == 1 {
                break OUTER // 跳出外层循环，整个嵌套循环结束
                // continue OUTER // 跳过外层循环的当前次，i直接变为2
            }
            fmt.Printf("i=%d, j=%d\n", i, j)
        }
    }
}
```

#### 4. goto：跳转到标签（慎用）
`goto` 可跳转到同一函数内的标签，适合简化多层循环退出，但易导致代码混乱，建议优先用标签+break：
```go
func main() {
    for i := 0; i < 10; i++ {
        if i == 5 {
            goto END // 跳转到END标签处
        }
        fmt.Println(i) // 输出 0-4
    }
END:
    fmt.Println("循环结束")
}
```

### 四、实战场景示例
#### 1. 遍历切片并修改元素（注意 for-range 的值是副本）
```go
func main() {
    nums := []int{1, 2, 3}
    // 错误：v是副本，修改v不会改变原切片
    // for _, v := range nums {
    //     v *= 2
    // }

    // 正确：通过索引修改原切片
    for i := range nums {
        nums[i] *= 2
    }
    fmt.Println(nums) // 输出 [2 4 6]
}
```

#### 2. 循环读取用户输入（无限循环+条件退出）
```go
package main

import (
    "bufio"
    "fmt"
    "os"
)

func main() {
    scanner := bufio.NewScanner(os.Stdin)
    fmt.Println("输入'exit'退出：")
    for {
        fmt.Print("> ")
        scanner.Scan()
        input := scanner.Text()
        if input == "exit" {
            fmt.Println("退出程序")
            break
        }
        fmt.Printf("你输入了：%s\n", input)
    }
}
```

#### 3. 遍历 map 并删除元素（安全操作）
```go
func main() {
    m := map[string]int{"a":1, "b":2, "c":3}
    // 遍历map时删除元素是安全的
    for k := range m {
        if k == "b" {
            delete(m, k)
        }
    }
    fmt.Println(m) // 输出 map[a:1 c:3]
}
```

### 五、常见坑点
1. **for-range 遍历的是值副本**：修改 `v` 不会改变原集合，需通过索引/键修改；
2. **遍历字符串用普通 for 会乱码**：普通 `for i:=0; i<len(str); i++` 按字节遍历，非 ASCII 字符需用 `for-range`；
3. **map 遍历顺序不固定**：Go 故意打乱 map 遍历顺序，不要依赖遍历顺序；
4. **无限循环忘记 break**：必须在循环体内加退出条件，否则程序卡死；
5. **循环变量复用问题**（Go 1.21 前）：
   ```go
   // 错误示例（Go 1.20及以前）：所有goroutine共用i的最终值（5）
   // for i := 0; i < 5; i++ {
   //     go func() {
   //         fmt.Println(i) // 大概率输出5个5
   //     }()
   // }

   // 正确：把i作为参数传入
   for i := 0; i < 5; i++ {
       go func(n int) {
           fmt.Println(n) // 输出 0-4（顺序不固定）
       }(i)
   }
   ```

### 总结
1. Go 只有 `for` 一种循环，通过不同语法形式实现计数、条件、无限循环，替代 `while`/`do-while`；
2. 核心特性：`for-range` 安全遍历切片/数组/字符串/map/通道，按 Unicode 处理字符串；
3. 循环控制：`break`/`continue` 控制单层循环，标签（label）控制多层循环，`goto` 慎用；
4. 避坑关键：for-range 的值是副本、字符串遍历用 for-range、map 遍历顺序不固定。

掌握这些用法后，你能实现所有常见的循环逻辑，覆盖日常开发中遍历、计数、条件循环等场景。

















