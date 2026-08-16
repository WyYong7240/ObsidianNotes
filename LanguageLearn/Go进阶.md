# Go语言圣经部分

## Part1:入门

1. 使用命令行参数

   `os.Args[0]`是程序本身的名字，其他的才是传递给命令行的参数

   ~~~go
   // Echo2 prints its command-line arguments.
   package main
   
   import (
       "fmt"
       "os"
   )
   
   func main() {
       s, sep := "", ""
       for _, arg := range os.Args[1:] {
           s += sep + arg
           sep = " "
       }
       fmt.Println(s)
   }
   ~~~

2. 命令行输入

   ~~~go
   // Dup1 prints the text of each line that appears more than
   // once in the standard input, preceded by its count.
   package main
   
   import (
       "bufio"
       "fmt"
       "os"
   )
   
   func main() {
       counts := make(map[string]int)
       input := bufio.NewScanner(os.Stdin)
       for input.Scan() {
           counts[input.Text()]++
       }
       // NOTE: ignoring potential errors from input.Err()
       for line, n := range counts {
           if n > 1 {
               fmt.Printf("%d\t%s\n", n, line)
           }
       }
   }
   ~~~

3. 文件读取（逐行读取）

   ~~~go
   // Dup2 prints the count and text of lines that appear more than once
   // in the input.  It reads from stdin or from a list of named files.
   package main
   
   import (
       "bufio"
       "fmt"
       "os"
   )
   
   func main() {
       counts := make(map[string]int)
       files := os.Args[1:]
       if len(files) == 0 {
           countLines(os.Stdin, counts)
       } else {
           for _, arg := range files {
               f, err := os.Open(arg)
               if err != nil {
                   fmt.Fprintf(os.Stderr, "dup2: %v\n", err)
                   continue
               }
               countLines(f, counts)
               f.Close()
           }
       }
       for line, n := range counts {
           if n > 1 {
               fmt.Printf("%d\t%s\n", n, line)
           }
       }
   }
   
   func countLines(f *os.File, counts map[string]int) {
       input := bufio.NewScanner(f)
       for input.Scan() {
           counts[input.Text()]++
       }
       // NOTE: ignoring potential errors from input.Err()
   }
   ~~~

4. 文件读取（一次读取所有数据）

   ~~~go
   package main
   
   import (
       "fmt"
       "io/ioutil"
       "os"
       "strings"
   )
   
   func main() {
       counts := make(map[string]int)
       for _, filename := range os.Args[1:] {
           data, err := ioutil.ReadFile(filename)
           if err != nil {
               fmt.Fprintf(os.Stderr, "dup3: %v\n", err)
               continue
           }
           for _, line := range strings.Split(string(data), "\n") {
               counts[line]++
           }
       }
       for line, n := range counts {
           if n > 1 {
               fmt.Printf("%d\t%s\n", n, line)
           }
       }
   }
   ~~~

5. 获取URL中的内容

   完成了书中的三个要求

   ~~~go
   // Fetch prints the content found at each specified URL.
   package main
   
   import (
   	"fmt"
   	"io"
   	"net/http"
   	"os"
   	"strings"
   )
   
   func main() {
   	for _, url := range os.Args[1:] {
   
   		if !strings.HasPrefix(url, "http://") {
   			url = "http://" + url
   		}
   
   		resp, err := http.Get(url)
   		if err != nil {
   			fmt.Fprintf(os.Stderr, "fetch: %v\n", err)
   			os.Exit(1)
   		}
   		// b, err := ioutil.ReadAll(resp.Body)
   		b, err := io.Copy(os.Stdout, resp.Body)
   		fmt.Printf("%d\n", b)
   		if err != nil {
   			fmt.Fprintf(os.Stderr, "fetch: %v\n", err)
   			os.Exit(1)
   		}
   
   		// c, err := io.Copy(os.Stdout, resp.Status)
   		fmt.Printf("%s\n", resp.Status)
   		resp.Body.Close()
   
   		// if err != nil {
   		// 	fmt.Fprintf(os.Stderr, "fetch: reading %s: %v\n", url, err)
   		// 	os.Exit(1)
   		// }
   		// fmt.Printf("%s", b)
   	}
   }
   ~~~

