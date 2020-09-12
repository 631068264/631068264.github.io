---
layout:     post
rewards: false
title:      docker服务使用docker命令控制其他docker服务
categories:
    - docker
---



image dockerfile 不需要安装docker 直接用宿主机的环境

```
version: '3'

services:
  serivice_name:
    image: image
    container_name: container_name
    network_mode: host
    volumes:
      - /usr/bin/docker:/usr/bin/docker
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/local/bin/docker-compose:/usr/bin/docker-compose
    restart: on-failure
```

进去容器里面使用`docker ps` 可以验证一下



如果想直接调用系统级别的命令如 systemctl，可以通过端口通信调用宿主机命令，效果会怪怪的。要确保服务没问题。


`docker-compose` 需要docker-compose.yml