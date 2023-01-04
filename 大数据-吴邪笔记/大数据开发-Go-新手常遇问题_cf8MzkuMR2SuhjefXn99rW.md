# 大数据开发-Go-新手常遇问题

函数声明和调用

# Go 中指针使用注意点

```go
// 1.空指针反向引用不合法
package main
func main() {
    var p *int = nil
    *p = 0
}
// in Windows: stops only with: <exit code="-1073741819" msg="process crashed"/>
// runtime error: invalid memory address or nil pointer dereference

// 2.文字或者常量引用也不合法
const i = 5
ptr := &i //error: cannot take the address of i
ptr2 := &10 //error: cannot take the address of 10

```

# Go语言常见内置函数

## sort

```go
// sort 包
import "sort"
sort.Strings(keys)

```

## close 用于管道通信，select 用于通信的switch

```go
type T int
func main() {
  c := make(chan T)
  close(c)
}

// select 用法
var c1, c2, c3 chan int
var i1, i2 int
select {
  case i1 = <-c1:
     fmt.Printf("received ", i1, " from c1\n")
  case c2 <- i2:
     fmt.Printf("sent ", i2, " to c2\n")
  case i3, ok := (<-c3):  // same as: i3, ok := <-c3
     if ok {
        fmt.Printf("received ", i3, " from c3\n")
     } else {
        fmt.Printf("c3 is closed\n")
     }
  default:
     fmt.Printf("no communication\n")
}    

```

## len、cap&#x20;

len 用于返回某个类型的长度或数量（字符串、数组、切片、map 和管道）；

cap 是容量的意思，用于返回某个类型的最大容量（只能用于切片和 map）

## new、make

new 和 make 均是用于分配内存：new 用于值类型和用户定义的类型，如自定义结构，make 用于内置引用类型（切片、map 和管道）。

它们的用法就像是函数，但是将类型作为参数：new (type)、make (type)。new (T) 分配类型 T 的零值并返回其地址，也就是指向类型 T 的指针。

它也可以被用于基本类型：v := new(int)。make (T) 返回类型 T 的初始化之后的值，因此它比 new 进行更多的工作，new () 是一个函数，不要忘记它的括号

## copy、append

用于复制和连接切片

## panic、recover

两者均用于错误处理机制

## print、println

底层打印函数，在部署环境中建议使用 fmt 包

## ~~complex、real、imag~~

~~操作复数，使用场景不多~~

# Go不支持函数重载

Go 语言不支持这项特性的主要原因是函数重载需要进行多余的类型匹配影响性能；没有重载意味着只是一个简单的函数调度。所以你需要给不同的函数使用不同的名字，我们通常会根据函数的特征对函数进行命名

如果需要申明一个在外部定义的函数，你只需要给出函数名与函数签名，不需要给出函数体：

```go
func flushICache(begin, end uintptr) // implemented externally
```

函数也可以以申明的方式被使用，作为一个函数类型，就像：

```go
type binOp func(int, int) int
```

# Go的map遍历时候变量地址一直用的是同一个

最佳实践：读数据可以用key，value，写数据用地址，如果要把地址赋值给另外的map，那么需要用临时变量

```go
  kvMap := make(map[int]int)
  kvMap[0] = 100
  kvMap[1] = 101
  kvMap[2] = 102
  kvMap[3] = 103
  
  for k, v := range kvMap {
    println(k, &k, v, &v)
  }
 
// 0 0xc000049e50 100 0xc000049e48
// 1 0xc000049e50 101 0xc000049e48
// 2 0xc000049e50 102 0xc000049e48
// 3 0xc000049e50 103 0xc000049e48 
```

# Go遍历的key，value是值，而不是地址

```go
// Version A:
items := make([]map[int]int, 5)
for i:= range items {
    items[i] = make(map[int]int, 1)
    items[i][1] = 2
}
fmt.Printf("Version A: Value of items: %v\n", items)

// Version B: NOT GOOD!
items2 := make([]map[int]int, 5)
for _, item := range items2 {
    item = make(map[int]int, 1) // item is only a copy of the slice element.
    item[1] = 2 // This 'item' will be lost on the next iteration.
}
fmt.Printf("Version B: Value of items: %v\n", items2)


```

应当像 A 版本那样通过索引使用切片的 map 元素。在 B 版本中获得的项只是 map 值的一个拷贝而已，所以真正的 map 元素没有得到初始化

# 锁和 sync 包

并发编程在大部分语言里面都会有，用来解决多个线程对临界资源的访问，经典的做法是一次只能让一个线程对共享变量进行操作。当变量被一个线程改变时 (临界区)，我们为它上锁，直到这个线程执行完成并解锁后，其他线程才能访问它，Go语言的这种锁机制是通过sync包中的Mutex来实现访问的，下面是一个例子，另外在sync包中还有一个读写锁，RWMutex ，其写互斥的方法与Mutex一样，读互斥采用下面的方法

```go
mu sync.Mutex

func update(a int) {
  mu.Lock()
  a = xxx
  mu.Unlock()
}

mu2 sync.RWMutex
mu2.RLock()
mu2.RUnlock()
```
