---

layout:     post
rewards: false
title:  elasticsearch 分片
categories:
    - es
tags:
    - big data
---

# 分片

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301017558.jpg)

![image-20201004111652659](https://cdn.jsdelivr.net/gh/631068264/img/202212301017559.jpg)

![image-20201004111812510](https://cdn.jsdelivr.net/gh/631068264/img/202212301017560.jpg)




# 主分片

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301017561.jpg)

# 副本分片

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301017562.jpg)


# 分片数设定

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301017563.jpg)

##  分片过多  增加节点 rebalance
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301017564.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301017565.jpg)

增加一台服务器，此时shard是如何分配的呢

Rebalance（再平衡），当集群中节点数量发生变化时，将会触发es集群的rebalance，即重新分配shard。Rebalance的原则就是尽量使shard在节点中分布均匀，**primary shard和replica shard不能分配到一个节点上的**，达到负载均衡的目的。

![image-20210530093202876](https://cdn.jsdelivr.net/gh/631068264/img/202212301017566.jpg)

## 故障转移 集群容灾

![image-20210530093334113](https://cdn.jsdelivr.net/gh/631068264/img/202212301017567.jpg)

- primary shard 所在节点发生故障（**red, 因为部分主分片不可用**）

- 当es集群中的master节点发生故障，重新选举master节点。
- master行驶其分片分配的任务。
- master会寻找node1节点上的P0分片的replica shard,  replica shard将被提升为primary shard。**这个升级过程是瞬间完成，集群的健康状态为yellow，因为不是每一个replica shard都是active的**。

- R0 也会重新分配，集群变绿

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301017568.jpg)



# 文档保存到分片

![image-20201004225303315](https://cdn.jsdelivr.net/gh/631068264/img/202212301017569.jpg)

所以设置好index后主分片数不能改，整个存储完全不一样

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301017570.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301017571.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301017572.jpg)



# 分片设计
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301017573.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301017574.jpg)

![image-20201005215651480](https://cdn.jsdelivr.net/gh/631068264/img/202212301017575.jpg)

![image-20201005215758649](https://cdn.jsdelivr.net/gh/631068264/img/202212301017576.jpg)

![image-20201005215824881](https://cdn.jsdelivr.net/gh/631068264/img/202212301017577.jpg)

![image-20201005215841945](https://cdn.jsdelivr.net/gh/631068264/img/202212301017578.jpg)

![image-20201005215956469](https://cdn.jsdelivr.net/gh/631068264/img/202212301017579.jpg)

# 分片机制 写入原理

[ES内部分片处理机制](https://my.oschina.net/LucasZhu/blog/1542850)

## 倒排索引不可变性

![image-20201004230145470](https://cdn.jsdelivr.net/gh/631068264/img/202212301017580.jpg)

删除的文档不会立即清理

多个segment（倒排索引文件）=> lucene index => es shard

![image-20201006102814479](https://cdn.jsdelivr.net/gh/631068264/img/202212301017581.jpg)

## refresh

数据先入buffer buffer 到 segment过程叫refresh

refresh后才数据会被搜索到

![image-20201004230453959](https://cdn.jsdelivr.net/gh/631068264/img/202212301017582.jpg)



## Transaction Log

防断电

![image-20201004231400579](https://cdn.jsdelivr.net/gh/631068264/img/202212301017583.jpg)

## flush

![image-20201004232216254](https://cdn.jsdelivr.net/gh/631068264/img/202212301017584.jpg)

## 总结
### 插入

![img](https://cdn.jsdelivr.net/gh/631068264/img/202212301017585.jpg)

**写入操作的延时**就等于latency = Latency(Primary Write) + Max(Replicas Write)。只要有副本在，写入延时最小也是两次单Shard的写入时延总和，写入效率会较低。

Elasticsearch是先写内存，最后才写TransLog，一种可能的原因是Lucene的内存写入会有很复杂的逻辑，很容易失败，比如分词，字段长度超过限制等，比较重，为了避免TransLog中有大量无效记录，减少recover的复杂度和提高速度，所以就把写Lucene放在了最前面。二是写Lucene内存后，并不是可被搜索的，需要通过Refresh把内存的对象转成完整的Segment后，然后再次reopen后才能被搜索，一般这个时间设置为1秒钟，导致写入Elasticsearch的文档，最快要1秒钟才可被从搜索到

### update

![img](https://cdn.jsdelivr.net/gh/631068264/img/202212301017586.jpg)



## merge

![image-20201004232327765](https://cdn.jsdelivr.net/gh/631068264/img/202212301017588.jpg)