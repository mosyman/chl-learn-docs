# Effective Go 详细解释

## 引言

### 概述

《Effective Go》是 Go 语言官方的编程指南，旨在帮助开发者编写清晰、地道且高效的 Go 代码。这份文档于 2009 年编写，虽然没有显著更新，但仍然是理解 Go 语言核心设计原则和最佳实践的重要资源。

### 重要说明

- 文档编写于 2009 年，主要关注语言本身的使用

- 对于 Go 生态系统的重大变化（如模块、构建系统、测试等）涉及较少

- 尽管如此，它仍然是学习 Go 语言核心概念的优秀指南

### 核心思想

Go 语言有其独特的设计理念，简单地将 C++ 或 Java 程序直接转换为 Go 代码通常不会产生令人满意的结果。要写好 Go 代码，需要理解其特性和惯用写法，以及遵循已建立的编程约定。

## 示例

### 标准库作为示例

Go 语言的包源代码不仅是核心库，也是使用语言的示例。许多包包含可直接运行的示例程序，可以在 go.dev 网站上运行。

### 学习建议

当你对如何解决某个问题或实现某个功能有疑问时，标准库中的文档、代码和示例可以提供答案、思路和背景知识。

## 格式化

### 自动化格式化

Go 语言采用了一种不同寻常的方法，让机器处理大部分格式化问题。`gofmt`程序（也可以通过`go fmt`命令使用）会读取 Go 程序并以标准格式输出源代码，包括缩进和垂直对齐，同时保留并在必要时重新格式化注释。

### 结构体字段注释对齐

例如，gofmt 会自动对齐结构体字段的注释：

```go

// 格式化前
type T struct {
    name string // name of the object
    value int // its value
}

// 格式化后
type T struct {
    name    string // name of the object
    value   int    // its value
}
```

### 格式化细节

#### 缩进

- 使用制表符进行缩进，gofmt 默认输出制表符

- 只有在必要时才使用空格

#### 行长度

- Go 没有行长度限制

- 如果一行感觉太长，可以换行并使用额外的制表符缩进

#### 括号

- Go 比 C 和 Java 需要更少的括号：控制结构（if、for、switch）的语法中不需要括号

- 运算符优先级层次更短更清晰，例如：

    ```go
    
    x<<8 + y<<16
    ```

  这个表达式的含义与空格暗示的一致，不同于其他语言。

## 注释

### 注释风格

Go 提供 C 风格的 /* */ 块注释和 C++ 风格的 // 行注释。行注释是常态；块注释主要出现在包注释中，但在表达式内部或禁用大量代码时也很有用。

### 文档注释

出现在顶层声明之前且没有空行分隔的注释被认为是该声明的文档。这些 "文档注释" 是 Go 包或命令的主要文档。

## 命名

### 命名的重要性

在 Go 语言中，命名和其他语言一样重要。它们甚至有语义效果：一个名称在包外的可见性由其首字母是否大写决定。

### 包名

#### 包名约定

- 导入包后，包名成为访问其内容的访问器

- 包名应该简短、简洁、有描述性

- 按照约定，包名使用小写、单单词名称，不需要使用下划线或混合大小写

- 倾向于简洁，因为使用你的包的每个人都要输入这个名称

#### 包名与目录名

另一个约定是包名是其源目录的基本名称。例如，src/encoding/base64 中的包导入为 "encoding/base64"，但包名为 base64，而不是 encoding_base64 或 encodingBase64。

#### 导出名称的命名

包的导入者会使用包名来引用其内容，因此包中的导出名称可以利用这一点避免重复。例如：

- bufio 包中的缓冲读取器类型称为 Reader，而不是 BufReader，因为用户看到的是 bufio.Reader

- ring 包中创建 ring.Ring 新实例的函数被称为 New，而不是 NewRing，因为客户端看到的是 ring.New

### Getter 方法

Go 不自动支持 getter 和 setter。自己提供 getter 和 setter 并没有错，而且通常是合适的，但在 getter 名称中加入 Get 既不符合习惯也没有必要。如果你有一个名为 owner 的字段（小写，未导出），getter 方法应该被称为 Owner（大写，导出），而不是 GetOwner。

```go

owner := obj.Owner()
if owner != user {
    obj.SetOwner(user)
}
```

### 接口名

按照约定，单方法接口的名称由方法名加上 - er 后缀或类似的修改来构成施动名词：Reader、Writer、Formatter、CloseNotifier 等。

这些名称有很多，遵循它们和它们所代表的函数名称是有益的。Read、Write、Close、Flush、String 等有规范的签名和含义。为了避免混淆，不要给你的方法起这些名称，除非它具有相同的签名和含义。

### 混合大小写

最后，Go 中的约定是使用 MixedCaps 或 mixedCaps 而不是下划线来编写多单词名称。

## 分号

### 自动分号插入

和 C 一样，Go 的形式语法使用分号来终止语句，但与 C 不同的是，这些分号不出现在源代码中。相反，词法分析器在扫描时使用简单的规则自动插入分号，因此输入文本中大多不需要分号。

### 插入规则

如果换行符前的最后一个标记是标识符（包括 int 和 float64 等词）、基本字面量（如数字或字符串常量），或者以下标记之一：

```Plain Text

break continue fallthrough return ++ -- ) }
```

词法分析器总是在该标记后插入分号。这可以总结为："如果换行符出现在可能结束语句的标记之后，插入分号"。

### 控制结构的大括号位置

分号插入规则的一个结果是，不能将控制结构（if、for、switch 或 select）的开括号放在下一行。如果这样做，会在括号前插入分号，可能导致意外效果。

```go

// 正确写法
if i < f() {
    g()
}

// 错误写法
if i < f()  // wrong!
{           // wrong!
    g()
}
```

## 控制结构

### 概述

Go 的控制结构与 C 相关，但在重要方面有所不同。没有 do 或 while 循环，只有稍微泛化的 for；switch 更灵活；if 和 switch 接受可选的初始化语句，类似于 for；break 和 continue 语句接受可选标签来标识要中断或继续的内容；还有新的控制结构，包括类型开关和多路通信多路复用器 select。语法也略有不同：没有括号，并且主体必须始终用大括号括起来。

### If 语句

#### 基本形式

```go

if x > 0 {
    return y
}
```

#### 强制大括号

强制使用大括号鼓励将简单的 if 语句写在多行上。即使主体包含 return 或 break 等控制语句，这样做也是良好的风格。

#### 初始化语句

由于 if 和 switch 接受初始化语句，常见的用法是用它来设置局部变量。

```go

if err := file.Chmod(0664); err != nil {
    log.Print(err)
    return err
}
```

#### 省略 Else

在 Go 库中，你会发现当 if 语句不流入下一个语句时 —— 即主体以 break、continue、goto 或 return 结束 —— 不必要的 else 会被省略。

```go

f, err := os.Open(name)
if err != nil {
    return err
}
codeUsing(f)
```

这是一种常见情况，代码必须防范一系列错误条件。如果成功的控制流程沿着页面向下运行，逐个消除错误情况，代码读起来会很顺畅。由于错误情况往往以 return 语句结束，生成的代码不需要 else 语句。

```go

f, err := os.Open(name)
if err != nil {
    return err
}
d, err := f.Stat()
if err != nil {
    f.Close()
    return err
}
codeUsing(f, d)
```

### 重新声明和重新赋值

#### 短声明的特性

最后一个示例展示了:= 短声明形式的一个细节。调用 os.Open 的声明是：

```go

f, err := os.Open(name)
```

这个语句声明了两个变量 f 和 err。几行之后，调用 f.Stat 的语句是：

```go

d, err := f.Stat()
```

这看起来像是声明了 d 和 err。但注意 err 出现在两个语句中。这种重复是合法的：err 由第一个语句声明，但在第二个语句中只是被重新赋值。

#### 重新声明规则

在:= 声明中，变量 v 即使已经被声明也可能出现，条件是：

1. 这个声明与 v 的现有声明在同一作用域中（如果 v 已经在外部作用域中声明，该声明将创建一个新变量）

2. 初始化中的对应值可以赋值给 v

3. 声明中至少有一个其他变量是被创建的

这种不寻常的特性纯粹是实用主义的，使得在长的 if-else 链中使用单个 err 值变得容易。

