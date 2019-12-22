---
layout:     post
rewards: false
title:  port 开放
categories:
    - Linux
---

# 服务端

`netstat -ntupl | grep port` 查看TCP/UDP类型的端口

- 0.0.0.0:port 外网可以访问 
- 127.0.0.1:port 只有服务器内部可以访问，不开放


# 客户端

- telnet ip port
- nmap ip -p port

