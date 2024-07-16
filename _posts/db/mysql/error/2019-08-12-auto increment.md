---
layout:     post
rewards: false
title:      auto increment
categories:
    - mysql
---

遇到mysql关于auto increment的重启坑


# 正常情况

定义表

```mysql
CREATE TABLE `user` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `user_name` varchar(10) NOT NULL,
  PRIMARY KEY (`id`),
) ENGINE=InnoDB AUTO_INCREMENT=101 DEFAULT CHARSET=utf8;
```

直接`insert into`的话id会是**101**


# 诡异情况

定义了`AUTO_INCREMENT`但没有插入数据，**重启mysql**后，在插入居然是`id=1`



# 关于AUTO_INCREMENT的原理

[官方 InnoDB AUTO_INCREMENT计数器初始化](https://dev.mysql.com/doc/refman/8.0/en/innodb-auto-increment-handling.html#InnoDB%20AUTO_INCREMENT%20Counter%20Initialization)


为InnoDB表某列指定AUTO_INCREMENT，在内存表对象中auto-increment
counter（自动增量计数器），为该列赋予新值。

MySQL 5.7及更早版本中，自动增量计数器仅存储在内存中，而不存储在磁盘上。所以服务重启时
，InnoDB会执行类似`SELECT MAX(ai_col) FROM table_name FOR
UPDATE;`的语句来重置**auto-increment counter**

MySQL 8.0中，此行为已更改。当**maximum auto-increment counter
value**每次改变时，都会写入redo
log，并保存到每个检查点上的引擎专用系统表中。保证重启后最大值不变。

在MySQL 5.7及更早版本中，服务器重新启动会取消AUTO_INCREMENT = Ntable选项的效果，
该选项可以在CREATE TABLEor ALTER TABLE语句中用于设置初始计数器值或分别更改现有计数器值。
在MySQL 8.0中，服务器重新启动不会取消AUTO_INCREMENT = N表选项的效果 。如果将自动递增计数器初始化为特定值，
或者将自动递增计数器值更改为更大的值，则新值将在服务器重新启动时保持不变。

