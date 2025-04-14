# Go概述

* Go 是一个开源的编程语言，它能让构造简单、可靠且高效的软件变得容易。

  Go是从2007年末由Robert Griesemer, Rob Pike, Ken Thompson主持开发，后来还加入了Ian Lance Taylor, Russ Cox等人，并最终于2009年11月开源，在2012年早些时候发布了Go 1稳定版本。现在Go的开发已经是完全开放的，并且拥有一个活跃的社区。

* Go特色

  - 简洁、快速、安全
  - 天生并行、有趣、开源
  - 内存管理、数组安全、编译迅速
  - 与编译型语言类似，是静态语言，但是开发可以感觉像动态语言

* 用途

  Go 语言被设计成一门应用于搭载 Web 服务器，存储集群或类似用途的巨型中央服务器的系统编程语言。

  对于高性能分布式系统领域而言，Go 语言无疑比大多数其它语言有着更高的开发效率。它提供了海量并行的支持，这对于游戏服务端的开发而言是再好不过了。

* 第一个Go语言代码
  > 注意，只有packge包名为main才可以执行，并且只能执行函数名为main的函数
  >	
  > ~~~Go
  > package main
  > 
  > import "fmt"
  > 
  > func main() {
  > 	fmt.Println("hello")
  > }
  > ~~~
  >
  
1. 使用`go run hello.go`
  
2. 使用`go build`生成二进制文件后执行
# Go语言结构

* 基础组成：包声明、引入包、函数、变量、语句&表达式、注释

  * 示例代码

    ~~~go
    package main
    
    import "fmt"
    
    func main() {
    	fmt.Println("hello")
    }
    ~~~

    * packge定义包名，package main表示一个可独立执行的程序，每个 Go 应用程序都包含一个名为 main 的包。
    * 下一行 *import "fmt"* 告诉 Go 编译器这个程序需要使用 fmt 包（的函数，或其他元素），fmt 包实现了格式化 IO（输入/输出）的函数。
    * 下一行 *func main()* 是程序开始执行的函数。main 函数是每一个可执行程序所必须包含的，一般来说都是在启动后第一个执行的函数**（如果有 init() 函数则会先执行该函数）**。
    * 下一行 *fmt.Println(...)* 可以将字符串输出到控制台，并在最后自动增加换行字符 \n。 <u>使用 fmt.Print("hello, world\n") 可以得到相同的结果。</u>
      Print 和 Println 这两个函数也支持使用变量，**如：fmt.Println(arr)。如果没有特别指定，它们会以默认的打印格式将变量 arr 输出到控制台。**
    * **当标识符（包括常量、变量、类型、函数名、结构字段等等）以一个大写字母开头**，如：Group1，那么使用**这种形式的标识符的对象就可以被外部包的代码所使用（客户端程序需要先导入这个包）**，这被称为导出（像面向对象语言中的 public）；**标识符如果以小写字母开头，则对包外是不可见的**，但是他们在整个包的内部是可见并且可用的（像面向对象语言中的 protected ）。

