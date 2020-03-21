---
layout:     post
rewards: false
title:      docker 常用
categories:
    - docker
---

# build

`docker build -t <仓库名 URL等>:<new_标签> <上下文路径/URL> -f <Dockerfile path>`

使用当前目录的 Dockerfile 创建镜像，标签为 runoob/ubuntu:v1

```
docker build -t runoob/ubuntu:v1 .
```

# manage

虚悬镜像
`docker rmi $(docker images -q -f dangling=true)`


所有的容器 ID
`docker ps -aq`

停止所有的容器
`docker stop $(docker ps -aq)`

删除所有的容器
`docker rm $(docker ps -aq)`

删除所有的镜像
`docker rmi $(docker images -q)`

删除stop container
`docker container prune`

删除dangling image 
`docker image prune`

# run/exec

- t:在新容器内指定一个伪终端或终端
- i:对容器内的标准输入
- d:后台run
- p:端口绑定

# 查看

docker ps -a

docker logs -f [container ID or NAMES]

docker stats

docker info

# 交互

`docker exec -it <container-name> bash`


# docker volume

`docker-compose stop && docker-compose rm -f`
`docker stop $(docker ps -aq) && docker container prune`

```
docker volume ls
docker volume rm <VOLUME NAME>

```