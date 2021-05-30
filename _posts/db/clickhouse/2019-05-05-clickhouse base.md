---
layout:     post
rewards: false
title:      概念
categories:
    - clickhouse
---

# what is OLAP

- 大多数是读请求
- 数据总是以相当大的批(> 1000 rows)进行写入
- 不修改已添加的数据
- 每次查询都从数据库中读取大量的行，但是同时又仅需要少量的列
- 宽表，即每个表包含着大量的列
- 较少的查询(通常每台服务器每秒数百个查询或更少)
- 对于简单查询，允许延迟大约50毫秒
- 列中的数据相对较小： 数字和短字符串(例如，每个URL 60个字节)
- 处理单个查询时需要高吞吐量（每个服务器每秒高达数十亿行）
- 事务不是必须的
- 对数据一致性要求低
- 每一个查询除了一个大表外都很小
- 查询结果明显小于源数据，换句话说，数据被过滤或聚合后能够被盛放在单台服务器的内存中

# vs mysql

数据压缩空间大，减少IO；处理单查询高吞吐量每台服务器每秒最多数十亿行；

高效的使用CPU，数据不仅仅按列存储，同时还按向量进行处理

索引非B树结构，不需要满足最左原则；只要过滤条件在索引列中包含即可；即使在使用的数据不在索引中，由于各种并行处理机制ClickHouse全表扫描的速度也很快

写入速度非常快，50-200M/s，对于大量的数据更新非常适用



1，不支持事务，不支持真正的删除/更新；

2，不支持高并发，官方建议qps为100，可以通过修改配置文件增加连接数，但是在服务器足够好的情况下；

3，SQL满足日常使用80%以上的语法，join写法比较特殊；最新版已支持类似SQL的join，但性能不好；

4，尽量做1000条以上批量的写入，避免逐行insert或小批量的insert，update，delete操作，因为ClickHouse底层会不断的做异步的数据合并，会影响查询性能，这个在做实时数据写入的时候要尽量避开；

5，Clickhouse快是因为采用了并行处理机制，即使一个查询，也会用服务器一半的CPU去执行，所以ClickHouse不能支持高并发的使用场景，默认单查询使用CPU核数为服务器核数的一半，安装时会自动识别服务器核数，可以通过配置文件修改该参数。



1，MySQL单条SQL是单线程的，只能跑满一个core，ClickHouse相反，有多少CPU，吃多少资源，所以飞快；

2，ClickHouse不支持事务，不存在隔离级别。ClickHouse的定位是分析性数据库，而不是严格的关系型数据库。

3，IO方面，MySQL是行存储，ClickHouse是列存储，后者在count()这类操作天然有优势，同时，在IO方面，MySQL需要大量随机IO，ClickHouse基本是顺序IO。






# why clickhouse
<span class='gp-2'>
    <img src='http://ww4.sinaimg.cn/large/006tNc79gy1g41p7txcy8j31iw0u0q7b.jpg' />
    <img src='http://ww3.sinaimg.cn/large/006tNc79gy1g41pc5fdj8j31ma0u0tg1.jpg' />
    <img src='http://ww1.sinaimg.cn/large/006tNc79gy1g41pdsxjgpj31h10u00v3.jpg' />
</span>

# what is clickhouse

ClickHouse是一个用于联机分析(OLAP)的列式数据库管理系统(DBMS)
<span class='gp-2'>
    <img src='http://ww3.sinaimg.cn/large/006tNc79gy1g41pkq41fkj318o0kugod.jpg' />
    <img src='http://ww2.sinaimg.cn/large/006tNc79gy1g41pl4hufhj314u0j2wg7.jpg' />
</span>

对于存储而言，列式数据库总是将同一列的数据存储在一起，不同列的数据也总是分开存储。

## 优点

- 每秒**几亿行**的吞吐能力
- 数据可以持续不断高效的写入到表中，并且写入的过程中不会存在任何加锁的行为。
- 为了高效的使用CPU，数据不仅仅按列存储，同时还按向量(列的一部分)进行处理。
- 支持SQL
- 多核心并行处理 多服务器分布式处理  异步的多主复制技术。当数据被写入任何一个可用副本后，系统会在后台将数据分发给其他副本，以保证系统在不同副本上保持相同的数据

## 缺点

- 仅能用于**批量删除或修改数据** 每次写入不少于1000行的批量写入，或每秒不超过一个写入请求
- 没有完整的事务支持
- 稀疏索引使得ClickHouse不适合通过其键检索单行的点查询。

# IO

- 针对分析类查询，通常只需要读取表的一小部分列。在列式数据库中你可以只读取你需要的数据。例如，如果只需要读取100列中的5列，这将帮助你最少减少20倍的I/O消耗。
- 由于数据总是打包成批量读取的，所以压缩是非常容易的。同时数据按列分别存储这也更容易压缩。这进一步降低了I/O的体积。
- 由于I/O的降低，这将帮助更多的数据被系统缓存。

