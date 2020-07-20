---
layout: post
rewards: false
title:  port 开放
categories:
    - Linux
---

# 服务端

`netstat -ntupl | grep port` 查看TCP/UDP类型的端口

- 0.0.0.0:port 外网可以访问 
- 127.0.0.1:port 只有服务器内部可以访问，不开放



- -t (tcp)仅显示tcp相关选项
- -u (udp)仅显示udp相关选项
- -p 显示建立相关链接的程序名
- -l 仅列出有在 Listen (监听) 的服務状态


# 客户端

- telnet ip port
- nmap ip -p port

