---
layout:     post
rewards: false
title:      docker network 与 本地 network 网段冲突
categories:
    - docker
---


# 背景

[docker-网络模式](https://docs.docker.com/network/#network-drivers)

get container ip

```
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name_or_id	
```





## **bridge**

虚拟机和主机是处于同等地位的机器



**Docker** 默认使用的模式, **Docker Daemon** 启动时默认会创建 **Docker0** 这个网桥, 网段为 **172.xx.0.0/16** 

同一网桥网络的容器进行通信，同时与未连接到该网桥网络的容器隔离，不同网桥网络上的容器无法直接相互通信。

## host

使用时容器不拥有自己的IP地址 ，端口映射不生效



#  docker重新分配网桥Ip

- stop and clear docker container  `docker stop $(docker ps -aq)   docker rm $(docker ps -aq)`

- `docker network ls`  ``docker network prune`  clear  network

- `/etc/docker/daemon.json`

  ```json
  {
    "default-address-pools" : [
      {
        "base" : "192.168.0.0/16",
        "size" : 24
      }
    ]
  }
  
  ```

  配置ip地址池  从**192.168.0.0 B段** 分出C段提供给docker网络

- `systemctl restart docker`

- restart other container


