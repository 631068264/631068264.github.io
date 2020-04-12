---
layout:     post
rewards: false
title:  elasticsearch
categories:
    - big data
tags:
    - big data
---

# 设置外网访问

```
[1]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
[2]: the default discovery settings are unsuitable for production use; at least one of [discovery.seed_hosts, discovery.seed_providers, cluster.initial_master_nodes] must be configured
```

- [安装到Ubuntu](https://linuxize.com/post/how-to-install-elasticsearch-on-ubuntu-18-04/)
- [外网访问](https://blog.csdn.net/wd2014610/article/details/89532638)


# 倒排索引

文档内容被表示为一系列关键词的集合

索引  文档id  关键词(记录它在文档中的出现次数和出现位置) 描述他们之间的索引关系

## 正向索引

通过关键词寻找 遍历文档 匹配关键词  打分 排序
![](https://tva1.sinaimg.cn/large/00831rSTgy1gdcy5k4ls3j312y0j242g.jpg)


## 反向索引


![](https://tva1.sinaimg.cn/large/00831rSTgy1gdcy6c8agyj311m0istb9.jpg)



# 概念



Elasticsearch使用称为倒排索引的数据结构，该结构支持非常快速的全文本搜索。默认情况下，Elasticsearch对每个字段中的所有数据建立索引

Elastic 数据管理的顶层单位就叫做 **Index**（索引）。它是单个数据库的同义词。每个 **Index** （即数据库）的名字必须是小写
**Index** 里面单条的记录称为 **Document**（文档）。许多条 **Document** 构成了一个 **Index**。
同一个 Index 里面的 Document，不要求有相同的结构（scheme），但是最好保持相同，这样有利于提高搜索效率。


Document 可以分组 **Type**，它是虚拟的逻辑分组，用来过滤 **Document**


# 请求 /Index/Type

- POST _id字段就是一个随机字符串  可以不指定id 新增
- PUT 指定_id  新增 更新
- DELETE 指定_id
- GET 指定_id


## 批量

**批量**批处理1,000至5,000个文档，总有效负载在5MB至15MB之间

`/Index/Type/_bulk`


格式
```
action_and_meta_data \ n   操作
optional_source \ n        数据体
action_and_meta_data \ n
optional_source \ n
```

Example
```
{"index":{"_id":"971"}}
{"account_number":971,"balance":22772,"firstname":"Gabrielle","lastname":"Reilly","age":32,"gender":"F","address":"964 Tudor Terrace","employer":"Blanet","email":"gabriellereilly@blanet.com","city":"Falmouth","state":"AL"}
```

- index 和 create  第二行是source数据体
- delete 没有第二行
- update 第二行可以是partial doc，upsert或者是script

修改操作

- index 添加新数据，同时替换（重新索引）现有数据（基于其ID）
- create 添加新数据-如果数据已经存在（基于其ID），则会引发异常
- update 更新现有数据（基于其ID）。如果找不到数据，则引发异常
- upsert 如果数据不存在，则 称为合并或插入；如果数据存在（根据其ID），则更新


```
{ "index" : { "_index" : "test", "_type" : "_doc", "_id" : "1" } }
{ "field1" : "value1" }
{ "delete" : { "_index" : "test", "_type" : "_doc", "_id" : "2" } }
{ "create" : { "_index" : "test", "_type" : "_doc", "_id" : "3" } }
{ "field1" : "value3" }
{ "update" : {"_id" : "1", "_type" : "_doc", "_index" : "test"} }
{ "doc" : {"field2" : "value2"} }
```



# query

- `/Index/Type/_search` 返回所有记录
    - took 操作的耗时（单位为毫秒）
        - Communication time between the coordinating node and data nodes
        - Time the request spends in the search thread pool, queued for execution
        - Actual execution time
    - timed_out字段表示是否超时
    - hits字段表示命中的记录
    - max_score：最高的匹配程度，本例是1.0。

## body search

```json
{
  "query" : { "match" : { "desc" : "管理" }},
  "size": 1
}
```

offset 0 limit x

```json
{
  "query" : { "match" : { "desc" : "管理" }},
  "from": 1,
  "size": 1
}
```

**or关系**

```json
{
  "query" : { "match" : { "desc" : "软件 系统" }}
}
```

**and关系**
```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "desc": "软件" } },
        { "match": { "desc": "系统" } }
      ]
    }
  }
}
```


**title**= and **content**= and **status** contains the exact word and **publish_date** field contains a date from 1 Jan 2015

```
{
  "query": { 
    "bool": { 
      "must": [
        { "match": { "title":   "Search"        }},
        { "match": { "content": "Elasticsearch" }}
      ],
      "filter": [ 
        { "term":  { "status": "published" }},
        { "range": { "publish_date": { "gte": "2015-01-01" }}}
      ]
    }
  }
}
```

### bool

- [bool](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html)

## q Lucene query string syntax

- [Apache Lucene](https://lucene.apache.org/core/2_9_4/queryparsersyntax.html)

```
q=destip:10.125.2.254 and destport:53

te?t

test*


mod_date:[20020101 TO 20030101]
title:{Aida TO Carmen}
包含范围的查询由方括号表示。排他范围查询由大括号表示


"jakarta apache" jakarta => "jakarta apache" OR jakarta
```

```
search for documents that contain "jakarta apache" and "Apache Lucene"
"jakarta apache" [AND,&&] "Apache Lucene"
```

```
search for documents that must contain "jakarta" and may contain "lucene"
+jakarta lucene

To search for a title that contains both the word "return" and the phrase "pink panther" use the query
title:(+return +"pink panther")
```

```
search for documents that contain "jakarta apache" but not "Apache Lucene"
"jakarta apache" [NOT,-] "Apache Lucene"
```

```
(jakarta OR apache) AND website
```

# support

- [spark support](https://www.elastic.co/guide/en/elasticsearch/hadoop/current/spark.html#spark)

