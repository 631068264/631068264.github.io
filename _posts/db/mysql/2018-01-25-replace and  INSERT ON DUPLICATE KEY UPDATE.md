---
layout:     post
rewards: false
title:      replace and  INSERT ON DUPLICATE KEY UPDATE
categories:
    - mysql
---

- 处理具有唯一键或主键的记录时，
REPLACE将执行DELETE，然后执行INSERT，或者只执行INSERT。
此函数将导致删除记录，并在末尾插入，这将导致索引分离，从而降低表的效率。

- 该记录具有与我们尝试更新的记录相同的UNIQUE或PRIMARY KEY。如果找到现有的，则为要更新的列指定一个子句。否则，它将执行正常的INSERT。