6. Web Server的创建

   ~~~go
   package main
   
   import (
   	"fmt"
   	"log"
   	"net/http"
   	"sync"
   )
   
   var mu sync.Mutex
   var count int
   
   func main() {
   	http.HandleFunc("/", handler)
   	http.HandleFunc("/count", counter)
   	log.Fatal(http.ListenAndServe("localhost:8000", nil))
   }
   
   // handler echoes the Path component of the requested URL.
   func handler(w http.ResponseWriter, r *http.Request) {
   	mu.Lock()
   	count++
   	mu.Unlock()
   	fmt.Fprintf(w, "URL.Path = %q\n", r.URL.Path)
   }
   
   // handler echoes the HTTP request.
   func handler2(w http.ResponseWriter, r *http.Request) {
   	fmt.Fprintf(w, "%s %s %s\n", r.Method, r.URL, r.Proto)
   	for k, v := range r.Header {
   		fmt.Fprintf(w, "Header[%q] = %q\n", k, v)
   	}
   	fmt.Fprintf(w, "Host = %q\n", r.Host)
   	fmt.Fprintf(w, "RemoteAddr = %q\n", r.RemoteAddr)
   	if err := r.ParseForm(); err != nil {
   		log.Print(err)
   	}
   	for k, v := range r.Form {
   		fmt.Fprintf(w, "Form[%q] = %q\n", k, v)
   	}
   }
   
   // counter echoes the number of calls so far.
   func counter(w http.ResponseWriter, r *http.Request) {
   	mu.Lock()
   	fmt.Fprintf(w, "Count %d\n", count)
   	mu.Unlock()
   }
   
   //!-
   ~~~

## Part2：复合数据类型

# Go语言高级编程

## 语言基础

### 1. 数组、字符串、切片

#### 数组

* Go 语言中数组是值语义。**一个数组变量即表示整个数组，它并不是隐式的指向第一个元素的指针（比如 C 语言的数组），而是一个完整的值。当一个数组变量被赋值或者被传递的时候（被赋值，指的是将该数组作为另一个数组的初值的情况），实际上会复制整个数组。如果数组较大的话，数组的赋值也会有较大的开销。为了避免复制数组带来的开销，可以传递一个指向数组的指针，但是数组指针并不是数组。**

  ~~~go
  var a = [...]int{1, 2, 3} // a 是一个数组
  var b = &a                // b 是指向数组的指针
  
  fmt.Println(a[0], a[1])   // 打印数组的前 2 个元素
  fmt.Println(b[0], b[1])   // 通过数组指针访问数组元素的方式和数组类似
  
  for i, v := range b {     // 通过数组指针迭代数组的元素
      fmt.Println(i, v)
  }
  ~~~

  其中 `b` 是指向 `a` 数组的指针，但是通过 `b` 访问数组中元素的写法和 `a` 类似的。还可以通过 `for range` 来迭代数组指针指向的数组元素。其实数组指针类型除了类型和数组不同之外，通过数组指针操作数组的方式和通过数组本身的操作类似**，而且数组指针赋值时只会拷贝一个指针。**但是数组指针类型依然不够灵活，**因为数组的长度是数组类型的组成部分，指向不同长度数组的数组指针类型也是完全不同的。**

  > 直接修改数组中某个元素的值，或者通过指针传参后在另一个函数中修改该数组某个元素的值的情况是不会导致原本的数组整体复制的

