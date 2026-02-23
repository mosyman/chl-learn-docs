

你想让我解释 Go 语言中这三个核心内置函数（`append`、`copy`、`delete`）的官方注释和用法，我会用通俗易懂的方式帮你拆解每个函数的作用、使用场景和关键特性。

### 1. append 函数：给切片追加元素
#### 核心含义
`append` 是用来给切片**末尾追加元素**的内置函数，它的核心特点是：
- 如果切片的**容量（capacity）足够**，直接在原有底层数组上扩展切片，修改原数组；
- 如果容量不足，会**重新分配一块更大的内存**作为新底层数组，把原数据拷贝过去，再追加新元素；
- 必须接收 `append` 的返回值（因为切片可能被重新分配），通常直接赋值给原切片变量。

#### 关键用法示例
```go
package main

import "fmt"

func main() {
    // 基础用法：追加单个/多个元素
    nums := []int{1, 2, 3}
    nums = append(nums, 4)         // 追加单个元素，nums 变为 [1,2,3,4]
    nums = append(nums, 5, 6)      // 追加多个元素，nums 变为 [1,2,3,4,5,6]
    fmt.Println("基础追加：", nums)

    // 追加另一个切片（注意末尾的 ...）
    otherNums := []int{7, 8}
    nums = append(nums, otherNums...) // 把 otherNums 拆成单个元素追加，nums 变为 [1,2,3,4,5,6,7,8]
    fmt.Println("追加切片：", nums)

    // 特殊场景：给字节切片追加字符串
    byteSlice := append([]byte("hello "), "world"...) // 结果是 []byte("hello world")
    fmt.Println("字节切片追加字符串：", string(byteSlice))
}
```

#### 关键说明
- `slice = append(...)` 是固定写法，因为如果切片扩容，原变量指向的还是旧地址，必须用返回值更新；
- `anotherSlice...` 中的 `...` 是“展开操作符”，把切片拆成单个元素传入；
- 字符串追加到字节切片的场景，常用于处理字节流（如网络传输、文件读写）。

### 2. copy 函数：拷贝切片元素
#### 核心含义
`copy` 用来把**源切片（src）的元素拷贝到目标切片（dst）**，核心特点是：
- 支持源/目标切片内存重叠（比如自己拷贝给自己）；
- 拷贝的元素数量是 `len(src)` 和 `len(dst)` 中的**较小值**（不会自动扩容目标切片）；
- 返回值是实际拷贝的元素个数；
- 特殊场景：可以把字符串拷贝到字节切片（字符串本质是不可变字节序列）。

#### 关键用法示例
```go
package main

import "fmt"

func main() {
    // 基础用法：拷贝切片
    src := []int{1, 2, 3, 4}
    dst := make([]int, 2) // 目标切片长度为2，容量默认2
    copiedNum := copy(dst, src) // 只能拷贝2个元素（dst长度限制）
    fmt.Println("拷贝结果：", dst)       // 输出 [1,2]
    fmt.Println("实际拷贝个数：", copiedNum) // 输出 2

    // 源切片比目标切片短
    src2 := []int{5, 6}
    dst2 := []int{0, 0, 0, 0}
    copy(dst2, src2) // 拷贝2个元素，dst2 变为 [5,6,0,0]
    fmt.Println("源短于目标：", dst2)

    // 特殊场景：字符串拷贝到字节切片
    str := "hello copy"
    byteDst := make([]byte, 5)
    copy(byteDst, str) // 拷贝前5个字节，byteDst 是 []byte("hello")
    fmt.Println("字符串拷贝到字节切片：", string(byteDst))

    // 内存重叠的场景（自己拷贝自己）
    nums := []int{1, 2, 3, 4, 5}
    copy(nums[1:], nums[:3]) // 把 [1,2,3] 拷贝到 [2,3,4] 的位置，nums 变为 [1,1,2,3,5]
    fmt.Println("内存重叠拷贝：", nums)
}
```

#### 关键说明
- `copy` 不会自动扩容目标切片，比如目标切片长度为0时，拷贝结果为空；
- 返回值可以用来判断是否完成全量拷贝（比如 `copiedNum == len(src)` 说明拷贝完成）；
- 相比 `append`，`copy` 更适合“精准控制拷贝范围”的场景，且不会修改原切片的底层数组（除非重叠）。

### 3. delete 函数：删除 map 中的键值对
#### 核心含义
`delete` 用来删除 map 中指定键（key）对应的键值对，核心特点是：
- 如果 map 是 `nil`，或者指定的 key 不存在，`delete` 不会报错，也不会做任何操作（no-op）；
- 没有返回值，直接修改原 map。

#### 关键用法示例
```go
package main

import "fmt"

func main() {
    // 基础用法：删除存在的key
    userAge := map[string]int{
        "张三": 20,
        "李四": 25,
        "王五": 30,
    }
    delete(userAge, "李四") // 删除 "李四" 对应的键值对
    fmt.Println("删除后：", userAge) // 输出 map[张三:20 王五:30]

    // 删除不存在的key（无操作）
    delete(userAge, "赵六")
    fmt.Println("删除不存在的key：", userAge) // 还是 map[张三:20 王五:30]

    // 删除 nil map 中的key（无操作，不报错）
    var nilMap map[string]int // 只声明，未初始化，值为nil
    delete(nilMap, "test")    // 不会panic
    fmt.Println("nil map删除：", nilMap) // 输出 map[]
}
```

#### 关键说明
- Go 中没有“直接删除切片元素”的内置函数，`delete` 仅针对 map；
- 删除 map 后，map 的长度（`len`）会减1（如果 key 存在）；
- 无需先判断 key 是否存在，直接调用 `delete` 即可（语言层面已做兼容）。

### 总结
1. `append`：给切片追加元素，**必须接收返回值**，容量不足时自动扩容，支持追加切片/字符串（字节切片场景）；
2. `copy`：拷贝切片元素，拷贝数量取源/目标长度的较小值，**不会自动扩容**，返回实际拷贝个数；
3. `delete`：仅用于删除 map 的键值对，对 nil map/不存在的 key 无操作，无返回值。






