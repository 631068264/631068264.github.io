---
layout:     post
rewards: false
title:  spark 参数调优
categories:
    - big data
tags:
    - big data
---

# spark on yarn

- [spark 参数详情](https://spark.apache.org/docs/latest/configuration.html)

example
```
spark-submit --class com.yizhisec.bigdata.TrafficEs 
--master yarn 
--deploy-mode cluster 
--executor-memory 512M 
--executor-cores 2 
--conf spark.streaming.concurrentJobs=5 
--num-executors 30 
--supervise bigdata-1.0.jar
```


- deploy-mode

    In cluster mode, the Spark driver runs inside an application master process which is managed by YARN on the cluster,
    and **the client can go away after initiating the application**. (部署完看log没啥问题就可以ctrl-c, application自动在后台跑)

    In client mode, the driver runs in the client process, and the application master is only used for requesting resources from YARN


- supervise

    make sure that the driver is automatically restarted if it fails with a non-zero exit code


- num-executors

    多少个executor进程来执行 (不设定，默认只会启动非常少的Executor。如果设得太小，无法充分利用计算资源。设得太大的话，又会抢占集群或队列的资源，导致其他作业无法顺利执行。)


- executor-cores

    每个个executor上的core数  一次能同时运行的task数。一个Spark应用最多可以同时运行的task数为**num-executors*executor-cores**

- executor-memory

    每个Executor的内存量

# 指标来源

- 通过ES API获取index详情 **docs.count**

```
http://node2:9200/_cat/indices/loh_traffic?format=json

[
    {
        "health": "yellow",
        "status": "open",
        "index": "loh_traffic",
        "uuid": "qmgjY2r4RFWbAneDZyzjSw",
        "pri": "1",
        "rep": "1",
        "docs.count": "5887176",
        "docs.deleted": "0",
        "store.size": "248.6mb",
        "pri.store.size": "248.6mb"
    }
]
```

- spark kafka **处理过的offset**

通过kafka脚本获取不到consumer的offset，因为没有[自动commit](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html#kafka-specific-configurations)
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301030371.jpg)

[最后通过checkpointLocation读取offset，读取hadoop文件](https://stackoverflow.com/a/55808666/5360312) 


# spark概念

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301030373.jpg)

使用`yarn application --list`通过**Tracking-URL**查看spark application工作情况

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301030374.jpg)


# 测试
4核+16G 服务器

- 推荐 task数量 = 设置成spark Application 总cpu core  = (num-executors * executor-cores)数量的2~3倍


-----

-  分区

这里只是针对**处理流程**分区

partition 约等于等于task数量

```
--executor-memory 512M --executor-cores 2 --num-executors 5 
```

partition=1000 kafka偏移很少去到3000

```
total: 6689868 总时间: 60127.31 ms store: 302.3mb
过滤后数量 435 avg 138.22370 ms/条
kafka偏移 2100 avg 28.63205 ms/条

total: 6691913 总时间: 60096.70 ms store: 302.4mb
过滤后数量 1240 avg 48.46508 ms/条
kafka偏移 3673 avg 16.36175 ms/条
```

partition=10

```
[I 200415 05:48:19 es_monitor:73]
total: 6978213 总时间: 60086.03 ms store: 310.7mb
过滤后数量 915 avg 65.66780 ms/条
kafka偏移 4722 avg 12.72470 ms/条

[I 200415 05:54:20 es_monitor:73]
total: 6981605 总时间: 60116.86 ms store: 314.3mb
过滤后数量 567 avg 106.02622 ms/条
kafka偏移 3248 avg 18.50889 ms/条
```

partition=30 

task 比较平均分配到各个executors
```
[I 200415 06:20:23 es_monitor:73]
total: 6995916 总时间: 60102.70 ms store: 309mb
过滤后数量 643 avg 93.47231 ms/条
kafka偏移 3133 avg 19.18375 ms/条

[I 200415 06:21:24 es_monitor:73]
total: 6996504 总时间: 60103.43 ms store: 309mb
过滤后数量 588 avg 102.21672 ms/条
kafka偏移 3078 avg 19.52678 ms/条
```

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301030375.jpg)


-  提高num-executors

内存从873.3M提高到2.5G

```
--executor-memory 512M --executor-cores 2 --num-executors 500
```
partition=3000

```
[I 200415 06:35:26 es_monitor:73]
total: 7005214 总时间: 60182.00 ms store: 309.6mb
过滤后数量 672 avg 89.55655 ms/条
kafka偏移 2436 avg 24.70526 ms/条

[I 200415 06:36:26 es_monitor:73]
total: 7005934 总时间: 60205.32 ms store: 310.1mb
过滤后数量 720 avg 83.61850 ms/条
kafka偏移 2721 avg 22.12617 ms/条

[I 200415 06:37:26 es_monitor:73]
total: 7006723 总时间: 60102.16 ms store: 311.3mb
过滤后数量 789 avg 76.17511 ms/条
kafka偏移 3462 avg 17.36053 ms/条

[I 200415 06:38:30 es_monitor:73]
total: 7007527 总时间: 63803.23 ms store: 309.7mb
过滤后数量 804 avg 79.35725 ms/条
kafka偏移 2470 avg 25.83127 ms/条
```

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301030376.jpg)

不能完全发挥理论上 cores = 2 * 500  上面的测试达到了

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301030377.jpg)