* 长度为0的数组，即空数组，是不占用内存空间的。并且，数组的len、cap是相同的，前者为长度，后者为容量；

  空数组的作用如下

  空数组虽然很少直接使用，但是可以用于强调某种特有类型的操作时避免分配额外的内存空间，比如用于管道的同步操作：

  ~~~go
      c1 := make(chan [0]int)
      go func() {
          fmt.Println("c1")
          c1 <- [0]int{}
      }()
      <-c1
  ~~~

  在这里，我们并不关心管道中传输数据的真实类型，其中管道接收和发送操作只是用于消息的同步。对于这种场景，我们用空数组来作为管道类型可以减少管道元素赋值时的开销。当然一般更倾向于用无类型的匿名结构体代替：

  ~~~go
      c2 := make(chan struct{})
      go func() {
          fmt.Println("c2")
          c2 <- struct{}{} // struct{} 部分是类型, {} 表示对应的结构体值
      }()
      <-c2
  ~~~

  > **空结构体（struct{}）的内存特性**
  >
  > - `struct{}` 是一个类型，**不占用内存空间**
  > - `struct{}{}` 是该类型的实例，**也不占用内存空间**（`unsafe.Sizeof(struct{}{}) == 0`）
  >
  > **当执行`c2 <- struct{}{}`时：**
  >
  > 1. 通道接收一个空结构体实例
  > 2. 通道状态变为"有数据可接收"
  > 3. `<-c2`会阻塞直到通道有数据，当数据到达后，`<-c2`会接收数据并继续执行
  >
  > 由于空结构体不占用内存，所以这种通道的内存开销**极小**，非常适合用于信号通知，如知识库[2]中所述：
  >
  > > "纯通知场景（如 done 信号）优先用 chan struct{}，零内存开销"

#### 字符串

* 和数组不同的是，字符串的元素不可修改，是一个只读的字节数组。每个字符串的长度虽然也是固定的，但是字符串的长度并不是字符串类型的一部分。

  Go 语言字符串的底层结构在 `reflect.StringHeader` 中定义：

  ```go
  type StringHeader struct {
      Data uintptr
      Len  int
  }
  ```

  字符串结构由两个信息组成：第一个是字符串指向的底层字节数组，第二个是字符串的字节的长度。字符串其实是一个结构体，因此字符串的赋值操作也就是 `reflect.StringHeader` 结构体的复制过程，并不会涉及底层字节数组的复制。在前面数组一节提到的 `[2]string` 字符串数组对应的底层结构和 `[2]reflect.StringHeader` 对应的底层结构是一样的，可以将字符串数组看作一个结构体数组。

  我们可以看看字符串“Hello, world”本身对应的内存结构：
  <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202603022047081.png" alt="image.png" style="zoom:80%;" />

  > 根据图片和你的问题，我们可以逐步解析来得到答案。
  >
  > 1. 字符串的存储结构
  >
  > 在 Go 语言中，字符串的存储结构并不是每个字符都使用 `StringHeader` 结构体来存储。实际上，**一个字符串整体使用一个 `StringHeader` 结构体来描述其存储结构**。
  >
  > - **`Data`** 字段是一个指针，指向字符串在内存中的起始位置。
  > - **`Len`** 字段是一个整数，表示字符串的长度（即字符的数量）。
  >
  > 如图所示：
  >
  > - `"hello, world"` 整个字符串有一个 `StringHeader`，其 `Data` 指向 `"hello, world"` 的首地址，`Len` 为 12。
  > - `"hello"` 子字符串也有一个 `StringHeader`，其 `Data` 指向 `"hello"` 的首地址（即 `"hello, world"` 中的 'h'），`Len` 为 5。
  > - `"world"` 子字符串同样有一个 `StringHeader`，其 `Data` 指向 `"world"` 的首地址（即 `"hello, world"` 中的 'w'），`Len` 为 5。
  >
  > 2. 字符串赋值操作
  >
  > 字符串的赋值操作实际上是 `StringHeader` 结构体的复制过程，**并不会涉及底层字节数组的复制**。这意味着：
  >
  > - 当你将一个字符串赋值给另一个字符串变量时，只是复制了 `StringHeader`（即 `Data` 和 `Len` 两个字段），而底层的字节数组（字符串内容）并没有被复制。
  >
  > 3. 与数组的对比
  >
  > 数组在被作为初值赋值给另一个新数组时，会整体复制这个数组，即复制数组中的每个元素。这与字符串的赋值操作确实有所不同，但并不矛盾，原因如下：
  >
  > - **数组和字符串的底层实现不同**：
  >   - **数组** 是一组相同类型的值的集合，每个元素独立存储。
  >   - **字符串** 在 Go 中是**不可变的字节序列**，其 `StringHeader` 只是描述信息，实际内容存储在连续的内存区域中。
  > - **赋值操作的语义不同**：
  >   - **数组赋值**：复制数组的所有元素，生成一个新的独立数组。
  >   - **字符串赋值**：复制 `StringHeader`，指向同一底层字节数组，因为字符串不可变，这样设计可以节省内存和提高效率。
  >
  > 4. 切片和字符串的关系
  >
  > 书中提到数组是切片和字符串的基础，这是因为：
  >
  > - **切片** 和 **字符串** 都是基于数组实现的，它们的 `Header` 结构体（`SliceHeader` 和 `StringHeader`）都包含对底层数组的引用。
  > - **切片** 可以修改其引用的数组内容（因为切片是可变的）。
  > - **字符串** 不能修改其引用的数组内容（因为字符串是不可变的）。
  >
  > ### 结论
  >
  > - **字符串的每个字符** 并不是使用 `StringHeader` 结构体来存储，**一个字符串整体使用一个 `StringHeader` 结构体来描述其存储结构**。
  > - **字符串的赋值操作** 只复制 `StringHeader`，不涉及底层字节数组的复制，这与数组的赋值操作（整体复制数组）并不矛盾，因为它们的底层实现和赋值语义不同。
  >
  > 因此，你的理解是正确的，字符串的赋值操作不会涉及底层字节数组的复制，这与数组的赋值操作并不矛盾。


