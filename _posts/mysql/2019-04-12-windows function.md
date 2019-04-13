---
layout:     post
rewards: false
title:      窗口函数 分析函数
categories:
    - mysql
---

# 概念
MySQL中的使用窗口函数的时候，是不允许使用*的，必须显式指定每一个字段。8.0后才有Window Function，其他数据库貌似一早就有

窗口函数有点像聚合函数，an aggregate operation groups query rows into a single result row, 
a window function produces a result for each query row

普通聚合函数
```sql
mysql> SELECT SUM(profit) AS total_profit
       FROM sales;
+--------------+
| total_profit |
+--------------+
|         7535 |
+--------------+
mysql> SELECT country, SUM(profit) AS country_profit
       FROM sales
       GROUP BY country
       ORDER BY country;
+---------+----------------+
| country | country_profit |
+---------+----------------+
| Finland |           1610 |
| India   |           1350 |
| USA     |           4575 |
+---------+----------------+
```

窗口函数
```sql
mysql> SELECT
         year, country, product, profit,
         SUM(profit) OVER() AS total_profit, -- OVER子句为空，它将整个查询行集视为单个分区。因此，窗函数产生全局和，但对每一行都这样做。
         SUM(profit) OVER(PARTITION BY country) AS country_profit -- 按国家/地区划分行，每个分区（每个国家/地区）生成一个总和。该函数为每个分区行生成此总和。
       FROM sales
       ORDER BY country, year, product, profit;
+------+---------+------------+--------+--------------+----------------+
| year | country | product    | profit | total_profit | country_profit |
+------+---------+------------+--------+--------------+----------------+
| 2000 | Finland | Computer   |   1500 |         7535 |           1610 |
| 2000 | Finland | Phone      |    100 |         7535 |           1610 |
| 2001 | Finland | Phone      |     10 |         7535 |           1610 |
| 2000 | India   | Calculator |     75 |         7535 |           1350 |
| 2000 | India   | Calculator |     75 |         7535 |           1350 |
| 2000 | India   | Computer   |   1200 |         7535 |           1350 |
| 2000 | USA     | Calculator |     75 |         7535 |           4575 |
| 2000 | USA     | Computer   |   1500 |         7535 |           4575 |
| 2001 | USA     | Calculator |     50 |         7535 |           4575 |
| 2001 | USA     | Computer   |   1200 |         7535 |           4575 |
| 2001 | USA     | Computer   |   1500 |         7535 |           4575 |
| 2001 | USA     | TV         |    100 |         7535 |           4575 |
| 2001 | USA     | TV         |    150 |         7535 |           4575 |
+------+---------+------------+--------+--------------+----------------+
```

# 执行方式

> Window functions are permitted only in the select list and ORDER BY clause. 
> Query result rows are determined from the FROM clause, after WHERE, GROUP BY, and HAVING processing,
> and windowing execution occurs before ORDER BY, LIMIT, and SELECT DISTINCT.

# 窗口函数

一般的聚合函数可以作为窗口函数

<span class='gp-2'>
    <img src='https://ws3.sinaimg.cn/large/006tNc79ly1g20tqixrt3j316c0kojso.jpg' />
    <img src='https://ws2.sinaimg.cn/large/006tNc79ly1g20tr48ybwj30ug0u00uj.jpg' />
</span>

非聚合窗口函数
```sql
mysql> SELECT
         year, country, product, profit,
         ROW_NUMBER() OVER(PARTITION BY country) AS row_num1,
         ROW_NUMBER() OVER(PARTITION BY country ORDER BY year, product) AS row_num2
       FROM sales;
+------+---------+------------+--------+----------+----------+
| year | country | product    | profit | row_num1 | row_num2 |
+------+---------+------------+--------+----------+----------+
| 2000 | Finland | Computer   |   1500 |        2 |        1 |
| 2000 | Finland | Phone      |    100 |        1 |        2 |
| 2001 | Finland | Phone      |     10 |        3 |        3 |
| 2000 | India   | Calculator |     75 |        2 |        1 |
| 2000 | India   | Calculator |     75 |        3 |        2 |
| 2000 | India   | Computer   |   1200 |        1 |        3 |
| 2000 | USA     | Calculator |     75 |        5 |        1 |
| 2000 | USA     | Computer   |   1500 |        4 |        2 |
| 2001 | USA     | Calculator |     50 |        2 |        3 |
| 2001 | USA     | Computer   |   1500 |        3 |        4 |
| 2001 | USA     | Computer   |   1200 |        7 |        5 |
| 2001 | USA     | TV         |    150 |        1 |        6 |
| 2001 | USA     | TV         |    100 |        6 |        7 |
+------+---------+------------+--------+----------+----------+
```

