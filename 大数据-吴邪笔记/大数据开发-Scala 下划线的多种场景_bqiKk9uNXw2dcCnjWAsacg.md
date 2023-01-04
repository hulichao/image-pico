# 大数据开发-Scala 下划线的多种场景

# 1.简述

Scala 的下划线在各种语法中几乎都要插一脚，其目的是代表某些特殊场合或者简化代码，不必去想命名该怎么取。下面介绍几种下划线的使用场合

# 2. \_ 有哪些使用方式

## 2.1 初始化变量

跟`Java `类似，成员变量未初始化会给一个默认值，`Scala`中也一样，只可以初始化成员变量，但是需要利用\_来特别说明，要注意的是\_如果初始化为`null `要特别指明变量的类型，否则变量类型就是`Null`, 初始化只针对`var `而不能是`val`, 其他情况使用变量类似和\_即可达到初始化的效果

```scala
// _ 对应的默认值:整型默认值0；浮点型默认值0.0；String与引用类型，默认值null; Boolean默认值false
class Student{
    //String类型的默认值为null
    var name : String = _
    var age: Int = _
    var amount: Double = _
    var mOrF: Boolean = _
}
```

## 2.2 方法转为函数

严格的说：使用 val 定义的是函数(function)，使用 def 定义的是方法(method)。二者在语义上的区别很小，在绝大
多数情况下都可以不去理会它们之间的区别，但是有时候有必要了解它们之间的转化，方法转换为函数使用下面的方式

```scala
scala> def f1 = ()=>{}
scala> val f2 = f1 _ 
```

## 2.3 导包

类似`Java `中的\*，可以通过此方式导入包中的所有内容

```scala
//Scala
import java.util._
//Java
import java.util.*; 
```

## 2.4 高阶函数中省去变量名

在Scala中的高阶函数如map , collection, count，sortWith, filter, reduce等，都需传入一个函数，函数的参数名字本身没有特别的用意，所以不必再起名上纠结，直接使用\_来代替参数，但是要注意单次使用，和多个参数时候的问题

```scala
val list = List(3,3,5)
list.reduce(_+_) //等同于list.sum()
list.map(_ * 2)
list.filter(_ > 3) 
```

## 2.5 访问元组

使用\_1 , \_2的方式来访问元组中的各个元素

```scala
val tu = (1,2,3)
tu._1
tu._2 
```

## 2.6 集合转为多个参数

可以再数组或者集合使用\_:\*来转为多个参数来使用

```scala
def addSum(nums: Int*) = {
  nums.sum
}

addSum(1 to 10: _*))
```

## 2.7 setter方法的实现

在变量名\_的方式定义`setter`方法,

可以看出来\_leg 是彻底的封装，而leg\_是leg方法的set版本

```scala
class Dog {
  private var _leg = 0
  def leg: Int = _leg
  def leg_=(newLag: Int) = {
    _leg = newLag
  }

  def get() = {
    _leg
  }
}
object GetterAndSettre {
  def main(args: Array[String]): Unit = {
    val dog = new Dog
    dog.leg_=(4) //等同于 dog.leg = 4 ,都是修改了_leg的值
    println(dog.get())
    dog.leg = 5
    println(dog.get())

  }

}
```

## 2.8 部分函数使用

部分应用函数（Partial Applied Function）也叫偏应用函数，部分应用函数是指缺少部分（甚至全部）参数的函数。如果一个函数有n个参数, 而为其提供少于n个参数, 那就得到了一个部分应用函数

```scala
// 定义一个函数
def add(x:Int, y:Int, z:Int) = x+y+z
// Int不能省略
def addX = add(1, _:Int, _:Int)
addX(2,3)
addX(3,4)
def addXAndY = add(10, 100, _:Int)
addXAndY(1)
def addZ = add(_:Int, _:Int, 10)
addZ(1,2)
// 省略了全部的参数，下面两个等价。第二个更常用
def add1 = add(_: Int, _: Int, _: Int)
def add2 = add _ 
```

## 2.9 模式匹配

在模式匹配中，\_可以指代默认值，类型匹配的时候，可以使用\_，可以省去起名字

```scala
val a = 10
a match {
  case _: Int => println("Int")
  case _ => println("defalult")
} 
```

## 3. 总结

上面主要讲了\_的九种用法，其大大简化了Scala的变量命名和开发过程，多用Scala 来简化代码，也是一个Scala程序员的必修课，当然写写Java式的Scala 可能更易懂把，哈哈哈哈
