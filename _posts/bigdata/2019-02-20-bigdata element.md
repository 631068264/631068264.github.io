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
![](https://ws4.sinaimg.cn/large/006tKfTcgy1g0jz6f0kp0j31gw0bk7fh.jpg)

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
![](https://ws3.sinaimg.cn/large/006tKfTcgy1g0egcywet8j30zu09m3yr.jpg)

#### FOREACH
处理行 有点像select

```
R = FOREACH A GENERATE *;
```
![](https://ws2.sinaimg.cn/large/006tKfTcgy1g0eghch92lj30ek0bidg5.jpg)

![](https://ws1.sinaimg.cn/large/006tKfTcgy1g0egi7gevyj31hw0imaav.jpg)

#### GROUP
```
B = GROUP A BY age;
```
![](https://ws4.sinaimg.cn/large/006tKfTcgy1g0egkxb7cpj31h60jswfo.jpg)

![](https://ws2.sinaimg.cn/large/006tKfTcgy1g0ehjnt6a6j31hg0jujsk.jpg)

#### STORE

```
STORE alias INTO 'directory' [USING function];
```

```
A = LOAD 'students' AS (name:chararray, age:int, gpa:float);
STORE A INTO 'output' USING PigStore('|');
CAT output;
```
![](https://ws2.sinaimg.cn/large/006tKfTcgy1g0ehsqlwbwj318b0u0wg2.jpg)


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
![](https://ws1.sinaimg.cn/large/006tKfTcgy1g0j0kby1r1j31sw0u07wh.jpg)
![](https://ws4.sinaimg.cn/large/006tKfTcgy1g0iufa0m5hj30ls0euq7x.jpg)
![](https://ws2.sinaimg.cn/large/006tKfTcgy1g0j0ofjbh9j31u80gyqfm.jpg)

### 表

![](https://ws3.sinaimg.cn/large/006tKfTcgy1g0j036smpmj31ai04g771.jpg)

### 行

![](https://ws2.sinaimg.cn/large/006tKfTcgy1g0j063g65gj31w40dutlm.jpg)

### 列族

![](https://ws1.sinaimg.cn/large/006tKfTcgy1g0j08zji2tj31e00u04qp.jpg)

### 限定字符

![](https://ws3.sinaimg.cn/large/006tKfTcgy1g0j0cbtd0uj31ti04ydk6.jpg)

### 单元格

![](https://ws1.sinaimg.cn/large/006tKfTcgy1g0j0davtrdj31ss07cdme.jpg)

### 时间戳

![](https://ws3.sinaimg.cn/large/006tKfTcgy1g0j0emqp1jj31tc0c2tlj.jpg)

## 列族存储

<span class='gp-2'>
    <img src='https://ws2.sinaimg.cn/large/006tKfTcgy1g0j0vyl9z4j31tc0o27hu.jpg' />
    <img src='https://ws4.sinaimg.cn/large/006tKfTcgy1g0j0xh0892j31460u0b29.jpg' />
</span>

![](https://ws1.sinaimg.cn/large/006tKfTcgy1g0j10j0m8lj31ti0gitpj.jpg)

- 降低IO
- 大并发查询
- 高数据压缩比

## 架构
与Hadoop访问过程，结构有点像
![](https://ws2.sinaimg.cn/large/006tKfTcgy1g0j1b6nlxdj31v40t84qp.jpg)
![](https://ws1.sinaimg.cn/large/006tKfTcgy1g0j27rn5jmj316g0owwrb.jpg)

###  zookeeper
![](https://ws1.sinaimg.cn/large/006tKfTcgy1g0j2e19djxj31dl0u0nof.jpg)

### master
![](https://ws3.sinaimg.cn/large/006tKfTcgy1g0j2k466laj31g00p0dzp.jpg)

### region
![](https://ws3.sinaimg.cn/large/006tKfTcgy1g0j2l8eelij31ie0dm14t.jpg)

#### 表 Region

![](https://ws1.sinaimg.cn/large/006tKfTcgy1g0j1frorkej31f80u04qp.jpg)
![](https://ws4.sinaimg.cn/large/006tKfTcgy1g0j1ltajqij31uk0b0qcx.jpg)

#### Region 定位

![](https://ws2.sinaimg.cn/large/006tKfTcgy1g0j1x3cpv4j31bc0u0no6.jpg)
![](https://ws1.sinaimg.cn/large/006tKfTcgy1g0j200u1u3j31260u01kx.jpg)
![](https://ws4.sinaimg.cn/large/006tKfTcgy1g0j22s6oo9j31800u0x51.jpg)

#### 结构
![](https://ws1.sinaimg.cn/large/006tKfTcgy1g0j2qvmpcij313l0u07v1.jpg)

**MemStore**容量有限，周期性写入到**StoreFile**,HLog写入一个标记。每次缓存刷新生成新的**StoreFile**，
当**StoreFile**数量到达某个阈值，会合并一个大**StoreFile**。当大**StoreFile**大小到达某个阈值，会分裂。
![](https://ws4.sinaimg.cn/large/006tKfTcgy1g0j3evboptj31gc0iiwqf.jpg)


![](https://ws1.sinaimg.cn/large/006tKfTcgy1g0j3j6dkl4j31hc0n8avu.jpg)
![](https://ws3.sinaimg.cn/large/006tKfTcgy1g0j31qmnmcj31g609ith9.jpg)



#### 读写
![](https://ws1.sinaimg.cn/large/006tKfTcgy1g0j2rfgaj2j31gu084tf3.jpg)


## 局限
![](https://ws3.sinaimg.cn/large/006tKfTcgy1g0j16237y9j31uw0bwans.jpg)

# NoSQL
不需要事务，读写实时性，没有复杂SQL查询。

## 种类

![](https://ws3.sinaimg.cn/large/006tKfTcgy1g0jl40p0b9j317u0u0nig.jpg)

# Spark

[Spark](/blog/2019/03/01/spark)

# 流计算

- 静态数据 批量计算 时间充足批量处理海量数据
- 流数据 实时计算
![](https://ws1.sinaimg.cn/large/006tKfTcgy1g0l0c3ng4ij31kk0f0gsc.jpg)

## 流数据特征
![](https://ws2.sinaimg.cn/large/006tKfTcly1g0kzn90nq6j31z60g67hy.jpg)

## 实时采集
![](https://ws3.sinaimg.cn/large/006tKfTcgy1g0l0iquglqj31620u0e81.jpg)

## 实时计算
![](https://ws1.sinaimg.cn/large/006tKfTcgy1g0l0la1nw4j31zs0f6nad.jpg)

## 实时查询
![](https://ws3.sinaimg.cn/large/006tKfTcgy1g0l0nnadogj32080r41kx.jpg)

## Storm

[Storm](/blog/2019/10/20/storm)

## Spark Streaming
将stream拆分成小量批处理, 做不到**毫秒级别**，**storm** 可以
![](https://ws2.sinaimg.cn/large/006tKfTcgy1g0l3u9pl8lj31y40r4e5z.jpg)

### 对比storm
做不到**毫秒级别**，**storm** 可以
![](https://ws3.sinaimg.cn/large/006tKfTcgy1g0l3xi6k1oj31xq0deh0q.jpg)

