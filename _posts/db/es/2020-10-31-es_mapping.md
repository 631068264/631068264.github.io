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

![image-20201004130722225](https://cdn.jsdelivr.net/gh/631068264/img/202212301023232.jpg)

![image-20201004130747669](https://cdn.jsdelivr.net/gh/631068264/img/202212301023234.jpg)

## Dynamic Mapping



![image-20201004130835841](https://cdn.jsdelivr.net/gh/631068264/img/202212301023235.jpg)

![image-20201004130857953](https://cdn.jsdelivr.net/gh/631068264/img/202212301023236.jpg)

![image-20201004131316898](https://cdn.jsdelivr.net/gh/631068264/img/202212301023237.jpg)



不能被索引，查询不了

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301023238.jpg)

## set mapping

![image-20201004132012061](https://cdn.jsdelivr.net/gh/631068264/img/202212301023239.jpg)

![image-20201004132130398](https://cdn.jsdelivr.net/gh/631068264/img/202212301023240.jpg)

![image-20201004132154528](https://cdn.jsdelivr.net/gh/631068264/img/202212301023241.jpg)

![image-20201004163851247](https://cdn.jsdelivr.net/gh/631068264/img/202212301023242.jpg)

# 倒排索引

![image-20201004113747882](https://cdn.jsdelivr.net/gh/631068264/img/202212301023243.jpg)

# reindex

![image-20201005170146242](https://cdn.jsdelivr.net/gh/631068264/img/202212301023244.jpg)

## 增加字段

修改mapping后查询不了

![image-20201005170942746](https://cdn.jsdelivr.net/gh/631068264/img/202212301023245.jpg)

![image-20201005171042287](https://cdn.jsdelivr.net/gh/631068264/img/202212301023246.jpg)

执行**_update_by_query** mapping更新前的数据可以搜索到，将原有索引重新索引

## 更改字段

![image-20201005171246045](https://cdn.jsdelivr.net/gh/631068264/img/202212301023247.jpg)

- 先创建一个新索引
- 将数据迁入到新索引

![image-20201005171839966](https://cdn.jsdelivr.net/gh/631068264/img/202212301023248.jpg)

![image-20201005172247881](https://cdn.jsdelivr.net/gh/631068264/img/202212301023249.jpg)

![image-20201005172317455](https://cdn.jsdelivr.net/gh/631068264/img/202212301023250.jpg)

新index存在数据

![image-20201005172350259](https://cdn.jsdelivr.net/gh/631068264/img/202212301023251.jpg)

![image-20201005172442768](https://cdn.jsdelivr.net/gh/631068264/img/202212301023252.jpg)

![image-20201005173108183](https://cdn.jsdelivr.net/gh/631068264/img/202212301023253.jpg)




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

![image-20201101113609497](https://cdn.jsdelivr.net/gh/631068264/img/202212301023254.jpg)

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