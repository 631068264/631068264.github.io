---
layout:     post
rewards: false
title:      zookeeper
categories:
    - big data
tags:
    - big data
---

# what

文件系统+监听通知机制

ZooKeeper 是一个典型的分布式数据一致性解决方案，分布式应用程序可以基于 ZooKeeper 实现诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master 选举、分布式锁和分布式队列等功能

![](http://ww2.sinaimg.cn/large/006tNc79gy1g5vl5va9nhj30go055mxc.jpg)

- 顺序一致性 - 客户端的更新将按发送顺序应用。
- 原子性 - 更新成功或失败。没有部分结果。
- 单系统映像 - 无论服务器连接到哪个服务器，客户端都将看到相同的服务视图。
- 可靠性 - 一旦应用了更新，它将从那时起持续到客户端覆盖更新。
- 及时性 - 系统的客户视图保证在特定时间范围内是最新的。


使用简单
- create : creates a node at a location in the tree
- delete : deletes a node
- exists : tests if a node exists at a location
- get data : reads the data from a node
- set data : writes data to a node
- get children : retrieves a list of children of a node
- sync : waits for data to be propagated

# znode

文件目录 znode(目录节点) 节点有两种 持久的，临时的
![](http://ww2.sinaimg.cn/large/006tNc79gy1g5vltbw0dpj30ca071t8o.jpg)

通过一个 Leader 选举过程来选定一台称为 **Leader** 的机器，Leader
既可以为客户端提供写服务又能提供读服务。除了 Leader 外，Follower 都只能提供读服务。

# 集群

ZAB（ZooKeeper Atomic Broadcast 原子广播）

实现分布式数据一致性，基于该协议，ZooKeeper 实现了一种主备模式的系统架构来保持集群中各个副本之间的数据一致性。

## 崩溃恢复

当整个服务框架在启动过程中，或是当 Leader 服务器出现网络中断、崩溃退出与重启等异常情况时，ZAB 协议就会进人恢复模式并选举产生新的Leader服务器。
当选举产生了新的 Leader 服务器，同时集群中已经**有过半的机器与该Leader服务器完成了状态同步**之后，ZAB协议就会退出恢复模式。


## 消息广播

集群中已经存在一个Leader服务器在负责进行消息广播，那么新加人的服务器就会自觉地进人数据恢复模式：
找到Leader所在的服务器，并与其进行数据同步，然后一起参与到消息广播流程中去。

