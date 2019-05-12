---
layout:     post
rewards: false
title:      Docker Compose
categories:
    - docker
---

# 概述

Compose是一个用于定义和**运行多容器Docker**应用程序的工具。使用Compose，您可以**使用YAML文件**来配置应用程序的服务。
然后，使用单个命令，您可以从配置中创建并启动所有服务。

使用Compose基本上是一个三步过程：

- 定义您的应用程序环境，Dockerfile以便可以在任何地方进行复制。
- 定义构成应用程序的服务，docker-compose.yml 以便它们可以在隔离的环境中一起运行。
- Run `docker-compose up`和Compose启动并运行整个应用程序。


# 特征

- 单个主机上的多个隔离环境
- 创建容器时保留卷数据
- 仅重新创建已更改的容器
- 变量和在环境之间移动合成


# write

Dockerfile
```
FROM python:3.4-alpine
COPY . /code
WORKDIR /code
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

docker-compose.yml

```
version: '3'
services:
  web:
    build: .
    ports:
     - "5000:5000"
  redis:
    image: "redis:alpine"
```

或者 允许您动态修改代码，而无需重建映像。
```
version: '3'
services:
  web:
    build: .
    ports:
     - "5000:5000"
    volumes:
     - .:/code
  redis:
    image: "redis:alpine"
```
Compose文件定义了两个服务：web和redis。

从**项目目录**中，键入`docker-compose up [-d 后台模式]`


[docker-compose.yml 配置参考](https://docs.docker.com/compose/compose-file/#service-configuration-reference) 

# cmd

查看
`docker-compose ps`

停止
`docker-compose stop`

`docker-compose --help`