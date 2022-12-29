---
layout:     post
rewards: false
title:  es机制内部原理
categories:
    - es
tags:
    - big data
---




# 搜索机制

两步 query + fetch  先从各分片拿到到doc id 再拿doc

## query

![image-20201005094515513](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gje7ra3j6bj31l70u01kx.jpg)

## fetch

![image-20201005094644093](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gje7stcrshj31lb0u0hbo.jpg)

## query then fetch 问题

![image-20201005095030221](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gje7wquvmrj31kx0u04ox.jpg)

![image-20201005095114548](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gje7xi7vyaj31ql0u0qt9.jpg)



# 聚合机制

![image-20201005133721163](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gjeegstygdj31hi0u04fl.jpg)

聚合不精准问题，数据分布在不同分片

![image-20201005134140153](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gjeeladw6xj31s50u04qp.jpg)

![image-20201005134218038](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gjeely4u7jj31hb0u0wv6.jpg)

![image-20201005134357342](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gjeennyq1cj31h80u0ni3.jpg)

![image-20201005134542926](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gjeepi1cy7j31iu0u04qp.jpg)

![image-20201005134607257](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gjeepwtwexj31nm0t2gxy.jpg)




# 并发处理机制

![image-20201005103851081](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gje9b23roaj31lp0u01kx.jpg)

通过**if_seq_no** 和 **if_primary_term** 控制

![image-20201005104042711](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gje9czev6vj31wd0u0kiv.jpg)

![image-20201005104155281](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gje9e8i3f8j31yd0u07wh.jpg)
