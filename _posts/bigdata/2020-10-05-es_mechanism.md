---
layout:     post
rewards: false
title:  es机制内部原理
categories:
    - big data
tags:
    - big data
---

# 集群

![image-20201004222432325](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjdo2zvavhj31tu0sgh57.jpg)

## 节点

![image-20201004222719097](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjdo5vp8szj317u0qkqm2.jpg)

![image-20201004222753980](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjdo6hweekj31lo0u07od.jpg)

![image-20201004222816774](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjdo6wdm5wj31ti0u0x0s.jpg)

## 集群故障

![image-20201004224827053](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjdorvly2mj31ku0u0qk7.jpg)

![image-20201004224921390](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjdostkb43j31i50u0ker.jpg)

![image-20201004225031868](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjdou28aswj31h50u04qp.jpg)





## 跨集群搜索

单master会成为性能瓶颈，集群meta信息（节点、索引、集群状态）过多，更新压力大

![image-20201004223220710](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjdob4z8pzj31mb0u01kx.jpg)

![image-20201004223248847](/Users/wyx/Library/Application Support/typora-user-images/image-20201004223248847.png)









# 分片

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjd4rwc21kj31720iamyt.jpg)

![image-20201004111652659](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjd4sckohtj31560ksabk.jpg)

![image-20201004111812510](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjd4toowa8j31m00u0e5f.jpg)



## 文档保存到分片

![image-20201004225303315](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjdownvlsmj31hf0u01bu.jpg)

所以设置好index后主分片数不能改，整个存储完全不一样

![image-20201004225405129](/Users/wyx/Library/Application Support/typora-user-images/image-20201004225405129.png)



# 倒排索引

![image-20201004113747882](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjd5e2eeclj31im0u0b29.jpg)

# reindex

![image-20201005170146242](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjekdh7zaoj314w0u0due.jpg)

## 增加字段

修改mapping后查询不了

![image-20201005170942746](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjeklqxat9j31p30u0h94.jpg)

![image-20201005171042287](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjekmrk8fnj31rq0iy7ms.jpg)

执行**_update_by_query** mapping更新前的数据可以搜索到，将原有索引重新索引

## 更改字段

![image-20201005171246045](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjekowutzgj31h50u01kx.jpg)

- 先创建一个新索引
- 将数据迁入到新索引

![image-20201005171839966](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjekv1ojakj31di0d20xq.jpg)

![image-20201005172247881](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjekzcppoqj31z40sgwxc.jpg)

![image-20201005172317455](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjekzviknej31pc0swdss.jpg)

新index存在数据

![image-20201005172350259](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjel0fqex7j31qw0u0nc6.jpg)

![image-20201005172442768](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjel1cgnz5j31oo0u0wyw.jpg)

![image-20201005173108183](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjel80wpqyj31p50u0e65.jpg)



# 分片机制

