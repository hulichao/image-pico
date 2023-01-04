# 大数据开发-Flume-频繁产生小文件原因和处理

# 1.问题背景

通过flume直接上传实时数据到hdfs，会常遇到的一个问题就是小文件，需要调参数来设置，往往在生产环境参数大小也不同

**1.flume滚动配置为何不起作用？**

**2.通过源码分析得出什么原因？**

**3.该如何解决flume小文件？**

# 2. 过程分析

接着上一篇，[https://blog.csdn.net/hu\_lichao/article/details/110358689](https://blog.csdn.net/hu_lichao/article/details/110358689 "https://blog.csdn.net/hu_lichao/article/details/110358689")

本人在测试hdfs的sink，发现sink端的文件滚动配置项起不到任何作用，配置如下：

```sql
a1.sinks.k1.type=hdfs  
a1.sinks.k1.channel=c1  
a1.sinks.k1.hdfs.useLocalTimeStamp=true  
a1.sinks.k1.hdfs.path=hdfs://linux121:9000/user/data/logs/%Y-%m-%d
a1.sinks.k1.hdfs.filePrefix=XXX  
a1.sinks.k1.hdfs.rollInterval=60  
a1.sinks.k1.hdfs.rollSize=0  
a1.sinks.k1.hdfs.rollCount=0  
a1.sinks.k1.hdfs.idleTimeout=0   
```

这里配置的是60秒，文件滚动一次，也就每隔60秒，会新产生一个文件【前提，flume的source端有数据来】这里注意 useLocalTimeStamp  是使用本地时间戳来对hdfs上的目录来命名，这个属性的目的就是相当于时间戳的拦截器，否则%Y 等等这些东西都识别不了

要么用上面这个属性，要么用时间戳拦截器。但是当我启动flume的时候，运行十几秒，不断写入数据，发现hdfs端频繁的产生文件，每隔几秒就有新文件产生而且在flume的日志输出可以频繁看到这句：

```sql
[WARN] Block Under-replication detected. Rotating file.
```

只要有这句，就会产生一个新的文件，意思就是检测到复制块正在滚动文件，结合源码看下：

```java
private boolean shouldRotate() {  

    boolean doRotate = false;    
    if (writer.isUnderReplicated()) {    
        this.isUnderReplicated = true;    
        doRotate = true;  
    
    } else {  
    
        this.isUnderReplicated = false;  
    
    }  
    
    if ((rollCount > 0) && (rollCount <= eventCounter)) {  
        LOG.debug("rolling: rollCount: {}, events: {}", rollCount, eventCounter); 
        doRotate = true;  
    
    }  
    
    if ((rollSize > 0) && (rollSize <= processSize)) {  
        LOG.debug("rolling: rollSize: {}, bytes: {}", rollSize, processSize);   
        doRotate = true;  
    }     
    return doRotate;  

}   
```

这是判断是否滚动文件，但是这里面的第一判断条件是判断是否当前的HDFSWriter正在复制块

```java
public boolean isUnderReplicated() {  
    try {  
    
        int numBlocks = getNumCurrentReplicas();  
        if (numBlocks == -1) {  
            return false;  
        }  
    
        int desiredBlocks;  
        if (configuredMinReplicas != null) {    
            desiredBlocks = configuredMinReplicas;  
        } else {   
            desiredBlocks = getFsDesiredReplication();     
        }  
    
        return numBlocks < desiredBlocks;  
    
    } catch (IllegalAccessException e) {  
        logger.error("Unexpected error while checking replication factor", e);   
    } catch (InvocationTargetException e) {   
        logger.error("Unexpected error while checking replication factor", e);      
    } catch (IllegalArgumentException e) {     
        logger.error("Unexpected error while checking replication factor", e);     
    }  
    
    return false;  

} 
```

通过读取的配置复制块数量和当前正在复制的块比较，判断是否正在被复制

```java
if (shouldRotate()) {  
    boolean doRotate = true;  
    
    if (isUnderReplicated) {  
    
        if (maxConsecUnderReplRotations > 0 &&  
            consecutiveUnderReplRotateCount >= maxConsecUnderReplRotations) {  
            doRotate = false;  
    
            if (consecutiveUnderReplRotateCount == maxConsecUnderReplRotations) {   
                LOG.error("Hit max consecutive under-replication rotations ({}); " +  
                    "will not continue rolling files under this path due to " +  
                    "under-replication", maxConsecUnderReplRotations);  
            }  
    
        } else {    
            LOG.warn("Block Under-replication detected. Rotating file.");  
        }  
    
        consecutiveUnderReplRotateCount++;  
    
    } else {  
    
        consecutiveUnderReplRotateCount = 0;  

}  
```

以上方法，入口是`shouldRotate()`方法，也就是如果你配置了`rollcount`,`rollsize`大于0，会按照你的配置来滚动的，但是在入口进来后，发现，又去判断了是否有块在复制；里面就读取了一个固定变量`maxConsecUnderReplRotations`=30，也就是正在复制的块，最多之能滚动出30个文件，如果超过了30次，该数据块如果还在复制中，那么数据也不会滚动了，`doRotate`=`false`，不会滚动了，所以有的人发现自己一旦运行一段时间，会出现30个文件，再结合上面的源码看一下：如果你配置了10秒滚动一次，写了2秒，恰好这时候该文件内容所在的块在复制中，那么虽然没到10秒，依然会给你滚动文件的，文件大小，事件数量的配置同理了。

为了解决上述问题，我们只要让程序感知不到写的文件所在块正在复制就行了，怎么做呢？？只要让`isUnderReplicated`()方法始终返回`false`就行了，该方法是通过当前正在被复制的块和配置中读取的复制块数量比较的，我们能改的就只有配置项中复制块的数量，而官方给出的flume配置项中有该项

```java
hdfs.minBlockReplicas
Specify minimum number of replicas per HDFS block. If not specified, it comes from the default Hadoop config in the classpath. 
```

默认读的是`hadoop`中的`dfs`.`replication`属性，该属性默认值是3，这里我们也不去该hadoop中的配置，在flume中添加上述属性为1即可

完整配置如下：

```java
a1.sources = r1
a1.sinks = k1
a1.channels = c1
# taildir source
a1.sources.r1.type = TAILDIR
a1.sources.r1.positionFile =
/data/lagoudw/conf/startlog_position.json
a1.sources.r1.filegroups = f1
a1.sources.r1.filegroups.f1 = /opt/hoult/servers/logs/start/.*log
a1.sources.r1.interceptors = i1
a1.sources.r1.interceptors.i1.type = com.hoult.flume.CustomerInterceptor$Builder
# memorychannel
a1.channels.c1.type = memory
a1.channels.c1.capacity = 100000
a1.channels.c1.transactionCapacity = 2000
# hdfs sink
a1.sinks.k1.type = hdfs
a1.sinks.k1.hdfs.path = /user/data/logs/start/dt=%{logtime}/
a1.sinks.k1.hdfs.filePrefix = startlog.
# 配置文件滚动方式（文件大小32M）
a1.sinks.k1.hdfs.rollSize = 33554432
a1.sinks.k1.hdfs.rollCount = 0
a1.sinks.k1.hdfs.rollInterval = 0
a1.sinks.k1.hdfs.idleTimeout = 0
a1.sinks.k1.hdfs.minBlockReplicas = 1
# 向hdfs上刷新的event的个数
a1.sinks.k1.hdfs.batchSize = 1000
# 使用本地时间
# a1.sinks.k1.hdfs.useLocalTimeStamp = true
# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
 
```

这样程序就永远不会因为文件所在块的复制而滚动文件了，只会根据你的配置项来滚动文件了。。。。

# 3.总结

设置`minBlockReplicas`=1 的时候可以保证会按你设置的几个参数来达到不会产生过多的小文件，因为这个参数在读取时候优先级较高，会首先判断到有没有Hdfs的副本复制，导致滚动文件的异常，另外`flume`接入数据时候可以通过过滤器尽可能把一些完全用不到的数据进行过滤，清洗时候就 省事一些了。
