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



# 宿主机网络

在使用 Docker 时，要注意平台之间实现的差异性，如 Docker For Mac 的实现和标准 Docker 规范有区别，Docker For Mac 的 Docker Daemon 是运行于虚拟机 (xhyve) 中的，而不是像 Linux 上那样作为进程运行于宿主机，因此 Docker For Mac 没有 docker0 网桥，不能实现 host 网络模式，host 模式会使 Container 复用 Daemon 的网络栈 (在 xhyve 虚拟机中)，而不是与 Host 主机网络栈







# 现象原理

和默认网桥**相同网段的机器** A访问不了宿主机B的服务，ping不通，grpc也连不上，不同网段可以访问。



因为docker的桥接模式修改宿主机上的路由表，A其实可以访问到B，但是得不到B的应答。

A的应答会根据**docker0**接口的网关，发送到对应ip的docker容器，而不会到B。

可以使用 在宿主机上面 **traceroute B的ip**



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


