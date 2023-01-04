# 大数据开发-Flink-窗口全解析

# Flink窗口背景

Flink认为Batch是Streaming的一个特例，因此Flink底层引擎是一个流式引擎，在上面实现了流处理和批处理。而Window就是从Streaming到Batch的桥梁。通俗讲，Window是用来对一个无限的流设置一个有限的集合，从而在有界的数据集上进行操作的一种机制。流上的集合由Window来划定范围，比如“计算过去10分钟”或者“最后50个元素的和”。Window可以由时间（Time Window）（比如每30s）或者数据（Count Window）（如每100个元素）驱动。DataStream API提供了Time和Count的Window。

一个Flink窗口应用的大致骨架结构如下所示：

```java
// Keyed Window
stream
  .keyBy(...)               <-  按照一个Key进行分组
  .window(...)              <-  将数据流中的元素分配到相应的窗口中
  [.trigger(...)]            <-  指定触发器Trigger（可选）
  [.evictor(...)]            <-  指定清除器Evictor(可选)
    .reduce/aggregate/process()      <-  窗口处理函数Window Function
// Non-Keyed Window
stream
  .windowAll(...)           <-  不分组，将数据流中的所有元素分配到相应的窗口中
  [.trigger(...)]            <-  指定触发器Trigger（可选）
  [.evictor(...)]            <-  指定清除器Evictor(可选)
    .reduce/aggregate/process()      <-  窗口处理函数Window Function
```

Flink窗口的骨架结构中有两个必须的两个操作：
\*   使用窗口分配器（WindowAssigner）将数据流中的元素分配到对应的窗口。
\*   当满足窗口触发条件后，对窗口内的数据使用窗口处理函数（Window Function）进行处理，常用的Window Function有`reduce`、`aggregate`、`process`

# 滚动窗口

## 基于时间驱动

将数据依据固定的窗口长度对数据进行切分，滚动窗口下窗口之间之间不重叠，且窗口长度是固定的。我们可以用`TumblingEventTimeWindows`和`TumblingProcessingTimeWindows`创建一个基于Event Time或Processing Time的滚动时间窗口。窗口的长度可以用`org.apache.flink.streaming.api.windowing.time.Time`中的`seconds`、`minutes`、`hours`和`days`来设置。

