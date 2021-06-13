---

layout:     post
rewards: false
title:  elasticsearch
categories:
    - es
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

# 倒排结构

![](https://tva1.sinaimg.cn/large/008i3skNgy1grfap6dxa5j31hc0tpq4o.jpg)

![](https://tva1.sinaimg.cn/large/008i3skNgy1grfawa0zrsj31q60u0n1l.jpg)

尽量少的读磁盘，有必要把一些数据缓存到内存里。但是整个 **term dictionary** 本身又太大了，无法完整地放到内存里。于是就有了 **term index**

>**term index** 有点像一本字典的大的章节表。比如：
>
>A 开头的 term ……………. Xxx 页
>
>C 开头的 term ……………. Xxx 页
>
>E 开头的 term ……………. Xxx 页



**Trem index** 

- term index 是一棵 **trie 树**（`字典树`）+ **内存存储**

- 它不存储所有的单词，**只存储单词前缀**

- 进一步节省内存，Lucene 还用了 FST（Finite State Transducers）对 Term Index 做进一步压缩
- 为 `O(m)`，其中 `m` 为关键字的字符数量，定位**Term dictionary**对应 block 的 offset

**Term dictionary**

- 分 block 存储，同一个 block 上，词项共享(**同一个前缀,都是 Ab 开头的单词就可以把 Ab 省去),节约磁盘空间**
- 其为**有序**的字典，正常情况，**二分查找**，查询效率 **O(logN)**
- 定位到最终的 term

**Posting list** 文档id

- 分块差值编码，再对每块进行 bit 压缩存储

  ```
  [73, 300, 302, 332, 343, 372]
  ```

  增量编码

  ```
  [73, 227, 2, 30, 11, 29] # 数据只记录元素与元素之间的增量
  ```

  用了 FOR(Frame Of Reference) 编码进行压缩

  ![](https://tva1.sinaimg.cn/large/008i3skNgy1grfc8eqp3ij30fp0cmdg6.jpg)

  **快速求交集**

  Roaring Bitmaps

  Lucene Posting List 的每个 Segement 最多放 65536 个文档ID

  **1个id**

  - 使用 bitmap 表示 需要 65536 个 bit，65536/8 = 8192 bytes

  - Integer 数组，只需要 2  = 2 bytes

   文档数量不多的时候，使用 Integer 数组更加节省内存

   当文档数量少于 8192/2=4096 时，用 Integer 数组，否则，用 bitmap。

  ![image-20210612122424083](https://tva1.sinaimg.cn/large/008i3skNgy1grfd9wolyej316g0sidkv.jpg)

  ![image-20210612122327223](https://tva1.sinaimg.cn/large/008i3skNgy1grfd8yq4kfj30zw0u0gr9.jpg)



- **Frame Of Reference** 是压缩数据，减少磁盘占用空间，所以当我们从磁盘取数据时，也需要一个反向的过程，**即解压**，解压后才有我们文档ID数组，对数据进行处理，**求交集或者并集**，这时候数据是需要放到**内存**进行处理的，更强有力的压缩算法，同时还要有利于快速的求交并集，于是有了**Roaring Bitmaps 算法**。

![](https://tva1.sinaimg.cn/large/008i3skNgy1grfd01nw5aj31vm0pajxk.jpg)

> Mysql 是以 b-tree 排序的方式存储在磁盘上的。检索一个 term 需要**若干次随机 IO** 的磁盘操作。而 Lucene 在 term dictionary 的基础上添加了term index来加速检索，term index 以树的形式缓存在内存中。从 term index 查到对应的 term dictionary 的 block 位置之后，再去磁盘上找 term，大大减少了**磁盘的随机IO次数**。



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
```json
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





```json
input {
    stdin {
    }
    jdbc {
      # mysql相关jdbc配置
      jdbc_connection_string => "jdbc:mysql://127.0.0.1:3306/loh"
      jdbc_user => "root"
      jdbc_password => "wuyuxi08"

      # jdbc连接mysql驱动的文件目录，可去官网下载:https://dev.mysql.com/downloads/connector/j/
      jdbc_driver_library => "/Users/wyx/project/docker-elk/logstash/mysql-connector-java-8.0.21.jar"
      # the name of the driver class for mysql
      jdbc_driver_class => "com.mysql.jdbc.Driver"
      jdbc_paging_enabled => "true"
      #jdbc_page_size => "50000"
      use_column_value => true

      # mysql文件, 也可以直接写SQL语句在此处，如下：
      # statement => "SELECT * from Table_test;"
      # statement_filepath => "C:/setup/logstash-7.0.1/config/myconfig/jdbc.sql"
      # statement => "SELECT * FROM table WHERE id >= :sql_last_value"
      statement => "SELECT * FROM suricata_rule;"

      # 这里类似crontab,可以定制定时操作，比如每10分钟执行一次同步(分 时 天 月 年)
      # schedule => "*/5 * * * *"
      # type => "jdbc"

      # 是否记录上次执行结果, 如果为真,将会把上次执行到的 tracking_column 字段的值记录下来,保存到 last_run_metadata_path 指定的文件中
      record_last_run => "true"

      # 是否需要记录某个column 的值,如果record_last_run为真,可以自定义我们需要 track 的 column 名称，此时该参数就要为 true. 否则默认 track 的是 timestamp 的值.
      use_column_value => "true"

      # 如果 use_column_value 为真,需配置此参数. track 的数据库 column 名,该 column 必须是递增的. 一般是mysql主键
      tracking_column => "id"

      last_run_metadata_path => "/Users/wyx/project/docker-elk/logstash/mysqlconf/last_id"

      # 是否清除 last_run_metadata_path 的记录,如果为真那么每次都相当于从头开始查询所有的数据库记录
      clean_run => "false"

      # 是否将 字段(column) 名称转小写
      lowercase_column_names => "false"

      columns_charset => {
        "message"=> "UTF-8"
        "name"=> "UTF-8"
      }
    }
}

# 此处我不做过滤处理,如果需要，也可参考elk安装那篇
filter {}

output {
    # 输出到elasticsearch的配置
    # 注意这里对type判断，若加载多个配置文件，要有这个判断才不会互相影响
    
     elasticsearch {
      hosts => ["127.0.0.1:9200"]
      index => "suricata_rule"

          # 将"_id"的值设为mysql的autoid字段
          # 注意这里的id，如果多个表输出到同一个index，它们的id有重复的，则这里的 document_id 要修改成不重复的，否则会覆盖数据
       document_id => "%{id}"
       template_overwrite => true
    }
    
    # 这里输出调试，正式运行时可以注释掉
    stdout {
        codec => json_lines
    }
}
```





# search



```json
{
  "query": {
    "multi_match" : {
      "query":    "guide", 
      "fields": [ "title", "summary" ] 
    }
  }
}
```

title or  summary  contains kw