#### 切片

* 因为动态数组的长度是不固定，切片的长度自然也就不能是类型的组成部分了。

  我们先看看切片的结构定义，`reflect.SliceHeader`：

  ```go
  type SliceHeader struct {
      Data uintptr
      Len  int
      Cap  int
  }
  ```

  可以看出切片的开头部分和 Go 字符串是一样的，但是切片多了一个 `Cap` 成员表示切片指向的内存空间的最大容量（对应元素的个数，而不是字节数）。下图是 `x := []int{2,3,5,7,11}` 和 `y := x[1:3]` 两个切片对应的内存结构。
  <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202603022050079.png" alt="image.png" style="zoom:80%;" />
  
  > 可以看到，y由于是对x中一些片段的引用，并没有复制，上述字符串应该的字串应该也是同样的含义
  >
  > 虽然y仅仅引用了x的1：3左闭右开，但是其cap容量还是4，也就是到了x的最后
  
  切片可以和 `nil` 进行比较，只有当切片底层数据指针为空时切片本身为 `nil`，这时候切片的长度和容量信息将是无效的。如果有切片的底层数据指针为空，但是长度和容量不为 0 的情况，那么说明切片本身已经被损坏了
  
  其实除了遍历之外，只要是切片的底层数据指针、长度和容量没有发生变化的话，对切片的遍历、元素的读取和修改都和数组是一样的。在对切片本身赋值或参数传递时，和数组指针的操作方式类似，只是复制切片头信息（`reflect.SliceHeader`），并不会复制底层的数据。对于类型，和数组的最大不同是，切片的类型和长度信息无关，只要是相同类型元素构成的切片均对应相同的切片类型。
  
  > 总结一下，数组只有在传参、赋值给别的数组时，才是整片数据复制，其他情况，例如直接修改、读取、遍历和切片是一样的，直接在原地操作；
  >
  > 切片在删除（s=s[1:]）是不复制底层数组；在追加元素时，如果cap够用，那就不复制，否则分配新内存地址，并将旧数据复制到新地址
  
