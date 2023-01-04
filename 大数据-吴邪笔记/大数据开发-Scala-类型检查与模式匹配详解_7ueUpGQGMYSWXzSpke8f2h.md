# 大数据开发-Scala-类型检查与模式匹配详解

# 0.前言

类型检查和类型转换在每个语言里面都有对应实现，比如`Java `中的`instanceof` 和 `isInstance `，当然Scala语言也有，但是相对于其他语言，`Scala `为了简化开发，产生了强大的模式匹配，其原理和`Java`中的`switch-case`很类似，但是其匹配能力更强，不仅仅可以匹配值，匹配类型，也可以进行类匹配，还可以进行前缀类匹配，而且在`Spark`源码中大量使用了模式匹配，另外的就是隐式转换，在另外一篇文章讲解，本文从类型检查和类型转换入手，引入`Scala`常见的几种模式匹配及其使用。

# 1.类型检查

要测试某个对象是否属于某个给定的类，可以用`isInstanceOf`方法。如果测试成功，可以用`asInstanceOf`方法进行类
型转换。

```scala
if(p.isInstanceOf[Employee]){
//s的类型转换为Employee
val s = p.asInstanceOf[Employee]
}
```

类型转换主要有下面几个点要注意的：

-   如果p指向的是`Employee`类及其子类的对象，则`p.isInstanceOf[Employee]`将会成功。
-   如果p是null，则`p.isInstanceOf[Employee]`将返回false，且`p.asInstanceOf[Employee]`将返回null。
-   如果p不是一个`Employee`，则`p.asInstanceOf[Employee]`将抛出异常。
-   如果想要测试p指向的是一个`Employee`对象但又不是其子类，可以用：`if(p.getClass == classOf[Employee])`
    `classOf`方法定义在`scala.Predef`对象中，因此会被自动引入。不过，与类型检查和转换相比，模式匹配通常是更好的选择。

```scala
p match{
//将s作为Employee处理
case s: Employee => ...
//p不是Employee的情况
case _ => ....
}
```

# 2.模式匹配

Scala 中的模式匹配总结来说支持下面几种匹配：值匹配，类型匹配，集合元素，元组匹配，有值或者无值匹配，下面从代码角度来看看这几种匹配如何使用，首先模式匹配的语法结构如下：

```scala
变量 match {
  case xx => 代码
}
```

和`Java ` 的不同点，不需要指定`break`,即有`break`的效果，使用占位符`_` 来代表默认值，另外`match `和 `if`一样有返回值。

## 2.1 值匹配

值匹配，即类似`Java` 中的整型，字符或者，字符串匹配。但是其之处守卫式匹配（可以理解为默认情况下的条件匹配）

```scala
//字符匹配
def main(args: Array[String]): Unit = {
  val charStr = '6'
  charStr match {
    case '+' => println("匹配上了加号")
    case '-' => println("匹配上了减号")
    case '*' => println("匹配上了乘号")
    case '/' => println("匹配上了除号")
    //注意：不满足以上所有情况，就执行下面的代码
    case _ => println("都没有匹配上，我是默认值")
  }
}

//字符串匹配
def main(args: Array[String]): Unit = {
  val arr = Array("hadoop", "zookeeper", "spark")
  val name = arr(Random.nextInt(arr.length))
  name match {
  case "hadoop" => println("大数据分布式存储和计算框架...")
  case "zookeeper" => println("大数据分布式协调服务框架...")
  case "spark" => println("大数据分布式内存计算框架...")
  case _ => println("我不认识你...")
  }
}

//守卫式匹配
def main(args: Array[String]): Unit = {
  //守卫式
  val character = '*'
  val num = character match {
    case '+' => 1
    case '-' => 2
    case _ if character.equals('*') => 3
    case _ => 4
  }
  println(character + " " + num)
} 
```

## 2.2 类型匹配

类型匹配是相对于`Java `来说的优势点，`Java`是做不到的,匹配格式如下：`case 变量名：类型` ，变量名可以用`_` 来代替

```scala
//类型匹配
  def typeMathc (x: Any) = {
    x match {
      case _: String => println("字符串")
      case _: Int => println("整形")
      case _: Array[Int] => println("正星星数组")
      case _ => println("nothing")
    }
  }
```

## 2.3 匹配数组，元组，集合

不同于类型匹配的是，类型匹配只能匹配到整个大的类型，而这种匹配可以匹配类似像某类，但是又可以限制匹配类的部分元素