* 注意：

  1. 需要注意的是 **{** 不能单独放在一行，所以以下代码在运行时会产生错误：

     ~~~go
     package main
     
     import "fmt"
     
     func main()  
     {  // 错误，{ 不能在单独的行上
         fmt.Println("Hello, World!")
     }
     ~~~

  2. 文件名和包名没有直接关系，不一定要将文件名和包名定成同一个

  3. 同一文件夹下只能有一个包名，否则编译报错

     * 文件结构：

       ![image-20241009151710511](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/image-20241009151710511.png)

     * hello.go

       ~~~go
       package main
       
       import (
       	"fmt"
       	"test/myMath"
       )
       
       func main() {
       	fmt.Println("hello")
       	fmt.Println(myMath.Add(2, 3))
       	fmt.Println(myMath.Sub(5, 3))
       }
       ~~~

     * myMath1.go

       ~~~go
       package myMath
       
       func Add(a, b int) int {
       	return a + b
       }
       ~~~

     * myMath2.go

       ~~~go
       package myMath
       
       func Sub(x, y int) int {
       	return x - y
       }
       ~~~



# Go语言基础语法

## Go标记

* Go 程序可以由多个标记组成，可以是关键字，标识符，常量，字符串，符号。如以下 GO 语句由 6 个标记组成

  ~~~go
  fmt.Println("Hello, World!")
  ~~~

  6 个标记是(每行一个)：

  ~~~go
  1. fmt
  2. .
  3. Println
  4. (
  5. "Hello, World!"
  6. )
  ~~~

## 行分隔符

* 在 Go 程序中，一行代表一个语句结束。每个语句不需要像 C 家族中的其它语言一样以分号 ; 结尾，因为这些工作都将由 Go 编译器自动完成。

  如果你打算将多个语句写在同一行，它们则必须使用 ; 人为区分，但在实际开发中我们并不鼓励这种做法。

## 标识符

标识符用来命名变量、类型等程序实体。一个标识符实际上就是一个或是多个字母(A~Z和a~z)数字(0~9)、下划线_组成的序列，但是第一个字符必须是字母或下划线而不能是数字。

以下是有效的标识符：

```go
mahesh   kumar   abc   move_name   a_123
myname50   _temp   j   a23b9   retVal
```



## 字符串连接

* Go 语言的字符串连接可以通过 **+** 实现：

    ~~~go
    package main
    import "fmt"
    func main() {
        fmt.Println("Google" + "Runoob")
    }
    ~~~

    以上实例输出结果为：

    ```go
    GoogleRunoob
    ```
    
    

## 关键字

* 下面列举了 Go 代码中会使用到的 25 个关键字或保留字：

  | break    | default     | func   | interface | select |
  | -------- | ----------- | ------ | --------- | ------ |
  | case     | defer       | go     | map       | struct |
  | chan     | else        | goto   | package   | switch |
  | const    | fallthrough | if     | range     | type   |
  | continue | for         | import | return    | var    |

* 除了以上介绍的这些关键字，Go 语言还有 36 个预定义标识符：

  | append | bool    | byte    | cap     | close  | complex | complex64 | complex128 | uint16  |
  | ------ | ------- | ------- | ------- | ------ | ------- | --------- | ---------- | ------- |
  | copy   | false   | float32 | float64 | imag   | int     | int8      | int16      | uint32  |
  | int32  | int64   | iota    | len     | make   | new     | nil       | panic      | uint64  |
  | print  | println | real    | recover | string | true    | uint      | uint8      | uintptr |

## Go中的空格

在 Go 语言中，空格通常用于分隔标识符、关键字、运算符和表达式，以提高代码的可读性。

Go 语言中变量的声明必须使用空格隔开，如：

```go
var x int
const Pi float64 = 3.14159265358979323846
```

在运算符和操作数之间要使用空格能让程序更易阅读：

无空格：

```go
fruit=apples+oranges;
```

在变量与运算符间加入空格，程序看起来更加美观，如：

```go
fruit = apples + oranges; 
```

在关键字和表达式之间要使用空格。

例如：

```go
if x > 0 {
    // do something
}
```

在函数调用时，函数名和左边等号之间要使用空格，参数之间也要使用空格。

例如：

```go
result := add(2, 3)
```



## 格式化字符串

* Go 语言中使用 **fmt.Sprintf** 或 **fmt.Printf** 格式化字符串并赋值给新串：

  - **Sprintf** 根据格式化参数生成格式化的字符串并**返回该字符串**。

    ~~~go
    package main
    
    import (
        "fmt"
    )
    
    func main() {
       // %d 表示整型数字，%s 表示字符串
        var stockcode=123
        var enddate="2020-12-31"
        var url="Code=%d&endDate=%s"
        var target_url=fmt.Sprintf(url,stockcode,enddate)
        fmt.Println(target_url)
    }
    ~~~

    **需要注意，Sprintf，后面的参数顺序要和匹配的%d、%s格式一致，例如上述中，stockcode对应url中的%d，enddate对应url中的%s格式**

  - **Printf** 根据格式化参数生成格式化的字符串并**写入标准输出**。

    ~~~go
    package main
    
    import (
        "fmt"
    )
    
    func main() {
       // %d 表示整型数字，%s 表示字符串
        var stockcode=123
        var enddate="2020-12-31"
        var url="Code=%d&endDate=%s"
        fmt.Printf(url,stockcode,enddate)
    }
    ~~~

* 其他详细输出方式见

  1. [Go fmt.Sprintf](https://www.runoob.com/go/go-fmt-sprintf.html)
  2. [Go fmt.Printf](https://www.runoob.com/go/go-fmt-printf.html)



## 其他

* 程序的一般结构

  ~~~go
  // 当前程序的包名
  package main
  
  // 为fmt起别名为fmt2
  import fmt2 "fmt"
  
  // 调用的时候只需要Println()，而不需要fmt.Println()
  import . "fmt"
  
  // 导入其他包
  import . "fmt"
  
  // 常量定义
  const PI = 3.14
  
  // ----全局变量----的声明和赋值
  var name = "gopher"
  
  // 一般类型声明
  type newType int
  
  // 结构的声明
  type gopher struct{}
  
  // 接口的声明
  type golang interface{}
  
  // 由main函数作为程序入口点启动
  func main() {
      Println("Hello World!")
  }
  
  //函数名首字母小写为private
  func getId(){}
  
  //函数名首字母大写为Public
  func Print(){}
  ~~~



# Go数据类型

* Go 语言按类别有以下几种数据类型

  | 序号 | 类型和描述                                                   |
  | :--- | :----------------------------------------------------------- |
  | 1    | **布尔型** 布尔型的值只可以是常量 true 或者 false。一个简单的例子：var b bool = true。 |
  | 2    | **数字类型** 整型 int 和浮点型 float32、float64，Go 语言支持整型和浮点型数字，并且支持复数，其中位的运算采用补码。 |
  | 3    | **字符串类型:** 字符串就是一串固定长度的字符连接起来的字符序列。Go 的字符串是由单个字节连接起来的。Go 语言的字符串的字节使用 UTF-8 编码标识 Unicode 文本。 |
  | 4    | **派生类型:** 包括：(a) 指针类型（Pointer）(b) 数组类型(c) 结构化类型(struct)(d) Channel 类型(e) 函数类型(f) 切片类型(g) 接口类型（interface）(h) Map 类型 |

## 数字类型

| 序号 | 类型和描述                                                   |
| :--- | :----------------------------------------------------------- |
| 1    | **uint8** 无符号 8 位整型 (0 到 255)                         |
| 2    | **uint16** 无符号 16 位整型 (0 到 65535)                     |
| 3    | **uint32** 无符号 32 位整型 (0 到 4294967295)                |
| 4    | **uint64** 无符号 64 位整型 (0 到 18446744073709551615)      |
| 5    | **int8** 有符号 8 位整型 (-128 到 127)                       |
| 6    | **int16** 有符号 16 位整型 (-32768 到 32767)                 |
| 7    | **int32** 有符号 32 位整型 (-2147483648 到 2147483647)       |
| 8    | **int64** 有符号 64 位整型 (-9223372036854775808 到 9223372036854775807) |

| 序号 | 类型和描述                        |
| :--- | :-------------------------------- |
| 1    | **float32** IEEE-754 32位浮点型数 |
| 2    | **float64** IEEE-754 64位浮点型数 |
| 3    | **complex64** 32 位实数和虚数     |
| 4    | **complex128** 64 位实数和虚数    |

| 序号 | 类型和描述                               |
| :--- | :--------------------------------------- |
| 1    | **byte** 类似 uint8                      |
| 2    | **rune** 类似 int32                      |
| 3    | **uint** 32 或 64 位                     |
| 4    | **int** 与 uint 一样大小                 |
| 5    | **uintptr** 无符号整型，用于存放一个指针 |

## 其他

* go 1.9版本对于数字类型，无需定义int、float32、float64，系统会自动识别

  ~~~go
  package main
  import "fmt"
  
  func main() {
     var a = 1.5
     var b =2
     fmt.Println(a,b)
  }
  ~~~

  结果：

  ~~~
  # go run int.go
  1.5 2
  ~~~

  

* 字符串去除空格和换行符

  ~~~go
  package main  
    
  import (  
      "fmt"  
      "strings"  
  )  
    
  func main() {  
      str := "这里是 www\n.runoob\n.com"  
      fmt.Println("-------- 原字符串 ----------")  
      fmt.Println(str)  
      // 去除空格  
      str = strings.Replace(str, " ", "", -1)  
      // 去除换行符  
      str = strings.Replace(str, "\n", "", -1)  
      fmt.Println("-------- 去除空格与换行后 ----------")  
      fmt.Println(str)  
  }
  ~~~

  结果：

  ~~~
  -------- 原字符串 ----------
  这里是 www
  .runoob
  .com
  -------- 去除空格与换行后 ----------
  这里是www.runoob.com
  ~~~



# Go语言变量

* 变量声明

  ~~~go
  package main
  import "fmt"
  func main() {
      var a string = "Runoob"
      fmt.Println(a)
  
      var b, c int = 1, 2
      fmt.Println(b, c)
  }
  ~~~

  * **第一种，指定变量类型，如果没有初始化，则变量默认为零值**。

    零值就是变量没有做初始化时系统默认设置的值

    ~~~
    var v_name v_type
    v_name = value
    ~~~

    实例：

    ~~~go
    package main
    import "fmt"
    func main() {
    
        // 声明一个变量并初始化
        var a = "RUNOOB"
        fmt.Println(a)
    
        // 没有初始化就为零值
        var b int
        fmt.Println(b)
    
        // bool 零值为 false
        var c bool
        fmt.Println(c)
    }
    ~~~

    以下几种类型的零值为nil：

    ~~~go
    var a *int
    var a []int
    var a map[string] int
    var a chan int
    var a func(string) int
    var a error // error 是接口
    ~~~

  * **第二种，根据值自行判定变量类型。**

    ~~~
    var v_name = value
    ~~~

    实例：

    ~~~go
    package main
    import "fmt"
    func main() {
        var d = true
        fmt.Println(d)
    }
    ~~~

  * **第三种，如果变量已经使用 var 声明过了，再使用 := 声明变量，就产生编译错误，格式：**

    ~~~go
    v_name := value
    ~~~

    例如：

    ```go
    var intVal int 
    intVal :=1 // 这时候会产生编译错误，因为 intVal 已经声明，不需要重新声明
    ```

    直接使用下面的语句即可：

    ```go
    intVal := 1 // 此时不会产生编译错误，因为有声明新的变量，因为 := 是一个声明语句
    ```

    **intVal := 1** 相等于：

    ```go
    var intVal int 
    intVal =1 
    ```

    可以将 var f string = "Runoob" 简写为 f := "Runoob"：

    ~~~go
    package main
    import "fmt"
    func main() {
        f := "Runoob" // var f string = "Runoob"
    
        fmt.Println(f)
    }
    ~~~

* 多变量声明

  ~~~go
  //类型相同多个变量, 非全局变量
  var vname1, vname2, vname3 type
  vname1, vname2, vname3 = v1, v2, v3
  
  var vname1, vname2, vname3 = v1, v2, v3 // 和 python 很像,不需要显示声明类型，自动推断
  
  vname1, vname2, vname3 := v1, v2, v3 // 出现在 := 左侧的变量不应该是已经被声明过的，否则会导致编译错误
  
  
  // 这种因式分解关键字的写法一般用于声明全局变量
  var (
      vname1 v_type1
      vname2 v_type2
  )
  
  ~~~

  实例：

  ~~~go
  package main
  import "fmt"
  
  var x, y int
  var (  // 这种因式分解关键字的写法一般用于声明全局变量
      a int
      b bool
  )
  
  var c, d int = 1, 2
  var e, f = 123, "hello"
  
  //这种不带声明格式的只能在函数体中出现
  //g, h := 123, "hello"
  
  func main(){
      g, h := 123, "hello"
      fmt.Println(x, y, a, b, c, d, e, f, g, h)
  }
  ~~~



## 值类型和引用类型

所有像 int、float、bool 和 string 这些基本类型都属于值类型，使用这些类型的变量直接指向存在内存中的值：

![4.4.2_fig4.1](https://www.runoob.com/wp-content/uploads/2015/06/4.4.2_fig4.1.jpgrawtrue)

当使用等号 `=` 将一个变量的值赋值给另一个变量时，如：`j = i`，实际上是在内存中将 i 的值进行了拷贝：

![4.4.2_fig4.2](https://www.runoob.com/wp-content/uploads/2015/06/4.4.2_fig4.2.jpgrawtrue)

你可以通过 &i 来获取变量 i 的内存地址，例如：0xf840000040（每次的地址都可能不一样）。

**值类型变量的值存储在栈中**。

内存地址会根据机器的不同而有所不同，甚至相同的程序在不同的机器上执行后也会有不同的内存地址。因为每台机器可能有不同的存储器布局，并且位置分配也可能不同。

更复杂的数据通常会需要使用多个字，这些数据一般使用引用类型保存。

**一个引用类型的变量 r1 存储的是 r1 的值所在的内存地址（数字），或内存地址中第一个字所在的位置。**

![4.4.2_fig4.3](https://www.runoob.com/wp-content/uploads/2015/06/4.4.2_fig4.3.jpgrawtrue)

这个内存地址称之为指针，这个指针实际上也被存在另外的某一个值中。

同一个引用类型的指针指向的多个字可以是在连续的内存地址中（内存布局是连续的），这也是计算效率最高的一种存储形式；也可以将这些字分散存放在内存中，每个字都指示了下一个字所在的内存地址。

**当使用赋值语句 r2 = r1 时，只有引用（地址）被复制。**

如果 r1 的值被改变了，那么这个值的所有引用都会指向被修改后的内容，在这个例子中，r2 也会受到影响。



## 简短形式，使用：=赋值操作符

我们知道可以在变量的初始化时省略变量的类型而由系统自动推断，声明语句写上 var 关键字其实是显得有些多余了，因此我们可以将它们简写为 a := 50 或 b := false。

a 和 b 的类型（int 和 bool）将由编译器自动推断。

这是使用变量的首选形式，但是它只能被用在函数体内，而不可以用于全局变量的声明与赋值。使用操作符 := 可以高效地创建一个新的变量，称之为初始化声明。

* 注意事项：

  如果在相同的代码块中，我们不可以再次对于相同名称的变量使用初始化声明，例如：a := 20 就是不被允许的，编译器会提示错误 no new variables on left side of :=，但是 a = 20 是可以的，因为这是给相同的变量赋予一个新的值。

  如果你在定义变量 a 之前使用它，则会得到编译错误 undefined: a。

  **如果你声明了一个局部变量却没有在相同的代码块中使用它，同样会得到编译错误，例如下面这个例子当中的变量 a：**

  ~~~go
  package main
  
  import "fmt"
  
  func main() {
     var a string = "abc"
     fmt.Println("hello, world")
  }
  ~~~

  此外，单纯地给 a 赋值也是不够的，这个值必须被使用；但是全局变量是允许声明但是不使用的，同一类型的多个变量可以声明在同一行，多变量也可以在同一行进行赋值：

  ~~~go
  var a, b int
  var c string
  a, b, c = 5, 7, "abc"
  ~~~

  如果上述变量没有被声明，那么应该这么用：

  ~~~go
  a, b, c := 5, 7, "abc"
  ~~~

* 其他

  1. 如果你想要交换两个变量的值，则可以简单地使用 **a, b = b, a**，两个变量的类型必须是相同。

  2. 空白标识符 _ 也被用于抛弃值，如值 5 在：_, b = 5, 7 中被抛弃。

     _ 实际上是一个只写变量，你不能得到它的值。**这样做是因为 Go 语言中你必须使用所有被声明的变量，但有时你并不需要使用从一个函数得到的所有返回值。**

     并行赋值也被用于当一个函数返回多个返回值时，比如这里的 val 和错误 err 是通过调用 Func1 函数同时得到：val, err = Func1(var1)。例如：

     ~~~go
     package main
     
     import "fmt"
     
     func main() {
       _,numb,strs := numbers() //只获取函数返回值的后两个
       fmt.Println(numb,strs)
     }
     
     //一个可以返回多个值的函数
     func numbers()(int,int,string){
       a , b , c := 1 , 2 , "str"
       return a,b,c
     }
     ~~~



# Go语言常量

- 显式类型定义： `const b string = "abc"`
- 隐式类型定义： `const b = "abc"`

* 多个相同类型的声明可以简写为：

  ~~~
  const c_name1, c_name2 = value1, value2
  ~~~

* 常量还可以用作枚举：

  ```go
  const (
      Unknown = 0
      Female = 1
      Male = 2
  )
  ```

  数字 0、1 和 2 分别代表未知性别、女性和男性。

  常量可以用len(), cap(), unsafe.Sizeof()函数计算表达式的值。常量表达式中，函数必须是内置函数，否则编译不过：

  ~~~go
  package main
  
  import "unsafe"
  const (
      a = "abc"
      b = len(a)
      c = unsafe.Sizeof(a)
  )
  
  func main(){
      println(a, b, c)
  }
  ~~~

  结果：

  ~~~
  abc 3 16
  ~~~

  **unsafe.Sizeof(a)=16,是因为字符串类型在go里是一个结构，包含一个指向底层数据的指针和一个表示字符串长度的整数，在64位系统上，每个指针占用8字节，整数占用8字节，因此一个字符串类型在64位系统上占用16字节空间**

* 在定义常量数组时，如果不提供初始值，则表示使用上行的表达式

  ~~~go
  package main
  
  import "fmt"
  
  const (
      a = 1
      b
      c
      d
  )
  
  func main() {
      fmt.Println(a)
      // b、c、d没有初始化，使用上一行(即a)的值
      fmt.Println(b)   // 输出1
      fmt.Println(c)   // 输出1
      fmt.Println(d)   // 输出1
  }
  ~~~

  

## iota

iota，特殊常量，可以认为是一个可以被编译器修改的常量。

iota 在 const关键字出现时将被重置为 0(const 内部的第一行之前)，**const 中每新增一行常量声明将使 iota 计数一次(iota 可理解为 const 语句块中的行索引)**。

iota 可以被用作枚举值：

```
const (
    a = iota
    b = iota
    c = iota
)
```

**第一个 iota 等于 0，每当 iota 在新的一行被使用时，它的值都会自动加 1；所以 a=0, b=1, c=2 可以简写为如下形式：**

> 但是，iota只是在同一个const常量组内递增，每当有新的的const关键字时，iota计数就会重新开始

```
const (
    a = iota
    b
    c
)
```

* 用法：

  ~~~go
  package main
  
  import "fmt"
  
  func main() {
      const (
              a = iota   //0
              b          //1
              c          //2
              d = "ha"   //独立值，iota += 1
              e          //"ha"   iota += 1
              f = 100    //iota +=1
              g          //100  iota +=1
              h = iota   //7,恢复计数
              i          //8
      )
      fmt.Println(a,b,c,d,e,f,g,h,i)
  }
  ~~~

  结果：

  ~~~
  0 1 2 ha ha 100 100 7 8
  ~~~

  用法2：

  ~~~go
  package main
  
  import "fmt"
  const (
      i=1<<iota
      j=3<<iota
      k
      l
  )
  
  func main() {
      fmt.Println("i=",i)
      fmt.Println("j=",j)
      fmt.Println("k=",k)
      fmt.Println("l=",l)
  }
  ~~~

  结果：

  ~~~
  i= 1
  j= 6
  k= 12
  l= 24
  ~~~

  iota 表示从 0 开始自动加 1，所以 **i=1<<0**, **j=3<<1**（**<<** 表示左移的意思），即：i=1, j=6，这没问题，关键在 k 和 l，从输出结果看 **k=3<<2**，**l=3<<3**。

  简单表述:

  - **i=1**：左移 0 位，不变仍为 1。
  - **j=3**：左移 1 位，变为二进制 **110**，即 6。
  - **k=3**：左移 2 位，变为二进制 **1100**，即 12。
  - **l=3**：左移 3 位，变为二进制 **11000**，即 24。

  注：<<n==*(2^n^)。



# Go语言运算符

## 算数运算符

| 运算符 | 描述 | 实例               |
| :----- | :--- | :----------------- |
| +      | 相加 | A + B 输出结果 30  |
| -      | 相减 | A - B 输出结果 -10 |
| *      | 相乘 | A * B 输出结果 200 |
| /      | 相除 | B / A 输出结果 2   |
| %      | 求余 | B % A 输出结果 0   |
| ++     | 自增 | A++ 输出结果 11    |
| --     | 自减 | A-- 输出结果 9     |

## 关系运算符

| 运算符 | 描述                                                         | 实例              |
| :----- | :----------------------------------------------------------- | :---------------- |
| ==     | 检查两个值是否相等，如果相等返回 True 否则返回 False。       | (A == B) 为 False |
| !=     | 检查两个值是否不相等，如果不相等返回 True 否则返回 False。   | (A != B) 为 True  |
| >      | 检查左边值是否大于右边值，如果是返回 True 否则返回 False。   | (A > B) 为 False  |
| <      | 检查左边值是否小于右边值，如果是返回 True 否则返回 False。   | (A < B) 为 True   |
| >=     | 检查左边值是否大于等于右边值，如果是返回 True 否则返回 False。 | (A >= B) 为 False |
| <=     | 检查左边值是否小于等于右边值，如果是返回 True 否则返回 False。 | (A <= B) 为 True  |

## 逻辑运算符

| 运算符 | 描述                                                         | 实例               |
| :----- | :----------------------------------------------------------- | :----------------- |
| &&     | 逻辑 AND 运算符。 如果两边的操作数都是 True，则条件 True，否则为 False。 | (A && B) 为 False  |
| \|\|   | 逻辑 OR 运算符。 如果两边的操作数有一个 True，则条件 True，否则为 False。 | (A \|\| B) 为 True |
| !      | 逻辑 NOT 运算符。 如果条件为 True，则逻辑 NOT 条件 False，否则为 True。 | !(A && B) 为 True  |

## 位运算符

| p    | q    | p & q | p \| q | p ^ q |
| :--- | :--- | :---- | :----- | :---- |
| 0    | 0    | 0     | 0      | 0     |
| 0    | 1    | 0     | 1      | 1     |
| 1    | 1    | 1     | 1      | 0     |
| 1    | 0    | 0     | 1      | 1     |

## 赋值运算符

| 运算符 | 描述                                           | 实例                                  |
| :----- | :--------------------------------------------- | :------------------------------------ |
| =      | 简单的赋值运算符，将一个表达式的值赋给一个左值 | C = A + B 将 A + B 表达式结果赋值给 C |
| +=     | 相加后再赋值                                   | C += A 等于 C = C + A                 |
| -=     | 相减后再赋值                                   | C -= A 等于 C = C - A                 |
| *=     | 相乘后再赋值                                   | C *= A 等于 C = C * A                 |
| /=     | 相除后再赋值                                   | C /= A 等于 C = C / A                 |
| %=     | 求余后再赋值                                   | C %= A 等于 C = C % A                 |
| <<=    | 左移后赋值                                     | C <<= 2 等于 C = C << 2               |
| >>=    | 右移后赋值                                     | C >>= 2 等于 C = C >> 2               |
| &=     | 按位与后赋值                                   | C &= 2 等于 C = C & 2                 |
| ^=     | 按位异或后赋值                                 | C ^= 2 等于 C = C ^ 2                 |
| \|=    | 按位或后赋值                                   | C \|= 2 等于 C = C \| 2               |

## 其他运算符

| 运算符 | 描述             | 实例                       |
| :----- | :--------------- | :------------------------- |
| &      | 返回变量存储地址 | &a; 将给出变量的实际地址。 |
| *      | 指针变量。       | *a; 是一个指针变量         |

## 其他

指针变量 ***** 和地址值 **&** 的区别：指针变量保存的是一个地址值，会分配独立的内存来存储一个整型数字。当变量前面有 ***** 标识时，才等同于 **&** 的用法，否则会直接输出一个整型数字。

~~~go
func main() {
   var a int = 4
   var ptr *int
   ptr = &a
   println("a的值为", a);    // 4
   println("*ptr为", *ptr);  // 4
   println("ptr为", ptr);    // 824633794744
}
~~~

# Go语言条件语句

* Go 语言提供了以下几种条件判断语句：

  | 语句                                                         | 描述                                                         |
  | :----------------------------------------------------------- | :----------------------------------------------------------- |
  | [if 语句](https://www.runoob.com/go/go-if-statement.html)    | **if 语句** 由一个布尔表达式后紧跟一个或多个语句组成。       |
  | [if...else 语句](https://www.runoob.com/go/go-if-else-statement.html) | **if 语句** 后可以使用可选的 **else 语句**, else 语句中的表达式在布尔表达式为 false 时执行。 |
  | [if 嵌套语句](https://www.runoob.com/go/go-nested-if-statements.html) | 你可以在 **if** 或 **else if** 语句中嵌入一个或多个 **if** 或 **else if** 语句。 |
  | [switch 语句](https://www.runoob.com/go/go-switch-statement.html) | **switch** 语句用于基于不同条件执行不同动作。                |
  | [select 语句](https://www.runoob.com/go/go-select-statement.html) | **select** 语句类似于 **switch** 语句，但是select会随机执行一个可运行的case。如果没有case可运行，它将阻塞，直到有case可运行。 |

  注意，Go语言没有三目运算符

## if

  ~~~go
  if 布尔表达式 {
     /* 在布尔表达式为 true 时执行 */
  }
  ~~~

  实例：

  ~~~go
  package main
  
  import "fmt"
  
  func main() {
     /* 定义局部变量 */
     var a int = 10
   
     /* 使用 if 语句判断布尔表达式 */
     if a < 20 {
         /* 如果条件为 true 则执行以下语句 */
         fmt.Printf("a 小于 20\n" )
     }
     fmt.Printf("a 的值为 : %d\n", a)
  }
  ~~~

  Go 的 if 还有一个强大的地方就是条件判断语句里面允许声明一个变量，这个变量的作用域只能在该条件逻辑块内，其他地方就不起作用了，如下所示:

  ~~~go
  package main
    
  import "fmt"
  func main() {
      if num := 9; num < 0 {
          fmt.Println(num, "is negative")
      } else if num < 10 {
          fmt.Println(num, "has 1 digit")
      } else {
          fmt.Println(num, "has multiple digits")
      }
  }
  ~~~

  **if 语句使用 tips**：

  **（1）** 不需使用括号将条件包含起来

  **（2）** 大括号{}必须存在，即使只有一行语句

  **（3）** 左括号必须在if或else的同一行

  **（4）** 在if之后，条件语句之前，可以添加变量初始化语句，使用；进行分隔

  **（5）** 在有返回值的函数中，最终的return不能在条件语句中

## switch语句

  ~~~go
  switch var1 {
      case val1:
          ...
      case val2:
          ...
      default:
          ...
  }
  ~~~

  变量 var1 可以是任何类型，而 val1 和 val2 则可以是同类型的任意值。类型不被局限于常量或整数，但必须是相同的类型；或者最终结果为相同类型的表达式。

  您可以同时测试多个可能符合条件的值，使用逗号分割它们，例如：case val1, val2, val3

  实例1：

  ~~~go
  package main
  
  import "fmt"
  
  func main() {
     /* 定义局部变量 */
     var grade string = "B"
     var marks int = 90
  
     switch marks {
        case 90: grade = "A"
        case 80: grade = "B"
        case 50,60,70 : grade = "C"
        default: grade = "D"  
     }
  
     switch {
        case grade == "A" :
           fmt.Printf("优秀!\n" )     
        case grade == "B", grade == "C" :
           fmt.Printf("良好\n" )      
        case grade == "D" :
           fmt.Printf("及格\n" )      
        case grade == "F":
           fmt.Printf("不及格\n" )
        default:
           fmt.Printf("差\n" );
     }
     fmt.Printf("你的等级是 %s\n", grade );      
  }
  ~~~

## typeSwitch

  switch 语句还可以被用于 type-switch 来判断某个 interface 变量中实际存储的变量类型。

  Type Switch 语法格式如下：

  ~~~go
  switch x.(type){
      case type:
         statement(s);      
      case type:
         statement(s); 
      /* 你可以定义任意个数的case */
      default: /* 可选 */
         statement(s);
  }
  ~~~

  实例2：

  ~~~go
  package main
  
  import "fmt"
  
  func main() {
     var x interface{}
       
     switch i := x.(type) {
        case nil:   
           fmt.Printf(" x 的类型 :%T",i)                
        case int:   
           fmt.Printf("x 是 int 型")                       
        case float64:
           fmt.Printf("x 是 float64 型")           
        case func(int) float64:
           fmt.Printf("x 是 func(int) 型")                      
        case bool, string:
           fmt.Printf("x 是 bool 或 string 型" )       
        default:
           fmt.Printf("未知型")     
     }   
  }
  ~~~

## fallthrough

  使用 fallthrough 会强制执行后面的 case 语句，fallthrough 不会判断下一条 case 的表达式结果是否为 true。

  ~~~go
  package main
  
  import "fmt"
  
  func main() {
  
      switch {
      case false:
              fmt.Println("1、case 条件语句为 false")
              fallthrough
      case true:
              fmt.Println("2、case 条件语句为 true")
              fallthrough
      case false:
              fmt.Println("3、case 条件语句为 false")
              fallthrough
      case true:
              fmt.Println("4、case 条件语句为 true")
      case false:
              fmt.Println("5、case 条件语句为 false")
              fallthrough
      default:
              fmt.Println("6、默认 case")
      }
  }
  ~~~

  结果：

  ~~~go
  2、case 条件语句为 true
  3、case 条件语句为 false
  4、case 条件语句为 true
  ~~~

  1. 不同的 case 之间不使用 break 分隔，默认只会执行一个 case。
  2. 如果想要执行多个 case，需要使用 fallthrough 关键字，也可用 break 终止。



## select

select 是 Go 中的一个控制结构，类似于 switch 语句。

**select 语句只能用于通道操作，每个 case 必须是一个通道操作，要么是发送要么是接收。**

**select 语句会监听所有指定的通道上的操作，一旦其中一个通道准备好就会执行相应的代码块。**

**如果多个通道都准备好，那么 select 语句会随机选择一个通道执行。如果所有通道都没有准备好，那么执行 default 块中的代码。**

* 语法：

  ~~~go
  select {
    case <- channel1:
      // 执行的代码
    case value := <- channel2:
      // 执行的代码
    case channel3 <- value:
      // 执行的代码
  
      // 你可以定义任意数量的 case
  
    default:
      // 所有通道都没有准备好，执行的代码
  }
  ~~~

  以下描述了 select 语句的语法：

  - 每个 case 都必须是一个通道

  - 所有 channel 表达式都会被求值

  - 所有被发送的表达式都会被求值

  - 如果任意某个通道可以进行，它就执行，其他被忽略。

  - 如果有多个 case 都可以运行，select 会随机公平地选出一个执行，其他不会执行。

    否则：

    1. 如果有 default 子句，则执行该语句。
    2. 如果没有 default 子句，select 将阻塞，直到某个通道可以运行；Go 不会重新对 channel 或值进行求值。

* 实例：

  ~~~go
  package main
  
  import (
      "fmt"
      "time"
  )
  
  func main() {
  
      c1 := make(chan string)
      c2 := make(chan string)
  
      go func() {
          time.Sleep(1 * time.Second)
          c1 <- "one"
      }()
      go func() {
          time.Sleep(2 * time.Second)
          c2 <- "two"
      }()
  
      for i := 0; i < 2; i++ {
          select {
          case msg1 := <-c1:
              fmt.Println("received", msg1)
          case msg2 := <-c2:
              fmt.Println("received", msg2)
          }
      }
  }
  ~~~

  以上代码执行结果为：

  ```
  received one
  received two
  ```

  以上实例中，我们创建了两个通道 c1 和 c2。

  **select 语句等待两个通道的数据。如果接收到 c1 的数据，就会打印 "received one"；如果接收到 c2 的数据，就会打印 "received two"。**

  以下实例中，我们定义了两个通道，并启动了两个协程（Goroutine）从这两个通道中获取数据。在 main 函数中，我们使用 select 语句在这两个通道中进行非阻塞的选择，如果两个通道都没有可用的数据，就执行 default 子句中的语句。

  以下实例执行后会不断地从两个通道中获取到的数据，当两个通道都没有可用的数据时，会输出 "no message received"。

  ~~~go
  package main
  
  import "fmt"
  
  func main() {
    // 定义两个通道
    ch1 := make(chan string)
    ch2 := make(chan string)
  
    // 启动两个 goroutine，分别从两个通道中获取数据
    go func() {
      for {
        ch1 <- "from 1"
      }
    }()
    go func() {
      for {
        ch2 <- "from 2"
      }
    }()
  
    // 使用 select 语句非阻塞地从两个通道中获取数据
    for {
      select {
      case msg1 := <-ch1:
        fmt.Println(msg1)
      case msg2 := <-ch2:
        fmt.Println(msg2)
      default:
        // 如果两个通道都没有可用的数据，则执行这里的语句
        fmt.Println("no message received")
      }
    }
  }
  ~~~

  结果：

  因为在两个goroutine中，都采用了无限循环，且select也被无限循环包含，所以结果是无穷的

* 注意：select 会随机检测条件，不是循环检测，是为了避免饥饿问题。如果有满足则执行并退出，否则一致随机检测。如下代码作为测试：

  ~~~go
  package main
  
  import (
     "fmt"   "time")
  
  func Chann(ch chan int, stopCh chan bool) {
     for j := 0; j < 10; j++ {
        ch <- j
        time.Sleep(time.Second)
     }
     stopCh <- true}
  
  func main() {
  
     ch := make(chan int)
     c := 0   stopCh := make(chan bool)
  
     go Chann(ch, stopCh)
  
     for {
        select {
        case c = <-ch:
           fmt.Println("Receive C", c)
        case s := <-ch:
           fmt.Println("Receive S", s)
        case _ = <-stopCh:
           goto end
        }
     }
  end:
  }
  ~~~

  这个代码可能会造成饥饿或死锁问题：

  ~~~go
  package main
  
  import (
  	"fmt"
  	"time"
  )
  
  func Chann(ch chan int, stopCh chan bool) {
  	for i := 0; i < 10; i++ {
  		ch <- i
  		time.Sleep(time.Second * 1)
  	}
  	stopCh <- true
  }
  
  func main() {
  	ch := make(chan int)
  	c := 0
  	stopCh := make(chan bool)
  
  	go Chann(ch, stopCh)
  
  	for {
  		select {
  		case c = <-ch:
  			fmt.Printf("received: C=%d \n", c)
  		case s := <-stopCh:
  			fmt.Printf("received: S=%d ", s)
  		case _ = <-stopCh:
  			goto end
  		}
  	}
  end:
  }
  ~~~




# Go语言循环语句

## for循环

Go 语言的 For 循环有 3 种形式，只有其中的一种使用分号。

和 C 语言的 for 一样：

```go
for init; condition; post { }
```

和 C 的 while 一样：

```go
for condition { }
```

和 C 的 for(;;) 一样：

```go
for { }
```

- init： 一般为赋值表达式，给控制变量赋初值；
- condition： 关系表达式或逻辑表达式，循环控制条件；
- post： 一般为赋值表达式，给控制变量增量或减量。

**for 循环的 range 格式可以对 slice、map、数组、字符串等进行迭代循环。**格式如下：

```go
for key, value := range oldMap {
    newMap[key] = value
}
```

以上代码中的 key 和 value 是可以省略。

如果只想读取 key，格式如下：

```go
for key := range oldMap
```

或者这样：

for key, _ := range oldMap

如果只想读取 value，格式如下：

```go
for _, value := range oldMap
```

* 实例1：

  ~~~go
  package main
  
  import "fmt"
  
  func main() {
     sum := 0
        for i := 0; i <= 10; i++ {
           sum += i
        }
     fmt.Println(sum)
  }
  ~~~

* 实例2：

  init 和 post 参数是可选的，我们可以直接省略它，类似 While 语句。

  以下实例在 sum 小于 10 的时候计算 sum 自相加后的值：

  ~~~GO
  package main
  
  import "fmt"
  
  func main() {
     sum := 1
     for ; sum <= 10; {
        sum += sum
     }
     fmt.Println(sum)
  
     // 这样写也可以，更像 While 语句形式
     for sum <= 10{
        sum += sum
     }
     fmt.Println(sum)
  }
  ~~~

* 实例3：

  无限循环

  ~~~go
  package main
  
  import "fmt"
  
  func main() {
     sum := 0
     for {
        sum++ // 无限循环下去
     }
     fmt.Println(sum) // 无法输出
  }
  ~~~

* 实例4：

  for-each range循环

  ~~~go
  package main
  import "fmt"
  
  func main() {
     strings := []string{"google", "runoob"}
     for i, s := range strings {
        fmt.Println(i, s)
     }
  
  
     numbers := [6]int{1, 2, 3, 5} 
     for i,x:= range numbers {
        fmt.Printf("第 %d 位 x 的值 = %d\n", i,x)
     }  
  }
  ~~~

  其输出为：

  ~~~
  0 google
  1 runoob
  第 0 位 x 的值 = 1
  第 1 位 x 的值 = 2
  第 2 位 x 的值 = 3
  第 3 位 x 的值 = 5
  第 4 位 x 的值 = 0
  第 5 位 x 的值 = 0
  ~~~

* 实例5：

  省略key、value的range格式for循环

  ~~~go
  package main
  import "fmt"
  
  func main() {
      map1 := make(map[int]float32)
      map1[1] = 1.0
      map1[2] = 2.0
      map1[3] = 3.0
      map1[4] = 4.0
      
      // 读取 key 和 value
      for key, value := range map1 {
        fmt.Printf("key is: %d - value is: %f\n", key, value)
      }
  
      // 读取 key
      for key := range map1 {
        fmt.Printf("key is: %d\n", key)
      }
  
      // 读取 value
      for _, value := range map1 {
        fmt.Printf("value is: %f\n", value)
      }
  }
  ~~~

  其输出为：

  ~~~go
  key is: 4 - value is: 4.000000
  key is: 1 - value is: 1.000000
  key is: 2 - value is: 2.000000
  key is: 3 - value is: 3.000000
  key is: 1
  key is: 2
  key is: 3
  key is: 4
  value is: 1.000000
  value is: 2.000000
  value is: 3.000000
  value is: 4.000000
  ~~~



## 循环控制语句

### break

* 与C语言不同的是，在go语言中，在多重循环中，可以用标号label标出想要break的循环

  * 实例：

    ~~~go
    package main
    
    import "fmt"
    
    func main() {
    
       // 不使用标记
       fmt.Println("---- break ----")
       for i := 1; i <= 3; i++ {
          fmt.Printf("i: %d\n", i)
          for i2 := 11; i2 <= 13; i2++ {
             fmt.Printf("i2: %d\n", i2)
             break
          }
       }
    
       // 使用标记
       fmt.Println("---- break label ----")
       re:
          for i := 1; i <= 3; i++ {
             fmt.Printf("i: %d\n", i)
             for i2 := 11; i2 <= 13; i2++ {
             fmt.Printf("i2: %d\n", i2)
             break re
          }
       }
    }
    ~~~

    其输出为：

    ~~~
    ---- break ----
    i: 1
    i2: 11
    i: 2
    i2: 11
    i: 3
    i2: 11
    ---- break label ----
    i: 1
    i2: 11    
    ~~~

* 与C语言相同，在switch中使用break语句的效果是和C语言一致的，但是在select语句中使用break就不是一回事了

  * 实例：

    ~~~go
    package main
    
    import (
        "fmt"
        "time"
    )
    
    func main() {
        ch1 := make(chan int)
        ch2 := make(chan int)
    
        go func() {
            time.Sleep(2 * time.Second)
            ch1 <- 1
        }()
    
        go func() {
            time.Sleep(1 * time.Second)
            ch2 <- 2
        }()
    
        select {
        case <-ch1:
            fmt.Println("Received from ch1.")
        case <-ch2:
            fmt.Println("Received from ch2.")
            break // 跳出 select 语句
        }
    }
    ~~~

    输出结果：

    ~~~
    Received from ch2.
    ~~~

  * 在 Go 语言中，break 语句在 select 语句中的应用是相对特殊的。由于 select 语句的特性，**break 语句并不能直接用于跳出 select 语句本身，因为 select 语句是非阻塞的，它会一直等待所有的通信操作都准备就绪。**如果需要提前结束 select 语句的执行，可以使用 return 或者 goto 语句来达到相同的效果。

    > 如何理解：
    >
    > 1. **阻塞行为**：在没有任何 `case` 可以执行的情况下，`select` 语句会阻塞，直到其中一个通道的操作可以进行。因此，即使你在 `select` 语句中使用 `break`，如果没有任何 `case` 准备好，程序也不会执行 `break` 语句，依然会在 `select` 语句中阻塞等待。
    > 2. **非阻塞性**：如果 `select` 中的某一个 `case` 被触发（例如接收到消息），那么这个 `case` 将会被执行并且 `select` 会立即退出。这种行为并不受 `break` 的影响，因为 `break` 只是在 `select` 被触发后才有意义。
    >
    > 其实意思就是，即使触发了含有break语句的select case，也只是在本轮中退出了select语句，并没能起到彻底退出select语句的作用，更别说，包含break语句的case不一定会被触发了

  * 使用return语句来提前结束select的执行

    ~~~go
    package main
    
    import (
        "fmt"
        "time"
    )
    
    func process(ch chan int) {
        for {
            select {
            case val := <-ch:
                fmt.Println("Received value:", val)
                // 执行一些逻辑
                if val == 5 {
                    return // 提前结束 select 语句的执行
                }
            default:
                fmt.Println("No value received yet.")
                time.Sleep(500 * time.Millisecond)
            }
        }
    }
    
    func main() {
        ch := make(chan int)
    
        go process(ch)
    
        time.Sleep(2 * time.Second)
        ch <- 1
        time.Sleep(1 * time.Second)
        ch <- 3
        time.Sleep(1 * time.Second)
        ch <- 5
        time.Sleep(1 * time.Second)
        ch <- 7
    
        time.Sleep(2 * time.Second)
    }
    ~~~

    输出结果：

    ~~~
    No value received yet.
    No value received yet.
    Received value: 1
    No value received yet.
    Received value: 3
    No value received yet.
    Received value: 5
    ~~~

    

### continue

* 与go中的break一致，可以使用标记label出想要continue的循环

  * 实例：

    ~~~go
    package main
    
    import "fmt"
    
    func main() {
    
        // 不使用标记
        fmt.Println("---- continue ---- ")
        for i := 1; i <= 3; i++ {
            fmt.Printf("i: %d\n", i)
                for i2 := 11; i2 <= 13; i2++ {
                    fmt.Printf("i2: %d\n", i2)
                    continue
                }
        }
    
        // 使用标记
        fmt.Println("---- continue label ----")
        re:
            for i := 1; i <= 3; i++ {
                fmt.Printf("i: %d\n", i)
                    for i2 := 11; i2 <= 13; i2++ {
                        fmt.Printf("i2: %d\n", i2)
                        continue re
                    }
            }
    }
    ~~~

    输出结果：

    ~~~go
    ---- continue ---- 
    i: 1
    i2: 11
    i2: 12
    i2: 13
    i: 2
    i2: 11
    i2: 12
    i2: 13
    i: 3
    i2: 11
    i2: 12
    i2: 13
    ---- continue label ----
    i: 1
    i2: 11
    i: 2
    i2: 11
    i: 3
    i2: 11
    ~~~

### goto

* 与C语言中的goto用法一致

  * 实例：

    ~~~go
    package main
    
    import "fmt"
    
    func main() {
       /* 定义局部变量 */
       var a int = 10
    
       /* 循环 */
       LOOP: for a < 20 {
          if a == 15 {
             /* 跳过迭代 */
             a = a + 1
             goto LOOP
          }
          fmt.Printf("a的值为 : %d\n", a)
          a++     
       }  
    }
    ~~~

    输出结果：

    ~~~g
    a的值为 : 10
    a的值为 : 11
    a的值为 : 12
    a的值为 : 13
    a的值为 : 14
    a的值为 : 16
    a的值为 : 17
    a的值为 : 18
    a的值为 : 19
    ~~~



# Go语言函数

## 函数定义

Go 语言函数定义格式如下：

```
func function_name( [parameter list] ) [return_types] {
   函数体
}
```

函数定义解析：

- func：函数由 func 开始声明

- function_name：函数名称，参数列表和返回值类型构成了函数签名。

- parameter list：参数列表，参数就像一个占位符，当函数被调用时，你可以将值传递给参数，这个值被称为实际参数。参数列表指定的是参数类型、顺序、及参数个数。参数是可选的，也就是说函数也可以不包含参数。

- return_types：返回类型，函数返回一列值。return_types 是该列值的数据类型。有些功能不需要返回值，这种情况下 return_types 不是必须的。

- 函数体：函数定义的代码集合。

- 实例：

  ~~~go
  /* 函数返回两个数的最大值 */
  func max(num1, num2 int) int {
     /* 声明局部变量 */
     var result int
  
     if (num1 > num2) {
        result = num1
     } else {
        result = num2
     }
     return result 
  }
  ~~~

## 函数返回多个值

* 实例：

  ~~~go
  package main
  
  import "fmt"
  
  func swap(x, y string) (string, string) {
     return y, x
  }
  
  func main() {
     a, b := swap("Google", "Runoob")
     fmt.Println(a, b)
  }
  ~~~

  结果：

  ~~~
  Runoob Google
  ~~~

## 函数参数

### 引用传递

* go的值传递和C没什么不同，go的引用传递其实就是C的地址传递，和C的指针的利用很像

  * 实例：

    ~~~go
    package main
    
    import "fmt"
    
    func main() {
       /* 定义局部变量 */
       var a int = 100
       var b int= 200
    
       fmt.Printf("交换前，a 的值 : %d\n", a )
       fmt.Printf("交换前，b 的值 : %d\n", b )
    
       /* 调用 swap() 函数
       * &a 指向 a 指针，a 变量的地址
       * &b 指向 b 指针，b 变量的地址
       */
       swap(&a, &b)
    
       fmt.Printf("交换后，a 的值 : %d\n", a )
       fmt.Printf("交换后，b 的值 : %d\n", b )
    }
    
    func swap(x *int, y *int) {
       var temp int
       temp = *x    /* 保存 x 地址上的值 */
       *x = *y      /* 将 y 值赋给 x */
       *y = temp    /* 将 temp 值赋给 y */
    }
    ~~~

    输出结果：

    ~~~
    交换前，a 的值 : 100
    交换前，b 的值 : 200
    交换后，a 的值 : 200
    交换后，b 的值 : 100
    ~~~

## 函数用法

### 函数作为另外一个函数的参数

* 并不是函数地址作为参数那种的

  * 实例：

    ~~~go
    package main
    
    import (
       "fmt"
       "math"
    )
    
    func main(){
       /* 声明函数变量 */
       getSquareRoot := func(x float64) float64 {
          return math.Sqrt(x)
       }
    
       /* 使用函数 */
       fmt.Println(getSquareRoot(9))
    
    }
    ~~~

    输出结果：

    ~~~
    3
    ~~~

* 将函数地址作为参数传递

  * 实例：

    ~~~go
    package main
    import "fmt"
    
    // 声明一个函数类型
    type cb func(int) int
    
    func main() {
        testCallBack(1, callBack)
        testCallBack(2, func(x int) int {
            fmt.Printf("我是回调，x：%d\n", x)
            return x
        })
    }
    
    func testCallBack(x int, f func(x int) int) {
        f(x)
    }
    
    func callBack(x int) int {
        fmt.Printf("我是回调，x：%d\n", x)
        return x
    }
    ~~~

    需要注意，**当函数地址作为参数传递时，参数类型要和被传入的函数参数类型、返回值类型一致，并且写明白**

* 匿名函数

  * 实例：

    ~~~go
    package main
    
    import "fmt"
    
    var (
        myRes = func (a int, b int) int {
            return a - b
        }
    )
    
    
    func main(){
        //匿名函数，只调用一次，定义时直接调用
        res1 := func (a int, b int) int {
            return a + b
        }(10,25)	//给匿名函数传参
        fmt.Printf("res1 =%d\n", res1)
        
        //匿名函数赋给变量用变量来调用,可多次使用，但作用域有限
        res2 := func (a int, b int) int {
            return a * b
        }
        res3 := res2(10,25)
        fmt.Printf("res3 =%d\n", res3)
    
        //将匿名函数用全局变量接收，则该函数为全局匿名函数
        res4 := myRes(10,25)
        fmt.Printf("res4 =%d\n", res4)
        
    }
    ~~~

    匿名函数，其实有点点内联函数的意思

### 闭包

* 废话不多说，看实例

  ~~~go
  package main
  
  import "fmt"
  
  func getSequence() func() int {
     i := 0
     return func() int {
        i += 1
        return i
     }
  }
  
  func main() {
     /* nextNumber 为一个函数，函数 i 为 0 */
     nextNumber := getSequence()
  
     /* 调用 nextNumber 函数，i 变量自增 1 并返回 */
     fmt.Println(nextNumber()) //这个执行结果是1
     fmt.Println(nextNumber()) //这个执行结果是2
     fmt.Println(nextNumber()) //这个执行结果是3
  
     /* 创建新的函数 nextNumber1，并查看结果 */
     nextNumber1 := getSequence() //当getSequence()被重新赋值之后，nextNumber的值应该销毁丢失的，但并没有
     fmt.Println(nextNumber1()) //这儿因为是新赋值的，所以是1
     fmt.Println(nextNumber()) //这一行代码是补充上例子的。这儿可不是新赋的值，重点说明这一个，这儿执行居然是4，这个值并没有被销毁，原因就是闭包导致的，尽管外面的函数销毁了，但是内部函数仍然存在，还可以继续走。这个就是闭包
     fmt.Println(nextNumber1()) //新赋值的，继续执行是2
  }
  ~~~

  输出结果：

  ~~~
  1
  2
  3
  1
  4
  2
  ~~~

  可见，`getSequence()`函数的返回值是一个返回值为int类型的匿名函数，注意学习创建匿名函数的过程，即`nextNumber := getSequence()`，可以发现，**创建匿名函数的时候，其中的临时变量i也被保存，并没有被释放，这就是函数闭包，所有回调函数都会存在的情况**

* 如何将匿名函数赋值给变量、在函数内部使用匿名函数、将匿名函数作为参数传递

  ~~~go
  package main
  
  import "fmt"
  
  func main() {
      // 定义一个匿名函数并将其赋值给变量add
      add := func(a, b int) int {
          return a + b
      }
  
      // 调用匿名函数
      result := add(3, 5)
      fmt.Println("3 + 5 =", result)
  
      // 在函数内部使用匿名函数
      multiply := func(x, y int) int {
          return x * y
      }
  
      product := multiply(4, 6)
      fmt.Println("4 * 6 =", product)
  
      // 将匿名函数作为参数传递给其他函数
      calculate := func(operation func(int, int) int, x, y int) int {
          return operation(x, y)
      }
  
      sum := calculate(add, 2, 8)
      fmt.Println("2 + 8 =", sum)
  
      // 也可以直接在函数调用中定义匿名函数
      difference := calculate(func(a, b int) int {
          return a - b
      }, 10, 4)
      fmt.Println("10 - 4 =", difference)
  }
  ~~~

* 闭包带参数

  * 实例1：

    ~~~go
    package main
    
    import "fmt"
    func main() {
        add_func := add(1,2)
        fmt.Println(add_func())
        fmt.Println(add_func())
        fmt.Println(add_func())
    }
    
    // 闭包使用方法
    func add(x1, x2 int) func()(int,int)  {
        i := 0
        return func() (int,int){
            i++
            return i,x1+x2
        }
    }
    ~~~

    输出

    ~~~
    1 3
    2 3
    3 3
    ~~~
  
    
  
  * 实例2：
  
    ~~~go
    package main
    import "fmt"
    func main() {
        add_func := add(1,2)
        fmt.Println(add_func(1,1))
        fmt.Println(add_func(0,0))
        fmt.Println(add_func(2,2))
    } 
    // 闭包使用方法
    func add(x1, x2 int) func(x3 int,x4 int)(int,int,int)  {
        i := 0
        return func(x3 int,x4 int) (int,int,int){ 
           i++
           return i,x1+x2,x3+x4
        }
    }
    ~~~
    
    输出
    
    ~~~
    1 3 2
    2 3 0
    3 3 4
    ~~~
    
    

### 方法

* Go 语言中同时有函数和方法。一个方法就是一个包含了接受者的函数，接受者可以是命名类型或者结构体类型的一个值或者是一个指针。所有给定类型的方法属于该类型的方法集。语法格式如下：

  ```
  func (variable_name variable_data_type) function_name() [return_type]{
     /* 函数体*/
  }
  ```

  * 实例

    ~~~go
    package main
    
    import (
       "fmt"  
    )
    
    /* 定义结构体 */
    type Circle struct {
      radius float64
    }
    
    func main() {
      var c1 Circle
      c1.radius = 10.00
      fmt.Println("圆的面积 = ", c1.getArea())
    }
    
    //该 method 属于 Circle 类型对象中的方法
    func (c Circle) getArea() float64 {
      //c.radius 即为 Circle 类型对象中的属性
      return 3.14 * c.radius * c.radius
    }
    ~~~

    可以发现，定义的method与function不一样的地方在于，其c Circle并不是参数，而是将该方法绑定到对应的类上，实现该类可以使用“.”的方式调用该方法

  * Go 没有面向对象，而我们知道常见的 Java。

    C++ 等语言中，**实现类的方法做法都是编译器隐式的给函数加一个 this 指针，而在 Go 里，这个 this 指针需要明确的申明出来，其实和其它 OO 语言并没有很大的区别。**

    在C++中是这样的

    ```C++
    class Circle {
      public:
        float getArea() {
           return 3.14 * radius * radius;
        }
      private:
        float radius;
    }
    
    // 其中 getArea 经过编译器处理大致变为
    float getArea(Circle *const c) {
      ...
    }
    ```

    在 Go 中则是如下:

    ```go
    func (c Circle) getArea() float64 {
      //c.radius 即为 Circle 类型对象中的属性
      return 3.14 * c.radius * c.radius
    }
    ```

  * 关于值和指针，如果想在方法中改变结构体类型的属性，需要对方法传递指针，体会如下对结构体类型改变的方法 changRadis() 和普通的函数 change() 中的指针操作:

    ~~~go
    package main
    
    import (
       "fmt"  
    )
    
    /* 定义结构体 */
    type Circle struct {
      radius float64
    }
    
    
    func main()  { 
       var c Circle
       fmt.Println(c.radius)
       c.radius = 10.00
       fmt.Println(c.getArea())
       c.changeRadius(20)
       fmt.Println(c.radius)
       change(&c, 30)
       fmt.Println(c.radius)
    }
    func (c Circle) getArea() float64  {
       return c.radius * c.radius
    }
    // 注意如果想要更改成功c的值，这里需要传指针
    func (c *Circle) changeRadius(radius float64)  {
       c.radius = radius
    }
    
    // 以下操作将不生效
    //func (c Circle) changeRadius(radius float64)  {
    //   c.radius = radius
    //}
    // 引用类型要想改变值需要传指针
    func change(c *Circle, radius float64)  {
       c.radius = radius
    }
    ~~~

    其输出为：

    ~~~
    0
    100
    20
    30
    ~~~



# Go语言变量作用域

* Go语言中，全局变量和局部变量名称可以相同，但是函数内部的局部变量会被优先考虑

  ~~~go
  package main
  
  import "fmt"
  
  /* 声明全局变量 */
  var g int = 20
  
  func main() {
     /* 声明局部变量 */
     var g int = 10
  
     fmt.Printf ("结果： g = %d\n",  g)
  }
  ~~~

  输出为10



# Go语言数组

> 建议了解Slice后再看

## 声明数组

Go 语言数组声明需要指定元素类型及元素个数，语法格式如下：

```
var arrayName [size]dataType
```

其中，**arrayName** 是数组的名称，**size** 是数组的大小，**dataType** 是数组中元素的数据类型。

以下定义了数组 balance 长度为 10 类型为 float32：

```
var balance [10]float32
```

## 初始化数组

 以下演示了数组初始化：

以下实例声明一个名为 numbers 的整数数组，其大小为 5，在声明时，数组中的每个元素都会根据其数据类型进行默认初始化，对于整数类型，初始值为 0。

```
var numbers [5]int
```

还可以使用初始化列表来初始化数组的元素：

```
var numbers = [5]int{1, 2, 3, 4, 5}
```

以上代码声明一个大小为 5 的整数数组，并将其中的元素分别初始化为 1、2、3、4 和 5。

另外，还可以使用 **:=** 简短声明语法来声明和初始化数组：

```
numbers := [5]int{1, 2, 3, 4, 5}
```

以上代码创建一个名为 numbers 的整数数组，并将其大小设置为 5，并初始化元素的值。

**注意：在 Go 语言中，数组的大小是类型的一部分，因此不同大小的数组是不兼容的，也就是说 [5]int 和 [10]int 是不同的类型。**

以下定义了数组 balance 长度为 5 类型为 float32，并初始化数组的元素：

```
var balance = [5]float32{1000.0, 2.0, 3.4, 7.0, 50.0}
```

我们也可以通过字面量在声明数组的同时快速初始化数组：

```
balance := [5]float32{1000.0, 2.0, 3.4, 7.0, 50.0}
```

**如果数组长度不确定，可以使用 ... 代替数组的长度，编译器会根据元素个数自行推断数组的长度**：

```
var balance = [...]float32{1000.0, 2.0, 3.4, 7.0, 50.0}
或
balance := [...]float32{1000.0, 2.0, 3.4, 7.0, 50.0}
```

**如果设置了数组的长度，我们还可以通过指定下标来初始化元素：**

```
//  将索引为 1 和 3 的元素初始化
balance := [5]float32{1:2.0,3:7.0}
```

初始化数组中 **{}** 中的元素个数不能大于 **[]** 中的数字。

**如果忽略 [] 中的数字不设置数组大小，Go 语言会根据元素的个数来设置数组的大小：**

```
 balance[4] = 50.0
```

以上实例读取了第五个元素。数组元素可以通过索引（位置）来读取（或者修改），索引从 0 开始，第一个元素索引为 0，第二个索引为 1，以此类推。

> `arr := []int{1: 2, 3: 0}`如此类定义，数组长度为4

## 多维数组

* 声明

  Go 语言支持多维数组，以下为常用的多维数组声明方式：

  ```
  var variable_name [SIZE1][SIZE2]...[SIZEN] variable_type
  ```

  以下实例声明了三维的整型数组：

  ```
  var threedim [5][10][4]int
  ```

* 初始化多维数组

  多维数组可通过大括号来初始值。以下实例为一个 3 行 4 列的二维数组：

  ```
  a := [3][4]int{  
   {0, 1, 2, 3} ,   /*  第一行索引为 0 */
   {4, 5, 6, 7} ,   /*  第二行索引为 1 */
   {8, 9, 10, 11},   /* 第三行索引为 2 */
  }
  ```

  > 注意：
  >
  > 以上代码中倒数第二行的 } 必须要有逗号，因为最后一行的 } 
  >
  > 不能单独一行，也可以写成这样：
  >
  > ```
  > a := [3][4]int{  
  >  {0, 1, 2, 3} ,   /*  第一行索引为 0 */
  >  {4, 5, 6, 7} ,   /*  第二行索引为 1 */
  >  {8, 9, 10, 11}}   /* 第三行索引为 2 */
  > ```

  以下实例初始化一个 2 行 2 列 的二维数组：

  ~~~go
  package main
  
  import "fmt"
  
  func main() {
      // 创建二维数组
      sites := [2][2]string{}
  
      // 向二维数组添加元素
      sites[0][0] = "Google"
      sites[0][1] = "Runoob"
      sites[1][0] = "Taobao"
      sites[1][1] = "Weibo"
  
      // 显示结果
      fmt.Println(sites)
  }
  ~~~

* 创建各个维度元素数量不一的二维数组

  ~~~go
  package main
  
  import "fmt"
  
  func main() {
      // 创建空的二维数组
      animals := [][]string{}
  
      // 创建三一维数组，各数组长度不同
      row1 := []string{"fish", "shark", "eel"}
      row2 := []string{"bird"}
      row3 := []string{"lizard", "salamander"}
  
      // 使用 append() 函数将一维数组添加到二维数组中
      animals = append(animals, row1)
      animals = append(animals, row2)
      animals = append(animals, row3)
  
      // 循环输出
      for i := range animals {
          fmt.Printf("Row: %v\n", i)
          fmt.Println(animals[i])
      }
  }
  ~~~

  此处代码中涉及的`[]string{}`和`[...]string{}`的区别，涉及到了切片slice的概念，见slice部分

## 向函数传递数组

* 方式1：

  形参设定数组大小：

  ```
  func myFunction(param [10]int) {
      ....
  }
  ```

* 方式2：

  形参未设定数组大小：

  ```
  func myFunction(param []int) {
      ....
  }
  ```

  > 未定义长度的数组只能传给不限制数组长度的函数，定义了长度的数组只能传给限制了相同数组长度的函数

  如果你想要在函数内修改原始数组，可以通过传递数组的指针来实现。

  ~~~go
  package main
  
  import "fmt"
  
  // 函数接受一个数组作为参数
  func modifyArray(arr [5]int) {
      for i := 0; i < len(arr); i++ {
          arr[i] = arr[i] * 2
      }
  }
  
  // 函数接受一个数组的指针作为参数
  func modifyArrayWithPointer(arr *[5]int) {
      for i := 0; i < len(*arr); i++ {
          (*arr)[i] = (*arr)[i] * 2
      }
  }
  
  func main() {
      // 创建一个包含5个元素的整数数组
      myArray := [5]int{1, 2, 3, 4, 5}
  
      fmt.Println("Original Array:", myArray)
  
      // 传递数组给函数，但不会修改原始数组的值
      modifyArray(myArray)
      fmt.Println("Array after modifyArray:", myArray)
  
      // 传递数组的指针给函数，可以修改原始数组的值
      modifyArrayWithPointer(&myArray)
      fmt.Println("Array after modifyArrayWithPointer:", myArray)
  }
  ~~~

  > 关于指针传递两种方式访问数组：
  >
  > * `(*nums)[0] = 5` 的解释
  >
  >   `(*nums)[0] = 5` 是一种显式的操作方式。`*nums` 表示对指针 `nums` 进行解引用，得到指针所指向的数组，然后通过 `[0]` 访问数组的第一个元素并将其赋值为 5。这是一种符合指针操作逻辑的标准写法，它明确地表达了先获取指针指向的数组，再访问数组元素的过程。
  >
  > * `nums[0] = 5` 的解释
  >
  >   `nums[0] = 5` 是 Go 语言提供的语法糖。在 Go 中，当你使用指针类型的数组时，可以直接通过指针来访问数组元素，而无需显式地进行解引用操作。Go 语言会自动处理指针的解引用，将 `nums[0]` 转换为 `(*nums)[0]`。这种语法糖的存在是为了让代码更加简洁易读。

* 数组与切片的不同

  \- Go 语言的数组是值，其长度是其类型的一部分，作为函数参数时，是 **值传递**，函数中的修改对调用者不可见

  \- Go 语言中对数组的处理，一般采用 **切片** 的方式，切片包含对底层数组内容的引用，作为函数参数时，类似于 **指针传递**，函数中的修改对调用者可见

  ~~~go
  package main
  import "fmt"
  // Go 语言的数组是值，其长度是其类型的一部分，作为函数参数时，是 值传递，函数中的修改对调用者不可见
  func change1(nums [3]int) {    
      nums[0] = 4
  }
  // 传递进来数组的内存地址，然后定义指针变量指向该地址，则会改变数组的值
  func change2(nums *[3]int) {    
      nums[0] = 5
  }
  // Go 语言中对数组的处理，一般采用 切片 的方式，切片包含对底层数组内容的引用，作为函数参数时，类似于 指针传递，函数中的修改对调用者可见
  func change3(nums []int) {    
      nums[0] = 6
  }
  func main() {    
      var nums1 = [3]int{1, 2, 3}   
      var nums2 = []int{1, 2, 3}    
      change1(nums1)    
      fmt.Println(nums1)  //  [1 2 3]     
      change2(&nums1)    
      fmt.Println(nums1)  //  [5 2 3]    
      change3(nums2)    
      fmt.Println(nums2)  //  [6 2 3]
  }
  ~~~

# Go语言指针

* 指针的声明

  指针声明格式如下：

  ```
  var var_name *var-type
  ```

  var-type 为指针类型，var_name 为指针变量名，* 号用于指定变量是作为一个指针。以下是有效的指针声明：

  ```
  var ip *int        /* 指向整型*/
  var fp *float32    /* 指向浮点型 */
  ```

  本例中这是一个指向 int 和 float32 的指针。

* 空指针

  当一个指针被定义后没有分配到任何变量时，它的值为 nil。

  nil 指针也称为空指针。

  nil在概念上和其它语言的null、None、nil、NULL一样，都指代零值或空值。

  一个指针变量通常缩写为 ptr。

  查看以下实例：

  ~~~go
  package main
  
  import "fmt"
  
  func main() {
     var  ptr *int
  
     fmt.Printf("ptr 的值为 : %x\n", ptr  )
  }
  ~~~

  输出结果为0
  
## GO语言指针数组

  在我们了解指针数组前，先看个实例，定义了长度为 3 的整型数组：

  ~~~go
  package main
  
  import "fmt"
  
  const MAX int = 3
  
  func main() {
     a := []int{10,100,200}
     var i int
     var ptr [MAX]*int;
  
     for  i = 0; i < MAX; i++ {
        ptr[i] = &a[i] /* 整数地址赋值给指针数组 */
     }
  
     for  i = 0; i < MAX; i++ {
        fmt.Printf("a[%d] = %d\n", i,*ptr[i] )
     }
  }
  ~~~

  > 创建指针数组的时候，不适合用 **range** 循环
  >
  > ~~~go
  > const max = 3
  > 
  > func main() {
  >     number := [max]int{5, 6, 7}
  >     var ptrs [max]*int //指针数组
  >     //将number数组的值的地址赋给ptrs
  >     for i, x := range &number {
  >         ptrs[i] = &x
  >     }
  >     for i, x := range ptrs {
  >         fmt.Printf("指针数组：索引:%d 值:%d 值的内存地址:%d\n", i, *x, x)
  >     }
  > }
  > ~~~
  >
  > 输出
  >
  > ~~~go
  > 指针数组：索引:0 值:7 值的内存地址:824634204304
  > 指针数组：索引:1 值:7 值的内存地址:824634204304
  > 指针数组：索引:2 值:7 值的内存地址:824634204304
  > ~~~
  >
  > 从结果中我们发现内存地址都一样，而且值也一样。怎么回事？
  >
  > 这个问题是range循环的实现逻辑引起的。跟for循环不一样的地方在于range循环中的x变量是临时变量。range循环只是将值拷贝到x变量中。因此内存地址都是一样的。**x临时变量仅被声明一次，此后都是将迭代 number 出的值赋值给 x ， x 变量的内存地址始终未变，这样再将 x 的地址发送给 ptrs 数组，自然也是相同的。**

## Go语言指向指针的指针

指向指针的指针变量声明格式如下：

```
var ptr **int;
```

用法和C的二层指针一样

## GO语言指针作为函数参数

~~~go
package main

import "fmt"

func main() {
   /* 定义局部变量 */
   var a int = 100
   var b int= 200

   fmt.Printf("交换前 a 的值 : %d\n", a )
   fmt.Printf("交换前 b 的值 : %d\n", b )

   /* 调用函数用于交换值
   * &a 指向 a 变量的地址
   * &b 指向 b 变量的地址
   */
   swap(&a, &b);

   fmt.Printf("交换后 a 的值 : %d\n", a )
   fmt.Printf("交换后 b 的值 : %d\n", b )
}

func swap(x *int, y *int) {
   var temp int
   temp = *x    /* 保存 x 地址的值 */
   *x = *y      /* 将 y 赋值给 x */
   *y = temp    /* 将 temp 赋值给 y */
}
~~~

# Go语言结构体

## 定义结构体

结构体定义需要使用 type 和 struct 语句。struct 语句定义一个新的数据类型，结构体中有一个或多个成员。type 语句设定了结构体的名称。结构体的格式如下：

```
type struct_variable_type struct {
   member definition
   member definition
   ...
   member definition
}
```

一旦定义了结构体类型，它就能用于变量的声明，语法格式如下：

```
variable_name := structure_variable_type {value1, value2...valuen}
或
variable_name := structure_variable_type { key1: value1, key2: value2..., keyn: valuen}
```

~~~go
package main

import "fmt"

type Books struct {
   title string
   author string
   subject string
   book_id int
}


func main() {

    // 创建一个新的结构体
    fmt.Println(Books{"Go 语言", "www.runoob.com", "Go 语言教程", 6495407})

    // 也可以使用 key => value 格式
    fmt.Println(Books{title: "Go 语言", author: "www.runoob.com", subject: "Go 语言教程", book_id: 6495407})

    // 忽略的字段为 0 或 空
   fmt.Println(Books{title: "Go 语言", author: "www.runoob.com"})
}
~~~

## 访问结构体成员

~~~go
package main

import "fmt"

type Books struct {
   title string
   author string
   subject string
   book_id int
}

func main() {
   var Book1 Books        /* 声明 Book1 为 Books 类型 */
   var Book2 Books        /* 声明 Book2 为 Books 类型 */

   /* book 1 描述 */
   Book1.title = "Go 语言"
   Book1.author = "www.runoob.com"
   Book1.subject = "Go 语言教程"
   Book1.book_id = 6495407

   /* book 2 描述 */
   Book2.title = "Python 教程"
   Book2.author = "www.runoob.com"
   Book2.subject = "Python 语言教程"
   Book2.book_id = 6495700

   /* 打印 Book1 信息 */
   fmt.Printf( "Book 1 title : %s\n", Book1.title)
   fmt.Printf( "Book 1 author : %s\n", Book1.author)
   fmt.Printf( "Book 1 subject : %s\n", Book1.subject)
   fmt.Printf( "Book 1 book_id : %d\n", Book1.book_id)

   /* 打印 Book2 信息 */
   fmt.Printf( "Book 2 title : %s\n", Book2.title)
   fmt.Printf( "Book 2 author : %s\n", Book2.author)
   fmt.Printf( "Book 2 subject : %s\n", Book2.subject)
   fmt.Printf( "Book 2 book_id : %d\n", Book2.book_id)
}
~~~

## 结构体作为函数参数

~~~go
package main

import "fmt"

type Books struct {
   title string
   author string
   subject string
   book_id int
}

func main() {
   var Book1 Books        /* 声明 Book1 为 Books 类型 */
   var Book2 Books        /* 声明 Book2 为 Books 类型 */

   /* book 1 描述 */
   Book1.title = "Go 语言"
   Book1.author = "www.runoob.com"
   Book1.subject = "Go 语言教程"
   Book1.book_id = 6495407

   /* book 2 描述 */
   Book2.title = "Python 教程"
   Book2.author = "www.runoob.com"
   Book2.subject = "Python 语言教程"
   Book2.book_id = 6495700

   /* 打印 Book1 信息 */
   printBook(Book1)

   /* 打印 Book2 信息 */
   printBook(Book2)
}

func printBook( book Books ) {
   fmt.Printf( "Book title : %s\n", book.title)
   fmt.Printf( "Book author : %s\n", book.author)
   fmt.Printf( "Book subject : %s\n", book.subject)
   fmt.Printf( "Book book_id : %d\n", book.book_id)
}
~~~



# Go语言切片

> Go 语言切片是对数组的抽象。
>
> Go 数组的长度不可改变，在特定场景中这样的集合就不太适用，Go 中提供了一种灵活，功能强悍的内置类型切片("动态数组")，与数组相比切片的长度是不固定的，可以追加元素，在追加时可能使切片的容量增大。

## 定义切片

你可以声明一个未指定大小的数组来定义切片：

```go
var identifier []type
```

切片不需要说明长度。

或使用 **make()** 函数来创建切片:

```go
var slice1 []type = make([]type, len)

//也可以简写为

slice1 := make([]type, len)
```

也可以指定容量，其中 **capacity** 为可选参数。

```
make([]T, length, capacity)
```

这里 len 是数组的长度并且也是切片的初始长度，capacity表示底层数组的长度

## 切片初始化

```
s :=[] int {1,2,3 } 
```

直接初始化切片，**[]** 表示是切片类型，**{1,2,3}** 初始化值依次是 **1,2,3**，其 **cap=len=3**。

```
s := arr[:] 
```

初始化切片 **s**，是数组 arr 的引用。

```
s := arr[startIndex:endIndex] 
```

将 arr 中从下标 startIndex 到 endIndex-1 下的元素创建为一个新的切片。

```
s := arr[startIndex:] 
```

默认 endIndex 时将表示一直到arr的最后一个元素。

```
s := arr[:endIndex] 
```

默认 startIndex 时将表示从 arr 的第一个元素开始。

```
s1 := s[startIndex:endIndex] 
```

通过切片 s 初始化切片 s1。

```
s :=make([]int,len,cap) 
```

通过内置函数 **make()** 初始化切片**s**，**[]int** 标识为其元素类型为 int 的切片。

## len()和cap()函数

~~~ go
package main

import "fmt"

func main() {
   var numbers = make([]int,3,5)

   printSlice(numbers)
}

func printSlice(x []int){
   fmt.Printf("len=%d cap=%d slice=%v\n",len(x),cap(x),x)
}
~~~

运行输出

~~~
len=3 cap=5 slice=[0 0 0]
~~~

## 空(nil)切片

一个切片在未初始化之前默认为 nil，长度为 0，实例如下：

~~~go
package main

import "fmt"

func main() {
   var numbers []int

   printSlice(numbers)

   if(numbers == nil){
      fmt.Printf("切片是空的")
   }
}

func printSlice(x []int){
   fmt.Printf("len=%d cap=%d slice=%v\n",len(x),cap(x),x)
}
~~~

输出

~~~go
len=0 cap=0 slice=[]
切片是空的
~~~

## 切片截取

可以通过设置下限及上限来设置截取切片 *[lower-bound:upper-bound]*，实例如下：

~~~go
package main

import "fmt"

func main() {
   /* 创建切片 */
   numbers := []int{0,1,2,3,4,5,6,7,8}   
   printSlice(numbers)

   /* 打印原始切片 */
   fmt.Println("numbers ==", numbers)

   /* 打印子切片从索引1(包含) 到索引4(不包含)*/
   fmt.Println("numbers[1:4] ==", numbers[1:4])

   /* 默认下限为 0*/
   fmt.Println("numbers[:3] ==", numbers[:3])

   /* 默认上限为 len(s)*/
   fmt.Println("numbers[4:] ==", numbers[4:])

   numbers1 := make([]int,0,5)
   printSlice(numbers1)

   /* 打印子切片从索引  0(包含) 到索引 2(不包含) */
   number2 := numbers[:2]
   printSlice(number2)

   /* 打印子切片从索引 2(包含) 到索引 5(不包含) */
   number3 := numbers[2:5]
   printSlice(number3)

}

func printSlice(x []int){
   fmt.Printf("len=%d cap=%d slice=%v\n",len(x),cap(x),x)
}
~~~

输出

~~~go
len=9 cap=9 slice=[0 1 2 3 4 5 6 7 8]
numbers == [0 1 2 3 4 5 6 7 8]
numbers[1:4] == [1 2 3]
numbers[:3] == [0 1 2]
numbers[4:] == [4 5 6 7 8]
len=0 cap=5 slice=[]
len=2 cap=9 slice=[0 1]
len=3 cap=7 slice=[2 3 4]
~~~

## append和copy函数

如果想增加切片的容量，我们必须创建一个新的更大的切片并把原分片的内容都拷贝过来。

下面的代码描述了从拷贝切片的 copy 方法和向切片追加新元素的 append 方法。

~~~go
package main

import "fmt"

func main() {
   var numbers []int
   printSlice(numbers)

   /* 允许追加空切片 */
   numbers = append(numbers, 0)
   printSlice(numbers)

   /* 向切片添加一个元素 */
   numbers = append(numbers, 1)
   printSlice(numbers)

   /* 同时添加多个元素 */
   numbers = append(numbers, 2,3,4)
   printSlice(numbers)

   /* 创建切片 numbers1 是之前切片的两倍容量*/
   numbers1 := make([]int, len(numbers), (cap(numbers))*2)

   /* 拷贝 numbers 的内容到 numbers1 */
   copy(numbers1,numbers)
   printSlice(numbers1)   
}

func printSlice(x []int){
   fmt.Printf("len=%d cap=%d slice=%v\n",len(x),cap(x),x)
}
~~~

输出

~~~go
len=0 cap=0 slice=[]
len=1 cap=1 slice=[0]
len=2 cap=2 slice=[0 1]
len=5 cap=6 slice=[0 1 2 3 4]
len=5 cap=12 slice=[0 1 2 3 4]
~~~

# Go语言Range

Go 语言中 range 关键字用于 for 循环中迭代数组(array)、切片(slice)、通道(channel)或集合(map)的元素。在数组和切片中它返回元素的索引和索引对应的值，在集合中返回 key-value 对。

for 循环的 range 格式可以对 slice、map、数组、字符串等进行迭代循环。格式如下：

```
for key, value := range oldMap {
    newMap[key] = value
}
```

以上代码中的 key 和 value 是可以省略。

如果只想读取 key，格式如下：

```
for key := range oldMap
```

或者这样：

for key, _ := range oldMap

如果只想读取 value，格式如下：

```
for _, value := range oldMap
```

## 数组和切片

~~~go
package main

import "fmt"

// 声明一个包含 2 的幂次方的切片
var pow = []int{1, 2, 4, 8, 16, 32, 64, 128}

func main() {
   // 遍历 pow 切片，i 是索引，v 是值
   for i, v := range pow {
      // 打印 2 的 i 次方等于 v
      fmt.Printf("2**%d = %d\n", i, v)
   }
}
~~~

## 字符串

range 迭代字符串时，返回每个字符的索引和 Unicode 代码点（rune）。

~~~go
package main

import "fmt"

func main() {
    for i, c := range "hello" {
        fmt.Printf("index: %d, char: %c\n", i, c)
    }
    
    //range也可以用来枚举 Unicode 字符串。第一个参数是字符的索引，第二个是字符（Unicode的值）本身。
    for i, c := range "go" {
        fmt.Println(i, c)
    }
}

~~~

输出

~~~go
index: 0, char: h
index: 1, char: e
index: 2, char: l
index: 3, char: l
index: 4, char: o
0 103
1 111
~~~

## 映射

~~~go
package main

import "fmt"

func main() {
    // 创建一个空的 map，key 是 int 类型，value 是 float32 类型
    map1 := make(map[int]float32)
    
    // 向 map1 中添加 key-value 对
    map1[1] = 1.0
    map1[2] = 2.0
    map1[3] = 3.0
    map1[4] = 4.0
   
    // 遍历 map1，读取 key 和 value
    for key, value := range map1 {
        // 打印 key 和 value
        fmt.Printf("key is: %d - value is: %f\n", key, value)
    }

    // 遍历 map1，只读取 key
    for key := range map1 {
        // 打印 key
        fmt.Printf("key is: %d\n", key)
    }

    // 遍历 map1，只读取 value
    for _, value := range map1 {
        // 打印 value
        fmt.Printf("value is: %f\n", value)
    }
}
~~~

## 通道

range 遍历从通道接收的值，直到通道关闭。

~~~go
package main

import "fmt"

func main() {
    ch := make(chan int, 2)
    ch <- 1
    ch <- 2
    close(ch)
    
    for v := range ch {
        fmt.Println(v)
    }
}
~~~

输出

~~~go
1
2
~~~

## 忽略值

在遍历时可以使用 **_** 来忽略索引或值。

~~~go
package main

import "fmt"

func main() {
    nums := []int{2, 3, 4}
    
    // 忽略索引
    for _, num := range nums {
        fmt.Println("value:", num)
    }
    
    // 忽略值
    for i := range nums {
        fmt.Println("index:", i)
    }
}
~~~

## 主函数参数列表

~~~go
package main

import (
    "fmt"
    "os"
)

func main() {
    fmt.Println(len(os.Args))
    for _, arg := range os.Args {
        fmt.Println(arg)
    }
}
~~~

## 中文字符串

Go 中的中文采用 **UTF-8** 编码，因此逐个遍历字符时必须采用 **for-each** 形式：

~~~go
package main

import "fmt"

func main() {   
   printStr("hello")   
   fmt.Println()   
   fmt.Println()   
   printStr("中国人")
}

func printStr(s string) {   
   fmt.Println("str: " + s)   
   for _, v := range s {      
      fmt.Printf("0x%x %c, ", v, v)   
   }   
   fmt.Println()   
   for i := 0; i < len(s); i++ {      
      fmt.Printf("0x%x, ", s[i])   
   }
}
~~~

# Go语言Map

Map 是一种无序的键值对的集合。

Map 最重要的一点是通过 key 来快速检索数据，key 类似于索引，指向数据的值。

Map 是一种集合，所以我们可以像迭代数组和切片那样迭代它。不过，Map 是无序的，遍历 Map 时返回的键值对的顺序是不确定的。

在获取 Map 的值时，如果键不存在，返回该类型的零值，例如 int 类型的零值是 0，string 类型的零值是 ""。

Map 是引用类型，如果将一个 Map 传递给一个函数或赋值给另一个变量，它们都指向同一个底层数据结构，因此对 Map 的修改会影响到所有引用它的变量。

## 定义Map

~~~go
/* 使用 make 函数 */
map_variable := make(map[KeyType]ValueType, initialCapacity)
~~~

其中 KeyType 是键的类型，ValueType 是值的类型，initialCapacity 是可选的参数，用于指定 Map 的初始容量。Map 的容量是指 Map 中可以保存的键值对的数量，当 Map 中的键值对数量达到容量时，Map 会自动扩容。如果不指定 initialCapacity，Go 语言会根据实际情况选择一个合适的值。

~~~go
// 创建一个空的 Map
m := make(map[string]int)

// 创建一个初始容量为 10 的 Map
m := make(map[string]int, 10)
~~~

也可以使用字面量创建 Map：

```
// 使用字面量创建 Map
m := map[string]int{
    "apple": 1,
    "banana": 2,
    "orange": 3,
}
```

获取元素：

```
// 获取键值对
v1 := m["apple"]
v2, ok := m["pear"]  // 如果键不存在，ok 的值为 false，v2 的值为该类型的零值
```

修改元素：

```
// 修改键值对
m["apple"] = 5
```

获取 Map 的长度：

```
// 获取 Map 的长度
len := len(m)
```

遍历 Map：

```
// 遍历 Map
for k, v := range m {
    fmt.Printf("key=%s, value=%d\n", k, v)
}
```

删除元素：

```
// 删除键值对
delete(m, "banana")
```

~~~go
package main

import "fmt"

func main() {
    var siteMap map[string]string /*创建集合 */
    siteMap = make(map[string]string)

    /* map 插入 key - value 对,各个国家对应的首都 */
    siteMap [ "Google" ] = "谷歌"
    siteMap [ "Runoob" ] = "菜鸟教程"
    siteMap [ "Baidu" ] = "百度"
    siteMap [ "Wiki" ] = "维基百科"

    /*使用键输出地图值 */ 
    for site := range siteMap {
        fmt.Println(site, "首都是", siteMap [site])
    }

    /*查看元素在集合中是否存在 */
    name, ok := siteMap [ "Facebook" ] /*如果确定是真实的,则存在,否则不存在 */
    /*fmt.Println(capital) */
    /*fmt.Println(ok) */
    if (ok) {
        fmt.Println("Facebook 的 站点是", name)
    } else {
        fmt.Println("Facebook 站点不存在")
    }
}
~~~

输出

~~~go
Wiki 首都是 维基百科
Google 首都是 谷歌
Runoob 首都是 菜鸟教程
Baidu 首都是 百度
Facebook 站点不存在
~~~

## delete函数

delete() 函数用于删除集合的元素, 参数为 map 和其对应的 key。实例如下：

~~~go
package main

import "fmt"

func main() {
        /* 创建map */
        countryCapitalMap := map[string]string{"France": "Paris", "Italy": "Rome", "Japan": "Tokyo", "India": "New delhi"}

        fmt.Println("原始地图")

        /* 打印地图 */
        for country := range countryCapitalMap {
                fmt.Println(country, "首都是", countryCapitalMap [ country ])
        }

        /*删除元素*/ delete(countryCapitalMap, "France")
        fmt.Println("法国条目被删除")

        fmt.Println("删除元素后地图")

        /*打印地图*/
        for country := range countryCapitalMap {
                fmt.Println(country, "首都是", countryCapitalMap [ country ])
        }
}
~~~

##  基于go实现简答HashMap

~~~go
package main

import (
    "fmt"
)

type HashMap struct {
    key string
    value string
    hashCode int
    next *HashMap
}

var table [16](*HashMap)

func initTable() {
    for i := range table{
        table[i] = &HashMap{"","",i,nil}
    }
}

func getInstance() [16](*HashMap){
    if(table[0] == nil){
        initTable()
    }
    return table
}

func genHashCode(k string) int{
    if len(k) == 0{
        return 0
    }
    var hashCode int = 0
    var lastIndex int = len(k) - 1
    for i := range k {
        if i == lastIndex {
            hashCode += int(k[i])
            break
        }
        hashCode += (hashCode + int(k[i]))*31
    }
    return hashCode
}

func indexTable(hashCode int) int{
    return hashCode%16
}

func indexNode(hashCode int) int {
    return hashCode>>4
}

func put(k string, v string) string {
    var hashCode = genHashCode(k)
    var thisNode = HashMap{k,v,hashCode,nil}

    var tableIndex = indexTable(hashCode)
    var nodeIndex = indexNode(hashCode)

    var headPtr [16](*HashMap) = getInstance()
    var headNode = headPtr[tableIndex]

    if (*headNode).key == "" {
        *headNode = thisNode
        return ""
    }

    var lastNode *HashMap = headNode
    var nextNode *HashMap = (*headNode).next

    for nextNode != nil && (indexNode((*nextNode).hashCode) < nodeIndex){
        lastNode = nextNode
        nextNode = (*nextNode).next
    }
    if (*lastNode).hashCode == thisNode.hashCode {
        var oldValue string = lastNode.value
        lastNode.value = thisNode.value
        return oldValue
    }
    if lastNode.hashCode < thisNode.hashCode {
        lastNode.next = &thisNode
    }
    if nextNode != nil {
        thisNode.next = nextNode
    }
    return ""
}

func get(k string) string {
    var hashCode = genHashCode(k)
    var tableIndex = indexTable(hashCode)

    var headPtr [16](*HashMap) = getInstance()
    var node *HashMap = headPtr[tableIndex]

    if (*node).key == k{
        return (*node).value
    }

    for (*node).next != nil {
        if k == (*node).key {
            return (*node).value
        }
        node = (*node).next
    }
    return ""
}

//examples 
func main() {
    getInstance()
    put("a","a_put")
    put("b","b_put")
    fmt.Println(get("a"))
    fmt.Println(get("b"))
    put("p","p_put")
    fmt.Println(get("p"))
}
~~~

# Go语言递归函数

Go 语言支持递归。但我们在使用递归时，开发者需要设置退出条件，否则递归将陷入无限循环中。

递归函数对于解决数学上的问题是非常有用的，就像计算阶乘，生成斐波那契数列等。

~~~go
package main

import "fmt"

func Factorial(n uint64)(result uint64) {
    if (n > 0) {
        result = n * Factorial(n-1)
        return result
    }
    return 1
}

func main() {  
    var i int = 15
    fmt.Printf("%d 的阶乘是 %d\n", i, Factorial(uint64(i)))
}
~~~

# Go语言类型转换

类型转换用于将一种数据类型的变量转换为另外一种类型的变量。

Go 语言类型转换基本格式如下：

```
type_name(expression)
```

type_name 为类型，expression 为表达式。

## 数值类型转换

~~~go
var a int = 10
var b float64 = float64(a)
~~~

~~~go
package main

import "fmt"

func main() {
   var sum int = 17
   var count int = 5
   var mean float32
   
   mean = float32(sum)/float32(count)
   fmt.Printf("mean 的值为: %f\n",mean)
}
~~~

## 字符串类型转换

将一个字符串转换成另一个类型，可以使用以下语法：

```
var str string = "10"
var num int
num, _ = strconv.Atoi(str)
```

以上代码将字符串变量 str 转换为整型变量 num。

注意，**strconv.Atoi** 函数返回两个值，第一个是转换后的整型值，第二个是可能发生的错误，我们可以使用空白标识符 **_** 来忽略这个错。

以下实例将字符串转换为整数

~~~go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    str := "123"
    num, err := strconv.Atoi(str)
    if err != nil {
        fmt.Println("转换错误:", err)
    } else {
        fmt.Printf("字符串 '%s' 转换为整数为：%d\n", str, num)
    }
}
~~~

以下实例将整数转换为字符串：

~~~go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    num := 123
    str := strconv.Itoa(num)
    fmt.Printf("整数 %d  转换为字符串为：'%s'\n", num, str)
}
~~~

以下实例将字符串转换为浮点数：

~~~go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    str := "3.14"
    num, err := strconv.ParseFloat(str, 64)
    if err != nil {
        fmt.Println("转换错误:", err)
    } else {
        fmt.Printf("字符串 '%s' 转为浮点型为：%f\n", str, num)
    }
}
~~~

以下实例将浮点数转换为字符串：

~~~go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    num := 3.14
    str := strconv.FormatFloat(num, 'f', 2, 64)
    fmt.Printf("浮点数 %f 转为字符串为：'%s'\n", num, str)
}
~~~

## 接口类型转换

接口类型转换有两种情况**：类型断言**和**类型转换**。

### 类型断言

类型断言用于将接口类型转换为指定类型，其语法为：

```
value.(type) 
或者 
value.(T)
```

其中 value 是接口类型的变量，type 或 T 是要转换成的类型。

如果类型断言成功，它将返回转换后的值和一个布尔值，表示转换是否成功。

~~~go
package main

import "fmt"

func main() {
    var i interface{} = "Hello, World"
    str, ok := i.(string)
    if ok {
        fmt.Printf("'%s' is a string\n", str)
    } else {
        fmt.Println("conversion failed")
    }
}
~~~

以上实例中，我们定义了一个接口类型变量 i，并将它赋值为字符串 "Hello, World"。然后，我们使用类型断言将 i 转换为字符串类型，并将转换后的值赋值给变量 str。最后，我们使用 ok 变量检查类型转换是否成功，如果成功，我们打印转换后的字符串；否则，我们打印转换失败的消息。

### 类型转换

类型转换用于将一个接口类型的值转换为另一个接口类型，其语法为：

```
T(value)
```

T 是目标接口类型，value 是要转换的值。

在类型转换中，我们必须保证要转换的值和目标接口类型之间是兼容的，否则编译器会报错。

~~~go
package main

import "fmt"

// 定义一个接口 Writer
type Writer interface {
    Write([]byte) (int, error)
}

// 实现 Writer 接口的结构体 StringWriter
type StringWriter struct {
    str string
}

// 实现 Write 方法
func (sw *StringWriter) Write(data []byte) (int, error) {
    sw.str += string(data)
    return len(data), nil
}

func main() {
    // 创建一个 StringWriter 实例并赋值给 Writer 接口变量
    var w Writer = &StringWriter{}
    
    // 将 Writer 接口类型转换为 StringWriter 类型，这里不明白是为什么
    sw := w.(*StringWriter)
    
    // 修改 StringWriter 的字段
    sw.str = "Hello, World"
    
    // 打印 StringWriter 的字段值
    fmt.Println(sw.str)
}
~~~

**解析：**

1. **定义接口和结构体**：
   - `Writer` 接口定义了 `Write` 方法。
   - `StringWriter` 结构体实现了 `Write` 方法。
2. **类型转换**：
   - 将 `StringWriter` 实例赋值给 `Writer` 接口变量 `w`。
   - 使用 `w.(*StringWriter)` 将 `Writer` 接口类型转换为 `StringWriter` 类型。
3. **访问字段**：
   - 修改 `StringWriter` 的字段 `str`，并打印其值。

## 空接口类型

空接口 **interface{}** 可以持有任何类型的值。在实际应用中，空接口经常被用来处理多种类型的值。

~~~go
package main

import (
    "fmt"
)

func printValue(v interface{}) {
    switch v := v.(type) {
    case int:
        fmt.Println("Integer:", v)
    case string:
        fmt.Println("String:", v)
    default:
        fmt.Println("Unknown type")
    }
}

func main() {
    printValue(42)
    printValue("hello")
    printValue(3.14)
}
~~~

## 其他

* GO不支持隐式类型转换

  ~~~go
  package main
  import "fmt"
  
  func main() {  
      var a int64 = 3
      var b int32
      b = a
      fmt.Printf("b 为 : %d", b)
  }
  ~~~

  报错

  ~~~go
  cannot use a (type int64) as type int32 in assignment
  cannot use b (type int32) as type string in argument to fmt.Printf
  ~~~

  改为

  ~~~go
  package main
  import "fmt"
  
  func main() {  
      var a int64 = 3
      var b int32
      b = int32(a)
      fmt.Printf("b 为 : %d", b)
  }
  ~~~

  

# Go语言接口

接口（interface）是 Go 语言中的一种类型，用于定义行为的集合，它通过描述类型必须实现的方法，规定了类型的行为契约。

Go 语言提供了另外一种数据类型即接口，它把所有的具有共性的方法定义在一起，任何其他类型只要实现了这些方法就是实现了这个接口。

Go 的接口设计简单却功能强大，是实现多态和解耦的重要工具。

接口可以让我们将不同的类型绑定到一组公共的方法上，从而实现多态和灵活的设计。

## 接口特点

* **隐式实现**：

  - Go 中没有关键字显式声明某个类型实现了某个接口。
  - 只要一个类型实现了接口要求的所有方法，该类型就自动被认为实现了该接口。

  **接口类型变量**：

  - 接口变量可以存储实现该接口的任意值。
  - 接口变量实际上包含了两个部分：
    - **动态类型**：存储实际的值类型。
    - **动态值**：存储具体的值。

  **零值接口**：

  - 接口的零值是 `nil`。
  - 一个未初始化的接口变量其值为 `nil`，且不包含任何动态类型或值。

  **空接口**：

  - 定义为 `interface{}`，可以表示任何类型。

  ## 接口的常见用法

  1. **多态**：不同类型实现同一接口，实现多态行为。
  2. **解耦**：通过接口定义依赖关系，降低模块之间的耦合。
  3. **泛化**：使用空接口 `interface{}` 表示任意类型。

## 接口定义和实现

~~~go
/* 定义接口 */
type interface_name interface {
   method_name1 [return_type]
   method_name2 [return_type]
   method_name3 [return_type]
   ...
   method_namen [return_type]
}

/* 定义结构体 */
type struct_name struct {
   /* variables */
}

/* 实现接口方法 */
func (struct_name_variable struct_name) method_name1() [return_type] {
   /* 方法实现 */
}
...
func (struct_name_variable struct_name) method_namen() [return_type] {
   /* 方法实现*/
}
~~~

实现接口：类型通过实现接口要求的所有方法来实现接口。

~~~go
package main

import (
        "fmt"
        "math"
)

// 定义接口
type Shape interface {
        Area() float64
        Perimeter() float64
}

// 定义一个结构体
type Circle struct {
        Radius float64
}

// Circle 实现 Shape 接口
func (c Circle) Area() float64 {
        return math.Pi * c.Radius * c.Radius
}

func (c Circle) Perimeter() float64 {
        return 2 * math.Pi * c.Radius
}

func main() {
        c := Circle{Radius: 5}
        var s Shape = c // 接口变量可以存储实现了接口的类型
        fmt.Println("Area:", s.Area())
        fmt.Println("Perimeter:", s.Perimeter())
}
~~~

> 这么来看的话，有点像只要类实现了接口的所有方法，接口成为了实现接口的类的“父类”？

## 空接口

空接口 `interface{}` 是 Go 的特殊接口，表示所有类型的超集。

- 任意类型都实现了空接口。
- 常用于需要存储任意类型数据的场景，如泛型容器、通用参数等。

~~~go
package main

import "fmt"

func printValue(val interface{}) {
        fmt.Printf("Value: %v, Type: %T\n", val, val)
}

func main() {
        printValue(42)         // int
        printValue("hello")    // string
        printValue(3.14)       // float64
        printValue([]int{1, 2}) // slice
}
~~~

> 可以通过%T来实现类型的输出，%v来实现接口值的输出

## 类型断言

类型断言用于从接口类型中提取其底层值。

基本语法:

```
value := iface.(Type)
```

- `iface` 是接口变量。
- `Type` 是要断言的具体类型。
- 如果类型不匹配，会触发 `panic`。

~~~go
package main

import "fmt"

func main() {
        var i interface{} = "hello"
        str := i.(string) // 类型断言
        fmt.Println(str)  // 输出：hello
}
~~~

带检查的类型断言

为了避免 panic，可以使用带检查的类型断言：

```
value, ok := iface.(Type)
```

- `ok` 是一个布尔值，表示断言是否成功。
- 如果断言失败，`value` 为零值，`ok` 为 `false`。

~~~go
package main

import "fmt"

func main() {
        var i interface{} = 42
        if str, ok := i.(string); ok {
                fmt.Println("String:", str)
        } else {
                fmt.Println("Not a string")
        }
}
~~~

## Type switch类型选择

type switch 是 Go 中的语法结构，用于根据接口变量的具体类型执行不同的逻辑。

~~~go
package main

import "fmt"

func printType(val interface{}) {
        switch v := val.(type) {
        case int:
                fmt.Println("Integer:", v)
        case string:
                fmt.Println("String:", v)
        case float64:
                fmt.Println("Float:", v)
        default:
                fmt.Println("Unknown type")
        }
}

func main() {
        printType(42)
        printType("hello")
        printType(3.14)
        printType([]int{1, 2, 3})
}
~~~

## 接口组合

接口可以通过嵌套组合，实现更复杂的行为描述。

~~~go
package main

import "fmt"

type Reader interface {
        Read() string
}

type Writer interface {
        Write(data string)
}

type ReadWriter interface {
        Reader
        Writer
}

type File struct{}

func (f File) Read() string {
        return "Reading data"
}

func (f File) Write(data string) {
        fmt.Println("Writing data:", data)
}

func main() {
        var rw ReadWriter = File{}
        fmt.Println(rw.Read())
        rw.Write("Hello, Go!")
}
~~~

## 动态值和动态类型

接口变量实际上包含了两部分：

1. **动态类型**：接口变量存储的具体类型。
2. **动态值**：具体类型的值。

动态值和动态类型示例：

~~~go
package main

import "fmt"

func main() {
        var i interface{} = 42
        fmt.Printf("Dynamic type: %T, Dynamic value: %v\n", i, i)
}
~~~

输出

~~~go
Dynamic type: int, Dynamic value: 42
~~~

## 接口的零值

接口的零值是 nil。

当接口变量的动态类型和动态值都为 nil 时，接口变量为 nil。

接口零值示例：

~~~go
package main

import "fmt"

func main() {
        var i interface{}
        fmt.Println(i == nil) // 输出：true
}
~~~

## 实例

~~~go
package main

import (
    "fmt"
)

type Phone interface {
    call()
}

type NokiaPhone struct {
}

func (nokiaPhone NokiaPhone) call() {
    fmt.Println("I am Nokia, I can call you!")
}

type IPhone struct {
}

func (iPhone IPhone) call() {
    fmt.Println("I am iPhone, I can call you!")
}

func main() {
    var phone Phone

    phone = new(NokiaPhone)
    phone.call()

    phone = new(IPhone)
    phone.call()

}
~~~

在上面的例子中，我们定义了一个接口 **Phone**，接口里面有一个方法 **call()**。然后我们在 **main** 函数里面定义了一个 **Phone** 类型变量，并分别为之赋值为 **NokiaPhone** 和 **IPhone**。然后调用 **call()** 方法，输出结果如下：

```
I am Nokia, I can call you!
I am iPhone, I can call you!
```

~~~go
package main

import "fmt"

type Shape interface {
    area() float64
}

type Rectangle struct {
    width  float64
    height float64
}

func (r Rectangle) area() float64 {
    return r.width * r.height
}

type Circle struct {
    radius float64
}

func (c Circle) area() float64 {
    return 3.14 * c.radius * c.radius
}

func main() {
    var s Shape

    s = Rectangle{width: 10, height: 5}
    fmt.Printf("矩形面积: %f\n", s.area())

    s = Circle{radius: 3}
    fmt.Printf("圆形面积: %f\n", s.area())
}
~~~

以上实例中，我们定义了一个 Shape 接口，它定义了一个方法 area()，该方法返回一个 float64 类型的面积值。然后，我们定义了两个结构体 Rectangle 和 Circle，它们分别实现了 Shape 接口的 area() 方法。在 main() 函数中，我们首先定义了一个 Shape 类型的变量 s，然后分别将 Rectangle 和 Circle 类型的实例赋值给它，并通过 area() 方法计算它们的面积并打印出来，输出结果如下：

```
矩形面积: 50.000000
圆形面积: 28.260000
```

需要注意的是，接口类型变量可以存储任何实现了该接口的类型的值。在示例中，我们将 Rectangle 和 Circle 类型的实例都赋值给了 Shape 类型的变量 s，并通过 area() 方法调用它们的面积计算方法。

# GO错误处理

Go 语言通过内置的错误接口提供了非常简单的错误处理机制。

Go 语言的错误处理采用显式返回错误的方式，而非传统的异常处理机制。这种设计使代码逻辑更清晰，便于开发者在编译时或运行时明确处理错误。

**Go 的错误处理主要围绕以下机制展开：**

1. **`error` 接口**：标准的错误表示。
2. **显式返回值**：通过函数的返回值返回错误。
3. **自定义错误**：可以通过标准库或自定义的方式创建错误。
4. **`panic` 和 `recover`**：处理不可恢复的严重错误。

## error接口

Go 标准库定义了一个 error 接口，表示一个错误的抽象。

error 类型是一个接口类型，这是它的定义：

```
type error interface {
    Error() string
}
```

- **实现 `error` 接口**：任何实现了 `Error()` 方法的类型都可以作为错误。

- `Error()` 方法返回一个描述错误的字符串。

- 使用error包创建错误

  ~~~go
  package main
  
  import (
      "errors"
      "fmt"
  )
  
  func main() {
      err := errors.New("this is an error")
      fmt.Println(err) // 输出：this is an error
  }
  ~~~

  函数通常在最后的返回值中返回错误信息，使用 errors.New 可返回一个错误信息：

  ```
  func Sqrt(f float64) (float64, error) {
      if f < 0 {
          return 0, errors.New("math: square root of negative number")
      }
      // 实现
  }
  ```

  在下面的例子中，我们在调用 Sqrt 的时候传递的一个负数，然后就得到了 non-nil 的 error 对象，将此对象与 nil 比较，结果为 true，所以 fmt.Println(fmt 包在处理 error 时会调用 Error 方法)被调用，以输出错误，请看下面调用的示例代码：

  ```
  result, err:= Sqrt(-1)
  
  if err != nil {
     fmt.Println(err)
  }
  ```

## 显式返回错误

Go 中，错误通常作为函数的返回值返回，开发者需要显式检查并处理。

~~~go
package main

import (
        "errors"
        "fmt"
)

func divide(a, b int) (int, error) {
        if b == 0 {
                return 0, errors.New("division by zero")
        }
        return a / b, nil
}

func main() {
        result, err := divide(10, 0)
        if err != nil {
                fmt.Println("Error:", err)
        } else {
                fmt.Println("Result:", result)
        }
}
~~~

输出

~~~go
Error: division by zero
~~~

## 自定义错误

通过定义自定义类型，可以扩展 error 接口。

~~~go
package main

import (
        "fmt"
)

type DivideError struct {
        Dividend int
        Divisor  int
}

func (e *DivideError) Error() string {
        return fmt.Sprintf("cannot divide %d by %d", e.Dividend, e.Divisor)
}

func divide(a, b int) (int, error) {
        if b == 0 {
                return 0, &DivideError{Dividend: a, Divisor: b}
        }
        return a / b, nil
}

func main() {
        _, err := divide(10, 0)
        if err != nil {
                fmt.Println(err) // 输出：cannot divide 10 by 0
        }
}
~~~

## fmt包与错误格式化

`fmt` 包提供了对错误的格式化输出支持：

- `%v`：默认格式。
- `%+v`：如果支持，显示详细的错误信息。
- `%s`：作为字符串输出。

~~~go
package main

import (
    "fmt"
)

// 定义一个 DivideError 结构
type DivideError struct {
    dividee int
    divider int
}

// 实现 `error` 接口
func (de *DivideError) Error() string {
    strFormat := `
    Cannot proceed, the divider is zero.
    dividee: %d
    divider: 0
`
    return fmt.Sprintf(strFormat, de.dividee)
}

// 定义 `int` 类型除法运算的函数
func Divide(varDividee int, varDivider int) (result int, errorMsg string) {
    if varDivider == 0 {
            dData := DivideError{
                    dividee: varDividee,
                    divider: varDivider,
            }
            errorMsg = dData.Error()
            return
    } else {
            return varDividee / varDivider, ""
    }

}

func main() {

    // 正常情况
    if result, errorMsg := Divide(100, 10); errorMsg == "" {
            fmt.Println("100/10 = ", result)
    }
    // 当除数为零的时候会返回错误信息
    if _, errorMsg := Divide(100, 0); errorMsg != "" {
            fmt.Println("errorMsg is: ", errorMsg)
    }

}
~~~

输出

~~~go
100/10 =  10
errorMsg is:  
    Cannot proceed, the divider is zero.
    dividee: 100
    divider: 0

~~~

## 使用errors.Is和errors.As

 **`errors.Is`**

检查某个错误是否是特定错误或由该错误包装而成。

~~~go
func Is(err, target error) bool
~~~

它接收两个 `error` 类型的参数 `err` 和 `target`，会递归地检查 `err` 链中的每一个错误，判断是否存在与 `target` 相等的错误。若存在，则返回 `true`；反之则返回 `false`。

~~~go
package main

import (
        "errors"
        "fmt"
)

var ErrNotFound = errors.New("not found")

func findItem(id int) error {
        return fmt.Errorf("database error: %w", ErrNotFound)
}

func main() {
        err := findItem(1)
        if errors.Is(err, ErrNotFound) {
                fmt.Println("Item not found")
        } else {
                fmt.Println("Other error:", err)
        }
}
~~~

errors.As

将错误转换为特定类型以便进一步处理。



~~~go
package main

import (
        "errors"
        "fmt"
)

type MyError struct {
        Code int
        Msg  string
}

func (e *MyError) Error() string {
        return fmt.Sprintf("Code: %d, Msg: %s", e.Code, e.Msg)
}

func getError() error {
        return &MyError{Code: 404, Msg: "Not Found"}
}

func main() {
        err := getError()
        var myErr *MyError
        if errors.As(err, &myErr) {
                fmt.Printf("Custom error - Code: %d, Msg: %s\n", myErr.Code, myErr.Msg)
        }
}
~~~

在 `main` 函数中，调用 `getError` 函数获取错误对象 `err`，定义一个 `MyError` 类型的指针 `myErr`。`errors.As(err, &myErr)` 尝试将 `err` 转换为 `MyError` 类型，若转换成功，`myErr` 就会指向 `err` 链中第一个 `MyError` 类型的错误实例，进而可以访问 `myErr` 的 `Code` 和 `Msg` 字段并输出详细信息。

## painc和recover？

Go 的 panic 用于处理不可恢复的错误，recover 用于从 panic 中恢复。

**panic:**

- 导致程序崩溃并输出堆栈信息。
- 常用于程序无法继续运行的情况。

**recover:**

- 捕获 `panic`，避免程序崩溃。

~~~go
package main

import "fmt"

func safeFunction() {
        defer func() {
                if r := recover(); r != nil {
                        fmt.Println("Recovered from panic:", r)
                }
        }()
        panic("something went wrong")
}

func main() {
        fmt.Println("Starting program...")
        safeFunction()
        fmt.Println("Program continued after panic")
}
~~~

* defer

  在 Go 语言里，`defer` 关键字用于修饰一个函数调用，其作用是让这个函数调用被延迟到当前函数执行结束前执行。无论当前函数是正常返回，还是因为 `panic` 异常退出，被 `defer` 修饰的函数都会被执行。

  `defer` 语句的特点如下：

  - **延迟执行**：当执行到 `defer` 语句时，并不会立即执行 `defer` 后面的函数，而是将其压入一个栈中，等包含 `defer` 语句的函数执行结束前，再从栈中依次取出并执行这些函数。
  - **后进先出（LIFO）顺序**：如果一个函数中有多个 `defer` 语句，它们会按照后进先出的顺序执行。

* defer与recover

  `defer` 和 `recover` 常常一起使用来处理 `panic` 异常。`panic` 是 Go 语言中用于抛出异常的机制，当调用 `panic` 函数时，程序会立即停止当前函数的执行，并开始回溯调用栈，依次执行每个函数中 `defer` 修饰的函数。

  `recover` 是 Go 语言中用于捕获 `panic` 异常的函数，它只能在 `defer` 修饰的函数中使用。当 `recover` 函数在 `defer` 函数中被调用时，如果当前函数正在处理 `panic` 异常，`recover` 会捕获这个 `panic` 并返回 `panic` 传入的参数，同时程序会停止回溯调用栈，继续执行后续的代码；如果当前函数没有发生 `panic`，`recover` 会返回 `nil`。

## 其他

* panic 与 recover 是 Go 的两个内置函数，这两个内置函数用于处理 Go 运行时的错误，panic 用于主动抛出错误，recover 用来捕获 panic 抛出的错误。

  - 引发`panic`有两种情况，一是程序主动调用，二是程序产生运行时错误，由运行时检测并退出。
  - 发生`panic`后，程序会从调用`panic`的函数位置或发生`panic`的地方立即返回，逐层向上执行函数的`defer`语句，然后逐层打印函数调用堆栈，直到被`recover`捕获或运行到最外层函数。
  - `panic`不但可以在函数正常流程中抛出，在`defer`逻辑里也可以再次调用`panic`或抛出`panic`。`defer`里面的`panic`能够被后续执行的`defer`捕获。
  - `recover`用来捕获`panic`，阻止`panic`继续向上传递。`recover()`和`defer`一起使用，但是`recover`只有在后面的函数体内直接被掉用才能捕获`panic`来终止异常，否则返回`nil`，异常继续向外传递。

  ~~~go
  //以下捕获失败
  defer recover()
  defer fmt.Prinntln(recover)
  defer func(){
      func(){
          recover() //无效，嵌套两层
      }()
  }()
  
  //以下捕获有效
  defer func(){
      recover()
  }()
  
  func except(){
      recover()
  }
  func test(){
      defer except()
      panic("runtime error")
  }
  ~~~

  多个panic只会捕捉最后一个：

  ~~~go
  package main
  import "fmt"
  func main(){
      defer func(){
          if err := recover() ; err != nil {
              fmt.Println(err)
          }
      }()
      defer func(){
          panic("three")
      }()
      defer func(){
          panic("two")
      }()
      panic("one")
  }
  ~~~

  分析：

  1. **`panic("one")`**：
     当执行到 `panic("one")` 时，程序开始抛出异常，停止 `main` 函数的正常执行，开始回溯调用栈，准备依次执行 `defer` 函数。
  2. **执行第三个 `defer` 函数**：
     由于 `defer` 函数是后进先出的顺序执行，所以首先执行第三个 `defer` 函数 `defer func(){ panic("two") }()`。在这个函数中，又调用了 `panic("two")`，这会导致新的 `panic` 异常，覆盖之前的 `panic("one")` 异常。此时程序会继续回溯调用栈，执行下一个 `defer` 函数。
  3. **执行第二个 `defer` 函数**：
     接着执行第二个 `defer` 函数 `defer func(){ panic("three") }()`。在这个函数中，再次调用了 `panic("three")`，新的 `panic` 异常又覆盖了之前的 `panic("two")` 异常。程序继续回溯调用栈，执行下一个 `defer` 函数。
  4. **执行第一个 `defer` 函数**：
     最后执行第一个 `defer` 函数 `defer func(){ if err := recover() ; err != nil { fmt.Println(err) } }()`。在这个函数中，调用了 `recover` 函数，它会捕获当前的 `panic` 异常（即 `panic("three")`），并将异常信息赋值给变量 `err`。由于 `err` 不为 `nil`，所以会执行 `fmt.Println(err)`，输出 `three`。

* 1、panic 在没有用 recover 前以及在 recover 捕获那一级函数栈，panic 之后的代码均不会执行；一旦被 recover 捕获后，外层的函数栈代码恢复正常，所有代码均会得到执行；

*  2、panic 后，不再执行后面的代码，立即按照逆序执行 defer，并逐级往外层函数栈扩散；defer 就类似 finally；

*  3、利用 recover 捕获 panic 时，defer 需要再 panic 之前声明，否则由于 panic 之后的代码得不到执行，因此也无法 recover；

# Go并发

并发是指程序同时执行多个任务的能力。

Go 语言支持并发，通过 goroutines 和 channels 提供了一种简洁且高效的方式来实现并发。

**Goroutines：**

- Go 中的并发执行单位，类似于轻量级的线程。
- Goroutine 的调度由 Go 运行时管理，用户无需手动分配线程。
- 使用 `go` 关键字启动 Goroutine。
- Goroutine 是非阻塞的，可以高效地运行成千上万个 Goroutine。

**Channel：**

- Go 中用于在 Goroutine 之间通信的机制。
- 支持同步和数据共享，避免了显式的锁机制。
- 使用 `chan` 关键字创建，通过 `<-` 操作符发送和接收数据。

**Scheduler（调度器）：**

Go 的调度器基于 GMP 模型，调度器会将 Goroutine 分配到系统线程中执行，并通过 M 和 P 的配合高效管理并发。

- **G**：Goroutine。
- **M**：系统线程（Machine）。
- **P**：逻辑处理器（Processor）。

## Goroutine

goroutine 是轻量级线程，goroutine 的调度是由 Golang 运行时进行管理的。

goroutine 语法格式：

```
go 函数名( 参数列表 )
```

例如：

```
go f(x, y, z)
```

开启一个新的 goroutine:

```
f(x, y, z)
```

Go 允许使用 go 语句开启一个新的运行期线程， 即 goroutine，以一个不同的、新创建的 goroutine 来执行一个函数。 同一个程序中的所有 goroutine 共享同一个地址空间。

~~~go
package main

import (
        "fmt"
        "time"
)

func sayHello() {
        for i := 0; i < 5; i++ {
                fmt.Println("Hello")
                time.Sleep(100 * time.Millisecond)
        }
}

func main() {
        go sayHello() // 启动 Goroutine
        for i := 0; i < 5; i++ {
                fmt.Println("Main")
                time.Sleep(100 * time.Millisecond)
        }
}
~~~

执行以上代码，你会看到输出的 Main 和 Hello。输出是没有固定先后顺序，因为它们是两个 goroutine 在执行：

```
Main
Hello
Main
Hello
...
```

## 通道Channel

通道（Channel）是用于 Goroutine 之间的数据传递。

通道可用于两个 goroutine 之间通过传递一个指定类型的值来同步运行和通讯。

使用 `make` 函数创建一个 channel，使用 `<-` 操作符发送和接收数据。如果未指定方向，则为双向通道。

```
ch <- v    // 把 v 发送到通道 ch
v := <-ch  // 从 ch 接收数据
           // 并把值赋给 v
```

声明一个通道很简单，我们使用chan关键字即可，通道在使用前必须先创建：

```
ch := make(chan int)
```

**注意**：**默认情况下，通道是不带缓冲区的。发送端发送数据，同时必须有接收端相应的接收数据。**

以下实例通过两个 goroutine 来计算数字之和，在 goroutine 完成计算后，它会计算两个结果的和：

~~~go
package main

import "fmt"

func sum(s []int, c chan int) {
    sum := 0
    for _, v := range s {
        sum += v
    }
    c <- sum // 把 sum 发送到通道 c
}

func main() {
    s := []int{7, 2, 8, -9, 4, 0}

    c := make(chan int)
    go sum(s[:len(s)/2], c)
    go sum(s[len(s)/2:], c)
    x, y := <-c, <-c // 从通道 c 中接收

    fmt.Println(x, y, x+y)
}
~~~

> * 在 Go 语言里，通道（`channel`）是一种用于在 `goroutine` 之间进行通信和同步的机制。当向通道发送数据（`channel <- data`）或者从通道接收数据（`data <- channel`）时，操作可能会阻塞。具体情况如下：
>
>   - **发送操作阻塞**：当通道是无缓冲通道且通道已满（对于缓冲通道而言）或者没有接收者时，发送操作会阻塞，直到有接收者从通道中接收数据。
>   - **接收操作阻塞**：当通道是无缓冲通道且通道为空或者没有发送者时，接收操作会阻塞，直到有发送者向通道发送数据。
>
>   `main` 函数会等待 `goroutine` 运行完成，这是因为无缓冲通道的接收操作具有阻塞特性。在 `main` 函数中，通过 `<-c` 从通道接收数据时，如果通道中没有数据，`main` 函数会被阻塞，直到 `goroutine` 向通道发送数据，从而确保 `main` 函数会等待 `goroutine` 完成计算任务。
>
> * 在 Go 语言中，`goroutine` 并不是一个独立的线程，它是 Go 程序中轻量级的并发执行单元，由 Go 运行时（runtime）进行调度，多个 `goroutine` 可以在同一个操作系统线程上复用执行。
>
>   如果主函数和 `goroutine` 之间没有通道或其他同步机制进行数据通信，主函数**通常不会等待** `goroutine` 运行完成。当主函数执行完毕后，Go 程序会直接退出，而不管其他 `goroutine` 是否还在运行。
>
>   ~~~go
>   package main
>     
>   import (
>       "fmt"
>       "time"
>   )
>     
>   func main() {
>       // 启动一个goroutine
>       go func() {
>           // 模拟一个耗时的任务，这里休眠2秒
>           time.Sleep(2 * time.Second)
>           fmt.Println("Goroutine finished")
>       }()
>     
>       // 主函数直接退出，不会等待goroutine完成
>       fmt.Println("Main function exited")
>   }
>   ~~~
>
>   如果 `goroutine` 没运行结束时主函数先结束了，`goroutine` 会因为主函数的结束而结束。
>
>   Go 程序在主函数返回时会直接退出，而不会等待其他 `goroutine` 完成。即使 `goroutine` 中还有未完成的任务，整个程序也会终止，这些 `goroutine` 会被强制停止，它们后续的代码将不会继续执行。

## 通道缓冲区

通道可以设置缓冲区，通过 make 的第二个参数指定缓冲区大小：

```
ch := make(chan int, 100)
```

**带缓冲区的通道允许发送端的数据发送和接收端的数据获取处于异步状态，就是说发送端发送的数据可以放在缓冲区里面，可以等待接收端去获取数据，而不是立刻需要接收端去获取数据。**

不过由于缓冲区的大小是有限的，所以还是必须有接收端来接收数据的，否则缓冲区一满，数据发送端就无法再发送数据了。

**注意**：如果通道不带缓冲，发送方会阻塞直到接收方从通道中接收了值。如果通道带缓冲，发送方则会阻塞直到发送的值被拷贝到缓冲区内；如果缓冲区已满，则意味着需要等待直到某个接收方获取到一个值。接收方在有值可以接收之前会一直阻塞。

~~~go
package main

import "fmt"

func main() {
    // 这里我们定义了一个可以存储整数类型的带缓冲通道
    // 缓冲区大小为2
    ch := make(chan int, 2)

    // 因为 ch 是带缓冲的通道，我们可以同时发送两个数据
    // 而不用立刻需要去同步读取数据
    ch <- 1
    ch <- 2

    // 获取这两个数据
    fmt.Println(<-ch)
    fmt.Println(<-ch)
}
~~~

## go遍历通道与关闭通道

Go 通过 range 关键字来实现遍历读取到的数据，类似于与数组或切片。格式如下：

```
v, ok := <-ch
```

如果通道接收不到数据后 ok 就为 false，这时通道就可以使用 **close()** 函数来关闭。

关闭通道并不会丢失里面的数据，只是让读取通道数据的时候不会读完之后一直阻塞等待新数据写入

~~~go
package main

import (
    "fmt"
)

func fibonacci(n int, c chan int) {
    x, y := 0, 1
    for i := 0; i < n; i++ {
        c <- x
        x, y = y, x+y
    }
    close(c)
}

func main() {
    c := make(chan int, 10)
    go fibonacci(cap(c), c)
    // range 函数遍历每个从通道接收到的数据，因为 c 在发送完 10 个
    // 数据之后就关闭了通道，所以这里我们 range 函数在接收到 10 个数据
    // 之后就结束了。如果上面的 c 通道不关闭，那么 range 函数就不
    // 会结束，从而在接收第 11 个数据的时候就阻塞了。
    for i := range c {
        fmt.Println(i)
    }
}
~~~

## select语句

`select` 语句使得一个 goroutine 可以等待多个通信操作。`select` 会阻塞，直到其中的某个 case 可以继续执行：

~~~go
package main

import "fmt"

func fibonacci(c, quit chan int) {
    x, y := 0, 1
    for {
        select {
        case c <- x:
            x, y = y, x+y
        case <-quit:
            fmt.Println("quit")
            return
        }
    }
}

func main() {
    c := make(chan int)
    quit := make(chan int)

    go func() {
        for i := 0; i < 10; i++ {
            fmt.Println(<-c)
        }
        quit <- 0
    }()
    fibonacci(c, quit)
}
~~~

以上代码中，`fibonacci` goroutine 在 channel `c` 上发送斐波那契数列，当接收到 `quit` channel 的信号时退出。

执行输出结果为：

~~~go
0
1
1
2
3
5
8
13
21
34
quit
~~~

## 使用waitGroup

sync.WaitGroup 用于等待多个 Goroutine 完成。

**同步多个 Goroutine：**

~~~go
package main

import (
        "fmt"
        "sync"
)

func worker(id int, wg *sync.WaitGroup) {
        defer wg.Done() // Goroutine 完成时调用 Done()
        fmt.Printf("Worker %d started\n", id)
        fmt.Printf("Worker %d finished\n", id)
}

func main() {
        var wg sync.WaitGroup

        for i := 1; i <= 3; i++ {
                wg.Add(1) // 增加计数器
                go worker(i, &wg)
        }

        wg.Wait() // 等待所有 Goroutine 完成
        fmt.Println("All workers done")
}
~~~

## 高级特性

**Buffered Channel：**

创建有缓冲的 Channel。

```
ch := make(chan int, 2)
```

**Context：**

用于控制 Goroutine 的生命周期。

```
context.WithCancel、context.WithTimeout。
```

**Mutex 和 RWMutex：**

sync.Mutex 提供互斥锁，用于保护共享资源。

```
var mu sync.Mutex
mu.Lock()
// critical section
mu.Unlock()
```