* 除了在切片的尾部追加，我们还可以在切片的开头添加元素：

  ```go
  var a = []int{1,2,3}
  a = append([]int{0}, a...)        // 在开头添加 1 个元素
  a = append([]int{-3,-2,-1}, a...) // 在开头添加 1 个切片
  ```
  
  在开头一般都会导致内存的重新分配，而且会导致已有的元素全部复制 1 次。因此，从切片的开头添加元素的性能一般要比从尾部追加元素的性能差很多。
  
  由于 `append` 函数返回新的切片，也就是它支持链式操作。我们可以将多个 `append` 操作组合起来，实现在切片中间插入元素：

  ```go
  var a []int
  a = append(a[:i], append([]int{x}, a[i:]...)...)     // 在第 i 个位置插入 x
  a = append(a[:i], append([]int{1,2,3}, a[i:]...)...) // 在第 i 个位置插入切片
  ```
  
  每个添加操作中的第二个 `append` 调用都会创建一个临时切片，并将 `a[i:]` 的内容复制到新创建的切片中，然后将临时创建的切片再追加到 `a[:i]`。
  
  可以用 `copy` 和 `append` 组合可以避免创建中间的临时切片，同样是完成添加元素的操作：
  
  ```go
  a = append(a, 0)     // 切片扩展 1 个空间
  copy(a[i+1:], a[i:]) // a[i:] 向后移动 1 个位置
  a[i] = x             // 设置新添加的元素
  ```
  
  第一句 `append` 用于扩展切片的长度，为要插入的元素留出空间。第二句 `copy` 操作将要插入位置开始之后的元素向后挪动一个位置。第三句真实地将新添加的元素赋值到对应的位置。操作语句虽然冗长了一点，但是相比前面的方法，可以减少中间创建的临时切片。
  
* 删除切片元素

  根据要删除元素的位置有三种情况：从开头位置删除，从中间位置删除，从尾部删除。其中删除切片尾部的元素最快

  删除开头的元素也可以不移动数据指针，但是将后面的数据向开头移动。可以用 `append` 原地完成（所谓原地完成是指在原有的切片数据对应的内存区间内完成，不会导致内存空间结构的变化）：

  ```go
  a = []int{1, 2, 3}
  a = append(a[:0], a[1:]...) // 删除开头 1 个元素
  a = append(a[:0], a[N:]...) // 删除开头 N 个元素
  ```

* 在本节开头的数组部分我们提到过有类似 `[0]int` 的空数组，空数组一般很少用到。但是对于切片来说，`len` 为 `0` 但是 `cap` 容量不为 `0` 的切片则是非常有用的特性。当然，如果 `len` 和 `cap` 都为 `0` 的话，则变成一个真正的空切片，虽然它并不是一个 `nil` 值的切片。在判断一个切片是否为空时，一般通过 `len` 获取切片的长度来判断，一般很少将切片和 `nil` 值做直接的比较。

  其实类似的根据过滤条件原地删除切片元素的算法都可以采用类似的方式处理（因为是删除操作不会出现内存不足的情形）：

  ```go
  func Filter(s []byte, fn func(x byte) bool) []byte {
      b := s[:0]
      for _, x := range s {
          if !fn(x) {
              b = append(b, x)
          }
      }
      return b
  }
  ```

  切片高效操作的要点是要降低内存分配的次数，尽量保证 `append` 操作不会超出 `cap` 的容量，降低触发内存分配的次数和每次分配内存大小。