解压缩的速度主要取决于未压缩数据的大小。

# MergeTree

表由按主键排序的**数据片段** 组成。

当数据被插入到表中时，会分成数据片段并按**主键的字典序排序**。

例如，主键是 (CounterID, Date) 时，片段中数据按 CounterID 排序，具有相同 CounterID 的部分按 Date 排序。
**不同分区的数据会被分成不同的片段**，ClickHouse 在后台合并数据片段以便更高效存储。
**不会合并来自不同分区的数据片段**。这个合并机制并**不保证相同主键的所有行都会合并到同一个数据片段中**。

ClickHouse 会为**每个数据片段创建一个索引文件**，**索引文件包含每个索引行（标记）的主键值**。
索引行号定义为 n * index_granularity 。最大的 n 等于总行数除以 index_granularity 的值的整数部分。
对于每列，跟主键相同的索引行处也会写入**标记**。这些**标记**让你可以直接找到数据所在的列。

你可以只用一单一大表并不断地一块块往里面加入数据 – MergeTree 引擎的就是为了这样的场景。

**ClickHouse 不要求主键唯一。所以，你可以插入多条具有相同主键的行。**

<span class='gp-2'>
    <img src='http://ww3.sinaimg.cn/large/006tNc79gy1g41q7tsr9uj31ld0u0q7g.jpg' />
    <img src='http://ww4.sinaimg.cn/large/006tNc79gy1g41q88nzx5j31ll0u075r.jpg' />
    <img src='http://ww3.sinaimg.cn/large/006tNc79gy1g41qfiihtlj31ko0u0wlz.jpg' />
    <img src='http://ww2.sinaimg.cn/large/006tNc79gy1g41qfz40htj31m30u041z.jpg' />
</span>

# 视图

普通视图与物化视图

```        
CREATE MATERIALIZED VIEW kafka_attack_event_consumer TO attack_event
	AS SELECT toDate(toDateTime(timestamp)) AS date, *
	FROM kafka_attack_event;
```

- 普通视图不存储任何数据，只是执行从另一个表中的读取。换句话说，普通视图只是保存了视图的查询，当从视图中查询时，此查询被作为子查询用于替换FROM子句。

- 物化视图存储的数据是由相应的SELECT查询转换得来的。

# Kafka 引擎

```
CREATE TABLE queue (
    timestamp UInt64,
    level String,
    message String
  ) ENGINE = Kafka('localhost:9092', 'topic', 'group1', 'JSONEachRow');

CREATE TABLE daily (
day Date,
level String,
total UInt64
) ENGINE = SummingMergeTree(day, (day, level), 8192);

CREATE MATERIALIZED VIEW consumer TO daily
AS SELECT toDate(toDateTime(timestamp)) AS day, level, count() as total
FROM queue GROUP BY day, level;

SELECT level, sum(total) FROM daily GROUP BY level;

```
- 使用引擎创建一个 Kafka 消费者并作为一条数据流。
- 创建一个结构表。
- 创建物化视图，改视图会在后台转换引擎中的数据并将其放入之前创建的表中。

当 MATERIALIZED VIEW 添加至引擎，它将会在后台收集数据。可以持续不断地从 Kafka 收集数据并通过 SELECT 将数据转换为所需要的格式。

# 配置

![](http://ww2.sinaimg.cn/large/006tNc79gy1g41qsxh7hgj31mr0u0n1k.jpg)
![](http://ww3.sinaimg.cn/large/006tNc79gy1g41qvabku6j31jp0u0diu.jpg)
![](http://ww1.sinaimg.cn/large/006tNc79gy1g41qx4h4ulj31mj0u0jtz.jpg)
![](http://ww2.sinaimg.cn/large/006tNc79gy1g41qxzqvntj31qe0u0myt.jpg)
![](http://ww4.sinaimg.cn/large/006tNc79gy1g41r0c8kl5j31m20u041b.jpg)
![](http://ww4.sinaimg.cn/large/006tNc79gy1g41r1lxyw2j31ey0u0439.jpg)
![](http://ww3.sinaimg.cn/large/006tNc79gy1g41r25bco9j31kl0u0qbf.jpg)
![](http://ww4.sinaimg.cn/large/006tNc79gy1g41r4f77j4j31c00u0q8s.jpg)
![](http://ww3.sinaimg.cn/large/006tNc79gy1g41r4n5p4rj31c00u0jvl.jpg)
![](http://ww2.sinaimg.cn/large/006tNc79gy1g41r5fgyxjj31c00u0n24.jpg)
![](http://ww2.sinaimg.cn/large/006tNc79gy1g41r68roxpj31c00u0775.jpg)
![](http://ww3.sinaimg.cn/large/006tNc79gy1g41r7ajwcdj31c00u0jsv.jpg)
![](http://ww3.sinaimg.cn/large/006tNc79gy1g41r8uwc0wj31c00u0q58.jpg)