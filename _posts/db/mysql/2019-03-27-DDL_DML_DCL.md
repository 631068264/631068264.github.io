---
layout:     post
rewards: false
title:      DDL, DML, DCL，TCL
categories:
    - mysql
---

![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1hcjank7ij310o0m8dhy.jpg)

# DDL
Data Definition Language 数据定义语言

这些语句定义了不同的数据段、数据库、表、列、索引等数据库对象的定义

**会自动提交 implict commit 隐式提交**

> 如果事务编写DML DDL夹杂   **DDL执行不管报错与否，都会执行一次commit**

```sql
SET AUTOCOMMIT = 1;
BEGIN;
INSERT INTO t1 VALUES (1);
CREATE TABLE t2 (pk int primary key);
INSERT INTO t2 VALUES (2);
ROLLBACK;
```
t1 插入 t2创建 t2插入

```sql
SET AUTOCOMMIT = 0;
BEGIN;
INSERT INTO t1 VALUES (1);
CREATE TABLE t2 (pk int primary key);
INSERT INTO t2 VALUES (2);
ROLLBACK;
```
t1 插入 t2创建   自动开启一个新事务 t2插入被回滚


# DML
Data Manipulation Language 数据操纵语句

用于添加、删除、更新和查询数据库记录，并检查数据完整性

delete 语句是数据库操作语言(dml)，这个操作会放到 **rollback segement** 中，事务提交之后才生效

# DCL

Data Control Language 数据控制语句

用于控制不同数据段直接的许可和访问级别的语句。这些语句定义了数据库、表、字段、用户的访问权限和安全级别。

# TCL
Transaction Control Language  事务控制

