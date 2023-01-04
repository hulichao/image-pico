# 大数据开发-数仓ads层指标计算

ads层数据往往是最终的结果指标数据，在大屏展示，或者实时流处理时候使用，通过下面两个例子来练习业务大屏展示sql该怎么写。

# 1.会员分析案例

## 1.1 数据准备

表结构如下，其中此表是dws层以天为维度的会员表，比如每天的会员信息汇总，

```sql
use dws;
drop table if exists dws.dws_member_start_day;
create table dws.dws_member_start_day(
`device_id` string, -- 设备id，来区分用户
`uid` string, -- uid
`app_v` string,
`os_type` string,
`language` string,
`channel` string,
`area` string,
`brand` string
) COMMENT '会员日启动汇总'
partitioned by(dt string)
stored as parquet;

```

## 1.2 会员指标计算

**沉默会员的定义**：只在安装当天启动过App，而且安装时间是在7天前

**流失会员的定义**：最近30天未登录的会员

### 1.2.1 如何计算沉默会员数

```sql
-- 拿到只启动一次的会员，后面再过滤安装时间是再7天前的，使用sum 窗口函数
SELECT count(*)
FROM
  (SELECT device_id,
          sum(device_id) OVER (PARTITION BY device_id) AS sum_num,
                     dt
   FROM dws.dws_member_start_day) tmp
WHERE dt <= date_add(CURRENT_DATE, -7)
  AND sum_num=1

```

### 1.2.2 如何计算流失会员数

```sql
-- 拿到会员最近一次登录时间，并用row_number来过滤
SELECT count(*)
FROM
  (SELECT device_id,
          dt,
          row_number() OVER (PARTITION BY device_id
                             ORDER BY dt DESC) ro
   FROM dws.dws_member_start_day) tmp
WHERE ro=1
  AND dt >= date_add(CURRENT_DATE, -30) 
```

## 2. 核心交易案例

### 2.1 数据准备

给定一个每日订单维度表，表结构如下图：

```sql
DROP TABLE IF EXISTS dwd.dwd_trade_orders;
create table dwd.dwd_trade_orders(
`orderId`    int,
`orderNo`   string,
`userId`    bigint,
`status`    tinyint,
`productMoney` decimal,
`totalMoney`  decimal,
`payMethod`   tinyint,
`isPay`     tinyint,
`areaId`    int,
`tradeSrc`   tinyint,
`tradeType`   int,
`isRefund`   tinyint,
`dataFlag`   tinyint,
`createTime`  string,
`payTime`   string,
`modifiedTime` string,
`start_date`  string,
`end_date`   string
) COMMENT '订单事实拉链表'
partitioned by (dt string)
STORED AS PARQUET;
```

其中，订单状态 -3 用户拒收 -2未付款的订单 -1用户取消 0 待发货 1配送中 2用户确认收货，订单有效标志 -1 删除 1 有效

数据预处理，在明细事实拉链表处理时不太方便，可以做一张中间表，`dws_trade_orders_day` 其表结构和加工如下：

```sql
DROP TABLE IF EXISTS dws.dws_trade_orders_day;

CREATE TABLE IF NOT EXISTS dws.dws_trade_orders_day(day_dt string COMMENT '日期：yyyy-MM-dd',
                                                   day_cnt decimal commnet '日订单笔数',
                                                   day_sum decimal COMMENT '日订单总额') COMMENT '日订单统计表';

SELECT dt,
       count(*) cnt,
       sum(totalMoney) sm
FROM
  (SELECT DISTINCT orderid,
                   dt,
                   totalMoney
   FROM dwd.dwd_trade_orders
   WHERE status >= 0
     AND dataFlag = '1') tmp
GROUP BY dt;


INSERT OVERWRITE TABLE dws.dws_trade_orders_day
SELECT dt,
       count(*) cnt,
       sum(totalMoney) sm
FROM
  (SELECT DISTINCT orderid,
                   dt,
                   totalMoney
   FROM dwd.dwd_trade_orders
   WHERE status >= 0
     AND dataFlag = '1') tmp
GROUP BY dt;


SELECT *
FROM dws.dws_trade_orders_day
WHERE day_dt BETWEEN '2020-01-01' AND '2020-12-31';
```

### 2.2 指标1，统计2020年每个季度的销售订单笔数、订单总额

先创建ads指标表：`dws_trade_orders_quarter`

