---
layout:     post
rewards: false
title:      MyISAM与InnoDB的主要区别
categories:
    - mysql
---

[MySQL存储引擎MyISAM与InnoDB的主要区别对比](http://www.ha97.com/4197.html)
[MYsql innodb](http://www.cnblogs.com/youxin/p/3359132.html)



# 文件存储结构

- innodb

  **.frm文件：**与表相关的**元数据信息**都存放在frm文件，**包括表结构的定义信息等**。
  **.ibd文件**: **独享表空间**存储方式使用，并且每个表一个ibd文件
  **.ibdata文件**: **共享表空间**存储方式使用.ibdata文件，所有表共同使用一个ibdata文件

- Myism

  **.frm文件：**与表相关的**元数据信息**都存放在frm文件，**包括表结构的定义信息等**。
  **.myd文件：**myisam存储引擎专用，用于存储myisam**表的数据**
  **.myi文件：**myisam存储引擎专用，用于存储myisam表的**索引相关信息**





# MyISAM

- 适合读多写少，不支持事务

- MyISAM的索引和数据是分开的，并且索引是有压缩的，内存使用率就对应提高了不少。能加载更多索引，而Innodb是索引和数据是紧密捆绑的，没有使用压缩从而会造成Innodb比MyISAM体积庞大不小

- AUTO_INCREMENT 更快，自增长的字段，InnoDB中必须包含只有该字段的索引，但是在MyISAM表中可以和其他字段一起建立联合索引

- 保存表具体行数 select count(*) from table，包含 where条件时，两种表的操作是一样的

- 表锁，并发性差

  

  

# InnoDB

- 支持事务，外键，故障恢复快
- 如果你的数据执行大量的**INSERT**或**UPDATE**，出于性能方面的考虑，应该使用InnoDB表
- 提供行锁
- InnoDB不支持FULLTEXT类型的索引