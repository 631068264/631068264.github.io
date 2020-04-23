---
layout:     post
rewards: false
title:      记一次kafka clickhouse 排错
categories:
    - clickhouse
---

# 背景

```
CREATE TABLE kafka_xx (
	...
) ENGINE = Kafka SETTINGS kafka_broker_list = 'localhost:9092', kafka_topic_list = 'xx', kafka_group_name = 'group1', kafka_format = 'JSONEachRow', kafka_max_block_size = 100;

CREATE MATERIALIZED VIEW kafka_xx_consumer TO xx
	AS SELECT toDate(toDateTime(end_time)) AS date, *
	FROM kafka_xx;
```

table xx 通过消费kafka插入数据


# 问题

发现前端没有数据展示，查看表只有前几天的数据


# 排查


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

    ssl
    ```
    bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic xx --consumer.config config/client_security.properties
    ```
    发现有数据可以消费，从数据带有的时间戳是**今天的**，那**clickhouse消费了啥**
    
    ```
    bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group group1 --command-config config/client_security.properties
    ```
    
    - **CURRENT-OFFSET**  消费到哪个offset
    - **LOG-END-OFFSET** 最新的offset 
    - **LAG** offset相差
    
    ![](https://tva1.sinaimg.cn/large/007S8ZIlgy1ge434ns6ojj321i0tyn91.jpg)
    
    > 原因找到了clickhouse消费速度跟不上生产速度，差距还越拉越大

# 解决

- 加大消费力

    增大**kafka_max_block_size**，可以直接[drop table](https://clickhouse.tech/docs/en/sql_reference/statements/misc/#drop)再重建，
    [不会影响offset](https://stackoverflow.com/a/49899391/5360312)

- 参数优化

    https://clickhouse.tech/docs/en/engines/table_engines/integrations/kafka/#description
    
    ![](https://tva1.sinaimg.cn/large/007S8ZIlgy1ge43l3jlebj31cs06ajrp.jpg)