```sql
DROP TABLE IF EXISTS dws.dws_trade_orders_quarter;


CREATE TABLE IF NOT EXISTS dws.dws_trade_orders_quarter(YEAR string COMMENT '年份',
                                                        QUARTER string COMMENT '季度',
                                                        cnt decimal COMMENT '订单总笔数',
                                                        SUM decimal COMMENT '订单总额') COMMENT '季度订单统计表';


INSERT OVERWRITE TABLE dws.dws_trade_orders_quarter WITH tmp AS
  (SELECT substr(day_dt, 0, 4) YEAR,
                               CASE WHEN substr(dat_dt, 6, 2)="01"
   OR substr(dat_dt, 6, 2)="02"
   OR substr(day_dt, 6, 2)="03" THEN "1" WHEN substr(dat_dt, 6, 2)="04"
   OR substr(dat_dt, 6, 2)="05"
   OR substr(day_dt, 6, 2)="06" THEN "2" WHEN substr(dat_dt, 6, 2)="07"
   OR substr(dat_dt, 6, 2)="08"
   OR substr(day_dt, 6, 2)="09" THEN "3" WHEN substr(dat_dt, 6, 2)="10"
   OR substr(dat_dt, 6, 2)="11"
   OR substr(day_dt, 6, 2)="12" THEN "4" AS QUARTER day_cnt,
                                     day_sum
   FROM dws.dws_trade_orders_day)
SELECT YEAR,
       QUARTER,
       sum(day_cnt),
       sum(day_sum)
FROM tmp
GROUP BY YEAR QUARTER;
```

### 2.3 统计2020年每个月的销售订单笔数、订单总额

先创建ads指标表：`dws_trade_orders_month`

```sql
DROP TABLE IF EXISTS dws.dws_trade_orders_month;

CREATE TABLE IF NOT EXISTS dws.dws_trade_orders_month(yearstring COMMENT '年份',
                                                      MONTH string COMMENT '月份',
                                                      month_cnt decimal COMMENT '月订单总笔数',
                                                      month_sum decimal COMMENT '月订单总额') COMMENT '月订单统计表';


INSERT OVERWRITE TABLE dws.dws_trade_orders_month WITH tmp AS
  (SELECT substr(day_dt, 0, 4) YEAR,
                               sunstr(day_dt, 6, 2) MONTH,
                                                    day_cnt,
                                                    day_sum
   FROM dws.dws_trade_orders_day)
SELECT YEAR,
       MONTH,
       sum(day_cnt) month_cnt,
       sum(day_sum) month_sum
FROM tmp
GROUP BY YEAR,
         MONTH;
```

### 2.4 统计2020年每周（周一到周日）的销售订单笔数、订单总额

创建ads层指标表：`dws_trade_orders_week` 利用到日期函数`weekofyear`

```sql
DROP TABLE IF EXISTS dws.dws_trade_orders_week;
CREATE TABLE IF NOT EXISTS dws.dws_trade_orders_week(YEAR string COMMENT '年份',
                                                     WEEK string COMMENT '一年中的第几周',
                                                     week_cnt decimal COMMENT '周订单总笔数',
                                                     week_sum decimal COMMENT '周订单总额') COMMENT '周订单统计表';


INSERT OVERWRITE TABLE dws.dws_trade_orders_week
SELECT substr(day_dt, 0, 4) YEAR,
                            weekofyear(day_dt) WEEK,
                                               sum(day_cnt),
                                               sum(day_sum)
FROM dws.dws_trade_orders_day
GROUP BY substr(day_dt, 0, 4) YEAR,
                              weekofyear(day_dt) WEEK;
```

### 2.5 统计2020年国家法定节假日、休息日、工作日的订单笔数、订单总额

创建日期信息维表：`dim_day_info`  并录入节假日信息数据（数据每年都不一样，需要国务院通知的公告，所以定期手动维护）

```sql
drop table if exists dim.dim_day_info;
create table if not exists dim.dim_day_info(
  day_dt string comment '日期',
  is_holidays int comment '节假日标识： 0不是 1是',
  is_workday int comment '工作日标识 0不是 1是'
) comment '日期信息表';
```

```sql
-- 统计2020节假日的订单笔数，订单总额

SELECT nvl(sum(day_cnt), 0) nvl(sum(day_sum), 0)
FROM dws.dws_trade_orders_day A
LEFT JOIN dim.dim_day_info B ON A.day_dt = B.day_dt
WHERE B.is_holiday = 1;

-- 统计2020年休息日的订单笔数，订单总额

SELECT nvl(sum(day_cnt), 0) nvl(sum(day_sum), 0)
FROM dws.dws_trade_orders_day A
LEFT JOIN dim.dim_day_info B ON A.day_dt = B.day_dt
WHERE B.is_workday = 0;

-- 统计2020节工作日的订单笔数，订单总额

SELECT nvl(sum(day_cnt), 0) nvl(sum(day_sum), 0)
FROM dws.dws_trade_orders_day A
LEFT JOIN dim.dim_day_info B ON A.day_dt = B.day_dt
WHERE B.is_workday = 1;
```
