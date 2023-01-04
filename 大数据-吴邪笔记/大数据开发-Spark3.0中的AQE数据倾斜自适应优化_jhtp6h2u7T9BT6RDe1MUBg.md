# 大数据开发-Spark3.0中的AQE数据倾斜自适应优化

# Spark3-AQE-数据倾斜Join优化

> Adaptive Query Exection(自适应查询计划)简称AQE，在最早在spark 1.6版本就已经有了AQE;到了spark 2.x版本，intel大数据团队进行了相应的原型开发和实践；到了spark 3.0时代，AQE终于面向用户可以使用了，关于倾斜问题，一直是大数据处理中头疼的点，但是其处理方式相对还是有很多套路的，这种方法论自然会被沉淀下来

注：以下代码分析基于Spark3.0.1版本

# 1 Join的自适应数据倾斜处理

> 代码位于sql.core模块的org.apache.spark.sql.execution.adaptive.OptimizeSkewedJoin &#x20;
> 主要原理就是基于需要进行join的两个RDD的每个partition信息，将数据量倾斜的分区进行切分出来再Join。

首先，是否能进行倾斜优化，有几点硬性要求：

-   必须是SortMergeJoin
-   必须是\[Inner,Cross,LeftSemi,LeftAnti,LeftOuter,RightOuter]中的一种Join
    -   left表:Inner,Cross,LeftSemi,LeftAnti,LeftOuter
    -   right表:Inner,Cross,RightOuter
-   left和right的分区数必须一致(这个只要不是异常情况一定可以保证，sortMergeJoin在Mapper端会确保左右两个rdd的partition函数一致，生成的分区数也一定是一致的)

主要流程为：

## 1.1 计算优化后的partition大小

> 根据left或right所有partition的数据分布情况，分别计算出left和right在优化后的partition大小

调用targetSize方法，sizes是每个partition的bytes大小，medianSize表示整个rdd中partition大小的中位数。

变量advisorySize通过spark.sql.adaptive.advisoryPartitionSizeInBytes设置，表示优化后的partition标准大小，默认64MB。

```scala
private def targetSize(sizes: Seq[Long], medianSize: Long): Long = {
  val advisorySize = conf.getConf(SQLConf.ADVISORY_PARTITION_SIZE_IN_BYTES)
  
  val nonSkewSizes = sizes.filterNot(isSkewed(_, medianSize))
  
  // It's impossible that all the partitions are skewed, as we use median size to define skew.
  
  assert(nonSkewSizes.nonEmpty) #要求必须有不倾斜的分片数
  
  math.max(advisorySize, nonSkewSizes.sum / nonSkewSizes.length)

}

```

首先通过调用isSkewed方法来过滤出不倾斜的分片，之后取advisorySize和整个分片的平均值大小作为优化后的分片大小，所以说targetSize也不一定就是我们设置的spark.sql.adaptive.advisoryPartitionSizeInBytes大小。

```scala
private def isSkewed(size: Long, medianSize: Long): Boolean = {

  size > medianSize * conf.getConf(SQLConf.SKEW_JOIN_SKEWED_PARTITION_FACTOR) &&

    size > conf.getConf(SQLConf.SKEW_JOIN_SKEWED_PARTITION_THRESHOLD)

}
```

isSkewed用来判断一个分片是否倾斜，当前分片必须满足大小大于`medianSize*spark.sql.adaptive.skewJoin.skewedPartitionFacto`r(默认5)，并且`size>spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes`(默认256MB)。medianSize表示所有partition的中位数大小。

## 1.2 判断partition是否进行倾斜处理

首先判断partition是否可切分，left的一个分片为例

```scala
val isLeftSkew = isSkewed(leftActualSizes(partitionIndex), leftMedSize) && canSplitLeft
```

直接调用isSkewed，canSplitLeft表示left是否满足\[Inner,Cross,LeftSemi,LeftAnti,LeftOuter]中的一种Join类型。right表判断同理。

然后需要判断一个partition是否是经过coalesce操作的：

```scala
val isLeftCoalesced = leftPartSpec.startReducerIndex + 1 < leftPartSpec.endReducerIndex
```

如果一个partition需要读取的reduceId>=2个，那么认为这个Partition经过AQE的coalesce操作，Spark3.0.1版本对于这种情况不再考虑倾斜处理。

## 1.3 partition的切分操作

以left的一个分区为例，必须满足isLeftSkew && !isLeftCoalesced才会进行split分区操作，否则返回leftPartSpec(原始的分区规则)。

