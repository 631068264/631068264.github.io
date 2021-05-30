---
layout:     post
rewards: false
title:  es 数据类型
categories:
    - es
tags:
    - big data

---

# mapping

![image-20201004130722225](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjd7z9lo99j316c0u07t8.jpg)

![image-20201004130747669](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjd7zpj3adj31880u04bu.jpg)

## Dynamic Mapping



![image-20201004130835841](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjd80jhydjj31ks0u07wh.jpg)

![image-20201004130857953](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjd80xft3dj31g30u0aux.jpg)

![image-20201004131316898](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjd85ey6aej31bo0u01kx.jpg)



不能被索引，查询不了

![](https://tva1.sinaimg.cn/large/008i3skNgy1gqso3n5la1j31cl0u0tbe.jpg)

## set mapping

![image-20201004132012061](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjd8cmdxb7j31i30u01kx.jpg)

![image-20201004132130398](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjd8dz8g51j31ww0u01kx.jpg)

![image-20201004132154528](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjd8ee5y0hj31pg0u0tvm.jpg)

![image-20201004163851247](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjde3b71i0j31k20u0qtb.jpg)

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




# 数据类型

- es整形支持到**long int64**，之前有些字段都是**uint64**的字段。

- string 类型字段查询结果有些值查询不出来，甚至保存,字符串将默认被同时映射成**text**和**keyword**类型

```json
{
    "foo": {
        "type": "text",

        "fields": {
            "keyword": {
                "type": "keyword",
                "ignore_above": 256

            }

        }
    }
}
```

Text：会分词，然后进行索引 支持**模糊**、精确查询     不支持聚合  （match*）

keyword：不进行分词，直接索引 支持模糊、**精确查询** **支持聚合**   (term agg)

# ignore_above

[ingore_above](https://www.elastic.co/guide/en/elasticsearch/reference/current/ignore-above.html)  [keyword](https://www.elastic.co/guide/en/elasticsearch/reference/current/keyword.html)

字符串长度**>**ignore_above的值，不会被索引（通过这个值查不到，聚合不了），但是可以保存。

对于长字符串，可以使用字符串数组（方便分词，ignore_above的值所用到数组的每个元素，而不是数组长度）

dynamic mapping 默认ignore_above: 256

keyword类型的最大支持的长度为32766个UTF-8类型的字符



# doc_values and fielddata

![image-20201101113609497](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9iozk4gqj31240gc0y4.jpg)

## search 、 sort and agg

搜索 倒排索引  

doc_values  正向索引 对不分词的字段默认开启，包括多域字段，对分词字段默认关闭。根据_source列式存储数据使得sort ,agg更高效，占据存储空间。**重新设置需要重建index**

fielddata 对分词的字段进行sort and agg。 即开即用，占内存。

# 搜索ip类型

```json
{
    "query":{
        "query_string":{
            "query":"clientIpAddress:[192.168.1.100 TO 192.168.1.102]"
        }
    }
}
```

```json
{
    "query":{
        "range":{
            "add":{
                "gte":"192.168.1.100",
                "lte":"192.168.1.102"
            }
        }
    }
}
```