![](https://files.catbox.moe/o7s958.png)

```java
//关键处理案例
KeyedStream<Tuple2<String, Integer>, Tuple> keyedStream = mapStream.keyBy(0);
// 基于时间驱动，每隔10s划分一个窗口
WindowedStream<Tuple2<String, Integer>, Tuple, TimeWindow> timeWindow =
keyedStream.timeWindow(Time.seconds(10));
// 基于事件驱动, 每相隔3个事件(即三个相同key的数据), 划分一个窗口进行计算
// WindowedStream<Tuple2<String, Integer>, Tuple, GlobalWindow> countWindow =
keyedStream.countWindow(3);
// apply是窗口的应用函数，即apply里的函数将应用在此窗口的数据上。
timeWindow.apply(new MyTimeWindowFunction()).print();
// countWindow.apply(new MyCountWindowFunction()).print();
```

## &#x20;基于事件驱动

当我们想要每100个用户的购买行为作为驱动，那么每当窗口中填满100个”相同”元素了，就会对窗口进行计算，很好理解，下面是一个实现案例

```java
public class MyCountWindowFunction implements WindowFunction<Tuple2<String, Integer>,
  String, Tuple, GlobalWindow> {
    @Override
    public void apply(Tuple tuple, GlobalWindow window, Iterable<Tuple2<String, Integer>>
      input, Collector<String> out) throws Exception {
      SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS");
      int sum = 0;
      for (Tuple2<String, Integer> tuple2 : input){
        sum += tuple2.f1;
      }
      
      //无用的时间戳，默认值为： Long.MAX_VALUE,因为基于事件计数的情况下，不关心时间。
      long maxTimestamp = window.maxTimestamp();
      out.collect("key:" + tuple.getField(0) + " value: " + sum + "| maxTimeStamp :"+ maxTimestamp + "," + format.format(maxTimestamp)
      );
  }
}
```

# 滑动时间窗口

动窗口是固定窗口的更广义的一种形式，滑动窗口由固定的窗口长度和滑动间隔组成，特点：窗口长度固定，可以有重叠，滑动窗口以一个步长（Slide）不断向前滑动，窗口的长度固定。使用时，我们要设置Slide和Size。Slide的大小决定了Flink以多大的频率来创建新的窗口，Slide较小，窗口的个数会很多。Slide小于窗口的Size时，相邻窗口会重叠，一个事件会被分配到多个窗口；Slide大于Size，有些事件可能被丢掉

![](https://files.catbox.moe/etmg7c.png)

## 基于时间的滚动窗口

```java
//基于时间驱动，每隔5s计算一下最近10s的数据
// WindowedStream<Tuple2<String, Integer>, Tuple, TimeWindow> timeWindow =
keyedStream.timeWindow(Time.seconds(10), Time.seconds(5));
SingleOutputStreamOperator<String> applyed = countWindow.apply(new WindowFunction<Tuple3<String, String, String>, String, String, GlobalWindow>() {
    @Override
    public void apply(String s, GlobalWindow window, Iterable<Tuple3<String, String, String>> input, Collector<String> out) throws Exception {
        Iterator<Tuple3<String, String, String>> iterator = input.iterator();
        StringBuilder sb = new StringBuilder();
        while (iterator.hasNext()) {
            Tuple3<String, String, String> next = iterator.next();
            sb.append(next.f0 + ".." + next.f1 + ".." + next.f2);
        }
//                window.
        out.collect(sb.toString());
    }
}); 
```

## 基于事件的滚动窗口

```java
/**
* 滑动窗口：窗口可重叠
* 1、基于时间驱动
* 2、基于事件驱动
*/
WindowedStream<Tuple3<String, String, String>, String, GlobalWindow> countWindow = keybyed.countWindow(3,2);

SingleOutputStreamOperator<String> applyed = countWindow.apply(new WindowFunction<Tuple3<String, String, String>, String, String, GlobalWindow>() {
    @Override
    public void apply(String s, GlobalWindow window, Iterable<Tuple3<String, String, String>> input, Collector<String> out) throws Exception {
        Iterator<Tuple3<String, String, String>> iterator = input.iterator();
        StringBuilder sb = new StringBuilder();
        while (iterator.hasNext()) {
            Tuple3<String, String, String> next = iterator.next();
            sb.append(next.f0 + ".." + next.f1 + ".." + next.f2);
        }
//                window.
        out.collect(sb.toString());
    }
});

```

# 会话时间窗口

由一系列事件组合一个指定时间长度的timeout间隙组成，类似于web应用的session，也就是一段时间没有接收到新数据就会生成新的窗口，在这种模式下，窗口的长度是可变的，每个窗口的开始和结束时间并不是确定的。我们可以设置定长的Session gap，也可以使用`SessionWindowTimeGapExtractor`动态地确定Session gap的长度。

```java
val input: DataStream[T] = ...
// event-time session windows with static gap
input
    .keyBy(...)
    .window(EventTimeSessionWindows.withGap(Time.minutes(10)))
    .<window function>(...)
// event-time session windows with dynamic gap
input
    .keyBy(...)
    .window(EventTimeSessionWindows.withDynamicGap(new SessionWindowTimeGapExtractor[T] {
      override def extract(element: T): Long = {
        // determine and return session gap
      }
    }))
    .<window function>(...)
// processing-time session windows with static gap
input
    .keyBy(...)
    .window(ProcessingTimeSessionWindows.withGap(Time.minutes(10)))
    .<window function>(...)
// processing-time session windows with dynamic gap
input
    .keyBy(...)
    .window(DynamicProcessingTimeSessionWindows.withDynamicGap(new SessionWindowTimeGapExtractor[T] {
      override def extract(element: T): Long = {
        // determine and return session gap
      }
    }))
    .<window function>(...)
```

# 窗口函数

在窗口划分完毕后，就是要对窗口内的数据进行处理，一是增量计算对应`reduce` 和`aggregate`，二是全量计算对应`process` ,增量计算指的是窗口保存一份中间数据，每流入一个新元素，新元素与中间数据两两合一，生成新的中间数据，再保存到窗口中。全量计算指的是窗口先缓存该窗口所有元素，等到触发条件后对窗口内的全量元素执行计算

# 参考

[https://cloud.tencent.com/developer/article/1584926](https://cloud.tencent.com/developer/article/1584926 "https://cloud.tencent.com/developer/article/1584926")