```scala
//数组模式匹配
  def arrayMatch(x: Array[Int]) = {
    x match {
      case Array(1,x,y) => println(x + ":" + y)
      case Array(1) => println("only 1 ....")
      case Array(1,_*) => println("1 开头的")
      case _ => println("nothing....")
    }

  }

  //list模式匹配
  def listMatch() = {
    val list = List(3, -1)
    //对List列表进行模式匹配，与Array类似，但是需要使用List特有的::操作符
    //构造List列表的两个基本单位是Nil和::，Nil表示为一个空列表
    //tail返回一个除了第一元素之外的其他元素的列表
    //分别匹配：带有指定个数元素的列表、带有指定元素的列表、以某元素开头的列表
    list match {
      case x :: y :: Nil => println(s"x: $x y: $y")
      case 0 :: Nil => println("only 0")
      case 1 :: _ => println("1 ...")
      case _ => println("something else")
    }
  }

  //元组匹配
  def tupleMatch() = {
    val tuple = (5, 3, 7)
    tuple match {
      case (1, x, y) => println(s"1, $x, $y")
      case (_, z, 5) => println(z)
      case _ => println("else")
    }
  }
```

当数组内没有写值，下面几种匹配等效，任意参数等于完全类型匹配

```scala
case Array(_*) => println("*")
case _: Array[Int] => println("整个数组")
```

## 2.4 样例类匹配

case class样例类是Scala中特殊的类。当声明样例类时，以下事情会自动发生：

-   主构造函数接收的参数通常不需要显式使用var或val修饰，Scala会自动使用val修饰
-   自动为样例类定义了伴生对象，并提供apply方法，不用new关键字就能够构造出相应的对象
-   将生成toString、equals、hashCode和copy方法，除非显示的给出这些方法的定义
-   继承了Product和Serializable这两个特质，也就是说样例类可序列化和可应用Product的方法

case class是多例的，后面要跟构造参数，case object是单例的。

此外，case class样例类中可以添加方法和字段，并且可用于模式匹配。怎么理解样例类的模式匹配呢，在使用动态绑定时候，从样例类的继承中可以判断，某个对象是否属于某个子类对象，而面向父类的接口，可以简化编程的设计。跟第一部分说到的`isInstanceOf` 类似，同时，样例类可以接受输入参数进行对应的子类操作。

```scala
class Amount
//定义样例类Dollar，继承Amount父类
case class Dollar(value: Double) extends Amount
//定义样例类Currency，继承Amount父类
case class Currency(value: Double, unit: String) extends Amount
//定义样例对象Nothing，继承Amount父类
case object Nothing extends Amount
object CaseClassDemo {
  def main(args: Array[String]): Unit = {
    judgeIdentity(Dollar(10.0))
    judgeIdentity(Currency(20.2,"100"))
    judgeIdentity(Nothing)
  }
  //自定义方法，模式匹配判断amt类型
  def judgeIdentity(amt: Amount): Unit = {
    amt match {
      case Dollar(value) => println(s"$value")
      case Currency(value, unit) => println(s"Oh noes,I got $unit")
      case _: Currency => println(s"Oh noes,I go")
      case Nothing => println("Oh,GOD!")
    }
  }
}
```

## 2.5 有值无值匹配

`Scala Option`选项类型用来表示一个值是可选的，有值或无值。
`Option[T]` 是一个类型为 T 的可选值的容器，可以通过`get()`函数获取`Option`的值。如果值存在，`Option[T] `就是一个
`Some`。如果不存在，`Option[T] `就是对象 `None `。
`Option`通常与模式匹配结合使用，用于判断某个变量是有值还是无值，下面以`Map`的自带返回值`Option`来看看这种匹配。

```scala
  val grades = Map("jacky" -> 90, "tom" -> 80, "jarry" -> 95)
  def getGrade(name: String): Unit = {
    val grade: Option[Int] = grades.get(name)
    grade match {
      case Some(grade) => println("成绩：" + grade)
      case None => println("没有此人成绩！")
    }
  }
  def main(args: Array[String]): Unit = {
    getGrade("jacky")
    getGrade("张三")
  }
```

# 3. 总结

`Scala` 中模式匹配，是用来简化代码，实现更多强大的不确定或确定的匹配，同时还可以自定义模式匹配，其匹配时候自动利用了`apply `和 `unapply`方法。
