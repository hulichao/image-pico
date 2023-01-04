# Hive-常见调优方式 && 两个面试sql

Hive作为大数据领域常用的数据仓库组件，在设计和开发阶段需要注意效率。影响Hive效率的不仅仅是数据量过大；数据倾斜、数据冗余、job或I/O过多、MapReduce分配不合理等因素都对Hive的效率有影响。对Hive的调优既包含对HiveQL语句本身的优化，也包含Hive配置项和MR方面的调
整。

从以下三个方面展开：
架构优化
参数优化
SQL优化

# 1.架构方面

执行引擎方面针对公司内平台的资源，选择更合适的更快的引擎，比如MR、TEZ、Spark等，

如果选择是TEZ引擎，可以在优化器时候开启向量化的优化器，另外可以选择成本优化器CBO，配置分别如下：

```sql
set hive.vectorized.execution.enabled = true; -
- 默认 false
set hive.vectorized.execution.reduce.enabled = true; -
- 默认 false
SET hive.cbo.enable=true; --从 v0.14.0默认
true
SET hive.compute.query.using.stats=true; -- 默认false
SET hive.stats.fetch.column.stats=true; -- 默认false
SET hive.stats.fetch.partition.stats=true; -- 默认true 
```

在表的设计上优化，比如选择分区表，分桶表，以及表的存储格式，为了减少数据传输，可以使用压缩的方式，下面给几个参数（更多参数可以查看官网）

```sql
-- 中间结果压缩
SET
hive.intermediate.compression.codec=org.apache.hadoop.io.compress.SnappyCodec ;
-- 输出结果压缩
SET hive.exec.compress.output=true;
SET mapreduce.output.fileoutputformat.compress.codec =org.apache.hadoop.io.compress.SnappyCodc
```

# 2.参数优化

第二部分是参数优化，其实上面架构部分，有部分也是通过参数来控制的，这一部分的参数控制主要有下面几个方面

本地模式、严格模式、JVM重用、并行执行、推测执行、合并小文件、Fetch模式

## 2.1 本地模式

当数据量较小的时候，启动分布式处理数据会比较慢，启动时间较长，不如本地模式快，用下面的参数来调整

```sql
SET hive.exec.mode.local.auto=true; -- 默认 false 
小
SET hive.exec.mode.local.auto.inputbytes.max=50000000; --输入文件的大小小于 hive.exec.mode.local.auto.inputbytes.max 配置的大
SET hive.exec.mode.local.auto.input.files.max=5; -- 默认 4  map任务的数量小于 hive.exec.mode.local.auto.input.files.max 配置的
大小
```

## 2.2 严格模式

这其实是个开关，满足下面三个语句时候，就会失败，如果不开启就正常执行，开启后就让这些语句自动失败

```sql
hive.mapred.mode=nostrict
 -- 查询分区表时不限定分区列的语句；
 -- 两表join产生了笛卡尔积的语句；
 -- 用order by来排序，但没有指定limit的语句 
```

## 2.3 Jvm重用

在mr里面，是以进程为单位的，一个进程就是一个Jvm,其实像短作业，这些进程能够重用就会很快，但是它的缺点是会等任务执行完毕后task插槽，这个在数据倾斜时候较为明显。开启这个使用下面的参数

```sql
SET mapreduce.job.jvm.numtasks=5;
```

## 2.4 并行执行

Hive的查询会转为stage，这些stage并不是相互依赖的，可以并行执行这些stage，使用下面的参数

```sql
SET hive.exec.parallel=true; -- 默认false
SET hive.exec.parallel.thread.number=16; -- 默认8
```

## 2.5 推测执行

这个参数的作用是，使用空间资源来换取得到最终结果的时间，比如由于网络，资源不均等原因，某些任务运行特别慢，会启动备份进程处理同一份数据，并最终选用最先成功的计算结果作为最终结果。

```sql
set mapreduce.map.speculative=true
set mapreduce.reduce.speculative=true
set hive.mapred.reduce.tasks.speculative.execution=true
```

## 2.6 合并小文件

在map执行前面，先合并小文件来减少map数

```sql
set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
```

在任务结束后，合并小文件

