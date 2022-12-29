---
layout:     post
rewards: false
title:      Bigdata Element
categories:
    - big data
tags:
    - big data
---


# Pig
[pig online doc](https://pig.apache.org/docs/latest/index.html)
- 高级数据流语言 Pig Latin
- 运行Pig Latin程序的执行环境

Pig能够让你专心于数据及业务本身，而不是纠结于数据的格式转换以及MapReduce程序的编写。
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027419.jpg)

## Execution Modes

### local
only requires a single machine. Pig will run on the local host and access the local filesystem.
`pig -x local ...`

### MapReduce
`pig ...`

## Interactive Mode
Pig can be run interactively in the Grunt shell. 
```
pig -x local
...
grunt>
```

## Batch Mode
use pig script
`pig -x local id.pig`

## pig 语法
Each statement is an operator that takes a relation as an input, performs a transformation on that relation,
and produces a relation as an out‐ put. Statements can span multiple lines,`;`结尾。

- A LOAD statement that reads the data from the filesystem
- One or more statements to transform the data
- A DUMP or STORE statement to view or store the results

### load
USING default keyword `\t` 
AS default not named and type bytearray
```pig
LOAD 'data' [USING function] [AS schema];
```

```pig
A = LOAD 'students' AS (name:chararray, age:int);
DUMP A; 

(john,21,3.89) 
(sally,19,2.56) 
(alice,22,3.76) 
(doug,19,1.98) 
(susan,26,3.25)
```

### Transforming Data
条件关系 and or not

#### FILTER
处理列
```
A = LOAD 'students' AS (name:chararray, age:int, gpa:float);

DUMP A; 
(john,21,3.89)
(sally,19,2.56) 
(alice,22,3.76)
(doug,19,1.98)
(susan,26,3.25)

R = FILTER A BY age>=20;
DUMP R; 

(john,21,3.89) 
(alice,22,3.76) 
(susan,26,3.25)
```
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027420.jpg)

#### FOREACH
处理行 有点像select

```
R = FOREACH A GENERATE *;
```
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027421.jpg)

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027422.jpg)

#### GROUP
```
B = GROUP A BY age;
```
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027423.jpg)

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027424.jpg)

#### STORE

```
STORE alias INTO 'directory' [USING function];
```

```
A = LOAD 'students' AS (name:chararray, age:int, gpa:float);
STORE A INTO 'output' USING PigStore('|');
CAT output;
```
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027425.jpg)


## UDF

[pig_util](https://github.com/apache/pig/tree/trunk/src/python/streaming)
```python
from pig.pig_util import outputSchema


@outputSchema('word:chararray')
def reverse(word):
    """
    Return the reverse text of the provided word """
    return word[::-1]


@outputSchema('length:int')
def num_chars(word):
    """
    Return the length of the provided word """
    return len(word)
```
```
REGISTER 'my_udf.py' USING streaming_python AS string_udf;
term_length = FOREACH unique_terms GENERATE word, string_udf.num_chars(word) as length;
```

# Hive

[Hive](/blog/2019/03/03/hive)

# HBase


## 数据模型概念
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027426.jpg)

![img](https://cdn.jsdelivr.net/gh/631068264/img/202212301039683.jpg)

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027427.jpg)

### 表

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027428.jpg)

### 行

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027429.jpg)

### 列族

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027430.jpg)

### 限定字符

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027431.jpg)

### 单元格

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027432.jpg)

### 时间戳

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027433.jpg)

## 列族存储

<span class='gp-2'>
    <img src='https://cdn.jsdelivr.net/gh/631068264/img/202212301027460.jpg' />
    <img src='https://cdn.jsdelivr.net/gh/631068264/img/202212301027461.jpg' />
</span>

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027434.jpg)

- 降低IO
- 大并发查询
- 高数据压缩比

## 架构
与Hadoop访问过程，结构有点像
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027435.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027436.jpg)

###  zookeeper
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027437.jpg)

### master
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027438.jpg)

### region
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027440.jpg)

#### 表 Region

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027441.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027442.jpg)

#### Region 定位

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027443.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027444.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027445.jpg)

#### 结构
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027446.jpg)

**MemStore**容量有限，周期性写入到**StoreFile**,HLog写入一个标记。每次缓存刷新生成新的**StoreFile**，
当**StoreFile**数量到达某个阈值，会合并一个大**StoreFile**。当大**StoreFile**大小到达某个阈值，会分裂。
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027447.jpg)


![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027448.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027449.jpg)



#### 读写
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027450.jpg)


## 局限
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027451.jpg)

# NoSQL
不需要事务，读写实时性，没有复杂SQL查询。

## 种类

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027452.jpg)

# Spark

[Spark](/blog/2019/03/01/spark)

# 流计算

- 静态数据 批量计算 时间充足批量处理海量数据
- 流数据 实时计算
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027453.jpg)

## 流数据特征
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027454.jpg)

## 实时采集
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027455.jpg)

## 实时计算
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027456.jpg)

## 实时查询
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027457.jpg)

## Storm

[Storm](/blog/2019/10/20/storm)

## Spark Streaming
将stream拆分成小量批处理, 做不到**毫秒级别**，**storm** 可以
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027458.jpg)

### 对比storm
做不到**毫秒级别**，**storm** 可以
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301027459.jpg)