```scala
// A skewed partition should never be coalesced, but skip it here just to be safe.

val leftParts:Seq[CoalescedPartitionSpec] = if (isLeftSkew && !isLeftCoalesced) {//倾斜 &非coalesc才进行自适应分区

  val reducerId = leftPartSpec.startReducerIndex

  val skewSpecs:Option[Seq[PartialReducerPartitionSpec]] = createSkewPartitionSpecs(left.mapStats.shuffleId, reducerId, leftTargetSize)

  if (skewSpecs.isDefined) {

    logDebug(s"Left side partition $partitionIndex is skewed, split it into " +

      s"${skewSpecs.get.length} parts.")

    leftSkewDesc.addPartitionSize(leftActualSizes(partitionIndex))

  }

  skewSpecs.getOrElse(Seq(leftPartSpec))

} else {

  Seq(leftPartSpec)

}
```

首先会调用createSkewPartitionSpecs函数来进行尝试split处理，如果返回Some表示能进行split，将该分区的大小add到leftSkewDesc，用于统计AQE信息。之后返回当前分区信息，如果skewSpecs=None，那么返回Seq长度为1的原始分区规则。

```scala
/**
  - Splits the skewed partition based on the map size and the target partition size
  - after split, and create a list of `PartialMapperPartitionSpec`. Returns None if can't split.
      - 为什么这里代码看起来像是做合并呢，因为mapPartitionSizes对应一个reducePart在上游需要读取的part分区，但是这里将其合并为多个子分区，
      - 每个子分区在AQE之后，都会单独启动一个分区；
      - 合并分片：[0,1,2,3,4,5]->[Part[0,3],Part[3,5]]

  */

private def createSkewPartitionSpecs(

  shuffleId: Int,

  reducerId: Int,

  targetSize: Long): Option[Seq[PartialReducerPartitionSpec]] = {

  val mapPartitionSizes:Array[Long] = getMapSizesForReduceId(shuffleId, reducerId) //获取每个分区的字节数

  // 尝试进行分区合并：比如[0,1,2,3,4,5]->[0,3,5]，012合并为一个分片，34合并为一个分片

  val mapStartIndices:Array[Int] = ShufflePartitionsUtil.splitSizeListByTargetSize(mapPartitionSizes, targetSize)

  if (mapStartIndices.length > 1) { //如果合并后的分区数大于1个，则转换为合并后的partitions，分别对应了mapstatus中的起始和结束索引

    [//mapStartIndices.sliding](//mapStartIndices.sliding)(2).map(t=>PartialReducerPartitionSpec(reducerId, t(0),t(1)))//这么写不是更好吗？

    Some(mapStartIndices.indices.map { i =>

      val startMapIndex = mapStartIndices(i)

      val endMapIndex = if (i == mapStartIndices.length - 1) {

        mapPartitionSizes.length

      } else {

        mapStartIndices(i + 1)

      }

      PartialReducerPartitionSpec(reducerId, startMapIndex, endMapIndex)

    })

  } else {//如果合并后只有一个索引，则不使用

    None

  }

}
```

判断一个split到底切分为多少片其实也是个费劲的过程。其中getMapSizesForReduceId这个方法需要解释一下，我们知道Shuffle过程中，上游的Mapper端生成数据后，是按照reduceId来排序的，并整体放在一个data文件中，同时生成一个索引文件。Reduce端可以根据索引文件中起始的reduceId来读取data中对应片段的数据，由于reduce端会依赖多个mapper，所以这个方法返回了一个Array\[Long]类型，代表需要从对应mapId拉取的bytes大小，mapId就是对应数组下标。通过这个方法我们起其实也可以知道，通过shuffleId+reduceId即可知道当前reduce都需要拉取哪些数据了。

ShufflePartitionsUtil.splitSizeListByTargetSize我们这里不深入讲解，有兴趣的可以自己阅读源码。具体就是按照targetSize来将一个reduce中需要拉取的mapPartitionSizes切分为适合的大小，方法内部主要是一个循环函数，计算相邻的mapSize，是否能合并为targetSize的大小。最终返回mapStartIndices:Array\[Int]，表示哪些mapId放到一个task中进行处理，比如\[0,3,5]，0-1-2合并为一个分片，3-4合并为一个分片，而5单独处理。

之后的流程就比较简单了，如果split完的分片数大于1个，则获取这个分片需要读取的startMapIndex和endMapIndex，封装为PartialReducerPartitionSpec，这样，在实际计算的时候，通过shuffleId+reduceId+startMapIndex+endMapIndex就可以得到对应task需要拉取的数据了。

**通过1.2和1.3的处理，可以分别将获取left和right切分后的part**

## 1.4 分配另一半Partition

> 小标题表述不准确，想表示的是left的某个partition假如split为3份，那么right需要将对应的partition复制3份，分别和left经过split的分片做join。

代码很精简，双层for循环，相当于做了个笛卡尔积：

