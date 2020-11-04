---
layout:     post
rewards: false
title:  es stat
categories:
    - es
tags:
    - big data

---

## 数据类型

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

