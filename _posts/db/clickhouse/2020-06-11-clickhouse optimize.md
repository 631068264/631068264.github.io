---
layout:     post
rewards: false
title:      clickhouse 优化
categories:
    - clickhouse
---

[ClickHouse Query Performance Tips and Tricks](https://www.altinity.com/presentations/2019/10/9/clickhouse-query-performance-tips-and-tricks)


# 建表

<span class='gp-2'>
    <img src='https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gfo535wuwcj31lk0u0gya.jpg' />
    <img src='https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gfo54ga7rjj31i30u07e7.jpg' />
</span>

# join 

![](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gfo59rz2yvj31hs0u0gy5.jpg)

![](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gfoel9scokj31kw0u0k6t.jpg)

# select 优化

![](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gfoeuq87l7j31o90u0drb.jpg)

![](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gfof4klklbj31z10u0alj.jpg)

![](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gfofcfp74qj31h40u07ic.jpg)

![](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gfoff16qh3j31x10u04cc.jpg)

![](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gfofm2osutj31g90u0dx2.jpg)

[lowcardinality](https://clickhouse.tech/docs/zh/sql-reference/data-types/lowcardinality/)

lowcardinality 是一种改变数据存储和数据处理方法的概念。 clickhouse会把 lowcardinality 所在的列进行dictionary coding。对很多应用来说，处理字典编码的数据可以显著的增加select查询速度。

使用 lowcarditality 数据类型的效率依赖于数据的多样性。如果一个字典包含少于10000个不同的值，那么clickhouse可以进行更高效的数据存储和处理。反之如果字典多于10000，效率会表现的更差。

当使用字符类型的时候，可以考虑使用 lowcardinality 代替enum。 lowcardinality 通常更加灵活和高效。


![](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gfofx5l97oj31lu0u0ap5.jpg)

![](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gfofydqrtzj31mm0u0gxj.jpg)

![](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gfog8k3xd6j31k10u0k3a.jpg)