### For 循环

#### 三种形式

Go 的 for 循环类似于但不同于 C 的 for 循环。它统一了 for 和 while，没有 do-while。有三种形式，其中只有一种使用分号。

```go

// 类似于C的for
for init; condition; post { }

// 类似于C的while
for condition { }

// 类似于C的for(;;)
for { }
```

#### 短声明

短声明使得在循环中声明索引变量变得容易。

```go

sum := 0
for i := 0; i < 10; i++ {
    sum += i
}
```

#### Range 子句

如果你遍历数组、切片、字符串、映射，或者从通道读取，可以使用 range 子句来管理循环。

```go

for key, value := range oldMap {
    newMap[key] = value
}
```

如果你只需要 range 中的第一个项（键或索引），可以省略第二个：

```go

for key := range m {
    if key.expired() {
        delete(m, key)
    }
}
```

如果你只需要 range 中的第二个项（值），可以使用空白标识符（下划线）来丢弃第一个：

```go

sum := 0
for _, value := range array {
    sum += value
}
```

#### 字符串 Range

对于字符串，range 会做更多工作，通过解析 UTF-8 来分解单个 Unicode 代码点。错误的编码会消耗一个字节并产生替换符文 U+FFFD。

```go

for pos, char := range "日本\x80語" { // \x80是非法的UTF-8编码
    fmt.Printf("character %#U starts at byte position %d\n", char, pos)
}
```

输出：

```Plain Text

character U+65E5 '日' starts at byte position 0
character U+672C '本' starts at byte position 3
character U+FFFD '�' starts at byte position 6
character U+8A9E '語' starts at byte position 7
```

#### 多变量循环

Go 没有逗号运算符，++ 和 -- 是语句而不是表达式。因此如果你想在 for 中运行多个变量，应该使用并行赋值。

```go

// 反转a
for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
    a[i], a[j] = a[j], a[i]
}
```

### Switch 语句

#### 更灵活的 Switch

Go 的 switch 比 C 的更通用。表达式不必是常量甚至整数，case 会从上到下求值直到找到匹配项，如果 switch 没有表达式，它会对 true 进行 switch。因此可以 —— 并且符合习惯 —— 将 if-else-if-else 链写成 switch。

```go

func unhex(c byte) byte {
    switch {
    case '0' <= c && c <= '9':
        return c - '0'
    case 'a' <= c && c <= 'f':
        return c - 'a' + 10
    case 'A' <= c && c <= 'F':
        return c - 'A' + 10
    }
    return 0
}
```

#### 无自动 Fallthrough

没有自动的 fallthrough，但 case 可以以逗号分隔的列表形式呈现。

```go

func shouldEscape(c byte) bool {
    switch c {
    case ' ', '?', '&', '=', '#', '+', '%':
        return true
    }
    return false
}
```

#### Break 语句

虽然 break 语句在 Go 中不像在其他 C 类语言中那样常见，但可以用来提前终止 switch。有时，需要跳出周围的循环而不是 switch，在 Go 中可以通过在循环上放置标签并 "break" 到该标签来实现。

```go

Loop:
    for n := 0; n < len(src); n += size {
        switch {
        case src[n] < sizeOne:
            if validateOnly {
                break
            }
            size = 1
            update(src[n])

        case src[n] < sizeTwo:
            if n+1 >= len(src) {
                err = errShortInput
                break Loop
            }
            if validateOnly {
                break
            }
            size = 2
            update(src[n] + src[n+1]<<shift)
        }
    }
```

### Type Switch

Switch 也可以用于发现接口变量的动态类型。这种类型 switch 使用类型断言的语法，在括号内使用 type 关键字。如果 switch 在表达式中声明了一个变量，该变量在每个子句中都会有相应的类型。在这种情况下重用名称也是符合习惯的，实际上是在每个 case 中声明一个具有相同名称但不同类型的新变量。

```go

var t interface{}
t = functionOfSomeType()
switch t := t.(type) {
default:
    fmt.Printf("unexpected type %T\n", t)     // %T打印t的任何类型
case bool:
    fmt.Printf("boolean %t\n", t)             // t的类型是bool
case int:
    fmt.Printf("integer %d\n", t)             // t的类型是int
case *bool:
    fmt.Printf("pointer to boolean %t\n", *t) // t的类型是*bool
case *int:
    fmt.Printf("pointer to integer %d\n", *t) // t的类型是*int
}
```

## 函数

### 多返回值

Go 的一个不寻常特性是函数和方法可以返回多个值。这种形式可以用来改进 C 程序中的一些笨拙用法：带内错误返回（如用 - 1 表示 EOF）和通过地址传递参数来修改参数。

在 C 中，写入错误通过负计数来表示，错误代码隐藏在 volatile 位置。在 Go 中，Write 可以返回计数和错误："是的，你写入了一些字节，但不是全部，因为设备已满"。

```go

func (file *File) Write(b []byte) (n int, err error)
```

类似的方法避免了需要传递返回值的指针来模拟引用参数。

```go

func nextInt(b []byte, i int) (int, int) {
    for ; i < len(b) && !isDigit(b[i]); i++ {
    }
    x := 0
    for ; i < len(b) && isDigit(b[i]); i++ {
        x = x*10 + int(b[i]) - '0'
    }
    return x, i
}
```

### 命名结果参数

Go 函数的返回或 "结果" 参数可以被命名并用作常规变量，就像传入的参数一样。命名后，它们在函数开始时被初始化为其类型的零值；如果函数执行没有参数的 return 语句，结果参数的当前值将被用作返回值。

这些名称不是强制性的，但它们可以使代码更短更清晰：它们是文档。如果我们给 nextInt 的结果命名，就可以清楚地知道返回的 int 是哪个。

```go

func nextInt(b []byte, pos int) (value, nextPos int) {
```

由于命名结果被初始化并与无参数的 return 绑定，它们可以简化并澄清代码。

```go

func ReadFull(r Reader, buf []byte) (n int, err error) {
    for len(buf) > 0 && err == nil {
        var nr int
        nr, err = r.Read(buf)
        n += nr
        buf = buf[nr:]
    }
    return
}
```

### Defer 语句

Go 的 defer 语句安排一个函数调用（被延迟的函数）在执行 defer 的函数返回之前立即运行。这是一种不寻常但有效的方式来处理必须释放资源的情况，无论函数通过哪个路径返回。典型的例子是解锁互斥锁或关闭文件。

```go

// Contents以字符串形式返回文件内容
func Contents(filename string) (string, error) {
    f, err := os.Open(filename)
    if err != nil {
        return "", err
    }
    defer f.Close()  // f.Close将在我们完成时运行

    var result []byte
    buf := make([]byte, 100)
    for {
        n, err := f.Read(buf[0:])
        result = append(result, buf[0:n]...) // append将在后面讨论
        if err != nil {
            if err == io.EOF {
                break
            }
            return "", err  // 如果我们在这里返回，f将被关闭
        }
    }
    return string(result), nil // 如果我们在这里返回，f将被关闭
}
```

#### Defer 的优势

延迟调用 Close 这样的函数有两个优势。首先，它保证你永远不会忘记关闭文件，如果你稍后编辑函数添加新的返回路径，这是一个容易犯的错误。其次，这意味着关闭操作位于打开操作附近，比放在函数末尾更清晰。

#### Defer 参数计算

被延迟函数的参数（如果函数是方法，则包括接收器）在 defer 执行时求值，而不是在调用执行时求值。除了避免担心变量在函数执行过程中改变值之外，这意味着单个延迟调用点可以延迟多个函数执行。

```go

for i := 0; i < 5; i++ {
    defer fmt.Printf("%d ", i)
}
```

延迟函数以 LIFO（后进先出）顺序执行，因此这段代码将在函数返回时打印 4 3 2 1 0。

## 数据

### 使用 new 分配

Go 有两个分配原语，内置函数 new 和 make。它们做不同的事情，适用于不同的类型，这可能会令人困惑，但规则很简单。先说说 new。它是一个内置函数，用于分配内存，但与其他语言中的同名函数不同，它不初始化内存，只将其置零。也就是说，new (T) 为类型 T 的新项分配置零的存储，并返回其地址，类型为 * T。用 Go 术语来说，它返回指向类型 T 新分配的零值的指针。

