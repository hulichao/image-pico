# 大数据开发-Hive-常用日期函数&&日期连续题sql套路

前面是常用日期函数总结，后面是一道连续日期的sql题目及其解法套路。

## 1.当前日期和时间

```java
select current_timestamp
-- 2020-12-05 19:16:29.284 
```

## 2.获取当前日期，当前是 2020-12-05

```sql
SELECT current_date; 
## OR 
SELECT current_date(); 
-- 2020-12-05 
```

## 3.获取unix系统下的时间戳

```sql
SELECT UNIX_TIMESTAMP();
-- 1524884881 
```

## 4.当前是 2020-12-05

```sql
select substr(current_timestamp, 0, 10);
-- current_timestamp 
```

## 5.当前是 2020-12-05

```sql
select date_sub(current_date, 1);
--2020-12-04 
```

## 6.yyyy-MM-dd HH:MM:ss 截取日期

```sql
select to_date("2017-10-22 10:10:10");
-- 2017-10-22 
select date_format("2017-10-22" "yyyy-MM")
-- 2017-10 
```

## 7.两个日期之间的天数差

```sql
select datediff("2017-10-22", "2017-10-12");
-- 10

select datediff("2017-10-22 10:10:10", "2017-10-12 23:10:10");

-- 10

select datediff("2017-10-22 01:10:10", "2017-10-12 23:10:10");

-- 10 
```

## 8.时间截取

```sql
select from_unixtime(cast(substr("1504684212155", 0,10) as int)) dt;
-- 2017-09-06 15:50:12 
```



## 9.时间戳转日期

**语法**: to\_date(string timestamp)

```sql
select to_date(from_unixtime(UNIX_TIMESTAMP()));
-- 2018-04-28

select FROM_UNIXTIME(UNIX_TIMESTAMP(),'yyyy-MM-dd 10:30:00');

-- 2018-04-28 10:30:00

select concat(date_sub(current_date,1),' 20:30:00');

-- 2018-04-27 20:30:00

-- hive version 1.2.0

select date_format(date_sub(current_date,1),'yyyy-MM-dd 20:30:00'); 
```

## 10.日期增加

**注意**：原始日期格式只支持两种：`yyyy-MM-dd  yyyy-MM-dd HH:mm:ss`

否则都需要`date_format`来转

```sql
date_add
next_day 
```

## 11. 附加题

有一个活跃会员表，每天分区维度是会员id，可以用device\_id来代替，问怎么计算最近七天连续三天活跃会员数，其中表(`dws.dws_member_start_day`)结构如下表（dt是分区,日期格式yyyy-MM-dd，每个分区有唯一`device_id`）：

```sql
device_id             string                                                                      
dt                    string                
```

### 解法套路

1.首先思考可以用到的日期函数datediff, date\_sub/date\_add

2.连续日期，连续问题都会用到一个排名函数，但是排名函数的值是数值，要与日期的连续性做到映射，才方便分组，比如可以把日期映射到连续数字，或者数字映射到连续日期，实现这两个的操作就是通过前面的datedff 和 date\_sub组合，原理就是日期与日期相减即可得到连续整数，整数随便与某个日期做相减即可得到连续的日期,其中date\_sub可以是反向排序得到连续日期。

3.通过连续的排序日期或者排序id相减，然后分组即可解决此类问题

#### 1.在原表基础上增加一列排序序号

```sql
SELECT device_id,
       dt,
       row_number() over(PARTITION BY device_id
                         ORDER BY dt) ro
FROM dws.dws_member_start_day

```

#### 2.将序号转为连续日期，或者把日期转为连续数字，后成为gid

```sql
-- 2.1 序号转为连续日期
SELECT device_id,
    dt,
    datediff(dt, date_add('2020-07-20', row_number() over(PARTITION BY device_id
        ORDER BY dt))) gid
FROM dws.dws_member_start_day 

-- 2.2 日期转为连续序号
SELECT device_id,
    dt,
    (datediff(dt, '2020-07-21') - row_number() over(PARTITION BY device_id
        ORDER BY dt)) gid
FROM dws.dws_member_start_day 
```

#### 3.分组筛选

```sql
SELECT device_id,count(1)
FROM
    (SELECT device_id,
        dt,
        datediff(dt, date_add('2020-07-20', row_number() over(PARTITION BY device_id
            ORDER BY dt))) gid
        FROM dws.dws_member_start_day
        WHERE datediff(dt, CURRENT_DATE) BETWEEN -7 AND 7 ) tmp
GROUP BY device_id,
    gid
HAVING count(1) < 3  
```
