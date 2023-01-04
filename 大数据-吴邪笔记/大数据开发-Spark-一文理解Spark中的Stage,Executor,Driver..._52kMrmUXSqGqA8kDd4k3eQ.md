# 大数据开发-Spark-一文理解Spark中的Stage,Executor,Driver...

# 1.引言吧

阿西吧，对于Spark新手来说，首先对于Spark的运行机制不了解，往往跟你交流的时候，互相都不知道在说什么，比如部署模式和运行模式，可能都混为一谈，对于有一定开发经验的老手，即使知道运行机制，可能在表述上，对Spark的各种术语也不是很懂，因此理解Spark术语，是Spark开发者之间沟通的必要之路，本文从Spark的运行机制开始，到WordCount案例来理解Spark中的各种术语。

# 2.Spark的运行机制

首先拿官网的一张图，来说明，其是分布式集群上spark应用程序的一般执行框架。主要由sparkcontext（spark上下文）、cluster manager(资源管理器)和▪executor（单个节点的执行进程）。其中cluster manager负责整个集群的统一资源管理。executor是应用执行的主要进程，内部含有多个task线程以及内存空间。

![](image/image_nfxD6UD77B.png)

Spark的主要运行流程如下：

1.  应用程序在使用spark-submit提交后，根据提交时的参数设置（deploy mode）在相应位置初始化sparkcontext，即spark的运行环境，并创建DAG Scheduler和Task Scheduer，Driver根据应用程序执行代码，将整个程序根据action算子划分成多个job，每个job内部构建DAG图，DAG Scheduler将DAG图划分为多个stage，同时每个stage内部划分为多个task，DAG Scheduler将taskset传给Task Scheduer，Task Scheduer负责集群上task的调度。至于stage和task的关系以及是如何划分的我们后面再详细讲。
2.  &#x20;Driver根据sparkcontext中的资源需求向resource manager申请资源，包括executor数及内存资源。
3.  资源管理器收到请求后在满足条件的work node节点上创建executor进程。
4.  Executor创建完成后会向driver反向注册，以便driver可以分配task给他执行。
5.  当程序执行完后，driver向resource manager注销所申请的资源。

# 3.理解Spark中的各个名词术语

从运行机制上，我们来继续解释下面的名词术语，

## 3.1 Driver program

driver就是我们编写的spark应用程序，用来创建sparkcontext或者sparksession，driver会和cluster mananer通信，并分配task到executor上执行

## 3.2 Cluster Manager

负责整个程序的资源调度，目前的主要调度器有：

YARN

Spark Standalone

Mesos

## 3.3 Executors

Executors其实是一个独立的JVM进程，在每个工作节点上会起一个，主要用来执行task，一个executor内，可以同时并行的执行多个task。

## 3.4 Job

Job是用户程序一个完整的处理流程，是逻辑的叫法。

## 3.5 Stage

一个Job可以包含多个Stage，Stage之间是串行的，State的触发是由一些shuffle，reduceBy，save动作产生的

## 3.6 Task

一个Stage可以包含多个task，比如sc.textFile("/xxxx").map().filter()，其中map和filter就分别是一个task。每个task的输出就是下一个task的输出。

## 3.7 Partition

partition是spark里面数据源的一部分，一个完整的数据源会被spark切分成多个partition以方便spark可以发送到多个executor上去并行执行任务。

## 3.8 RDD

RDD是分布式弹性数据集，在spark里面一个数据源就可以看成是一个大的RDD，RDD由多个partition组成，spark加载的数据就会被存在RDD里面，当然在RDD内部其实是切成多个partition了。

那么问题来了一个spark job是如何执行的？

（1）我们写好的spark程序，也称驱动程序，会向Cluster Manager提交一个job

（2）Cluster Manager会检查数据本地行并寻找一个最合适的节点来调度任务

（3）job会被拆分成不同stage，每个stage又会被拆分成多个task

（4）驱动程序发送task到executor上执行任务

（5）驱动程序会跟踪每个task的执行情况，并更新到master node节点上，这一点我们可以在spark master UI上进行查看

（6）job完成，所有节点的数据会被最终再次聚合到master节点上，包含了平均耗时，最大耗时，中位数等等指标。

## 3.9 部署模式和运行模式

部署模式 就是说的，Cluster Manager，一般有Standalone, Yarn ,而运行模式说的是Drvier的运行机器，是集群还是提交任务的机器，分别对应Cluster和Client模式，区别在于运行结果，日志，稳定性等。

# 4. 从WordCount案例来理解各个术语

### 再次理解相关概念

-   Job：Job是由Action触发的，因此一个Job包含一个Action和N个Transform操作；
-   Stage：Stage是由于shuffle操作而进行划分的Task集合，Stage的划分是根据其宽窄依赖关系；
-   Task：最小执行单元，因为每个Task只是负责一个分区的数据 &#x20;

    处理，因此一般有多少个分区就有多少个Task，这一类的Task其实是在不同的分区上执行一样的动作；

下面是一段WordCount程序

```scala
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.rdd.RDD

object WordCount {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setMaster("yarn").setAppName("WordCount")
    val sc = new SparkContext(conf)
    val lines1: RDD[String] = sc.textFile("data/spark/wc.txt")
    val lines2: RDD[String] = sc.textFile("data/spark/wc2.txt")
    val j1 = lines1.flatMap(_.split(" ")).map((_, 1)).reduceByKey(_+_)
    val j2 = lines2.flatMap(_.split(" ")).map((_, 1)).reduceByKey(_+_)
    j1.join(j2).collect()
    sc.stop()
  }
}
```

Yarn模式在生产环境用的较多，因此从Yarn的部署模式来看，代码上只有一个action操作collect，所以只有一个Job, Job又由于Shuffle的原因被划分为3个stage, 分别是flatMap 和 map 和 reduceBykey 算一个Stage0,  另外的line2又算一个，Stage1, 而Stage3 是前面两个结果join，然后collect, 且stage3依赖于 stage1 和 stage0, 但stage0 和 stage1 是并行的，在实际的生产环境下，要去看依赖stage的依赖图，可以明显看到依赖的关系。

