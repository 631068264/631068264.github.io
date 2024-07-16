---
layout:     post
rewards: false
title:      MySQL Server has gone away
categories:
    - mysql
---

# Mysql 曾经挂过

查看mysql的运行时长

```mysql
mysql -uroot -p -e "show global status like 'uptime';"

+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Uptime        | 68928 |
+---------------+-------+
1 row in set (0.04 sec)
```

# 连接超时

**长连接** 长连接很久没有新的请求发起，达到了server端的timeout时长会
被server强行关闭。 此后再通过这个connection发起查询的时候，就会报错server has
gone away。

```
mysql -uroot -p -e "show global variables like '%timeout';"

+---------------------+----------------+
| variable_name       | variable_value |
+---------------------+----------------+
| INTERACTIVE_TIMEOUT | 28800          |
| WAIT_TIMEOUT        | 28800          |
+---------------------+----------------+
```

**interactive_timeout**和**wait_timeout**的区别

- interactive_timeout针对交互式连接，wait_timeout针对非交互式连接
- mysql客户端连接数据库是交互式连接，通过jdbc连接数据库是非交互式连接。 
- 连接启动的时候，根据连接的类型，来确认会话变量wait_timeout的值是继承于全局变量wait_timeout，还是interactive_timeout。
- 如果是交互式连接，则继承全局变量interactive_timeout的值，如果是非交互式连接，则继承全局变量wait_timeout的值

[验证](https://www.cnblogs.com/ivictor/p/5979731.html)

```py
代码中检查是否超时，是否需要重连   pool_recycle< WAIT_TIMEOUT
sqlalchemy  create_engine(pool_recycle)
```

# 结果集超大

通常发生在查询字段包含数据量较大 > max_allowed_packet

```
show global variables like 'max_allowed_packet';

+--------------------+---------+
| Variable_name      | Value   |
+--------------------+---------+
| max_allowed_packet | 1048576 |
+--------------------+---------+
1 row in set (0.00 sec)
```