* 避免切片内存泄漏

  如前面所说，切片操作并不会复制底层的数据。底层的数组会被保存在内存中，直到它不再被引用。但是有时候可能会因为一个小的内存引用而导致底层整个数组处于被使用的状态，这会延迟自动内存回收器对底层数组的回收。

  类似的问题，在删除切片元素时可能会遇到。假设切片里存放的是指针对象，那么下面删除末尾的元素后，被删除的元素依然被切片底层数组引用，从而导致不能及时被自动垃圾回收器回收（这要依赖回收器的实现方式）：

  ```go
  var a []*int{ ... }
  a = a[:len(a)-1]    // 被删除的最后一个元素依然被引用, 可能导致 GC 操作被阻碍
  ```

  保险的方式是先将需要自动内存回收的元素设置为 `nil`，保证自动回收器可以发现需要回收的对象，然后再进行切片的删除操作：

  ```go
  var a []*int{ ... }
  a[len(a)-1] = nil // GC 回收最后一个元素内存
  a = a[:len(a)-1]  // 从切片删除最后一个元素
  ```

  当然，如果切片存在的周期很短的话，可以不用刻意处理这个问题。因为如果切片本身已经可以被 GC 回收的话，切片对应的每个元素自然也就是可以被回收的了。

* 切片强制类型转换

  为了安全，当两个切片类型 `[]T` 和 `[]Y` 的底层原始切片类型不同时，Go 语言是无法直接转换类型的。不过安全都是有一定代价的，有时候这种转换是有它的价值的——可以简化编码或者是提升代码的性能。比如在 64 位系统上，需要对一个 `[]float64` 切片进行高速排序，我们可以将它强制转为 `[]int` 整数切片，然后以整数的方式进行排序（因为 `float64` 遵循 IEEE754 浮点数标准特性，当浮点数有序时对应的整数也必然是有序的）。

  下面的代码通过两种方法将 `[]float64` 类型的切片转换为 `[]int` 类型的切片：

  ```go
  // +build amd64 arm64
  
  import "sort"
  
  var a = []float64{4, 2, 5, 7, 2, 1, 88, 1}
  
  func SortFloat64FastV1(a []float64) {
      // 强制类型转换
      var b []int = ((*[1 << 20]int)(unsafe.Pointer(&a[0])))[:len(a):cap(a)]
  
      // 以 int 方式给 float64 排序
      sort.Ints(b)
  }
  
  func SortFloat64FastV2(a []float64) {
      // 通过 reflect.SliceHeader 更新切片头部信息实现转换
      var c []int
      aHdr := (*reflect.SliceHeader)(unsafe.Pointer(&a))
      cHdr := (*reflect.SliceHeader)(unsafe.Pointer(&c))
      *cHdr = *aHdr
  
      // 以 int 方式给 float64 排序
      sort.Ints(c)
  }
  ```

  第一种强制转换是先将切片数据的开始地址转换为一个较大的数组的指针，然后对数组指针对应的数组重新做切片操作。中间需要 `unsafe.Pointer` 来连接两个不同类型的指针传递。需要注意的是，Go语言实现中非0大小数组的长度不得超过 2GB，因此需要针对数组元素的类型大小计算数组的最大长度范围（`[]uint8` 最大 2GB，`[]uint16` 最大 1GB，以此类推，但是 `[]struct{}` 数组的长度可以超过 2GB）。

  第二种转换操作是分别取到两个不同类型的切片头信息指针，任何类型的切片头部信息底层都是对应 `reflect.SliceHeader` 结构，然后通过更新结构体方式来更新切片信息，从而实现 `a` 对应的 `[]float64` 切片到 `c` 对应的 `[]int` 类型切片的转换。

  > 个人觉得第二种方法更好理解

  通过基准测试，我们可以发现用 `sort.Ints` 对转换后的 `[]int` 排序的性能要比用 `sort.Float64s` 排序的性能好一点。不过需要注意的是，这个方法可行的前提是要保证 `[]float64` 中没有 NaN 和 Inf 等非规范的浮点数（因为浮点数中 NaN 不可排序，正 0 和负 0 相等，但是整数中没有这类情形）。