由于 new 返回的内存是置零的，在设计数据结构时，最好安排每种类型的零值无需进一步初始化即可使用。这意味着数据结构的用户可以用 new 创建一个并立即开始工作。例如，bytes.Buffer 的文档指出 "Buffer 的零值是一个空缓冲区，可以直接使用"。类似地，sync.Mutex 没有显式的构造函数或 Init 方法。相反，sync.Mutex 的零值被定义为未锁定的互斥锁。

零值有用的特性是传递性的。考虑这个类型声明：

```go

type SyncedBuffer struct {
    lock    sync.Mutex
    buffer  bytes.Buffer
}
```

SyncedBuffer 类型的值在分配或声明后也可以立即使用。

```go

p := new(SyncedBuffer)  // 类型*SyncedBuffer
var v SyncedBuffer      // 类型 SyncedBuffer
```

### 构造函数和复合字面量

有时零值不够好，需要一个初始化构造函数，就像这个来自 os 包的例子：

```go

func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := new(File)
    f.fd = fd
    f.name = name
    f.dirinfo = nil
    f.nepipe = 0
    return f
}
```

这里有很多样板代码。我们可以使用复合字面量来简化它，复合字面量是每次求值时创建新实例的表达式。

```go

func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := File{fd, name, nil, 0}
    return &f
}
```

注意，与 C 不同，返回局部变量的地址是完全可以的；与变量相关的存储在函数返回后仍然存在。实际上，取复合字面量的地址每次求值都会分配一个新实例，所以我们可以将最后两行合并：

```go

return &File{fd, name, nil, 0}
```

复合字面量的字段按顺序排列，并且必须全部存在。但是，通过将元素显式标记为 field:value 对，初始化器可以以任何顺序出现，缺失的元素将保留其各自的零值。因此我们可以说：

```go

return &File{fd: fd, name: name}
```

作为极限情况，如果复合字面量不包含任何字段，它将创建该类型的零值。表达式 new (File) 和 & File {} 是等价的。

复合字面量也可以为数组、切片和映射创建，字段标签适当地是索引或映射键。

```go

a := [...]string   {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
s := []string      {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
m := map[int]string{Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
```

### 使用 make 分配

回到分配。内置函数 make (T, args) 的目的与 new (T) 不同。它只创建切片、映射和通道，并返回类型 T 的初始化（非置零）值（而不是 * T）。区分的原因是这三种类型在底层代表对必须初始化的数据结构的引用。例如，切片是一个三项描述符，包含指向数据（数组内部）的指针、长度和容量，在这些项初始化之前，切片是 nil。对于切片、映射和通道，make 初始化内部数据结构并准备好值供使用。

例如：

```go

make([]int, 10, 100)
```

分配一个包含 100 个 int 的数组，然后创建一个切片结构，长度为 10，容量为 100，指向数组的前 10 个元素。（创建切片时，可以省略容量；有关更多信息，请参见切片部分。）相比之下，new ([] int) 返回指向新分配的、置零的切片结构的指针，即指向 nil 切片值的指针。

这些例子说明了 new 和 make 之间的区别：

```go

var p *[]int = new([]int)       // 分配切片结构；*p == nil；很少有用
var v  []int = make([]int, 100) // 切片v现在引用一个包含100个int的新数组

// 不必要的复杂：
var p *[]int = new([]int)
*p = make([]int, 100, 100)

// 符合习惯的写法：
v := make([]int, 100)
```

记住，make 仅适用于映射、切片和通道，不返回指针。要获得显式指针，请使用 new 分配或显式获取变量的地址。

### 数组

数组在规划内存的详细布局时很有用，有时可以帮助避免分配，但它们主要是切片的构建块。

Go 中的数组与 C 中的数组有很大不同：

1. 数组是值类型。将一个数组赋值给另一个数组会复制所有元素。

2. 特别是，如果你将数组传递给函数，它将接收数组的副本，而不是指针。

3. 数组的大小是其类型的一部分。类型 [10] int 和 [20] int 是不同的。

值属性可能有用，但也可能代价高昂；如果你想要类似 C 的行为和效率，可以传递数组的指针。

```go

func Sum(a *[3]float64) (sum float64) {
    for _, v := range *a {
        sum += v
    }
    return
}

array := [...]float64{7.0, 8.5, 9.1}
x := Sum(&array)  // 注意显式的取地址运算符
```

但即使这种风格也不符合 Go 的习惯用法。应该使用切片。

### 切片

切片包装数组，为序列数据提供更通用、强大和方便的接口。除了具有显式维度的项（如变换矩阵），Go 中的大多数数组编程都是使用切片而不是简单数组完成的。

切片持有对底层数组的引用，如果你将一个切片赋值给另一个切片，两者都引用同一个数组。如果函数接受切片参数，它对切片元素所做的更改将对调用者可见，类似于传递指向底层数组的指针。

```go

func (f *File) Read(buf []byte) (n int, err error)
```

要读取较大缓冲区 buf 的前 32 个字节，可以对缓冲区进行切片：

```go

n, err := f.Read(buf[0:32])
```

#### 切片的长度和容量

切片的长度可以更改，只要它仍然适合底层数组的限制；只需将其赋值给自身的切片。切片的容量可以通过内置函数 cap 获取，它报告切片可以假设的最大长度。

```go

func Append(slice, data []byte) []byte {
    l := len(slice)
    if l + len(data) > cap(slice) {  // 重新分配
        // 分配所需空间的两倍，以备将来增长
        newSlice := make([]byte, (l+len(data))*2)
        // copy函数是预声明的，适用于任何切片类型
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0:l+len(data)]
    copy(slice[l:], data)
    return slice
}
```

我们必须返回切片，因为虽然 Append 可以修改 slice 的元素，但切片本身（持有指针、长度和容量的运行时数据结构）是按值传递的。

#### 内置 append 函数

内置的 append 函数捕获了向切片追加数据的想法。它的签名是：

```go

func append(slice []T, elements ...T) []T
```

append 将元素追加到切片的末尾并返回结果。结果需要返回，因为底层数组可能会改变。

```go

x := []int{1,2,3}
x = append(x, 4, 5, 6)
fmt.Println(x) // 打印 [1 2 3 4 5 6]
```

如果你想将一个切片追加到另一个切片，可以使用...：

```go

x := []int{1,2,3}
y := []int{4,5,6}
x = append(x, y...)
fmt.Println(x) // 打印 [1 2 3 4 5 6]
```

### 二维切片

Go 的数组和切片是一维的。要创建等效的二维数组或切片，需要定义数组的数组或切片的切片：

```go

type Transform [3][3]float64  // 一个3x3数组，实际上是数组的数组
type LinesOfText [][]byte     // 一个字节切片的切片
```

由于切片是可变长度的，每个内部切片可以有不同的长度。这在 LinesOfText 示例中很常见：每行有独立的长度。

```go

text := LinesOfText{
    []byte("Now is the time"),
    []byte("for all good gophers"),
    []byte("to bring some fun to the party."),
}
```

### 映射

映射是一种方便且强大的内置数据结构，它将一种类型的值（键）与另一种类型的值（元素或值）相关联。键可以是任何定义了相等运算符的类型，例如整数、浮点数和复数、字符串、指针、接口（只要动态类型支持相等）、结构体和数组。切片不能用作映射键，因为它们没有定义相等性。

像切片一样，映射持有对底层数据结构的引用。如果你将映射传递给更改其内容的函数，更改将对调用者可见。

```go

var timeZone = map[string]int{
    "UTC":  0*60*60,
    "EST": -5*60*60,
    "CST": -6*60*60,
    "MST": -7*60*60,
    "PST": -8*60*60,
}
```

#### 访问映射值

尝试使用映射中不存在的键获取映射值将返回映射中条目类型的零值。例如，如果映射包含整数，查找不存在的键将返回 0。

```go

attended := map[string]bool{
    "Ann": true,
    "Joe": true,
    // ...
}

if attended[person] { // 如果person不在映射中，将为false
    fmt.Println(person, "was at the meeting")
}
```

#### Comma Ok 惯用法

有时你需要区分缺失的条目和零值。你可以使用多赋值的形式来区分：

```go

var seconds int
var ok bool
seconds, ok = timeZone[tz]
```