```scala
for {

  leftSidePartition <- leftParts

  rightSidePartition <- rightParts

} {

  leftSidePartitions += leftSidePartition

  rightSidePartitions += rightSidePartition

}
```

如果left和right都没有经过倾斜优化，那么这段代码中leftSidePartitions和rightSidePartitions分别只有各自原始的分区。看下面的例子，p1代表left或这right的index=1的分片。

如果left split为3份，那么leftSidePartitions=\[p1\_0,p1\_1,p1\_2]，rightSidePartitions=\[p1,p1,p1]。

如果left split为3份，right split为2份，那么leftSidePartitions=\[p1\_0,p1\_0,p1\_1,p1\_1,p1\_2,p1\_2]，rightSidePartitions=\[p1\_0,p1\_1,p1\_0,p1\_1,p1\_0,p1\_1]。left将被fetch和处理2次，而right是3次。

是不是看到了什么缺点？

## 1.5 更新Join计划

新的执行计划建立在left或right有经过倾斜优化的分区，smj代表SortMergeJoin，将优化后的分区规则更新到执行计划中即可。

```scala
if (leftSkewDesc.numPartitions > 0 || rightSkewDesc.numPartitions > 0) {

  val newLeft = CustomShuffleReaderExec(

    left.shuffleStage, leftSidePartitions, leftSkewDesc.toString)

  val newRight = CustomShuffleReaderExec(

    right.shuffleStage, rightSidePartitions, rightSkewDesc.toString)

  smj.copy(

    left = s1.copy(child = newLeft), right = s2.copy(child = newRight), isSkewJoin = true)

} else {

  smj

}
```

# 2 补充

贴上optimizeSkewJoin的核心方法，推荐大家看看源代码：

