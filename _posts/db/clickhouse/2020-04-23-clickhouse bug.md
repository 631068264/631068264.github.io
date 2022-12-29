---
layout:     post
rewards: false
title:      记一次kafka clickhouse 排错
categories:
    - clickhouse
---

# 背景

```sql
CREATE TABLE kafka_xx (
	...
) ENGINE = Kafka SETTINGS kafka_broker_list = 'localhost:9092', kafka_topic_list = 'xx', kafka_group_name = 'group1', kafka_format = 'JSONEachRow', kafka_max_block_size = 100;

CREATE MATERIALIZED VIEW kafka_xx_consumer TO xx
	AS SELECT toDate(toDateTime(end_time)) AS date, *
	FROM kafka_xx;
```

table xx 通过消费kafka插入数据


# 问题一

发现前端没有数据展示，查看表只有前几天的数据



## 初步

初步看log,认为是数据是没有，或者程序哪里有bug

- 看后台log

- clickhouse log

    `/var/log/clickhouse-server/clickhouse-server.log` log路径
    
    ```
    2020.04.23 15:20:02.326403 [ 34194 ] {} <Trace> StorageKafka (kafka_xx): Polled batch of 100 messages. Offset position: [ xx[0:7223895730] ]
    。。。。
    2020.04.23 15:20:02.851093 [ 34195 ] {} <Trace> StorageKafka (kafka_xx): Committed offset 7223895830 (topic: xx, partition: 0)
    ```
    
    发现**clickhouse**在消费kafka 瞄了下消费记录大概**200 rows/sec**
    
- 使用kafka命令

    查看 consumer-groups list
    
    ```shell
    bin/kafka-consumer-groups.sh  --list --bootstrap-server localhost:9092 --command-config config/client_security.properties
    ```
    
    ssl   消费某个topic
    ```shell
    bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic xx --consumer.config config/client_security.properties
    ```
    发现有数据可以消费，从数据带有的时间戳是**今天的**，那**clickhouse消费了啥**
    
    ```shell
    bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group group1 --command-config config/client_security.properties
    ```
    
    - **CURRENT-OFFSET**  消费到哪个offset
    - **LOG-END-OFFSET** 最新的offset 
    - **LAG** offset相差
    
    ![](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1ge434ns6ojj321i0tyn91.jpg)
    
    > 原因找到了clickhouse消费速度跟不上生产速度，差距还越拉越大

## 解决

- 加大消费力

    增大**kafka_max_block_size**，可以直接[drop table](https://clickhouse.tech/docs/en/sql_reference/statements/misc/#drop)再重建，
    [不会影响offset](https://stackoverflow.com/a/49899391/5360312)

- 参数优化

    https://clickhouse.tech/docs/en/engines/table_engines/integrations/kafka/#description
    
    ![](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1ge43l3jlebj31cs06ajrp.jpg)


# engine 改变

- [How to change PARTITION in clickhouse](https://stackoverflow.com/questions/61452077/how-to-change-partition-in-clickhouse)

```
CREATE TABLE traffic2 (
	`date` Date,
	。。。
) ENGINE = MergeTree()
PARTITION BY date
ORDER BY (end_time);

detach table kafka_traffic_consumer;

insert into traffic2 select * from traffic;

drop table traffic;

rename table traffic2 to traffic;

attach table kafka_traffic_consumer;
```

# 问题二

数据库没有缺数据查不出来

## 背景

配置文件

```
/etc/clickhouse-server/config.xml
/etc/clickhouse-server/users.xml
```

因为数据量比较大在**users.xml**设置了几个user，然后里面设置了**max_rows_to_read**  `<max_rows_to_read>20000000</max_rows_to_read>`

- clickhouse查询会使用**多线程**，没有使用**PARTITION key**，会有多个线程一起查，到了**max_rows_to_read**就停止了。

    这样会导致一种境况，可能每次查询到的**结果都有些不一样**

- 使用了**PARTITION key**可以限制查询分区，**按月分区**
  
    如果分区里面的数量太大，也可能查询不了，应该再细分分区**按日**，减少分区数据量，但是**这样会使磁盘空间变大，提高按日的性能。**

