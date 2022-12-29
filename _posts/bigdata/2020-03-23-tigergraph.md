---
layout:     post
rewards: false
title:  Tiger graph
categories:
    - big data
tags:
    - big data
---

# 安装

- `tar -xzvf *.tar.gz`
- `sudo install.sh -n`

检查安装状况

- `gadmin status` `gadmin start`和`gadmin stop`启动和停止服务
- `gsql version`

TigerGraph使用gadmin命令进行管理，需要在tigergraph用户下执行，安装在/home/tigergraph/tigergraph/下



从图存储中删除所有数据
`clear graph store –HARD`

![](https://cdn.jsdelivr.net/gh/631068264/img/00831rSTgy1gd3q6wolnxj30v30u0tae.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/00831rSTgy1gd3q7i0tqvj30ul0hjmxq.jpg)


# Schema

加载前数据前要定义**图模型(Graph Schema)**就像一本字典一样，定义了图形中的各种实体（顶点和边）的类型以及它们之间的关系。

定义顶点类和边类

## 点

```
CREATE VERTEX vertex_type_name (PRIMARY_ID id_name type
      [, attribute_name type [DEFAULT default_value] ]*)
      [WITH STATS="none"|"outdegree_by_edgetype"]
      
```

WITH STATS 该顶点与其他顶点的连接数

## 边
```
CREATE UNDIRECTED|DIRECTED EDGE edge_type_name (FROM vertex_type_name , TO vertex_type_name ,
edge_attribute_list ) [ edge_options ]
```

声明CREATE DIRECTED EDGE时同时声明了参数WITH REVERSE_EDGE=**rev_name**，则一个额外的有向型边**rev_name**会自动生成。
该边的起点与终点与原始创建边相反。之后，每当一个新的边生成，就会自动生成一个反向的边。反向的边拥有与原始边相同的属性。同时，当原始的边有变更时，对应反向的边也同时会变更。

在TigerGraph系统中，反向的边可以大幅增加图查询的效率，特别是需要回溯的查询。

使用通配符*来表示任意顶点类

`CREATE DIRECTED EDGE any_edge (FROM *, TO *, label STRING)`


# 图

`CREATE GRAPH gname (vertex_or_edge_type, vertex_or_edge_type...) [WITH ADMIN username]`

```
CREATE GRAPH everythingGraph (*)
CREATE GRAPH emptyGraph ()
```


## use graph

`USE GRAPH gname`

```
附带-g <graph_name>选项调用gsql

gsql -g Book_rating book_load.gsql
```

`DROP GRAPH gname`

`DROP ALL`

任何一个不与其他图共享的顶点类和边类以及它们对应的数据都会被删除。
但如果该类与其它的图共享，则该类不会被删除。如果只是想删除某个指定的边类或顶点类。


# 修改graph


- 创建SCHEMA_CHANGE JOB，它定义ADD，ALTER和/或DROP语句的顺序。
- 运行SCHEMA_CHANGE JOB（即RUN JOB job_name），它将执行以下操作：
    - 尝试更改图形数据库Schema
    - 如果更改成功，则任何与新图形数据库Schema不兼容的任何加载作业或查询失效。
    - 如果更改失败，则报告错误并返回到尝试更改之前的状态。


```
CREATE VERTEX User (PRIMARY_ID user_id UINT, name STRING, age UINT, gender STRING, postalCode STRING)
CREATE VERTEX Occupation (PRIMARY_ID occ_id UINT, occ_name STRING) WITH STATS="outdegree_by_edgetype"
CREATE VERTEX Book  (PRIMARY_ID bookcode UINT, title STRING, pub_year UINT) WITH STATS="none"
CREATE VERTEX Genre (PRIMARY_ID genre_id STRING, genre_name STRING)
CREATE UNDIRECTED EDGE user_occupation (FROM User, TO Occupation)
CREATE UNDIRECTED EDGE book_genre (FROM Book, TO Genre)
CREATE UNDIRECTED EDGE user_book_rating (FROM User, TO Book, rating UINT, date_time UINT)
CREATE UNDIRECTED EDGE friend_of (FROM User, TO User, on_date UINT)
CREATE UNDIRECTED EDGE user_book_read (FROM User, To Book, on_date UINT)
CREATE DIRECTED EDGE sequel_of (FROM Book, TO Book) WITH REVERSE_EDGE="preceded_by"
CREATE GRAPH Book_rating (*)
```
# 编程

[tg-jdbc](https://github.com/tigergraph/ecosys/tree/master/etl/tg-jdbc-driver)

jdbc功能简陋，性能确实比ne4j好


```
CREATE QUERY test_demo(/* Parameters here */) FOR GRAPH traffic { 
  /* Write query logic here */ 
	STRING d = datetime_format(now(),"％Y-％m-％d％");
  PRINT d; 
}


CREATE QUERY GetAppProto(STRING traffic_date) FOR GRAPH traffic RETURNS (GroupByAccum<STRING ip,STRING date,STRING app_proto ,SumAccum<INT>num>){

	GroupByAccum<STRING ip,STRING date,STRING app_proto ,SumAccum<INT>num > @@DstAccum;
	machine = {MACHINE.*};
	results = SELECT p
	          FROM machine-(TRAFFIC:l)->:p
	          WHERE l.traffic_date == traffic_date
	          ACCUM
	            @@DstAccum += (p.ip,l.traffic_date,l.app_proto->1);
  RETURN @@DstAccum;
}
```