```scala
/*
  - This method aim to optimize the skewed join with the following steps:
  - 1. Check whether the shuffle partition is skewed based on the median size
  - and the skewed partition threshold in origin smj.
  - 2. Assuming partition0 is skewed in left side, and it has 5 mappers (Map0, Map1...Map4).
  - And we may split the 5 Mappers into 3 mapper ranges [(Map0, Map1), (Map2, Map3), (Map4)]
  - based on the map size and the max split number.
  - 3. Wrap the join left child with a special shuffle reader that reads each mapper range with one
  - task, so total 3 tasks.
  - 4. Wrap the join right child with a special shuffle reader that reads partition0 3 times by
  - 3 tasks separately.

  */

def optimizeSkewJoin(plan: SparkPlan): SparkPlan = plan.transformUp {

  case smj @ SortMergeJoinExec(_, _, joinType, _,

    s1 @ SortExec(_, _, ShuffleStage(left: ShuffleStageInfo), _),

    s2 @ SortExec(_, _, ShuffleStage(right: ShuffleStageInfo), _), _)

    if supportedJoinTypes.contains(joinType) =>

    // 要求left和right分片数必须相同，这在sortMergeJoin中是可以实现的

    assert(left.partitionsWithSizes.length == right.partitionsWithSizes.length)

    val numPartitions = left.partitionsWithSizes.length

    // 获取两个rdd的所有part的bytes大小的中位数

    // Use the median size of the actual (coalesced) partition sizes to detect skewed partitions.

    val leftMedSize = medianSize(left.partitionsWithSizes.map(_._2))

    val rightMedSize = medianSize(right.partitionsWithSizes.map(_._2))

    logDebug(

      s"""

        |Optimizing skewed join.

        |Left side partitions size info:

        |${getSizeInfo(leftMedSize, left.partitionsWithSizes.map(_._2))}

        |Right side partitions size info:

        |${getSizeInfo(rightMedSize, right.partitionsWithSizes.map(_._2))}

      """.stripMargin)

    // 是否可以切分：满足一定的join条件

    val canSplitLeft = canSplitLeftSide(joinType)

    val canSplitRight = canSplitRightSide(joinType)

    // We use the actual partition sizes (may be coalesced) to calculate target size, so that

    // the final data distribution is even (coalesced partitions + split partitions).

    // 分别对两个rdd的每个partition进行数据倾斜优化后的大小

    val leftActualSizes = left.partitionsWithSizes.map(_._2)

    val rightActualSizes = right.partitionsWithSizes.map(_._2)

    val leftTargetSize = targetSize(leftActualSizes, leftMedSize)

    val rightTargetSize = targetSize(rightActualSizes, rightMedSize)

    val leftSidePartitions = mutable.ArrayBuffer.empty[ShufflePartitionSpec]

    val rightSidePartitions = mutable.ArrayBuffer.empty[ShufflePartitionSpec]

    val leftSkewDesc = new SkewDesc

    val rightSkewDesc = new SkewDesc

    // 遍历所有partition，进行合并或者切分

    for (partitionIndex <- 0 until numPartitions) {

      /** 遍历每个part，如果是倾斜，就按照map端分区的index进行split，子分区不会超过targetSize*1.2的大小
        - 左右rdd都需要执行这样的操作，
        - */

      //是否倾斜及是否可切分

      val isLeftSkew = isSkewed(leftActualSizes(partitionIndex), leftMedSize) && canSplitLeft

      val leftPartSpec:CoalescedPartitionSpec = left.partitionsWithSizes(partitionIndex)._1

      //如果一个分片是经过coalesc操作的，那么他的startIndex+1 != endIndex也就是至少读取两个分片

      val isLeftCoalesced = leftPartSpec.startReducerIndex + 1 < leftPartSpec.endReducerIndex

      val isRightSkew = isSkewed(rightActualSizes(partitionIndex), rightMedSize) && canSplitRight

      val rightPartSpec = right.partitionsWithSizes(partitionIndex)._1

      val isRightCoalesced = rightPartSpec.startReducerIndex + 1 < rightPartSpec.endReducerIndex

      // A skewed partition should never be coalesced, but skip it here just to be safe.

      val leftParts = if (isLeftSkew && !isLeftCoalesced) {//倾斜 &非coalesc才进行自适应分区

        val reducerId = leftPartSpec.startReducerIndex

        val skewSpecs:Option[Seq[PartialReducerPartitionSpec]] = createSkewPartitionSpecs(left.mapStats.shuffleId, reducerId, leftTargetSize)

        if (skewSpecs.isDefined) {

          logDebug(s"Left side partition $partitionIndex is skewed, split it into " +

            s"${skewSpecs.get.length} parts.")

          leftSkewDesc.addPartitionSize(leftActualSizes(partitionIndex))

        }

        skewSpecs.getOrElse(Seq(leftPartSpec))

      } else {

        Seq(leftPartSpec)

      }

      // A skewed partition should never be coalesced, but skip it here just to be safe.

      val rightParts:Seq[CoalescedPartitionSpec] = if (isRightSkew && !isRightCoalesced) {

        val reducerId = rightPartSpec.startReducerIndex

        val skewSpecs :Option[Seq[PartialReducerPartitionSpec]]= createSkewPartitionSpecs(

          right.mapStats.shuffleId, reducerId, rightTargetSize)

        if (skewSpecs.isDefined) {

          logDebug(s"Right side partition $partitionIndex is skewed, split it into " +

            s"${skewSpecs.get.length} parts.")

          rightSkewDesc.addPartitionSize(rightActualSizes(partitionIndex))

        }

        skewSpecs.getOrElse(Seq(rightPartSpec))

      } else {

        Seq(rightPartSpec)

      }

        /** 这里做了一个笛卡尔积操作：
          - leftParts和rightParts分别存放了经过split的分区或未经过split的分区
          - 1 left right都没有skew：则结果各自只有一个SidePartition
          - 2 left.skew right.notskew:rightSidePartition会将相同的part进行复制(left.skew切分的part数量)
          - 3 left.notSkew +right.skew：leftSidePartition会将相同的part进行复制(right.skew切分的part数量)
          - 4 left.skew+right.skew: 笛卡尔积，若left有3个part，right有2个part，那么left每个part会复制2份，right每个part复制3份
          - */

      for {

        leftSidePartition <- leftParts

        rightSidePartition <- rightParts

      } {

        leftSidePartitions += leftSidePartition

        rightSidePartitions += rightSidePartition

      }

    }

    logDebug("number of skewed partitions: " +

      s"left ${leftSkewDesc.numPartitions}, right ${rightSkewDesc.numPartitions}")

    //修改经过自适应的join计划

    if (leftSkewDesc.numPartitions > 0 || rightSkewDesc.numPartitions > 0) {

      val newLeft = CustomShuffleReaderExec(

        left.shuffleStage, leftSidePartitions, leftSkewDesc.toString)

      val newRight = CustomShuffleReaderExec(

        right.shuffleStage, rightSidePartitions, rightSkewDesc.toString)

      smj.copy(

        left = s1.copy(child = newLeft), right = s2.copy(child = newRight), isSkewJoin = true)

    } else {

      smj

    }
}
```

# 3.  总结

对于倾斜问题，一般想到的就是随机前缀key，而Spark3.0做的这个优化主要原理就是基于需要进行join的两个RDD的每个partition信息，将数据量倾斜的分区进行切分出来再Join。其实整体看下来，逻辑不是那么的复杂。自适应倾斜Join优化并没有使用我们熟知的分配随机前缀Key来进行，可能是需要进行二次Join引入的复杂度无法预料，所以这里采用split分片，进行partition级别笛卡尔积的操作，这可能会导致数据传输量变大的问题，但是整体来说还是比较可控的。

