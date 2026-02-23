你想了解 Go 语言中 `if` 语句的完整用法，我会从基础语法到进阶特性，用通俗易懂的方式帮你讲清楚 `if` 的所有核心用法和注意事项。

### 1. if 语句的核心定义
`if` 是 Go 中最基础的条件判断语句，用于根据**布尔表达式的结果**（`true`/`false`）执行不同的代码块。Go 对 `if` 的设计相比其他语言（如 Java/Python）有一些独特的简化和规范。

### 2. 基础用法（从简单到复杂）
#### 2.1 最基础的 if
**语法**：
```go
if 布尔表达式 {
    // 表达式为true时执行的代码
}
```
**示例**：
```go
package main

import "fmt"

func main() {
    age := 18
    if age >= 18 {
        fmt.Println("你已成年") // 输出：你已成年
    }
}
```
**关键说明**：
- 布尔表达式**不能省略括号**（和 Python 不同）；
- 代码块的大括号 `{}` **必须有**，即使只有一行代码（和 Java 不同，Go 不允许省略）；
- 大括号的左括号 `{` 必须和 `if` 写在同一行（Go 语法规范，否则编译报错）。

#### 2.2 if-else
**语法**：
```go
if 布尔表达式 {
    // 表达式为true时执行
} else {
    // 表达式为false时执行
}
```
**示例**：
```go
func main() {
    score := 59
    if score >= 60 {
        fmt.Println("及格")
    } else {
        fmt.Println("不及格") // 输出：不及格
    }
}
```

#### 2.3 if-else if-else（多条件判断）
**语法**：
```go
if 表达式1 {
    // 表达式1为true时执行
} else if 表达式2 {
    // 表达式1为false、表达式2为true时执行
} else {
    // 所有表达式都为false时执行
}
```
**示例**：
```go
func main() {
    score := 85
    if score >= 90 {
        fmt.Println("优秀")
    } else if score >= 80 {
        fmt.Println("良好") // 输出：良好
    } else if score >= 60 {
        fmt.Println("及格")
    } else {
        fmt.Println("不及格")
    }
}
```
**关键说明**：
- `else if` 可以有多个，执行顺序是“从上到下”，匹配到第一个 `true` 的表达式后，后续条件不再判断；
- 最后的 `else` 是可选的。

### 3. Go 特有的 if 初始化语句（核心特性）
这是 Go 区别于其他语言的重要特性：`if` 可以在判断表达式前加一个**初始化语句**，用于定义局部变量（仅在 `if-else` 块内有效）。

#### 语法
```go
if 初始化语句; 布尔表达式 {
    // 执行代码
}
```

#### 示例
```go
package main

import "fmt"

func main() {
    // 初始化语句：定义变量 num，仅在这个if-else块内有效
    if num := 10; num%2 == 0 {
        fmt.Printf("%d 是偶数\n", num) // 输出：10 是偶数
    } else {
        fmt.Printf("%d 是奇数\n", num)
    }

    // 错误：num 是if块内的局部变量，外部无法访问
    // fmt.Println(num)
}
```

#### 实用场景（最常用）
结合函数调用，直接判断返回值，避免额外定义变量：
```go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    // 初始化语句调用 strconv.Atoi，同时判断转换是否成功
    if num, err := strconv.Atoi("123"); err == nil {
        fmt.Println("转换成功：", num) // 输出：转换成功：123
    } else {
        fmt.Println("转换失败：", err)
    }
}
```
**关键说明**：
- 初始化语句和判断表达式之间用**分号 `;`** 分隔；
- 初始化的变量作用域仅限于当前 `if-else` 代码块，外部不可见；
- 这种写法能让代码更简洁，符合 Go “简洁至上”的设计理念。

### 4. 注意事项
1. **布尔表达式必须是纯布尔值**：Go 不允许隐式类型转换（如 `if 1` 是错误的），必须明确写 `if 1 != 0`；
   ```go
   // 错误示例
   // if 1 { // 编译报错：non-bool 1 (type int) used as if condition
   //     fmt.Println("错误")
   // }

   // 正确示例
   if 1 != 0 {
       fmt.Println("正确")
   }
   ```
2. **大括号的位置**：左括号 `{` 必须和 `if`/`else` 在同一行，否则编译失败（Go 编译器的强制规范）；
   ```go
   // 错误示例
   // if age > 18
   // { // 编译报错：syntax error: unexpected newline, expecting { after if clause
   //     fmt.Println("成年")
   // }

   // 正确示例
   if age > 18 {
       fmt.Println("成年")
   }
   ```
3. **避免冗余的 else**：如果 `if` 块内有 `return`/`break`/`continue` 等终止语句，`else` 可以省略，代码更简洁。
   ```go
   func checkAge(age int) string {
       if age >= 18 {
           return "成年"
       }
       // 无需写 else，因为上面的 if 满足时已经 return 了
       return "未成年"
   }
   ```

### 总结
1. Go 的 `if` 语句语法规范严格：括号不可省、大括号必须有、左括号和 `if` 同行；
2. 核心特色是**初始化语句 + 判断表达式**的组合写法，能简化变量定义，限定变量作用域；
3. 布尔表达式必须是纯布尔值，不支持隐式转换，多条件判断用 `else if` 实现。