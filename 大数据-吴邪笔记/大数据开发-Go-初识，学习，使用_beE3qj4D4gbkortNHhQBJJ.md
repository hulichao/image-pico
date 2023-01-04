# 大数据开发-Go-初识，学习，使用

2018年，初识Go，用它写过一个区块链项目，不过也仅仅止步于此，后续工作都是Java和Scala居多，不过Go发展算是较晚的语言，以其简洁，快等特点很快被大众所熟知，一直认为它是一门非常优秀的语言，后续的工作场景也使用较多Go，重拾起来也不算太难，本文学习主要参考Go 101 系列，在文末有资料链接

# 为什么是Go - Go的特点

Go提供了几种基本但非必需的类型，比如切片，接口和通道。

Go简单不是它的主要卖点，做为一门静态语言，Go却和很多动态脚本语言一样得灵活是Go的主要卖点，节省内存、程序启动快和代码执行速度快合在一块儿是Go的另一个主要卖点，Go是一门编译型的和静态的编程语言。 Go诞生于谷歌研究院

-   内置并发编程支持：
    -   使用协程（goroutine）做为基本的计算单元。轻松地创建协程。
    -   使用通道（channel）来实现协程间的同步和通信。
-   内置了映射（map）和切片（slice）类型。
-   支持多态（polymorphism）。
-   使用接口（interface）来实现裝盒（value boxing）和反射（reflection）。
-   支持指针。
-   支持函数闭包（closure）。
-   支持方法。
-   支持延迟函数调用（defer）。
-   支持类型内嵌（type embedding）。
-   支持类型推断（type deduction or type inference）。
-   内存安全。
-   自动垃圾回收。
-   良好的代码跨平台性。

&#x20;编译时间的长短是开发愉悦度的一个重要因素。 编译时间短是很多程序员喜欢Go的一个原因

# 运行第一个go程序

```bash
## 运行go程序
go run 

## 打包go程序，生成可执行文件
go build 

## 来安装一个第三方Go程序的最新版本，（至GOBIIN目录） 在Go官方工具链1.16版本之前，对应的命令是go get -u example.com/program（现在已经被废弃而不再推荐被使用了
go install 

## 检查可能的代码逻辑错误
go vet

## 获取网络的依赖包,用拉添加、升级、降级或者删除单个依赖,不如go mod tidy常用？
go get -u

## 生成go.mod 文件，依赖到的模块
go mod init Demo.go

## 扫描当前项目中的所有代码来添加未被记录的依赖至go.mod文件或从go.mod文件中删除不再被使用的依赖
go mod tidy

## 格式化源文件代码
go fmt Demo.go 

##  运行单元和基准测试用例
go test

##  查看Go代码库包的文档
go doc 

## 运行go help aSubCommand来查看一个子命令aSubCommand的帮助信息 
go help 
```

# GOROOT和GOPATH

GOROOT是Go语言环境的安装路径，在安装开发环境时已经确定
GOPATH是当前项目工程的开发路径，GOPATH可以有多个，每个GOPATH下的一般有三个包，pkg、src和bin，src用于存放项目工程的源代码文件，pkg文件夹下的文件在编译时自动生成，bin目录下生成\*.exe的可执行文件。
PS：每一个GOPATH下都可以有pkg、src、bin三个文件夹，当设置多个GOPATH时，当前GOPATH的src源文件编译结果和生成的可执行文件会存储在最近路径的GOPATH的pkg和bin文件夹下，即当前GOPATH下,开发时在src目录下新建目录并建立源代码文件，目录名称和源文件名称可以不同，源文件内第一行代码package pkgName中的pkgName也可以和源文件所在文件夹名称不同。但是，如果此包需要在其他包中使用，编译器会报错，建议package 后的名称和文件所在文件夹的名称相同。一般只有main函数所在的源文件下才会出现所在包和“package 包名”声明的包名不同的情况

最终import 的包需要时package中写的而不是目录名，否则会报错

# Go的25个关键字

