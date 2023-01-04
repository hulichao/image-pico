# 大数据开发-Spark-闭包的理解

# 1.从Scala中理解闭包

闭包是一个函数，返回值依赖于声明在函数外部的一个或多个变量。闭包通常来讲可以简单的认为是可以访问一个函数里面局部变量的另外一个函数。

如下面这段匿名的函数：

```java
val multiplier = (i:Int) => i * 10  
```

函数体内有一个变量 i，它作为函数的一个参数。如下面的另一段代码：

```java
val multiplier = (i:Int) => i * factor
```

在 `multiplier `中有两个变量：i 和 factor。其中的一个 i 是函数的形式参数，在 `multiplier `函数被调用时，i 被赋予一个新的值。然而，factor不是形式参数，而是自由变量，考虑下面代码：

```java
var factor = 3  val multiplier = (i:Int) => i * factor 
```

这里我们引入一个自由变量 `factor`，这个变量定义在函数外面。

这样定义的函数变量 `multiplier `成为一个"闭包"，因为它引用到函数外面定义的变量，定义这个函数的过程是将这个自由变量捕获而构成一个封闭的函数

完整的例子：

```scala
object Test {  
   def main(args: Array[String]) {  
      println( "muliplier(1) value = " +  multiplier(1) )  
      println( "muliplier(2) value = " +  multiplier(2) )  
   }  
   var factor = 3  
   val multiplier = (i:Int) => i * factor  
}
```

# 2.Spark中的闭包理解

先来看下面一段代码：

```scala
val data=Array(1, 2, 3, 4, 5)
var counter = 0
var rdd = sc.parallelize(data)

// ？？？？ 这样做会怎么样
rdd.foreach(x => counter += x)

println("Counter value: " + counter)
```

首先肯定的是上面输出的结果是0，park将RDD操作的处理分解为tasks，每个task由`Executor`执行。在执行之前，Spark会计算task的闭包。闭包是`Executor`在RDD上进行计算的时候必须可见的那些变量和方法（在这种情况下是foreach()）。闭包会被序列化并发送给每个Executor，但是发送给Executor的是副本，所以在Driver上输出的依然是`counter`本身，如果想对全局的进行更新，用累加器，在`spark-streaming`里面使用`updateStateByKey`来更新公共的状态。

另外在Spark中的闭包还有别的作用，

1.清除Driver发送到Executor上的无用的全局变量等，只复制有用的变量信息给Executor，更多可以参考 [读懂Spakr的闭包清理机制](https://toutiao.io/posts/1o5qde/preview "读懂Spakr的闭包清理机制")

可以看下面的map方法：

```scala
def map[U: ClassTag](f: T => U): RDD[U] = withScope {
  val cleanF = sc.clean(f)
  new MapPartitionsRDD[U, T](this, (context, pid, iter) => iter.map(cleanF))
}
```

2.保证发送到Executor上的是序列化以后的数据

比如在使用DataSet时候 case class的定义必须在类下，而不能是方法内，即使语法上没问题，如果使用过json4s来序列化，`implicit val formats = DefaultFormats` 的引入最好放在类下，否则要单独将这个format序列化，即使你没有使用到它别的东西。

# 3.总结

闭包在Spark的整个生命周期中处处可见，就比如从`Driver`上拷贝的所有数据都需要序列化 + 闭包的方式到`Executor`上的。

