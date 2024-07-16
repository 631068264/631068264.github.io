---
layout:     post
rewards: false
title:      引擎介绍
categories:
- clickhouse
---


![total-table-engine.png](https://cdn.jsdelivr.net/gh/631068264/img/008i3skNgy1gsva6l6t2cj313x0ohjtl.jpg)

# log 系列

主要用于快速写入小表（1百万行左右的表），然后全部读出的场景。

Log表引擎的共性是：

- 数据被顺序append写到磁盘上；
- 不支持delete、update；
- 不支持index；
- 不支持原子性写；
- insert会阻塞select操作。

区别是：

- TinyLog：不支持并发读取数据文件，查询性能较差；格式简单，适合用来暂存中间数据；
- StripLog：支持并发读取数据文件，查询性能比TinyLog好；将所有列存储在同一个大文件中，减少了文件个数；
- Log：支持并发读取数据文件，查询性能比TinyLog好；每个列会单独存储在一个独立文件中。

# MergeTree系列

```sql
-- 强制后台Compaction
optimize table [taleName] final;
```



##  MergeTree

MergeTree表引擎主要用于海量数据分析，支持数据分区、存储有序、主键索引、稀疏索引、数据TTL等。MergeTree支持所有ClickHouse SQL语法，但是有些功能与MySQL并不一致，比如在MergeTree中主键并不用于去重。

MergeTree虽然有主键索引，但是其主要作用是加速查询，而不是类似MySQL等数据库用来保持记录唯一。即便在Compaction完成后，主键相同的数据行也仍旧共同存在。

## ReplacingMergeTree

为了解决MergeTree相同主键无法去重的问题，ClickHouse提供了ReplacingMergeTree引擎，用来做去重。

虽然ReplacingMergeTree提供了**主键去重的能力，但是仍旧有以下限制**：

- 在没有彻底optimize之前，可能无法达到主键去重的效果，比如部分数据已经被去重，而另外一部分数据仍旧有主键重复；
- 在分布式场景下，相同primary key的数据可能被sharding到不同节点上，不同shard间可能无法去重；
- optimize是后台动作，无法预测具体执行时间点；
- 手动执行optimize在海量数据场景下要消耗大量时间，无法满足业务即时查询的需求；

因此ReplacingMergeTree更多被用于**确保数据最终被去重，而无法保证查询过程中主键不重复。**

## CollapsingMergeTree

ClickHouse实现了CollapsingMergeTree来消除ReplacingMergeTree的限制。该引擎要求在建表语句中指定一个标记列Sign，后台Compaction时会将主键相同、Sign相反的行进行折叠，也即删除。



CollapsingMergeTree将行按照Sign的值分为两类：Sign=1的行称之为状态行，Sign=-1的行称之为取消行。

**每次需要新增状态时，写入一行状态行；需要删除状态时，则写入一行取消行。** 有先后次序。



在后台Compaction时，状态行与取消行会自动做折叠（删除）处理。而尚未进行Compaction的数据，状态行与取消行同时存在。



**业务逻辑改造**

- 执行删除操作需要写入取消行，而取消行中需要包含与原始状态行一样的数据（Sign列除外）。所以在应用层需要记录原始状态行的值，或者在执行删除操作前先查询数据库获取原始状态行。
- 由于后台Compaction时机无法预测，在发起查询时，状态行和取消行可能尚未被折叠。ClickHouse无法保证primary key相同的行落在同一个节点上，不在同一节点上的数据无法折叠。
- 因此在进行count(*)、sum(col)等聚合计算时，可能会存在数据冗余的情况。为了获得正确结果，业务层需要改写SQL，将`count()、sum(col)`分别改写为`sum(Sign)、sum(col * Sign)`。

**缺点**

CollapsingMergeTree虽然解决了主键相同的数据即时删除的问题，但是状态持续变化且多线程并行写入情况下，**状态行与取消行位置可能乱序，导致无法正常折叠。**

## VersionedCollapsingMergeTree

解决CollapsingMergeTree乱序写入情况下无法正常折叠问题



建表语句中**新增了一列Version**，用于在乱序情况下记录状态行与取消行的对应关系。主键相同，且Version相同、Sign相反的行，在Compaction时会被删除。



**业务层同让要改写。和CollapsingMergeTree一样**

## SummingMergeTree

SummingMergeTree支持**对主键列进行预先聚合**。在后台Compaction时，会将**主键相同的多行进行sum求和**，然后使用一行数据取而代之，从而大幅度降低存储空间占用，提升聚合计算性能。



- ClickHouse**只在后台Compaction时才会进行数据的预先聚合**，而compaction的执行时机无法预测，所以可能存在部分数据已经被预先聚合、部分数据尚未被聚合的情况。因此，**在执行聚合计算时，SQL中仍需要使用GROUP BY子句**。
- 在预先聚合时，ClickHouse会对主键列之外的其他所有列进行预聚合。如果这些列是可聚合的（比如数值类型），则直接sum；如果不可聚合（比如String类型），则随机选择一个值。
- 通常建议将SummingMergeTree与MergeTree配合使用，使用MergeTree来存储具体明细，使用SummingMergeTree来存储预先聚合的结果加速查询。

## AggregatingMergeTree

与SummingMergeTree的区别在于：

- SummingMergeTree对非主键列进行sum聚合

- AggregatingMergeTree则可以指定各种聚合函数

结合物化视图或ClickHouse的特殊数据类型AggregateFunction一起使用。在insert和select时，也有独特的写法和要求：写入时需要使用-State语法，查询时使用-Merge语法。



示例一：配合物化视图使用。

```sql
-- 建立明细表
CREATE TABLE visits
(
    UserID UInt64,
    CounterID UInt8,
    StartDate Date,
    Sign Int8
)
ENGINE = CollapsingMergeTree(Sign)
ORDER BY UserID;

-- 对明细表建立物化视图，该物化视图对明细表进行预先聚合
-- 注意：预先聚合使用的函数分别为： sumState, uniqState。对应于写入语法<agg>-State.
CREATE MATERIALIZED VIEW visits_agg_view
ENGINE = AggregatingMergeTree() PARTITION BY toYYYYMM(StartDate) ORDER BY (CounterID, StartDate)
AS SELECT
    CounterID,
    StartDate,
    sumState(Sign)    AS Visits,
    uniqState(UserID) AS Users
FROM visits
GROUP BY CounterID, StartDate;

-- 插入明细数据
INSERT INTO visits VALUES(0, 0, '2019-11-11', 1);
INSERT INTO visits VALUES(1, 1, '2019-11-12', 1);

-- 对物化视图进行最终的聚合操作
-- 注意：使用的聚合函数为 sumMerge， uniqMerge。对应于查询语法<agg>-Merge.
SELECT
    StartDate,
    sumMerge(Visits) AS Visits,
    uniqMerge(Users) AS Users
FROM visits_agg_view
GROUP BY StartDate
ORDER BY StartDate;

-- 普通函数 sum, uniq不再可以使用
-- 如下SQL会报错： Illegal type AggregateFunction(sum, Int8) of argument 
SELECT
    StartDate,
    sum(Visits),
    uniq(Users)
FROM visits_agg_view
GROUP BY StartDate
ORDER BY StartDate;
```

示例二：配合特殊数据类型AggregateFunction使用。

```sql
-- 建立明细表
CREATE TABLE detail_table
(   CounterID UInt8,
    StartDate Date,
    UserID UInt64
) ENGINE = MergeTree() 
PARTITION BY toYYYYMM(StartDate) 
ORDER BY (CounterID, StartDate);

-- 插入明细数据
INSERT INTO detail_table VALUES(0, '2019-11-11', 1);
INSERT INTO detail_table VALUES(1, '2019-11-12', 1);

-- 建立预先聚合表，
-- 注意：其中UserID一列的类型为：AggregateFunction(uniq, UInt64)
CREATE TABLE agg_table
(   CounterID UInt8,
    StartDate Date,
    UserID AggregateFunction(uniq, UInt64)
) ENGINE = AggregatingMergeTree() 
PARTITION BY toYYYYMM(StartDate) 
ORDER BY (CounterID, StartDate);

-- 从明细表中读取数据，插入聚合表。
-- 注意：子查询中使用的聚合函数为 uniqState， 对应于写入语法<agg>-State
INSERT INTO agg_table
select CounterID, StartDate, uniqState(UserID)
from detail_table
group by CounterID, StartDate

-- 不能使用普通insert语句向AggregatingMergeTree中插入数据。
-- 本SQL会报错：Cannot convert UInt64 to AggregateFunction(uniq, UInt64)
INSERT INTO agg_table VALUES(1, '2019-11-12', 1);

-- 从聚合表中查询。
-- 注意：select中使用的聚合函数为uniqMerge，对应于查询语法<agg>-Merge
SELECT uniqMerge(UserID) AS state 
FROM agg_table 
GROUP BY CounterID, StartDate;
```