这个惯用法被称为 "comma ok"。如果 tz 存在，seconds 将被适当设置，ok 将为 true；否则，seconds 将被设置为零，ok 将为 false。

```go

func offset(tz string) int {
    if seconds, ok := timeZone[tz]; ok {
        return seconds
    }
    log.Println("unknown time zone:", tz)
    return 0
}
```

如果你只想测试映射中是否存在某个键而不关心实际值，可以使用空白标识符 (_)：

```go

_, present := timeZone[tz]
```

#### 删除映射条目

要删除映射条目，请使用内置函数 delete，其参数是映射和要删除的键。即使键已经不在映射中，这样做也是安全的。

```go

delete(timeZone, "PDT")  // 现在使用标准时间
```

### 打印

Go 中的格式化打印使用类似于 C 的 printf 家族的风格，但更丰富、更通用。函数位于 fmt 包中，名称大写：fmt.Printf、fmt.Fprintf、fmt.Sprintf 等。字符串函数（如 Sprintf）返回字符串而不是填充提供的缓冲区。

#### 默认格式

你不需要提供格式字符串。对于每个 Printf、Fprintf 和 Sprintf，都有另一对函数，例如 Print 和 Println。这些函数不接受格式字符串，而是为每个参数生成默认格式。Println 版本还在参数之间插入空格，并在输出末尾追加换行符，而 Print 版本仅在两边的操作数都不是字符串时才添加空格。

```go

fmt.Printf("Hello %d\n", 23)
fmt.Fprint(os.Stdout, "Hello ", 23, "\n")
fmt.Println("Hello", 23)
fmt.Println(fmt.Sprint("Hello ", 23))
```

#### 格式说明符

与 C 不同，数值格式（如 % d）不接受有符号或大小的标志；相反，打印例程使用参数的类型来决定这些属性。

```go

var x uint64 = 1<<64 - 1
fmt.Printf("%d %x; %d %x\n", x, x, int64(x), int64(x))
```

输出：

```Plain Text

18446744073709551615 ffffffffffffffff; -1 -1
```

#### 通用格式 % v

如果你只想要默认转换，例如整数的十进制，可以使用通用格式 % v（代表 "value"）；结果与 Print 和 Println 产生的完全相同。此外，该格式可以打印任何值，甚至数组、切片、结构体和映射。

```go

fmt.Printf("%v\n", timeZone)  // 或者直接fmt.Println(timeZone)
```

输出：

```Plain Text

map[CST:-21600 EST:-18000 MST:-25200 PST:-28800 UTC:0]
```

#### 结构体格式

打印结构体时，修改后的格式 %+v 会用字段名注释结构体的字段，而对于任何值，备用格式 %#v 会以完整的 Go 语法打印值。

```go

type T struct {
    a int
    b float64
    c string
}
t := &T{ 7, -2.35, "abc\tdef" }
fmt.Printf("%v\n", t)
fmt.Printf("%+v\n", t)
fmt.Printf("%#v\n", t)
```

输出：

```Plain Text

&{7 -2.35 abc   def}
&{a:7 b:-2.35 c:abc     def}
&main.T{a:7, b:-2.35, c:"abc\tdef"}
```

#### 自定义 String 方法

如果你想控制自定义类型的默认格式，只需要为该类型定义一个签名为 String () string 的方法。

```go

func (t *T) String() string {
    return fmt.Sprintf("%d/%g/%q", t.a, t.b, t.c)
}
fmt.Printf("%v\n", t)
```

输出：

```Plain Text

7/-2.35/"abc\tdef"
```

注意：在实现 String 方法时要小心避免无限递归。例如：

```go

type MyString string

func (m MyString) String() string {
    return fmt.Sprintf("MyString=%s", m) // 错误：将无限递归
}
```

解决方法是将参数转换为基本字符串类型：

```go

type MyString string
func (m MyString) String() string {
    return fmt.Sprintf("MyString=%s", string(m)) // 正确：注意转换
}
```

## 初始化

### 常量

Go 中的常量就是常量。它们在编译时创建，即使在函数中定义为局部变量，也只能是数字、字符（符文）、字符串或布尔值。由于编译时的限制，定义它们的表达式必须是常量表达式，可以由编译器求值。例如，1<<3 是常量表达式，而 math.Sin (math.Pi/4) 不是，因为对 math.Sin 的函数调用需要在运行时发生。

#### Iota 枚举

在 Go 中，枚举常量使用 iota 枚举器创建。由于 iota 可以是表达式的一部分，并且表达式可以隐式重复，因此很容易构建复杂的值集。

```go

type ByteSize float64

const (
    _           = iota // 通过赋值给空白标识符忽略第一个值
    KB ByteSize = 1 << (10 * iota)
    MB
    GB
    TB
    PB
    EB
    ZB
    YB
)
```

#### 常量的 String 方法

为任何用户定义类型附加 String 方法的能力使得任意值可以自动格式化以便打印。

```go

func (b ByteSize) String() string {
    switch {
    case b >= YB:
        return fmt.Sprintf("%.2fYB", b/YB)
    case b >= ZB:
        return fmt.Sprintf("%.2fZB", b/ZB)
    case b >= EB:
        return fmt.Sprintf("%.2fEB", b/EB)
    case b >= PB:
        return fmt.Sprintf("%.2fPB", b/PB)
    case b >= TB:
        return fmt.Sprintf("%.2fTB", b/TB)
    case b >= GB:
        return fmt.Sprintf("%.2fGB", b/GB)
    case b >= MB:
        return fmt.Sprintf("%.2fMB", b/MB)
    case b >= KB:
        return fmt.Sprintf("%.2fKB", b/KB)
    }
    return fmt.Sprintf("%.2fB", b)
}
```

### 变量

变量可以像常量一样初始化，但初始化器可以是在运行时计算的通用表达式。

```go

var (
    home   = os.Getenv("HOME")
    user   = os.Getenv("USER")
    gopath = os.Getenv("GOPATH")
)
```

### Init 函数

最后，每个源文件可以定义自己的无参数 init 函数来设置所需的任何状态。（实际上每个文件可以有多个 init 函数。）最后意味着最后：init 在包中的所有变量声明都计算了它们的初始化器之后调用，而这些初始化器只有在所有导入的包都初始化之后才会计算。

除了不能表示为声明的初始化之外，init 函数的常见用途是在实际执行开始之前验证或修复程序状态的正确性。

```go

func init() {
    if user == "" {
        log.Fatal("$USER not set")
    }
    if home == "" {
        home = "/home/" + user
    }
    if gopath == "" {
        gopath = home + "/go"
    }
    // gopath可以通过命令行上的--gopath标志覆盖
    flag.StringVar(&gopath, "gopath", gopath, "override default GOPATH")
}
```

## 方法

### 指针与值接收器

正如我们在 ByteSize 中看到的，方法可以为任何命名类型定义（指针或接口除外）；接收器不必是结构体。

在上面关于切片的讨论中，我们编写了一个 Append 函数。我们可以将其定义为切片上的方法。为此，我们首先声明一个命名类型，我们可以将方法绑定到该类型，然后使方法的接收器成为该类型的值。

```go

type ByteSlice []byte

func (slice ByteSlice) Append(data []byte) []byte {
    // 主体与上面定义的Append函数完全相同
}
```

这仍然要求方法返回更新后的切片。我们可以通过重新定义方法以接受指向 ByteSlice 的指针作为其接收器来消除这种笨拙，这样方法就可以覆盖调用者的切片。

```go

func (p *ByteSlice) Append(data []byte) {
    slice := *p
    // 主体与上面相同，没有返回
    *p = slice
}
```

实际上，我们可以做得更好。如果我们修改函数使其看起来像标准的 Write 方法：

```go

func (p *ByteSlice) Write(data []byte) (n int, err error) {
    slice := *p
    // 再次与上面相同
    *p = slice
    return len(data), nil
}
```

然后类型 * ByteSlice 满足标准接口 io.Writer，这很方便。例如，我们可以向其中打印：

```go

var b ByteSlice
fmt.Fprintf(&b, "This hour has %d days\n", 7)
```

我们传递 ByteSlice 的地址，因为只有 * ByteSlice 满足 io.Writer。关于接收器的指针与值的规则是：值方法可以在指针和值上调用，但指针方法只能在指针上调用。

