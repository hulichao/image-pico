# 算法-一个经典sql 题和一个Java算法题

# 1.sql题描述

话说有一个日志表，只有两列，分别是连续id和num 至于啥意思，把它当金额把。现在想知道连续次数3次及以上的num，数据如下

| id | num |
| -- | --- |
| 1  | 1   |
| 2  | 1   |
| 3  | 1   |
| 4  | 2   |
| 5  | 3   |
| 6  | 4   |
| 7  | 4   |
| 8  | 4   |

那么结果只有1，4满足条件，问这个sql该怎么写？

# 2.思路和解法

分析：题目简单，没有歧义，能看得懂，像连续几次的这种问题一定是用到窗口函数，首先想到的是排名`row_number`  然后`lag`  怎么体现连续呢，肯定是需要用到一个排序的id，由于题目给了id是连续递增的，可以省去row\_number了

所以第一步，上lag，结果就是如下：

| id | num | lagid |
| -- | --- | ----- |
| 1  | 1   | null  |
| 2  | 1   | 0     |
| 3  | 1   | 0     |
| 4  | 2   | 1     |
| 5  | 3   | 1     |
| 6  | 4   | 1     |
| 7  | 4   | 0     |
| 8  | 4   | 0     |

得到lagid后，连续怎么用呢，首先只有为0的才满足条件，所以可以做一个筛选，结果就如下表去掉`xxx`的，下面观察0的行，怎么区分3 行的 0 和 7行的 0呢，想到使用新分组，rid 这样就把lagid 相同，num相同的排序，最后再加一列，id-rid 相同的分为一组

| id | num | lagid        | rid | gid |
| -- | --- | ------------ | --- | --- |
| 1  | 1   | null   xxx   |     |     |
| 2  | 1   | 0            | 1   | 1   |
| 3  | 1   | 0            | 2   | 1   |
| 4  | 2   | 1        xxx |     |     |
| 5  | 3   | 1        xxx |     |     |
| 6  | 4   | 1        xxx |     |     |
| 7  | 4   | 0            | 1   | 6   |
| 8  | 4   | 0            | 2   | 6   |

```sql
-- 完整sql
## 解法1
SELECT num
FROM
  (SELECT id,
          num,
          lagid,
          (id-row_number() over(PARTITION BY num, lagid
                                ORDER BY id)) AS gid
   FROM
     (SELECT id,
             num,
             num- lag(num) (OVER PARTITION BY 1
                            ORDER BY id) AS lagid) tmp1
   WHERE lagid=0 ) tmp2
GROUP BY num,
         gid
HAVING count(*) >= 2


## 解法2  
select
  num,
  gid,
  count(1) as c
from
(
select
id,
num,
id-row_number() over(PARTITION BY num ORDER BY id) as gid
from 
(select * from logs order by num,id) a
) b
group by num,gid
```

后面想到了更好的，其实不用`lag`，也不用`order by `全局排序,id 的作用和日期一样，一般是用来配合`row_number`来解决连续问题的，所以`row_number`必不可少，那么可以这样写（神他妈简单是不是，别想复杂了）：

```sql
SELECT num,
       gid
FROM
  (SELECT num,
          id-row_number() OVER (PARTITION BY num
                                ORDER BY id) gid
   FROM logs)
GROUP BY num,
         gid
HAVING count(1) >= 3
 
```



# 3. Java题描述

首先，给你一个初始数组 arr。然后，每天你都要根据前一天的数组生成一个新的数组。第 i 天所生成的数组，是由你对第 i-1 天的数组进行如下操作所得的：假如一个元素小于它的左右邻居，那么该元素自增 1。假如一个元素大于它的左右邻居，那么该元素自减 1。

首、尾元素 永不 改变。

过些时日，你会发现数组将会不再发生变化，请返回最终所得到的数组。

示例 1：

输入：\[6,2,3,4]

输出：\[6,3,3,4]

解释：

第一天，数组从 \[6,2,3,4] 变为 \[6,3,3,4]。

无法再对该数组进行更多操作。

示例 2：

输入：\[1,6,3,4,3,5]

输出：\[1,4,4,4,4,5]

解释：

第一天，数组从 \[1,6,3,4,3,5] 变为 \[1,5,4,3,4,5]。

第二天，数组从 \[1,5,4,3,4,5] 变为 \[1,4,4,4,4,5]。

无法再对该数组进行更多操作。

# 3.3 分析和解法

1.  首先考虑一轮遍历怎么写，应该很简单把，思路就是一个大小为3的窗口
2.  用一个flag来标志每一轮是否有改过数据。那么代码如下：

```sql
public int[] get(int[] input) {
    if (input == null || input.length <=2)
        return input;
    boolean flag = false;
        do {
            flag = false;
            for (int i=1;i+1 < input.length;i++){
                if (input[i] < input[i+1] && input[i] < input[i-1] ) {
                    input[i] +=1;
                    if (!flag)
                        flag = true;
                }
                if (input[i] > input[i+1] && input[i] > input[i-1] ) {
                    input[i] -=1;
                    if (!flag)
                        flag = true;
                    }
            }
        } while(flag)
        return input;
}
```
