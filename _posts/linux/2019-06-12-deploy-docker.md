---
layout:     post
rewards: false
title:      Ubuntu 下docker 部署
categories:
    - Linux
---


# 安装

```shell
apt-get install -y docker.io nginx docker-compose 
```


```shell
systemctl start docker

systemctl enable docker


# 开机启动
systemctl enable nginx

systemctl start nginx 
systemctl stop nginx 
systemctl restart nginx
```

# 配置启动 service

```
cd etc/systemd/system/
```

```
[Unit]
Description={xxxxx}
After=network.target
After=docker.service

[Service]
Type=simple
User=root
ExecStart={where is docker-compose} up
WorkingDirectory={project}
Restart=on-failure
RestartSec=15

StandardOutput=syslog
StandardError=syslog

[Install]
WantedBy=multi-user.target
``` 

```
# Service 状态
systemctl status endoscope

# 启动失败 查看log
journalctl -u xxxx.service -f

# 更新xxx.service
systemctl daemon-reload
```