[ES内部分片处理机制](https://my.oschina.net/LucasZhu/blog/1542850)

## 倒排索引不可变性

![image-20201004230145470](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjdp5q6y6bj31pg0u0nn6.jpg)

删除的文档不会立即清理

![image-20201006102814479](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjfemb5pphj31jr0u0b29.jpg)

## refresh

refresh后才数据会被搜索到

![image-20201004230453959](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjdp906xsxj31r20u01kx.jpg)



## Transaction Log

防断电

![image-20201004231400579](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjdpih3khmj31k40u04qp.jpg)

## flush

![image-20201004232216254](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjdpr3557sj31pu0u01as.jpg)



## merge

![image-20201004232327765](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjdpsbp64yj31ey0u0dpa.jpg)



# 搜索机制

两步 query + fetch  先从各分片拿到到doc id 再拿doc

## query

![image-20201005094515513](https://tva1.sinaimg.cn/large/007S8ZIlgy1gje7ra3j6bj31l70u01kx.jpg)

## fetch

![image-20201005094644093](https://tva1.sinaimg.cn/large/007S8ZIlgy1gje7stcrshj31lb0u0hbo.jpg)

## query then fetch 问题

![image-20201005095030221](https://tva1.sinaimg.cn/large/007S8ZIlgy1gje7wquvmrj31kx0u04ox.jpg)

![image-20201005095114548](https://tva1.sinaimg.cn/large/007S8ZIlgy1gje7xi7vyaj31ql0u0qt9.jpg)



# 聚合机制

![image-20201005133721163](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjeegstygdj31hi0u04fl.jpg)

聚合不精准问题，数据分布在不同分片

![image-20201005134140153](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjeeladw6xj31s50u04qp.jpg)

![image-20201005134218038](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjeely4u7jj31hb0u0wv6.jpg)

![image-20201005134357342](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjeennyq1cj31h80u0ni3.jpg)

![image-20201005134542926](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjeepi1cy7j31iu0u04qp.jpg)

![image-20201005134607257](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjeepwtwexj31nm0t2gxy.jpg)









# 并发处理机制

![image-20201005103851081](https://tva1.sinaimg.cn/large/007S8ZIlgy1gje9b23roaj31lp0u01kx.jpg)

通过**if_seq_no** 和 **if_primary_term** 控制

![image-20201005104042711](https://tva1.sinaimg.cn/large/007S8ZIlgy1gje9czev6vj31wd0u0kiv.jpg)

![image-20201005104155281](https://tva1.sinaimg.cn/large/007S8ZIlgy1gje9e8i3f8j31yd0u07wh.jpg)

# 关联关系处理

![image-20201005161420618](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjej04f24hj31me0u04qp.jpg)

![image-20201005161436967](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjej0e9tghj31ib0u0qlz.jpg)

![image-20201005161324631](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjeiz77lwdj31jz0u0axt.jpg)

## example

![image-20201005161825725](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjej4d36s5j321c0sinkb.jpg)

![image-20201005161837806](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjej4l1c7zj31ke0f6qci.jpg)

![image-20201005161901941](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjej4zvw9rj31va0twhb5.jpg)

使用错误条件查询会出现不该出现的结果

![image-20201005161924263](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjej5e35z0j31uu0kyh3j.jpg)

![image-20201005162427557](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjejan7p35j31se0swwx6.jpg)

## nested type

![image-20201005162536762](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjejbusfwfj31yv0u0b22.jpg)

![image-20201005162716385](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjejdl03ndj31p00u01kx.jpg)

![image-20201005163022490](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjejgt096fj31v00s8x43.jpg)

## 父子关系

![image-20201005164120043](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjejs7bx2mj31pz0u0trf.jpg)

![image-20201005164539498](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjejwpcy7jj31ki0u0axu.jpg)

![image-20201005164638010](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjejxq6uuij31hc0jek5g.jpg)

![image-20201005164852064](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjek01nizhj31ls0u07wh.jpg)

![image-20201005165636129](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjek82zjdsj31rq0ich6n.jpg)

![image-20201005165558663](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjek7g6rt1j31qu0gw1bt.jpg)

![image-20201005165756902](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjek9i32bjj31xo0pm7vq.jpg)

![image-20201005165935170](/Users/wyx/Library/Application Support/typora-user-images/image-20201005165935170.png)

![image-20201005170003349](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjekbor5l6j31mk0u0h6k.jpg)

# ingest node

数据预处理

![image-20201005173249367](/Users/wyx/Library/Application Support/typora-user-images/image-20201005173249367.png)

![image-20201005173330305](/Users/wyx/Library/Application Support/typora-user-images/image-20201005173330305.png)

![image-20201005173346663](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjelasb8ycj31620u0h7j.jpg)

![image-20201005173956016](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjelh6mb6vj31mo0mgniz.jpg)

![image-20201005174050976](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjeli4s1vrj31o20jqwvw.jpg)

![image-20201005174110740](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjelihe91fj316k0k8ton.jpg)

![image-20201005174807250](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjelpowcorj31v00kiayj.jpg)

![image-20201005174953420](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjelrjjss7j31q30u0nlt.jpg)

# painless script

![image-20201005175104080](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjelsrt5cdj31cj0u0any.jpg)

![image-20201005175738153](/Users/wyx/Library/Application Support/typora-user-images/image-20201005175738153.png)

![image-20201005175846876](/Users/wyx/Library/Application Support/typora-user-images/image-20201005175846876.png)

![image-20201005175933842](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjem1lwgzfj318w0kkgxe.jpg)

![image-20201005180015611](/Users/wyx/Library/Application Support/typora-user-images/image-20201005180015611.png)

![image-20201005180034132](/Users/wyx/Library/Application Support/typora-user-images/image-20201005180034132.png)

# 数据建模

![image-20201005180212497](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjem4fn8qpj31p80u0e0v.jpg)

![image-20201005180237667](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjem4syymgj31gu0u0000.jpg)

![image-20201005180254469](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjem54a47uj323m0sc7ed.jpg)

![image-20201005180346414](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjem5zc7bmj31ji0u0e7b.jpg)

![image-20201005180416110](/Users/wyx/Library/Application Support/typora-user-images/image-20201005180416110.png)

![image-20201005180449097](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjem72tr1gj315o0u0qeq.jpg)

![image-20201005180507181](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjem7e2ewej313u0u0ao7.jpg)

![image-20201005180935176](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjemc0wh5fj31r40u0np8.jpg)

![image-20201005181348224](/Users/wyx/Library/Application Support/typora-user-images/image-20201005181348224.png)

![image-20201005181456415](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjemhlobmsj31xw0u0qrw.jpg)

![image-20201005193450892](/Users/wyx/Library/Application Support/typora-user-images/image-20201005193450892.png)

![image-20201005193530594](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjeotfwl41j31ou0u04p8.jpg)

![image-20201005193604549](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjeou19zxoj318g0u0ao5.jpg)

![image-20201005193706900](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjeov4ecp2j31js0u07wh.jpg)

![image-20201005193927998](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjeoxjqap3j319g0u0h2z.jpg)

![image-20201005193945751](/Users/wyx/Library/Application Support/typora-user-images/image-20201005193945751.png)

![image-20201005194110223](/Users/wyx/Library/Application Support/typora-user-images/image-20201005194110223.png)

![image-20201005194230930](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjep0pz2m9j31nk0u07wh.jpg)