---
layout:     post
rewards: false
title:      Swarm
categories:
    - docker
---

# 概述

由多个Docker主机组成，这些主机以**群集模式**运行，一个主机可以充当managers，workers或者同时扮演两个角色。

manager node 指派task to worker nodes,负责集群管理，工作调度。
worker nodes执行被委派的task, 默认情况下manager node也会作为worker，可配置仅执行manage task。worker node会汇报工作状态到manager node.

task 包含指定的image and 有执行命令的container 是原子的一旦分配 不能转移到另一个node

Swarm模式有一个内部DNS组件，可以自动为swarm中的每个服务分配一个DNS条目。群集管理器使用内部负载平衡来根据服务的DNS名称在群集内的服务之间分发请求。

集群中的每个节点都强制执行TLS相互身份验证和加密，以保护自身与所有其他节点之间的通信。您可以选择使用自签名根证书或自定义根CA的证书。


<span class='gp-2'>
    <img src='https://ws2.sinaimg.cn/large/006tNc79ly1g2ydpbd34rj30m80fc0tj.jpg' />
    <img src='https://ws4.sinaimg.cn/large/006tNc79ly1g2ydq6mkb4j30o40h8wf8.jpg' />
</span>


# set


## add manager
```
docker-machine ls     /   docker-machine ip <MACHINE-NAME>
```

指定manager
```
# 使用名为的机器manager1。如果您使用Docker Machine，则可以使用以下命令通过SSH连接到它：

docker-machine ssh manager1

# 创建新的swarm  得到token 让work join

docker swarm init --advertise-addr <MANAGER-IP>
```

节点信息
```
docker node ls
```

## add work

在 work node 也run `ssh`
```
docker swarm join \
  --token  SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
  192.168.99.100:2377
  
# 在manage node 上获得命令
docker swarm join-token worker
```

## set task

```
docker service create --replicas 1 --name helloworld alpine ping docker.com
```

- 该docker service create命令创建服务。
- 该--name标志命名该服​​务helloworld。
- 该--replicas标志指定1个正在运行的实例的所需状态。
- 参数alpine ping docker.com将服务定义为执行命令的Alpine Linux容器ping docker.com。

`docker service ls`