# 大数据开发 - mysql 根据自表更新错误的解决方法

mysql出现You can’t specify target table for update in FROM clause 这个错误的意思是不能在同一个sql语句中，先select同一个表的某些值，然后再update这个表。

例如：message表保存了多个用户的消息

# 创建表

```sql
CREATE TABLE `message` (

  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,

  `uid` int(10) unsigned NOT NULL,

  `content` varchar(255) NOT NULL,

  `addtime` datetime NOT NULL,

  PRIMARY KEY (`id`),

  KEY `uid` (`uid`),

  KEY `addtime` (`addtime`)

) ENGINE=InnoDB DEFAULT CHARSET=utf8;

```

# 插入数据

```sql
insert into message(uid,content,addtime) values
(1,'content1','2016-09-26 00:00:01'),
(2,'content2','2016-09-26 00:00:02'),
(3,'content3','2016-09-26 00:00:03'),
(1,'content4','2016-09-26 00:00:04'),
(3,'content5','2016-09-26 00:00:05'),
(2,'content6','2016-09-26 00:00:06'),
(2,'content7','2016-09-26 00:00:07'),
(4,'content8','2016-09-26 00:00:08'),
(4,'content9','2016-09-26 00:00:09'),
(1,'content10','2016-09-26 00:00:10');


```

表结构及数据如下：

```sql
mysql> select * from message;
+----+-----+-----------+---------------------+
| id | uid | content   | addtime             |
+----+-----+-----------+---------------------+
|  1 |   1 | content1  | 2016-09-26 00:00:01 |
|  2 |   2 | content2  | 2016-09-26 00:00:02 |
|  3 |   3 | content3  | 2016-09-26 00:00:03 |
|  4 |   1 | content4  | 2016-09-26 00:00:04 |
|  5 |   3 | content5  | 2016-09-26 00:00:05 |
|  6 |   2 | content6  | 2016-09-26 00:00:06 |
|  7 |   2 | content7  | 2016-09-26 00:00:07 |
|  8 |   4 | content8  | 2016-09-26 00:00:08 |
|  9 |   4 | content9  | 2016-09-26 00:00:09 |
| 10 |   1 | content10 | 2016-09-26 00:00:10 |
+----+-----+-----------+---------------------+
10 rows in set (0.00 sec)
```

# 问题

然后执行将每个用户第一条消息的内容更新为Hello World

```sql
mysql> update message set content='Hello World' where id in(select min(id) from message group by uid);
ERROR 1093 (HY000): You can't specify target table 'message' for update in FROM clause

```

因为在同一个sql语句中，先select出message表中每个用户消息的最小id值，然后再更新message表，因此会出现 ERROR 1093 (HY000): You can’t specify target table ‘message’ for update in FROM clause 这个错误。

# 解决办法

解决方法：select的结果再通过一个中间表select多一次，就可以避免这个错误

```sql
update message set content='Hello World' where id in( select min_id from ( select min(id) as min_id from message group by uid) as a );

```

执行：

```sql
mysql> update message set content='Hello World' where id in( select min_id from ( select min(id) as min_id from message group by uid) as a );
Query OK, 4 rows affected (0.01 sec)
Rows matched: 4  Changed: 4  Warnings: 0
```

```sql
mysql> select * from message;
+----+-----+-------------+---------------------+
| id | uid | content     | addtime             |
+----+-----+-------------+---------------------+
|  1 |   1 | Hello World | 2016-09-26 00:00:01 |
|  2 |   2 | Hello World | 2016-09-26 00:00:02 |
|  3 |   3 | Hello World | 2016-09-26 00:00:03 |
|  4 |   1 | content4    | 2016-09-26 00:00:04 |
|  5 |   3 | content5    | 2016-09-26 00:00:05 |
|  6 |   2 | content6    | 2016-09-26 00:00:06 |
|  7 |   2 | content7    | 2016-09-26 00:00:07 |
|  8 |   4 | Hello World | 2016-09-26 00:00:08 |
|  9 |   4 | content9    | 2016-09-26 00:00:09 |
| 10 |   1 | content10   | 2016-09-26 00:00:10 |
+----+-----+-------------+---------------------+


```

注意，只有mysql会有这个问题，mssql与oracle都没有这个问题
