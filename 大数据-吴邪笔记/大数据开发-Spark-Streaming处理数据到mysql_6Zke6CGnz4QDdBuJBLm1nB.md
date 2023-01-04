# 大数据开发-Spark-Streaming处理数据到mysql

前面一篇讲到streamin读取kafka数据加工处理后写到kafka数据，[大数据开发-Spark-开发Streaming处理数据 && 写入Kafka](<大数据开发-Spark-开发Streaming处理数据 && 写入Kafka_32EKKxHaKH8cpRVjEdEbfz.md> "大数据开发-Spark-开发Streaming处理数据 && 写入Kafka")是针对比如推荐领域，实时标签等场景对于实时处理结果放到mysql也是一种常用方式，假设一些车辆调度的地理位置信息处理后写入到mysql

# 1.说明

数据表如下：

```sql
create database test;
use test;
DROP TABLE IF EXISTS car_gps;
CREATE TABLE IF NOT EXISTS car_gps(
deployNum VARCHAR(30) COMMENT '调度编号',
plateNum VARCHAR(10) COMMENT '车牌号',
timeStr VARCHAR(20) COMMENT '时间戳',
lng VARCHAR(20) COMMENT '经度',
lat VARCHAR(20) COMMENT '纬度',
dbtime TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '数据入库时间',
PRIMARY KEY(deployNum, plateNum, timeStr)) 
```

# 2.编写程序

首先引入mysql的驱动

```xml
  <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>5.1.44</version>
  </dependency>
```

## 2.1 jdbc写入mysql

```scala
package com.hoult.Streaming.work

import java.sql.{Connection, DriverManager, PreparedStatement}
import java.util.Properties

import com.hoult.structed.bean.BusInfo
import org.apache.spark.sql.ForeachWriter

class JdbcHelper extends ForeachWriter[BusInfo] {
  var conn: Connection = _
  var statement: PreparedStatement = _
  override def open(partitionId: Long, epochId: Long): Boolean = {
    if (conn == null) {
      conn = JdbcHelper.openConnection
    }
    true
  }

  override def process(value: BusInfo): Unit = {
    //把数据写入mysql表中
    val arr: Array[String] = value.lglat.split("_")
    val sql = "insert into car_gps(deployNum,plateNum,timeStr,lng,lat) values(?,?,?,?,?)"
    statement = conn.prepareStatement(sql)
    statement.setString(1, value.deployNum)
    statement.setString(2, value.plateNum)
    statement.setString(3, value.timeStr)
    statement.setString(4, arr(0))
    statement.setString(5, arr(1))
    statement.executeUpdate()
  }

  override def close(errorOrNull: Throwable): Unit = {
    if (null != conn) conn.close()
    if (null != statement) statement.close()
  }
}

object JdbcHelper {
  var conn: Connection = _
  val url = "jdbc:mysql://hadoop1:3306/test?useUnicode=true&characterEncoding=utf8"
  val username = "root"
  val password = "123456"
  def openConnection: Connection = {
    if (null == conn || conn.isClosed) {
      val p = new Properties
      Class.forName("com.mysql.jdbc.Driver")
      conn = DriverManager.getConnection(url, username, password)
    }
    conn
  }
}

```

## 2.2 通过foreach来写入mysql

```scala
package com.hoult.Streaming.work
import com.hoult.structed.bean.BusInfo
import org.apache.spark.sql.{Column, DataFrame, Dataset, SparkSession}

object KafkaToJdbc {
  def main(args: Array[String]): Unit = {
    System.setProperty("HADOOP_USER_NAME", "root")
    //1 获取sparksession
    val spark: SparkSession = SparkSession.builder()
      .master("local[*]")
      .appName(KafkaToJdbc.getClass.getName)
      .getOrCreate()
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    //2 定义读取kafka数据源
    val kafkaDf: DataFrame = spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "linux121:9092")
      .option("subscribe", "test_bus_info")
      .load()
    //3 处理数据
    val kafkaValDf: DataFrame = kafkaDf.selectExpr("CAST(value AS STRING)")
    //转为ds
    val kafkaDs: Dataset[String] = kafkaValDf.as[String]
    //解析出经纬度数据，写入redis
    //封装为一个case class方便后续获取指定字段的数据
    val busInfoDs: Dataset[BusInfo] = kafkaDs.map(BusInfo(_)).filter(_ != null)

    //将数据写入MySQL表
    busInfoDs.writeStream
      .foreach(new JdbcHelper)
      .outputMode("append")
      .start()
      .awaitTermination()
  }
}

```

## 2.4 创建topic和从消费者端写入数据

```bash
kafka-topics.sh --zookeeper linux121:2181/myKafka --create --topic test_bus_info --partitions 2 --replication-factor 1
kafka-console-producer.sh --broker-list linux121:9092 --topic test_bus_info 
```