```sql
# 在 map-only 任务结束时合并小文件，默认true
SET hive.merge.mapfiles = true;
# 在 map-reduce 任务结束时合并小文件，默认false
SET hive.merge.mapredfiles = true;
# 合并文件的大小，默认256M
SET hive.merge.size.per.task = 268435456;
# 当输出文件的平均大小小于该值时，启动一个独立的map-reduce任务进行文件merge
SET hive.merge.smallfiles.avgsize = 16777216;
```

## 2.7 Fetch模式

最后一种fetch模式，则是在某些情况下尽量不跑mr，比如查询几个字段，全局查找，字段查，limit查等情况

```sql
hive.fetch.task.conversion=more
```

# 3.sql优化

这一部分较复杂，可能涉及到数据倾斜问题，至于数据倾斜问题一直是大数据处理的不可比避免的一个问题，处理方式也较多

## 3.1 sql优化

sql优化是开发人员最容易控制的部分，往往是经验使之，大约总结一下又下面的方式

列，分区拆解，sort by 代替 order by, group by 代替count(distinct) ，group by的预聚合（通过参数来控制），倾斜配置项，map join，单独过滤空值，适当调整map 和 reduces数，这些在工作中几乎都会碰到，尽可能去优化他们呢是你要做的

## 3.2 倾斜均衡配置项

这个配置与 group by 的倾斜均衡配置项异曲同工，通过 hive.optimize.skewjoin来配置，默认false。如果开启了，在join过程中Hive会将计数超过阈值 hive.skewjoin.key （默认100000）的倾斜key对应的行临时写进文件中，然后再启动另一个job做map join生成结果。通过 hive.skewjoin.mapjoin.map.tasks 参数还可以控制第二个job的mapper数量，默认1000

## 3.3 单独处理倾斜key

如果倾斜的 key 有实际的意义，一般来讲倾斜的key都很少，此时可以将它们单独抽取出来，对应的行单独存入临时表中，然后打上一个较小的随机数前缀（比如0\~9），最后再进行聚合。不要一个Select语句中，写太多的Join。一定要了解业务，了解数据。(A0-A9)分成多条语句，分步执行；(A0-A4; A5-A9)；先执行大表与小表的关联；

## 4.两个SQL

### &#x20;4.1 找出全部夺得3连贯的队伍

team,year
活塞,1990
公牛,1991
公牛,1992

```sql

--
 -- 1 排名
select team, year, 
row_number() over (partition by team order by year) as rank
  from t1;

-- 2 获取分组id
select team, year, 
row_number() over (partition by team order by year) as rank,
(year -row_number() over (partition by team order by year)) as groupid
  from t1;

-- 3 分组求解
select team, count(1) years
  from (select team, 
        (year -row_number() over (partition by team order by year)) as groupid
          from t1
       ) tmp
group by team, groupid
having count(1) >= 3;
```

### 4.2 找出每个id在在一天之内所有的波峰与波谷值

> 波峰：
> 这一时刻的值 > 前一时刻的值
> 这一时刻的值 > 后一时刻的值
> 波谷：
> 这一时刻的值 < 前一时刻的值
> 这一时刻的值 < 后一时刻的值
> id        time price  前一时刻的值（lag）    后一时刻的值(lead)
> sh66688, 9:35, 29.48       null                  28.72
> sh66688, 9:40, 28.72        29.48                27.74
> sh66688, 9:45, 27.74  &#x20;
> sh66688, 9:50, 26.75  &#x20;
> sh66688, 9:55, 27.13
> sh66688, 10:00, 26.30
> sh66688, 10:05, 27.09
> sh66688, 10:10, 26.46
> sh66688, 10:15, 26.11
> sh66688, 10:20, 26.88
> sh66688, 10:25, 27.49
> sh66688, 10:30, 26.70
> sh66688, 10:35, 27.57
> sh66688, 10:40, 28.26
> sh66688, 10:45, 28.03

```sql
-- 思路：关键是找到波峰波谷的特征
-- 波峰的特征: 大于前一个时间段、后一个时间段的值
-- 波谷的特征: 小于前一个时间段、后一个时间段的值
-- 找到这个特征SQL就好写了

select id, time, price,
       case when price > beforeprice and price > afterprice then "波峰"
            when price < beforeprice and price < afterprice then "波谷" end as feature
  from (select id, time, price,
               lag(price) over (partition by id order by time) beforeprice,
               lead(price) over (partition by id order by time) afterprice
          from t2
        )tmp
 where (price > beforeprice and price > afterprice) or
       (price < beforeprice and price < afterprice);
```

