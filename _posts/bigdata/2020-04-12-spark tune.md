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