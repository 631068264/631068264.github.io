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

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjd4rwc21kj31720iamyt.jpg)

![image-20201004111652659](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjd4sckohtj31560ksabk.jpg)

![image-20201004111812510](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjd4toowa8j31m00u0e5f.jpg)




# 主分片

![](https://tva1.sinaimg.cn/large/008i3skNgy1gr06work5qj31d20u044v.jpg)

# 副本分片

![](https://tva1.sinaimg.cn/large/008i3skNgy1gr072xm56ij31sn0u0qah.jpg)


# 分片数设定

![](https://tva1.sinaimg.cn/large/008i3skNgy1gr07590g3vj31mm0u0tgj.jpg)

分片过多
![](https://tva1.sinaimg.cn/large/008i3skNgy1gr077j7kywj31jf0u0n1c.jpg)
![](https://tva1.sinaimg.cn/large/008i3skNgy1gr0783s295j32370u0jwa.jpg)

![image-20210530093202876](https://tva1.sinaimg.cn/large/008i3skNgy1gr078kp9zbj320o0u0to1.jpg)

故障转移

![image-20210530093334113](https://tva1.sinaimg.cn/large/008i3skNgy1gr07a5z51nj31rk0u01kx.jpg)

![](https://tva1.sinaimg.cn/large/008i3skNgy1gr07b6pkmfj32010u0aei.jpg)



# 文档保存到分片

![image-20201004225303315](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjdownvlsmj31hf0u01bu.jpg)

所以设置好index后主分片数不能改，整个存储完全不一样

![](https://tva1.sinaimg.cn/large/008i3skNgy1gr07qu1zdjj31hk0u0n4w.jpg)
![](https://tva1.sinaimg.cn/large/008i3skNgy1gr07r8d5elj31j90u0gqk.jpg)
![](https://tva1.sinaimg.cn/large/008i3skNgy1gr07suyndvj31hf0u0te5.jpg)



# 分片设计
![](https://tva1.sinaimg.cn/large/008i3skNgy1gr09laa87pj31d70u0n1v.jpg)
![](https://tva1.sinaimg.cn/large/008i3skNgy1gr09ll161vj31jv0u077s.jpg)

![image-20201005215651480](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjeswi6atij31fo0u04d7.jpg)

![image-20201005215758649](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjesxoa6klj31lh0u0tta.jpg)

![image-20201005215824881](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjesy4axdvj31870u0qip.jpg)

![image-20201005215841945](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjesyf2jx7j31950u0tr6.jpg)

![image-20201005215956469](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjeszpvv22j31ts0u07wh.jpg)

# 分片机制 写入原理

[ES内部分片处理机制](https://my.oschina.net/LucasZhu/blog/1542850)

## 倒排索引不可变性

![image-20201004230145470](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjdp5q6y6bj31pg0u0nn6.jpg)

删除的文档不会立即清理

多个segment（倒排索引文件）=> lucene index => es shard

![image-20201006102814479](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjfemb5pphj31jr0u0b29.jpg)

## refresh

数据先入buffer buffer 到 segment过程叫refresh

refresh后才数据会被搜索到

![image-20201004230453959](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjdp906xsxj31r20u01kx.jpg)



## Transaction Log

防断电

![image-20201004231400579](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjdpih3khmj31k40u04qp.jpg)

## flush

![image-20201004232216254](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjdpr3557sj31pu0u01as.jpg)

## 总结
### 插入

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjx1ldd79kj30w00hyq3o.jpg)

**写入操作的延时**就等于latency = Latency(Primary Write) + Max(Replicas Write)。只要有副本在，写入延时最小也是两次单Shard的写入时延总和，写入效率会较低。

Elasticsearch是先写内存，最后才写TransLog，一种可能的原因是Lucene的内存写入会有很复杂的逻辑，很容易失败，比如分词，字段长度超过限制等，比较重，为了避免TransLog中有大量无效记录，减少recover的复杂度和提高速度，所以就把写Lucene放在了最前面。二是写Lucene内存后，并不是可被搜索的，需要通过Refresh把内存的对象转成完整的Segment后，然后再次reopen后才能被搜索，一般这个时间设置为1秒钟，导致写入Elasticsearch的文档，最快要1秒钟才可被从搜索到

### update

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjx1vgeal8j31400g7go8.jpg)



## merge

![image-20201004232327765](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjdpsbp64yj31ey0u0dpa.jpg)