# 大数据开发-Spark-拷问灵魂的5个问题

# 1.Spark计算依赖内存，如果目前只有10g内存，但是需要将500G的文件排序并输出，需要如何操作？

&#x20;    ①、把磁盘上的500G数据分割为100块（chunks），每份5GB。（注意，要留一些系统空间！）  &#x20;

②、顺序将每份5GB数据读入内存，使用quick sort算法排序。  &#x20;

③、把排序好的数据（也是5GB）存放回磁盘。  &#x20;

④、循环100次，现在，所有的100个块都已经各自排序了。（剩下的工作就是如何把它们合并排序！）  &#x20;

⑤、从100个块中分别读取5G/100=0.05 G入内存（100input buffers）。  &#x20;

⑥、执行100路合并，并将合并结果临时存储于5g基于内存的输出缓冲区中。当缓冲区写满5GB时，写入硬盘上最终文件，并清空输出缓冲区；当100个输入缓冲区中任何一个处理完毕时，写入该缓冲区所对应的块中的下一个0.05 GB，直到全部处理完成。

# 2.countByValue和countByKey的区别

首先从源码角度来看：

```scala
// PairRDDFunctions.scala
def countByKey(): Map[K, Long] = self.withScope {
  self.mapValues(_ => 1L).reduceByKey(_ + _).collect().toMap
}

// RDD.scala
def countByValue()(implicit ord: Ordering[T] = null): Map[T, Long] = withScope {
  map(value => (value, null)).countByKey()
} 
```

`countByValue（RDD.scala）`

-   作用在普通的`RDD`上
-   其实现过程调用了 `countByKey`

`countByKey（PairRDDFunctions.scala）`

-   作用在 PairRDD 上
-   对 key 进行计数
-   数据要收到Driver端，结果集大时，不适用

问题：

-   `countByKey `可以作用在 普通的`RDD`上吗
-   `countByValue `可以作用在 `PairRDD `上吗

```scala
val rdd1: RDD[Int] = sc.makeRDD(1 to 10)
val rdd2: RDD[(Int, Int)] = sc.makeRDD((1 to 10).toList.zipWithIndex)

val result1 = rdd1.countByValue() //可以
val result2 = rdd1.countByKey() //语法错误

val result3 = rdd2.countByValue() //可以
val result4 = rdd2.countByKey() //可以
```

# 3.两个rdd join 什么时候有shuffle什么时候没有shuffle

其中join操作是考验所有数据库性能的一项重要指标，对于Spark来说，考验join的性能就是Shuffle,Shuffle 需要经过磁盘和网络传输，Shuffle数据越少性能越好，有时候可以尽量避免程序进行Shuffle ,那么什么情况下有Shuffle ，什么情况下没有Shuffle 呢

## 3.1 Broadcast join

broadcast join 比较好理解，除了自己实现外，`Spark SQL` 已经帮我们默认来实现了，其实就是小表分发到所有`Executors`，控制参数是：`spark.sql.autoBroadcastJoinThreshold` 默认大小是10m, 即小于这个阈值即自动使用`broadcast join`.

# 3.2 Bucket join

其实rdd方式和table类似，不同的是后者要写入Bucket表，这里主要讲rdd的方式，原理就是，当两个rdd根据相同分区方式，预先做好分区，分区结果是一致的，这样就可以进行Bucket join, 另外这种join没有预先的算子，需要在写程序时候自己来开发，对于表的这种join可以看一下 [字节跳动在Spark SQL上的核心优化实践](https://juejin.cn/post/6844903989557854216 "字节跳动在Spark SQL上的核心优化实践") 。可以看下下面的例子

rdd1、rdd2都是Pair RDD

rdd1、rdd2的数据完全相同

> 一定有shuffle

rdd1 => 5个分区

rdd2 => 6个分区



rdd1 => 5个分区 => (1, 0), (2,0), || (1, 0), (2,0), || (1, 0), (2,0), || (1, 0), (2,0),(1, 0), || (2,0),(1, 0), (2,0)

rdd2 => 5个分区 => (1, 0), (2,0), || (1, 0), (2,0), || (1, 0), (2,0), || (1, 0), (2,0),(1, 0), || (2,0),(1, 0), (2,0)

> 一定没有shuffle



rdd1 => 5个分区 => （1,0), （1,0), （1,0), （1,0), （1,0), || (2,0), (2,0), (2,0), (2,0), (2,0), (2,0), (2,0) || 空 || 空 || 空

rdd2 => 5个分区 => （1,0), （1,0), （1,0), （1,0), （1,0), || (2,0), (2,0), (2,0), (2,0), (2,0), (2,0), (2,0) || 空 || 空 || 空

这样所有`Shuffle`的算子，如果数据提前做好了分区（`partitionBy`），很多情况下没有`Shuffle`.

除上面两种方式外，一般就是有`Shuffle`的`join`, 关于spark的join原理可以查看：[大数据开发-Spark Join原理详解](<大数据开发-Spark Join原理详解_tYc7uVdj1arV952G85EF3U.md> "大数据开发-Spark Join原理详解")

# 4..transform 是不是一定不触发action

有个算子例外，那就是sortByKey,其底层有个抽样算法，水塘抽样，最后需要根据抽样的结果，进行RangePartition的,所以从job角度来说会看到两个job，除了触发action的本身算子之外，记住下面的&#x20;

sortByKey  → 水塘抽样→ collect

# 5.广播变量是怎么设计的

我们都知道，广播变量是把数据放到每个excutor上，也都知道广播变量的数据一定是从driver开始出去的，什么意思呢，如果广播表放在hive表中，那么它的存储就是在各个block块上，也对应多个excutor (不一样的叫法)，首先将数据拉到driver上，然后再进行广播，广播时候不是全部广播，是根据excutor预先用到数据的，首先拿数据，然后通过bt协议进行传输，什么是bt协议呢，就是数据在分布式点对点网络上，根据网络距离来去拉对应的数据，下载者也是上传者，这样就不同每个task （excutor）都从driver上来拉数据，这样就减少了压力，另外在spark1.几的时候还是task级别，现在是共同的一个锁，整个excutor上的task共享这份数据。

# 参考

[https://juejin.cn/post/6844903989557854216](https://juejin.cn/post/6844903989557854216 "https://juejin.cn/post/6844903989557854216")

[https://www.jianshu.com/p/6bf887bf52b2](https://www.jianshu.com/p/6bf887bf52b2 "https://www.jianshu.com/p/6bf887bf52b2")



<https://www.youtube.com/watch?v=6BD-Vv-ViBw>
