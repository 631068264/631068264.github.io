---
layout:     post
rewards: false
title:      docker remote debug
categories:
    - docker
---
Pycharm local debug docker


# Build, Execution, Deployment
![](https://cdn.jsdelivr.net/gh/631068264/img/006tNc79ly1g3sc1hgc0fj31jy0u0gmp.jpg)

# Python Interpreter

<span class='gp-2'>
    <img src='https://cdn.jsdelivr.net/gh/631068264/img/006tNc79ly1g3sc9a2lnij31co0u00tm.jpg' />
    <img src='https://cdn.jsdelivr.net/gh/631068264/img/006tNc79ly1g3scamb6anj31dd0u0jsx.jpg' />
</span>

基本弄好这个就可以下断点

# docker log
![](https://cdn.jsdelivr.net/gh/631068264/img/006tNc79gy1g3seaxhfujj31ps0u0aca.jpg)


# update project

重构 修改的部分

```shell
docker-compose -f .debug.docker-compose.yml up --build -d
```

# compose config

compose services host name
```dockerfile
version: '3'

services:
  endoscope-web:
    build:
      context: .
      dockerfile: .debug.Dockerfile
    depends_on:
      - mysql
      - redis
    ports:
      - 9000:9000
    privileged: true
    volumes:
      - ./.etc/localtime:/etc/localtime
      - ./conf:/endoscope/conf
    restart: on-failure

  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: ""
      MYSQL_USER: ""
      MYSQL_PASSWORD: ""
      MYSQL_DATABASE: ""
    volumes:
      - ./.etc/localtime:/etc/localtime
      - ./sql/create.sql:/docker-entrypoint-initdb.d/create.sql
      - mysql_data:/var/lib/mysql
    ports:
      - 3306:3306
    restart: on-failure

  redis:
    image: redis
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    ports:
      - 6379:6379
    restart: on-failure
```

