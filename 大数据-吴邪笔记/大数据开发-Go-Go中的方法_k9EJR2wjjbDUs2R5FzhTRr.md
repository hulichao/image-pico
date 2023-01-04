# 大数据开发-Go-Go中的方法

我们知道Go中没有继承，接口的用法也与Java中的用法大相径庭，很多适合，我们需要使用OO的思想来组织我们达到项目，但是将Java的oo思想在Go中会比较痛苦，Go中的方法和面向对象的类似，但是定义上也是很多初学者不好理解的，比如方法的定义竟然在结构体外部，组装起来看起来也很随意，其实Go只是有面向对象特性，其实不算是面向对象语言。

# 方法

Go支持一些面向对象编程特性，方法是这些所支持的特性之一。 本篇文章将介绍在Go中和方法相关的各种概念

# 方法声明

在Go中，我们可以为类型`T`和`*T`显式地声明一个方法，其中类型`T`必须满足四个条件：

1.  `T`必须是一个[定义类型](https://gfw.go101.org/article/type-system-overview.html#non-defined-type "定义类型")；
2.  `T`必须和此方法声明定义在同一个代码包中；
3.  `T`不能是一个指针类型；
4.  `T`不能是一个接口类型。

类型`T`和`*T`称为它们各自的方法的属主类型（receiver type）。 类型`T`被称作为类型`T`和`*T`声明的所有方法的属主基类型（receiver base type）

如果我们为某个类型声明了一个方法，以后我们可以说此类型拥有此方法，一个方法声明和一个函数声明很相似，但是比函数声明多了一个额外的参数声明部分。 此额外的参数声明部分只能含有一个类型为此方法的属主类型的参数，此参数称为此方法声明的属主参数（receiver parameter）。 此属主参数声明必须包裹在一对小括号()之中。 此属主参数声明部分必须处于func关键字和方法名之间

下面是一个方法声明的例子：

```go
// Age和int是两个不同的类型。我们不能为int和*int
// 类型声明方法，但是可以为Age和*Age类型声明方法。
type Age int
func (age Age) LargerThan(a Age) bool {
  return age > a
}
func (age *Age) Increase() {
  *age++
}

// 为自定义的函数类型FilterFunc声明方法。
type FilterFunc func(in int) bool
func (ff FilterFunc) Filte(in int) bool {
  return ff(in)
}

// 为自定义的映射类型StringSet声明方法。
type StringSet map[string]struct{}
func (ss StringSet) Has(key string) bool {
  _, present := ss[key]
  return present
}
func (ss StringSet) Add(key string) {
  ss[key] = struct{}{}
}
func (ss StringSet) Remove(key string) {
  delete(ss, key)
}

// 为自定义的结构体类型Book和它的指针类型*Book声明方法。
type Book struct {
  pages int
}
func (b Book) Pages() int {
  return b.pages
}
func (b *Book) SetPages(pages int) {
  b.pages = pages
}
```

# 每个方法对应着一个隐式声明的函数

对每个方法声明，编译器将自动隐式声明一个相对应的函数。 比如对于上一节的例子中为类型`Book`和`*Book`声明的两个方法，编译器将自动声明下面的两个函数：

```go
func Book.Pages(b Book) int {
  return b.pages // 此函数体和Book类型的Pages方法体一样
}

func (*Book).SetPages(b *Book, pages int) {
  b.pages = pages // 此函数体和*Book类型的SetPages方法体一样
}
```

在上面的两个隐式函数声明中，它们各自对应的方法声明的属主参数声明被插入到了普通参数声明的第一位。 它们的函数体和各自对应的显式方法的方法体是一样的。

两个隐式函数名`Book.Pages`和`(*Book).SetPages`都是`aType.MethodName`这种形式的。 我们不能显式声明名称为这种形式的函数，因为这种形式不属于合法标识符。这样的函数只能由编译器隐式声明。 但是我们可以在代码中调用这些隐式声明的函数

# 方法原型（method prototype）和方法集（method set）

一个方法原型可以看作是一个不带func关键字的函数原型。 我们可以把每个方法声明看作是由一个func关键字、一个属主参数声明部分、一个方法原型和一个方法体组成

# 方法值和方法调用

方法事实上是特殊的函数。方法也常被称为成员函数。 当一个类型拥有一个方法，则此类型的每个值将拥有一个不可修改的函数类型的成员（类似于结构体的字段）。 此成员的名称为此方法名，它的类型和此方法的声明中不包括属主部分的函数声明的类型一致。 一个值的成员函数也可以称为此值的方法。

一个方法调用其实是调用了一个值的成员函数。假设一个值`v`有一个名为`m`的方法，则此方法可以用选择器语法形式`v.m`来表示。

下面这个例子展示了如何调用为`Book`和`*Book`类型声明的方法：

```go
package main

import "fmt"

type Book struct {
  pages int
}

func (b Book) Pages() int {
  return b.pages
}

func (b *Book) SetPages(pages int) {
  b.pages = pages
}

func main() {
  var book Book

  fmt.Printf("%T \n", book.Pages)       // func() int
  fmt.Printf("%T \n", (&book).SetPages) // func(int)
  // &book值有一个隐式方法Pages。
  fmt.Printf("%T \n", (&book).Pages)    // func() int

  // 调用这三个方法。
  (&book).SetPages(123)
  book.SetPages(123)           // 等价于上一行
  fmt.Println(book.Pages())    // 123
  fmt.Println((&book).Pages()) // 123
}
```

和普通参数传参一样，属主参数的传参也是一个值复制过程。 所以，在方法体内对属主参数的直接部分的修改将不会反映到方法体外

# 参考

更多详细内容可以查看 [https://gfw.go101.org/article/101.html](https://gfw.go101.org/article/101.html "https://gfw.go101.org/article/101.html")