```bash
break     default      func    interface  select
case      defer        go      map        struct
chan      else         goto    package    switch
const     fallthrough  if      range      type
continue  for          import  return     var
```

-   `const`、`func`、`import`、`package`、`type`和`var`用来声明各种代码元素。
-   `chan`、`interface`、`map`和`struct`用做 一些组合类型的字面表示中。
-   `break`、`case`、`continue`、`default`、 `else`、`fallthrough`、`for`、 `goto`、`if`、`range`、 `return`、`select`和`switch`用在流程控制语句中。 详见[基本流程控制语法](https://gfw.go101.org/article/control-flows.html "基本流程控制语法")。
-   `defer`和`go`也可以看作是流程控制关键字， 但它们有一些特殊的作用。详见[协程和延迟函数调用](https://gfw.go101.org/article/control-flows-more.html "协程和延迟函数调用")。

**ps**: uintptr、int以及uint类型的值的尺寸依赖于具体编译器实现。 通常地，在64位的架构上，int和uint类型的值是64位的；在32位的架构上，它们是32位的。 编译器必须保证uintptr类型的值的尺寸能够存下任意一个内存地址

# 常量自动补全和iota

在一个包含多个常量描述的常量声明中，除了第一个常量描述，其它后续的常量描述都可以只有标识符部分。 Go编译器将通过照抄前面最紧挨的一个完整的常量描述来自动补全不完整的常量描述。 比如，在编译阶段，编译器会将下面的代码

```go
const (
  X float32 = 3.14
  Y                // 这里必须只有一个标识符
  Z                // 这里必须只有一个标识符

  A, B = "Go", "language"
  C, _
  // 上一行中的空标识符是必需的（如果
  // 上一行是一个不完整的常量描述）。
)
```

自动补全为

```go
const (
  X float32 = 3.14
  Y float32 = 3.14
  Z float32 = 3.14

  A, B = "Go", "language"
  C, _ = "Go", "language"
)
```

`iota`是Go中预声明（内置）的一个特殊的有名常量。 `iota`被预声明为`0`，但是它的值在编译阶段并非恒定。 当此预声明的`iota`出现在一个常量声明中的时候，它的值在第n个常量描述中的值为`n`（从0开始）。 所以`iota`只对含有多个常量描述的常量声明有意义。

`iota`和常量描述自动补全相结合有的时候能够给Go编程带来很大便利。 比如，下面是一个使用了这两个特性的例子

```go
const (
  Failed = iota - 1 // == -1
  Unknown           // == 0
  Succeeded         // == 1
)

const (
  Readable = 1 << iota // == 1
  Writable             // == 2
  Executable           // == 4
)
```

可以看出，iota可以在状态值常量的应用上很方便，但是注意是在编译时候使用哦

# 变量声明方式

go语言中主要有下面两种声明方式

## 标准变量声明方式

```go
var (
  lang, bornYear, compiled     = "Go", 2007, true
  announceAt, releaseAt    int = 2009, 2012
  createdBy, website       string
)
```

注意，Go声明的局部变量要被有效使用一次，包变量没有限制，另外不支持其他语言所支持的连等赋值，如下图：

```go
var a, b int
a = b = 123 // 语法错误
```

## 短变量声明方式

```go
package main

func main() {
  // 变量lang和year都为新声明的变量。
  lang, year := "Go language", 2007

  // 这里，只有变量createdBy是新声明的变量。
  // 变量year已经在上面声明过了，所以这里仅仅
  // 改变了它的值，或者说它被重新声明了。
  year, createdBy := 2009, "Google Research"

  // 这是一个纯赋值语句。
  lang, year = "Go", 2012

  print(lang, "由", createdBy, "发明")
  print("并发布于", year, "年。")
  println()
}
```

`：= ` 这种方式声明的还有个特点就是，短声明语句中必须至少有一个新声明的变量，可以支持前面部分声明过的变量再继续声明赋值

# 函数声明和调用

分别为 `func` 函数名 参数 返回值 函数体（包含返回值或者不包含）

其他都好理解，有一点需要注意在Go中和其他语言不同之处，就是返回值如果匿名声明，那么return 的时候需要明确返回变量或者某个值，如果是非匿名声明，那么就可以只写`return` 这个和其他语言还是有一点区别的，如果函数没有返回值 ，那么`return`就不用写，Go不支持输入参数默认值。每个返回结果的默认值是它的类型的零值。 比如，下面的函数在被调用时将打印出（和返回）0 false

```go
func f() (x int, y bool) {
  println(x, y) // 0 false
  return
}

// 个人不是很懂这个，理解不了 
```

# Go流程控制语法

三种基本的流程控制代码块：

-   `if-else`条件分支代码块；
-   `for`循环代码块；
-   `switch-case`多条件分支代码块。

还有几种和特定种类的类型相关的流程控制代码块：

-   [容器类型](https://gfw.go101.org/article/container.html#iteration "容器类型")相关的`for-range`循环代码块。
-   [接口类型](https://gfw.go101.org/article/interface.html#type-switch "接口类型")相关的`type-switch`多条件分支代码块。
-   [通道类型](https://gfw.go101.org/article/channel.html#select "通道类型")相关的`select-case`多分支代码块。

# Go并发同步 - `sync.WaitGroup`

并发编程的一大任务就是要调度不同计算，控制它们对资源的访问时段，以使数据竞争的情况不会发生。 此任务常称为并发同步（或者数据同步）。Go支持几种并发同步技术，先学习最简单的一种，`sync`标准库中的`WaitGroup`来同步主协程和创建的协程

`WaitGroup`类型有三个方法（特殊的函数，将在以后的文章中详解）：`Add`、`Done`和`Wait`。 此类型将在后面的某篇文章中详细解释，目前我们可以简单地认为：

-   `Add`方法用来注册新的需要完成的任务数。
-   `Done`方法用来通知某个任务已经完成了。
-   一个`Wait`方法调用将阻塞（等待）到所有任务都已经完成之后才继续执行其后的语句

```go
package main

import (
  "log"
  "math/rand"
  "time"
  "sync"
)

var wg sync.WaitGroup

func SayGreetings(greeting string, times int) {
  for i := 0; i < times; i++ {
    log.Println(greeting)
    d := time.Second * time.Duration(rand.Intn(5)) / 2
    time.Sleep(d)
  }
  wg.Done() // 通知当前任务已经完成。
}

func main() {
  rand.Seed(time.Now().UnixNano())
  log.SetFlags(0)
  wg.Add(2) // 注册两个新任务。
  go SayGreetings("hi!", 10)
  go SayGreetings("hello!", 10)
  wg.Wait() // 阻塞在这里，直到所有任务都已完成。
}
```

可以看到一个活动中的协程可以处于两个状态：运行状态和阻塞状态。一个协程可以在这两个状态之间切换

![](https://files.catbox.moe/8n4fb0.png)

# 协程的调度

编译器采纳了一种被称为[M-P-G模型](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw "M-P-G模型")的算法来实现协程调度。 其中，M表示系统线程，P表示逻辑处理器（并非上述的逻辑CPU），G表示协程。 大多数的调度工作是通过逻辑处理器（P）来完成的。 逻辑处理器像一个监工一样通过将不同的处于运行状态协程（G）交给不同的系统线程（M）来执行。 一个协程在同一时刻只能在一个系统线程中执行。一个执行中的协程运行片刻后将自发地脱离让出一个系统线程，从而使得其它处于等待子状态的协程得到执行机会，如下图，P可以理解为总司令，G是程序写好的逻辑协程，而M是具体的系统线程执行器

&#x20;&#x20;

![](https://files.catbox.moe/jnuopa.png)

# 参考：

Go语言101 ： [https://gfw.go101.org/article/101-about.html](https://gfw.go101.org/article/101-about.html "https://gfw.go101.org/article/101-about.html")