这个规则产生的原因是指针方法可以修改接收器；在值上调用它们会导致方法接收值的副本，因此任何修改都会被丢弃。因此语言不允许这个错误。不过，有一个方便的例外。当值是可寻址的时，语言会通过自动插入地址运算符来处理在值上调用指针方法的常见情况。在我们的例子中，变量 b 是可寻址的，所以我们可以用 b.Write 调用它的 Write 方法。编译器会将其重写为 (&b).Write。

顺便说一下，在字节切片上使用 Write 的想法是 bytes.Buffer 实现的核心。

### 接口与其他类型

#### 接口

Go 中的接口提供了一种指定对象行为的方式：如果某物能做这个，那么它可以在这里使用。我们已经看到了几个简单的例子；自定义打印机可以通过 String 方法实现，而 Fprintf 可以向任何具有 Write 方法的对象生成输出。只有一两个方法的接口在 Go 代码中很常见，通常以方法的名称命名，如 io.Writer 表示实现 Write 的东西。

一个类型可以实现多个接口。例如，一个集合如果实现了 sort.Interface（包含 Len ()、Less (i, j int) bool 和 Swap (i, j int)），就可以被 sort 包中的例程排序，它也可以有一个自定义格式化器。

```go

type Sequence []int

// sort.Interface要求的方法
func (s Sequence) Len() int {
    return len(s)
}
func (s Sequence) Less(i, j int) bool {
    return s[i] < s[j]
}
func (s Sequence) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}

// Copy返回Sequence的副本
func (s Sequence) Copy() Sequence {
    copy := make(Sequence, 0, len(s))
    return append(copy, s...)
}

// 打印方法 - 打印前先排序元素
func (s Sequence) String() string {
    s = s.Copy() // 制作副本；不覆盖参数
    sort.Sort(s)
    str := "["
    for i, elem := range s { // 循环是O(N²)；将在下一个例子中修复
        if i > 0 {
            str += " "
        }
        str += fmt.Sprint(elem)
    }
    return str + "]"
}
```

#### 类型转换

Sequence 的 String 方法正在重复 Sprint 已经为切片做的工作。我们可以通过在调用 Sprint 之前将 Sequence 转换为普通的 [] int 来分享努力（并加快速度）。

```go

func (s Sequence) String() string {
    s = s.Copy()
    sort.Sort(s)
    return fmt.Sprint([]int(s))
}
```

这是调用 Sprintf 安全地从 String 方法转换技术的另一个例子。因为这两个类型（Sequence 和 [] int）如果忽略类型名称是相同的，所以在它们之间转换是合法的。转换不会创建新值，它只是临时表现得好像现有值有了新类型。

在 Go 程序中，转换表达式的类型以访问不同的方法集是一种习惯用法。例如，我们可以使用现有的类型 sort.IntSlice 将整个例子简化为：

```go

type Sequence []int

// 打印方法 - 打印前先排序元素
func (s Sequence) String() string {
    s = s.Copy()
    sort.IntSlice(s).Sort()
    return fmt.Sprint([]int(s))
}
```

现在，不是让 Sequence 实现多个接口（排序和打印），我们使用数据项可以转换为多个类型（Sequence、sort.IntSlice 和 [] int）的能力，每个类型都做一部分工作。这在实践中不太常见，但可能很有效。

#### 接口转换和类型断言

类型开关是一种转换形式：它们接受一个接口，并在某种意义上，对于开关中的每个 case，将其转换为该 case 的类型。

```go

type Stringer interface {
    String() string
}

var value interface{} // 调用者提供的值
switch str := value.(type) {
case string:
    return str
case Stringer:
    return str.String()
}
```

如果你只关心一种类型，可以使用类型断言：

```go

str := value.(string)
```

但如果值不包含字符串，程序会崩溃。为了防范这种情况，可以使用 "comma ok" 惯用语安全地测试值是否为字符串：

```go

str, ok := value.(string)
if ok {
    fmt.Printf("string value is: %q\n", str)
} else {
    fmt.Printf("value is not a string\n")
}
```

#### 通用性

如果一个类型仅为实现接口而存在，并且永远不会有超出该接口的导出方法，则无需导出该类型本身。仅导出接口可以清楚地表明该值除了接口中描述的内容外没有其他有趣的行为。这也避免了在每个公共方法的实例上重复文档。

在这种情况下，构造函数应该返回接口值而不是实现类型。例如，在哈希库中，crc32.NewIEEE 和 adler32.New 都返回接口类型 hash.Hash32。在 Go 程序中用 CRC-32 算法替换 Adler-32 只需要改变构造函数调用；代码的其余部分不受算法变化的影响。

### 接口与方法

由于几乎任何东西都可以附加方法，几乎任何东西都可以满足接口。一个说明性的例子是在 http 包中，它定义了 Handler 接口。任何实现 Handler 的对象都可以处理 HTTP 请求。

```go

type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

ResponseWriter 本身是一个接口，提供访问返回响应给客户端所需的方法。这些方法包括标准的 Write 方法，因此 http.ResponseWriter 可以在任何使用 io.Writer 的地方使用。Request 是一个结构体，包含客户端请求的解析表示。

#### 示例：计数器服务器

```go

// 简单的计数器服务器
type Counter struct {
    n int
}

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ctr.n++
    fmt.Fprintf(w, "counter = %d\n", ctr.n)
}
```

我们可以将这样的服务器附加到 URL 树的一个节点上：

```go

import "net/http"
...
ctr := new(Counter)
http.Handle("/counter", ctr)
```

#### 更简单的计数器服务器

为什么要让 Counter 成为结构体？只需要一个整数就够了。（接收器需要是指针，这样增量对调用者可见。）

```go

// 更简单的计数器服务器
type Counter int

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    *ctr++
    fmt.Fprintf(w, "counter = %d\n", *ctr)
}
```

#### 通道作为服务器

如果你希望程序有一些内部状态需要在页面被访问时得到通知，可以将通道绑定到网页。

```go

// 每次访问时发送通知的通道
// （可能希望通道是带缓冲的）
type Chan chan *http.Request

func (ch Chan) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ch <- req
    fmt.Fprint(w, "notification sent")
}
```

#### 函数作为服务器

最后，假设我们想在 /args 上显示调用服务器二进制文件时使用的参数。编写一个打印参数的函数很容易。

```go

func ArgServer() {
    fmt.Println(os.Args)
}
```

我们如何将其变成 HTTP 服务器？我们可以让 ArgServer 成为某个我们忽略其值的类型的方法，但有更简洁的方法。由于我们可以为除指针和接口之外的任何类型定义方法，我们可以为函数编写方法。http 包包含这段代码：

```go

// HandlerFunc类型是一个适配器，允许将普通函数用作HTTP处理器。
// 如果f是具有适当签名的函数，HandlerFunc(f)是一个调用f的Handler对象。
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP调用f(w, req)。
func (f HandlerFunc) ServeHTTP(w ResponseWriter, req *Request) {
    f(w, req)
}
```

HandlerFunc 是一个具有 ServeHTTP 方法的类型，因此该类型的值可以处理 HTTP 请求。

```go

// 参数服务器
func ArgServer(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintln(w, os.Args)
}
```

ArgServer 现在具有与 HandlerFunc 相同的签名，因此可以转换为该类型以访问其方法。

```go

http.Handle("/args", http.HandlerFunc(ArgServer))
```

当有人访问 /args 页面时，安装在该页面的处理器的值为 ArgServer，类型为 HandlerFunc。HTTP 服务器将调用该类型的 ServeHTTP 方法，以 ArgServer 作为接收器，这将反过来调用 ArgServer（通过 HandlerFunc.ServeHTTP 内部的 f (w, req) 调用）。然后参数将被显示。

在本节中，我们用结构体、整数、通道和函数制作了 HTTP 服务器，这一切都是因为接口只是方法的集合，可以为（几乎）任何类型定义。

## 空白标识符

### 概述

我们已经在 for range 循环和映射的上下文中提到了空白标识符。空白标识符可以被赋值或声明为任何类型的任何值，值会被无害地丢弃。这有点像写入 Unix 的 /dev/null 文件：它表示一个只写值，用作需要变量但实际值无关紧要的占位符。它还有其他用途。

### 多赋值中的空白标识符

在 for range 循环中使用空白标识符是多赋值的一种特殊情况。

如果赋值需要左边有多个值，但其中一个值不会被程序使用，左边的空白标识符避免了创建哑变量的需要，并明确表示该值将被丢弃。例如，当调用返回值和错误的函数，但只有错误重要时，使用空白标识符丢弃无关的值。

```go

