---
layout:     post
rewards: false
title:      docker for mac network host failed
categories:
    - docker
---

想docker容器访问本地服务

```sh
docker run --name test --net="host" -e "MYSQL_ADDRESS=host.docker.internal:3306" -d go-example
```

https://docs.docker.com/network/host/

> 主机网络驱动程序仅适用于 Linux 主机，不支持 Docker for Mac、Docker for Windows 或 Docker EE for Windows Server。

docker 守护程序实际上是在包含 Linux 内核的虚拟机中运行，虚拟机的网络与主机不同。

当使用`--net host`交换机运行 docker 时，您将容器连接到虚拟机网络，而不是连接到您的主机网络，因为它在 Linux 上运行。然后尝试连接到 127.0.0.1 或 localhost 不允许连接到正在运行的容器。

根据https://docs.docker.com/desktop/mac/networking/

**从容器连接到主机上的服务**

可以使用`host.docker.internal`代替`127.0.0.1`

连接到 `host.docker.internal`解析为主机使用的内部 IP 地址的特殊 DNS 名称。这是出于开发目的，不适用于 Docker Desktop for Mac 之外的生产环境