# 语法
```sql
over_clause:
    {OVER (window_spec) | OVER window_name}
```
## OVER (window_spec)
```sql
window_spec:
    [window_name] [partition_clause] [order_clause] [frame_clause]
```
- partition_clause

> `PARTITION BY`子句指示如何将查询行分区，执行窗口函数的结果是基于该分区的所有行，没有`PARTITION BY`代表所有行是一个单独分区。

- frame_clause

> frame当前分区的子集 帧是根据当前行确定的，这使得**帧能够在分区内移动**，具体取决于其分区内当前行的位置 => 就是**移动窗口**

```sql
mysql> SELECT
         time, subject, val,
         SUM(val) OVER (PARTITION BY subject ORDER BY time
                        ROWS UNBOUNDED PRECEDING) -- 该分区内 sum(当前值,sum(前一个值))
           AS running_total,
         AVG(val) OVER (PARTITION BY subject ORDER BY time
                        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING) -- 该分区内 avg(后一个值,当前值,前一个值)
           AS running_average
       FROM observations;
+----------+---------+------+---------------+-----------------+
| time     | subject | val  | running_total | running_average |
+----------+---------+------+---------------+-----------------+
| 07:00:00 | st113   |   10 |            10 |          9.5000 |
| 07:15:00 | st113   |    9 |            19 |         14.6667 |
| 07:30:00 | st113   |   25 |            44 |         18.0000 |
| 07:45:00 | st113   |   20 |            64 |         22.5000 |
| 07:00:00 | xh458   |    0 |             0 |          5.0000 |
| 07:15:00 | xh458   |   10 |            10 |          5.0000 |
| 07:30:00 | xh458   |    5 |            15 |         15.0000 |
| 07:45:00 | xh458   |   30 |            45 |         20.0000 |
| 08:00:00 | xh458   |   25 |            70 |         27.5000 |
+----------+---------+------+---------------+-----------------+
```

只有聚合函数和`FIRST_VALUE() LAST_VALUE() NTH_VALUE()`可以使用，其他的用了会被ignore，use the entire partition even if a frame is specified

```sql
mysql> SELECT
         time, subject, val,
         FIRST_VALUE(val)  OVER w AS 'first',
         LAST_VALUE(val)   OVER w AS 'last',
         NTH_VALUE(val, 2) OVER w AS 'second',
         NTH_VALUE(val, 4) OVER w AS 'fourth'
       FROM observations
       WINDOW w AS (PARTITION BY subject ORDER BY time
                    ROWS UNBOUNDED PRECEDING);
+----------+---------+------+-------+------+--------+--------+
| time     | subject | val  | first | last | second | fourth |
+----------+---------+------+-------+------+--------+--------+
| 07:00:00 | st113   |   10 |    10 |   10 |   NULL |   NULL |
| 07:15:00 | st113   |    9 |    10 |    9 |      9 |   NULL |
| 07:30:00 | st113   |   25 |    10 |   25 |      9 |   NULL |
| 07:45:00 | st113   |   20 |    10 |   20 |      9 |     20 |
| 07:00:00 | xh458   |    0 |     0 |    0 |   NULL |   NULL |
| 07:15:00 | xh458   |   10 |     0 |   10 |     10 |   NULL |
| 07:30:00 | xh458   |    5 |     0 |    5 |     10 |   NULL |
| 07:45:00 | xh458   |   30 |     0 |   30 |     10 |     30 |
| 08:00:00 | xh458   |   25 |     0 |   25 |     10 |     30 |
+----------+---------+------+-------+------+--------+--------+
```


## OVER window_name

```sql
WINDOW window_name AS (window_spec)
    [, window_name AS (window_spec)] ...
    
```

按名称引用窗口，可以更简单地编写查询
```sql
SELECT
  val,
  ROW_NUMBER() OVER w AS 'row_number',
  RANK()       OVER w AS 'rank',
  DENSE_RANK() OVER w AS 'dense_rank'
FROM numbers
WINDOW w AS (ORDER BY val);

```
子句中使用以不同方式修改窗口
```sql
SELECT
  DISTINCT year, country,
  FIRST_VALUE(year) OVER (w ORDER BY year ASC) AS first,
  FIRST_VALUE(year) OVER (w ORDER BY year DESC) AS last
FROM sales
WINDOW w AS (PARTITION BY country);
```

# CTE
CTE有两种用法，非递归的CTE和递归的CTE。

非递归的CTE可以用来增加代码的可读性，增加逻辑的结构化表达。
```sql
WITH cte as (
select row_number()over(partition by course order by score desc) as row_num, 
		id,
		name,
		course,
		score 
	from tb )


select id,name,course,score from cte where row_num=1;
```