if _, err := os.Stat(path); os.IsNotExist(err) {
    fmt.Printf("%s does not exist\n", path)
}
```

偶尔你会看到为了忽略错误而丢弃错误值的代码；这是糟糕的做法。始终检查错误返回；它们被提供是有原因的。

```go

// 糟糕！如果path不存在，这段代码会崩溃。
fi, _ := os.Stat(path)
if fi.IsDir() {
    fmt.Printf("%s is a directory\n", path)
}
```

### 未使用的导入和变量

导入包或声明变量而不使用它是错误的。未使用的导入会使程序膨胀并减慢编译速度，而已初始化但未使用的变量至少是浪费计算，可能表明存在更大的错误。然而，当程序处于活跃开发中时，未使用的导入和变量经常出现，删除它们只是为了让编译继续，之后又需要它们，这可能很烦人。空白标识符提供了一种解决方法。

```go

package main

import (
    "fmt"
    "io"
    "log"
    "os"
)

var _ = fmt.Printf // 用于调试；完成后删除
var _ io.Reader    // 用于调试；完成后删除

func main() {
    fd, err := os.Open("test.go")
    if err != nil {
        log.Fatal(err)
    }
    // TODO: 使用fd
    _ = fd
}
```

按照约定，用于消除导入错误的全局声明应该紧跟在导入之后，并加上注释，以便于查找，并作为以后清理的提醒。

### 为副作用导入

像前面例子中的 fmt 或 io 这样的未使用导入最终应该被使用或删除：空白赋值将代码标识为进行中的工作。但有时只导入包的副作用而不进行任何显式使用是有用的。例如，net/http/pprof 包在其 init 函数中注册了提供调试信息的 HTTP 处理器。它有一个导出的 API，但大多数客户端只需要处理器注册，并通过网页访问数据。要仅为副作用导入包，将包重命名为空白标识符：

```go

import _ "net/http/pprof"
```

这种导入形式明确表示包是为其副作用导入的，因为在这个文件中，它没有名称。（如果它有名称，而我们没有使用该名称，编译器会拒绝该程序。）

### 接口检查

正如我们在上面关于接口的讨论中看到的，类型不需要显式声明它实现了接口。相反，类型只要实现了接口的方法就实现了该接口。实际上，大多数接口转换是静态的，因此在编译时检查。例如，将*os.File 传递给期望 io.Reader 的函数将无法编译，除非*os.File 实现了 io.Reader 接口。

有些接口检查确实在运行时进行。一个实例是在 encoding/json 包中，它定义了 Marshaler 接口。当 JSON 编码器接收到实现该接口的值时，编码器会调用该值的编组方法将其转换为 JSON，而不是进行标准转换。编码器使用类型断言在运行时检查此属性：

```go

m, ok := val.(json.Marshaler)
```

如果只需要询问类型是否实现了接口，而不实际使用接口本身，可能作为错误检查的一部分，可以使用空白标识符忽略类型断言的值：

```go

if _, ok := val.(json.Marshaler); ok {
    fmt.Printf("value %v of type %T implements json.Marshaler\n", val, val)
}
```

这种情况出现的一个地方是在实现类型的包内需要保证它实际上满足接口。如果一个类型 —— 例如 json.RawMessage—— 需要自定义 JSON 表示，它应该实现 json.Marshaler，但没有静态转换会导致编译器自动验证这一点。如果类型无意中未能满足接口，JSON 编码器仍然可以工作，但不会使用自定义实现。为了保证实现正确，可以在包中使用空白标识符进行全局声明：

```go

var _ json.Marshaler = (*RawMessage)(nil)
```

在这个声明中，涉及将*RawMessage 转换为 Marshaler 的赋值要求*RawMessage 实现 Marshaler，并且该属性将在编译时检查。如果 json.Marshaler 接口改变，这个包将不再编译，我们会注意到它需要更新。

空白标识符在这个构造中的出现表明声明仅用于类型检查，而不是创建变量。不过，不要对每个满足接口的类型都这样做。按照约定，这样的声明仅在代码中没有静态转换的情况下使用，这是罕见的情况。

## 嵌入

### 概述

Go 不提供典型的、类型驱动的子类化概念，但它确实具有通过在结构体或接口中嵌入类型来 "借用" 实现部分的能力。

### 接口嵌入

接口嵌入非常简单。我们之前提到了 io.Reader 和 io.Writer 接口：

```go

type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

io 包还导出了其他几个指定可以实现多个此类方法的对象的接口。例如，有 io.ReadWriter，一个包含 Read 和 Write 的接口。我们可以通过显式列出两个方法来指定 io.ReadWriter，但嵌入两个接口来形成新接口更容易、更有表现力：

```go

// ReadWriter是组合了Reader和Writer接口的接口
type ReadWriter interface {
    Reader
    Writer
}
```

这看起来就像它所说的：ReadWriter 可以做 Reader 和 Writer 能做的事情；它是嵌入接口的并集。只有接口可以嵌入在接口中。

### 结构体嵌入

同样的基本思想适用于结构体，但具有更深远的影响。bufio 包有两个结构体类型 bufio.Reader 和 bufio.Writer，它们当然都实现了来自 io 包的类似接口。bufio 还实现了一个缓冲的读写器，它通过使用嵌入将读取器和写入器组合到一个结构体中：它在结构体中列出类型但不给它们字段名。

```go

// ReadWriter存储指向Reader和Writer的指针
// 它实现了io.ReadWriter
type ReadWriter struct {
    *Reader  // *bufio.Reader
    *Writer  // *bufio.Writer
}
```

嵌入的元素是指向结构体的指针，当然必须初始化以指向有效的结构体才能使用。

通过直接嵌入结构体，我们避免了这种簿记。嵌入类型的方法会自动成为外部类型的方法，这意味着 bufio.ReadWriter 不仅具有 bufio.Reader 和 bufio.Writer 的方法，还满足所有三个接口：io.Reader、io.Writer 和 io.ReadWriter。

#### 嵌入与子类化的区别

嵌入与子类化有一个重要的区别。当我们嵌入一个类型时，该类型的方法成为外部类型的方法，但当它们被调用时，方法的接收器是内部类型，而不是外部类型。在我们的例子中，当调用 bufio.ReadWriter 的 Read 方法时，它与上面写出的转发方法具有完全相同的效果；接收器是 ReadWriter 的 reader 字段，而不是 ReadWriter 本身。

#### 嵌入作为便利

嵌入也可以是一种简单的便利。这个例子展示了一个嵌入字段和一个普通的命名字段：

```go

type Job struct {
    Command string
    *log.Logger
}
```

Job 类型现在具有 * log.Logger 的 Print、Printf、Println 和其他方法。当然，我们可以给 Logger 一个字段名，但没有必要这样做。现在，一旦初始化，我们可以记录到 Job：

```go

job.Println("starting now...")
```

Logger 是 Job 结构体的常规字段，所以我们可以在 Job 的构造函数中以通常的方式初始化它：

```go

func NewJob(command string, logger *log.Logger) *Job {
    return &Job{command, logger}
}
```

或者使用复合字面量：

```go

job := &Job{command, log.New(os.Stderr, "Job: ", log.Ldate)}
```

如果我们需要直接引用嵌入字段，字段的类型名（忽略包限定符）充当字段名。例如，如果我们需要访问 Job 变量 job 的 * log.Logger，我们会写 job.Logger，这在我们想要细化 Logger 的方法时很有用：

```go

func (job *Job) Printf(format string, args ...interface{}) {
    job.Logger.Printf("%q: %s", job.Command, fmt.Sprintf(format, args...))
}
```

#### 名称冲突

嵌入类型引入了名称冲突的问题，但解决它们的规则很简单。首先，字段或方法 X 隐藏任何在类型更深嵌套部分中的其他 X 项。如果 log.Logger 包含名为 Command 的字段或方法，Job 的 Command 字段将优先于它。

