---
layout:     post
rewards: false
title:  spark 读取es
categories:
    - big data
tags:
    - big data
---

# 介绍

- [spark sql 操作example](https://blog.csdn.net/dabokele/article/details/52802150)
- [spark sql es](https://www.elastic.co/guide/en/elasticsearch/hadoop/current/spark.html#spark-sql)
- [lucene query 语法](https://lucene.apache.org/core/2_9_4/queryparsersyntax.html) es支持底层支持lucene可以做简单查询


尽量使用DataSet/DataFrame 不用Rdd 底层也是用rdd 但是做了优化
不用type es会弃用这个概念

- api灵活
- 高效序列化反序列化、压缩，减少GC
- 适合操作结构化数据可以通过名字或字段来处理或访问数据


# code

## 连接方式

```java
SparkSession spark = SparkSession.builder()
        .master("local[*]")
        .appName("demo")
        .getOrCreate();

Dataset<Row> df = spark.read().format("es")
        .option("es.nodes", properties.getProperty("es.nodes"))
        .option("es.port", properties.getProperty("es.port"))
        .load("index");
```

这种比较灵活

```java

SparkSession spark = SparkSession.builder()
        .master("local[*]")
        .config("es.nodes", properties.getProperty("es.nodes"))
        .config("es.port", properties.getProperty("es.port"))
        .appName("demo")
        .getOrCreate();

Dataset<Row> df = JavaEsSparkSQL.esDF(spark, "accounts");
```
  
## sql操作example
schema
```
root
 |-- account_number: long (nullable = true)
 |-- address: string (nullable = true)
 |-- age: long (nullable = true)
 |-- balance: long (nullable = true)
 |-- city: string (nullable = true)
 |-- email: string (nullable = true)
 |-- employer: string (nullable = true)
 |-- firstname: string (nullable = true)
 |-- gender: string (nullable = true)
 |-- lastname: string (nullable = true)
 |-- state: string (nullable = true)
```


```java
Dataset<Row> temp = df.filter("age == 32");
Dataset<Row> temp = df.filter(df.col("age").equalTo(32));

df.filter(df.col("city").equalTo("Sunnyside"));
df.filter("31<age and age <33");
df.groupBy("age").min("balance");
df.filter(df.col("balance").geq(39225));
```

可以打印结果

```
temp.show(1)
```

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdwz3vmn1xj31mc08wt95.jpg)