其次，如果相同的名称出现在相同的嵌套级别，通常是错误的；如果 Job 结构体包含另一个名为 Logger 的字段或方法，嵌入 log.Logger 是错误的。但是，如果重复的名称在程序中除了类型定义之外从未被提及，则是可以的。这个限定条件提供了一些保护，防止对从外部嵌入的类型所做的更改；如果添加的字段与另一个子类型中的另一个字段冲突，但两个字段都从未被使用，则没有问题。

## 并发

### 通过通信共享内存

并发编程在许多环境中变得困难，因为实现对共享变量的正确访问需要微妙的处理。Go 鼓励一种不同的方法，其中共享值通过通道传递，实际上，永远不会被单独的执行线程主动共享。任何时候只有一个 goroutine 可以访问该值。数据竞争在设计上是不可能发生的。为了鼓励这种思维方式，我们将其简化为一个口号：

> 不要通过共享内存进行通信；相反，通过通信来共享内存。
>
>

这种方法可能会被过度使用。例如，引用计数最好通过在整数变量周围放置互斥锁来完成。但作为一种高级方法，使用通道来控制访问使编写清晰、正确的程序变得更容易。

### Goroutines

它们被称为 goroutines，因为现有的术语 —— 线程、协程、进程等等 —— 传达了不准确的含义。goroutine 有一个简单的模型：它是一个与其他 goroutine 在同一地址空间中并发执行的函数。它是轻量级的，成本只比分配栈空间多一点。栈开始很小，所以它们很便宜，并且根据需要通过分配（和释放）堆存储来增长。

Goroutines 被多路复用到多个 OS 线程上，因此如果一个 goroutine 阻塞，例如在等待 I/O 时，其他 goroutine 继续运行。它们的设计隐藏了线程创建和管理的许多复杂性。

在函数或方法调用前加上 go 关键字可以在新的 goroutine 中运行该调用。调用完成后，goroutine 会静默退出。（效果类似于 Unix shell 中运行命令的 & 符号。）

```go

go list.Sort()  // 并发运行list.Sort；不等待它完成
```

函数字面量在 goroutine 调用中很方便：

```go

func Announce(message string, delay time.Duration) {
    go func() {
        time.Sleep(delay)
        fmt.Println(message)
    }()  // 注意括号 - 必须调用函数
}
```

在 Go 中，函数字面量是闭包：实现确保函数引用的变量在它们活跃期间一直存在。

### 通道

像映射一样，通道使用 make 分配，结果值充当对底层数据结构的引用。如果提供可选的整数参数，它会设置通道的缓冲区大小。默认值为零，用于无缓冲或同步通道。

```go

ci := make(chan int)            // 无缓冲的整数通道
cj := make(chan int, 0)         // 无缓冲的整数通道
cs := make(chan *os.File, 100)  // 指向文件的指针的缓冲通道
```

无缓冲通道将通信（值的交换）与同步（保证两个计算（goroutines）处于已知状态）结合在一起。

#### 通道同步

有许多使用通道的好习惯。这里有一个例子。在上一节中，我们在后台启动了一个排序。通道可以允许启动 goroutine 等待排序完成。

```go

c := make(chan int)  // 分配一个通道
// 在goroutine中启动排序；完成后，在通道上发送信号
go func() {
    list.Sort()
    c <- 1  // 发送信号；值不重要
}()
doSomethingForAWhile()
<-c   // 等待排序完成；丢弃发送的值
```

接收器总是阻塞直到有数据可接收。如果通道是无缓冲的，发送方会阻塞直到接收器接收到值。如果通道有缓冲区，发送方只会阻塞直到值被复制到缓冲区；如果缓冲区已满，这意味着要等待某个接收器检索到一个值。

#### 通道作为信号量

缓冲通道可以用作信号量，例如限制吞吐量。在这个例子中，传入的请求被传递给 handle，它将一个值发送到通道，处理请求，然后从通道接收一个值以使 "信号量" 准备好下一个消费者。通道缓冲区的容量限制了同时调用 process 的数量。

```go

var sem = make(chan int, MaxOutstanding)

func handle(r *Request) {
    sem <- 1    // 等待活动队列排空
    process(r)  // 可能需要很长时间
    <-sem       // 完成；允许下一个请求运行
}

func Serve(queue chan *Request) {
    for {
        req := <-queue
        go handle(req)  // 不等待handle完成
    }
}
```

一旦 MaxOutstanding 个处理器正在执行 process，任何更多的处理器都会在尝试发送到已满的通道缓冲区时阻塞，直到其中一个现有处理器完成并从缓冲区接收。

### 通道的通道

Go 最重要的属性之一是通道是一等值，可以像其他任何值一样分配和传递。这个属性的一个常见用途是实现安全、并行的多路分解。

```go

type Request struct {
    args        []int
    f           func([]int) int
    resultChan  chan int
}
```

客户端提供一个函数及其参数，以及请求对象内部的一个通道，用于接收答案。

```go

func sum(a []int) (s int) {
    for _, v := range a {
        s += v
    }
    return
}

request := &Request{[]int{3, 4, 5}, sum, make(chan int)}
// 发送请求
clientRequests <- request
// 等待响应
fmt.Printf("answer: %d\n", <-request.resultChan)
```

在服务器端，处理器函数是唯一改变的东西：

```go

func handle(queue chan *Request) {
    for req := range queue {
        req.resultChan <- req.f(req.args)
    }
}
```

显然，要使其现实还需要做更多工作，但这段代码是一个速率限制、并行、非阻塞 RPC 系统的框架，而且没有互斥锁。

### 并行化

这些想法的另一个应用是跨多个 CPU 核心并行化计算。如果计算可以分解为可以独立执行的单独部分，就可以并行化，使用通道来表示每个部分何时完成。

```go

type Vector []float64

// 对v[i], v[i+1] ... 直到v[n-1]应用操作
func (v Vector) DoSome(i, n int, u Vector, c chan int) {
    for ; i < n; i++ {
        v[i] += u.Op(v[i])
    }
    c <- 1    // 表示这个部分完成
}
```

我们在循环中独立启动这些部分，每个 CPU 一个。它们可以以任何顺序完成，但这没关系；我们只需要在启动所有 goroutines 后通过排空通道来计数完成信号。

```go

const numCPU = 4 // CPU核心数

func (v Vector) DoAll(u Vector) {
    c := make(chan int, numCPU)  // 缓冲是可选的但合理的
    for i := 0; i < numCPU; i++ {
        go v.DoSome(i*len(v)/numCPU, (i+1)*len(v)/numCPU, u, c)
    }
    // 排空通道
    for i := 0; i < numCPU; i++ {
        <-c    // 等待一个任务完成
    }
    // 全部完成
}
```

### 泄漏缓冲区

并发编程的工具甚至可以使非并发思想更容易表达。这是一个从 RPC 包抽象出来的例子。客户端 goroutine 循环从某个源接收数据，可能是网络。为了避免分配和释放缓冲区，它保留一个空闲列表，并使用缓冲通道来表示它。如果通道为空，则分配一个新缓冲区。一旦消息缓冲区准备好，它就会通过 serverChan 发送到服务器。

```go

var freeList = make(chan *Buffer, 100)
var serverChan = make(chan *Buffer)

func client() {
    for {
        var b *Buffer
        // 如果可用则获取缓冲区；否则分配
        select {
        case b = <-freeList:
            // 得到一个；无需做其他事情
        default:
            // 没有空闲的，所以分配一个新的
            b = new(Buffer)
        }
        load(b)              // 从网络读取下一条消息
        serverChan <- b      // 发送到服务器
    }
}
```

服务器循环从客户端接收每条消息，处理它，并将缓冲区返回到空闲列表。

```go

func server() {
    for {
        b := <-serverChan    // 等待工作
        process(b)
        // 如果有空间则重用缓冲区
        select {
        case freeList <- b:
            // 缓冲区在空闲列表中；无需做其他事情
        default:
            // 空闲列表已满，继续
        }
    }
}
```

客户端尝试从 freeList 获取缓冲区；如果没有可用的，则分配一个新的。服务器发送到 freeList 会将 b 放回空闲列表，除非列表已满，在这种情况下缓冲区被丢弃，由垃圾回收器回收。（select 语句中的 default 子句在没有其他 case 准备好时执行，这意味着 select 永远不会阻塞。）这个实现只用几行代码就构建了一个漏桶空闲列表，依靠缓冲通道和垃圾回收器进行簿记。

## 错误

### 错误处理概述

库例程通常必须向调用者返回某种错误指示。如前所述，Go 的多值返回使得返回详细错误描述以及正常返回值变得容易。使用此功能提供详细错误信息是良好的风格。

### Error 接口

按照约定，错误具有 error 类型，这是一个简单的内置接口：

```go

type error interface {
    Error() string
}
```

库编写者可以自由地在底层使用更丰富的模型实现此接口，不仅可以查看错误，还可以提供一些上下文。

### 自定义错误类型

例如，os.Open 返回的错误值是 os.PathError 类型：

```go

// PathError记录错误以及导致错误的操作和文件路径
type PathError struct {
    Op string    // "open", "unlink"等
    Path string  // 相关文件
    Err error    // 系统调用返回的错误
}

func (e *PathError) Error() string {
    return e.Op + " " + e.Path + ": " + e.Err.Error()
}
```

PathError 的 Error 生成如下字符串：

```Plain Text

open /etc/passwx: no such file or directory
```

这样的错误包含有问题的文件名、操作和触发的操作系统错误，即使在远离导致它的调用的地方打印也很有用；它比简单的 "没有这样的文件或目录" 提供了更多信息。

### 错误字符串约定

在可行的情况下，错误字符串应该标识它们的来源，例如通过添加生成错误的操作或包的前缀。例如，在 image 包中，由于未知格式导致的解码错误的字符串表示是 "image: unknown format"。

### 错误类型断言

关心精确错误细节的调用者可以使用类型开关或类型断言来查找特定错误并提取详细信息。对于 PathErrors，这可能包括检查内部的 Err 字段以查找可恢复的失败。

```go

for try := 0; try < 2; try++ {
    file, err = os.Create(filename)
    if err == nil {
        return
    }
    if e, ok := err.(*os.PathError); ok && e.Err == syscall.ENOSPC {
        deleteTempFiles()  // 恢复一些空间
        continue
    }
    return
}
```

### Panic

向调用者报告错误的通常方式是将错误作为额外的返回值返回。典型的 Read 方法是一个众所周知的例子；它返回字节计数和错误。但如果错误是不可恢复的呢？有时程序根本无法继续。

为此，有一个内置函数 panic，它实际上创建一个运行时错误，将停止程序（但请看下一节）。该函数接受任意类型的单个参数 —— 通常是字符串 —— 在程序终止时打印。它也是一种表明发生了不可能的事情的方式，例如退出无限循环。

```go

// 使用牛顿法计算立方根的玩具实现
func CubeRoot(x float64) float64 {
    z := x/3   // 任意初始值
    for i := 0; i < 1e6; i++ {
        prevz := z
        z -= (z*z*z-x) / (3*z*z)
        if veryClose(z, prevz) {
            return z
        }
    }
    // 一百万次迭代仍未收敛；出了问题
    panic(fmt.Sprintf("CubeRoot(%g) did not converge", x))
}
```

这只是一个例子，但实际的库函数应该避免 panic。如果问题可以被掩盖或解决，让事情继续运行总是比让整个程序崩溃更好。一个可能的例外是在初始化期间：如果库确实无法设置自己，panic 可能是合理的。

```go

var user = os.Getenv("USER")

func init() {
    if user == "" {
        panic("no value for $USER")
    }
}
```

### Recover

当调用 panic 时，包括隐式的运行时错误（如索引切片超出范围或类型断言失败），它会立即停止当前函数的执行，并开始展开 goroutine 的栈，同时运行任何延迟的函数。如果展开到达 goroutine 栈的顶部，程序就会终止。然而，可以使用内置函数 recover 来重新获得 goroutine 的控制权并恢复正常执行。

调用 recover 会停止展开并返回传递给 panic 的参数。由于在展开期间运行的唯一代码是在延迟函数内部，recover 只在延迟函数内部有用。

recover 的一个应用是在服务器中关闭失败的 goroutine 而不杀死其他正在执行的 goroutine。

```go

func server(workChan <-chan *Work) {
    for work := range workChan {
        go safelyDo(work)
    }
}

func safelyDo(work *Work) {
    defer func() {
        if err := recover(); err != nil {
            log.Println("work failed:", err)
        }
    }()
    do(work)
}
```

在这个例子中，如果 do (work) panic，结果将被记录，goroutine 将干净地退出而不干扰其他 goroutine。在延迟闭包中不需要做其他事情；调用 recover 完全处理了这种情况。

由于 recover 除非直接从延迟函数中调用否则总是返回 nil，延迟代码可以调用本身使用 panic 和 recover 的库例程而不会失败。例如，safelyDo 中的延迟函数可以在调用 recover 之前调用日志函数，而该日志代码将不受 panic 状态的影响。

## 示例 Web 服务器

### 概述

让我们用一个完整的 Go 程序来结束，一个 Web 服务器。这实际上是一种 Web 重新服务器。Google 在[chart.apis.google.com](https://chart.apis.google.com)提供了一项服务，可以自动将数据格式化为图表和图形。但交互式使用很困难，因为你需要将数据作为查询放入 URL。这里的程序为一种数据提供了更好的界面：给定一段短文本，它调用图表服务器生成一个 QR 码，一个编码文本的方框矩阵。这个图像可以用手机的摄像头捕捉并解释为 URL，省去了你在手机的小键盘上输入 URL 的麻烦。

### 完整代码

```go

package main

import (
    "flag"
    "html/template"
    "log"
    "net/http"
)

var addr = flag.String("addr", ":1718", "http service address") // Q=17, R=18

var templ = template.Must(template.New("qr").Parse(templateStr))

func main() {
    flag.Parse()
    http.Handle("/", http.HandlerFunc(QR))
    err := http.ListenAndServe(*addr, nil)
    if err != nil {
        log.Fatal("ListenAndServe:", err)
    }
}

func QR(w http.ResponseWriter, req *http.Request) {
    templ.Execute(w, req.FormValue("s"))
}

const templateStr = `
<html>
<head>
QR Link Generator
</head>
<body>
{{if .}}
<img src="http://chart.apis.google.com/chart?chs=300x300&cht=qr&choe=UTF-8&chl={{.}}" />
<br>
{{.}}
<br>
<br>
{{end}}
<form action="/" name=f method="GET">
    <input maxLength=1024 size=70 name=s value="" title="Text to QR Encode">
    <input type=submit value="Show QR" name=qr>
</form>
</body>
</html>
`
```

### 代码解释

#### 包和导入

- 程序使用 main 包，表示它是一个可执行程序

- 导入了四个包：

    - flag：用于解析命令行标志

    - html/template：用于处理 HTML 模板

    - log：用于记录错误

    - net/http：用于 HTTP 服务器功能

#### 全局变量

- `addr`：一个命令行标志，设置 HTTP 服务的地址，默认值为 ":1718"

- `templ`：一个预解析的 HTML 模板，使用`template.Must`确保模板解析不会失败

#### Main 函数

- `flag.Parse()`：解析命令行标志

- `http.Handle("/", http.HandlerFunc(QR))`：将根路径 "/" 处理程序设置为 QR 函数

- `http.ListenAndServe(*addr, nil)`：启动 HTTP 服务器，监听指定地址

#### QR 函数

- 这是 HTTP 请求处理函数

- 它执行模板，将表单值 "s" 作为数据传递给模板

#### HTML 模板

- 模板使用 Go 的模板语法

- `{{if .}}`：如果数据非空，则显示 QR 码图像

- `<img src="http://chart.apis.google.com/chart?chs=300x300&cht=qr&choe=UTF-8&chl={{.}}" />`：生成 QR 码图像的 URL

- 表单允许用户输入文本并提交生成 QR 码

### 运行程序

运行程序后，你可以在浏览器中访问[http://localhost:1718](http://localhost:1718)，输入文本并点击 "Show QR" 按钮，程序将生成对应的 QR 码图像。

这个示例展示了 Go 语言的强大之处：用几行代码加上一些数据驱动的 HTML 文本，就能创建一个有用的 Web 服务器。
> （注：文档部分内容可能由 AI 生成）