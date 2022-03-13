---
layout:     post
rewards: false
title:   正向代理配置
categories:
    - k8s

---

# squid

http/https正向代理

配置路径

```
/etc/squid/squid.conf
```

log目录

```
/var/log/squid
```

启动

```sh
cp squid.conf /etc/squid/squid.conf
# 初始化缓存空间 不做这步systemctl start squid监听进程起不来，神经病
squid -z
# chown -R squid:squid /var/spool/squid
# chown -R squid:squid /var/log/squid

# 启动
systemctl enable squid
systemctl start squid
```

命令

```sh
# 验证squid.conf的语法和配置 有语法或配置错误，这里会返回提示。如果没有返回，如果没有返回则启动成功
squid -k parse
# 载入新的配置文件
squid -k reconfigure

# 测试启动
squid -N -d1

# 停止
squid -k shutdown
```

权限

```sh
# 修改cache 缓存目录的权限
chown -R squid:squid /var/spool/squid
# 修改squid 日志目录的权限
chown -R squid:squid /var/log/squid

#more /var/log/squid/access.log | grep TCP_MEM_HIT    
该指令可以看到在squid运行过程中，有那些文件被squid缓存到内存中，并返回给访问用户。    
#more /var/log/squid/access.log | grep TCP_HIT    
该指令可以看到在squid运行过程中，有那些文件被squid缓存到cache目录中，并返回给访问用户。    
#more /var/log/squid/access.log | grep TCP_MISS    
该指令可以看到在squid运行过程中，有那些文件没有被squid缓存，而是现重原始服务器获取并返回给访问用户。

```

设置crontab

```sh
0 4 * * * /usr/sbin/squid -k rotate
```

另外，要特别提示一下swap.state文件。当squid应用运行了一段时间以后，cache_dir对应的swap.state文件就会变得越来越大，里面的无效接口数据越来越多，这可能在一定程度上影响squid的响应时间，此时我们可以使用rotate命令来使squid清理swap.state里面的无效数据，减少swap.state的大小。



配置

```sh
# Deny CONNECT to other than secure SSL ports
# 默认ssl拒绝非443端口
#http_access deny CONNECT !SSL_ports

# And finally deny all other access to this proxy
http_access allow all

# Squid normally listens to port 3128
http_port 3128

# 解除visible_hostname的wanring
visible_hostname squid.packet-pushers.net
# 使得hosts起效
dns_v4_first on
```

# dnate

SOCKS v4 and v5 proxy server [下载对应的dnate和dnate-server](https://pkgs.org/search/?q=dante)

```sh
rpm -Uvh *.rpm
# 不做这步启动不了
mkdir -p /var/run/sockd

systemctl enable sockd
systemctl start sockd
```

配置目录

```
/etc/sockd.conf
```



配置

```sh
# the server will log both via syslog, to stdout and to /var/log/sockd.log
#logoutput: syslog stdout /var/log/sockd.log
logoutput: stderr

# The server will bind to the address 10.1.1.1, port 1080 and will only
# accept connections going to that address.
# 使用本地所有可用网络接口的 1080 端口
internal: 0.0.0.0 port = 1080

# all outgoing connections from the server will use the IP address
# 输出接口设置为 eth0
external: eth0

# methods for socks-rules.
socksmethod: username none #rfc931

# methods for client-rules.
clientmethod: none

# when doing something that can require privilege, it will use the
# userid "sockd".
user.privileged: root

# when running as usual, it will use the unprivileged userid of "sockd".
user.unprivileged: nobody

# 访问规则
client pass {
    from: 0.0.0.0/0 port 1-65535 to: 0.0.0.0/0 port 1-65535
    log: ioop
}

socks pass {
    from: 0.0.0.0/0 to: 0.0.0.0/0
    log: ioop
}
```

| 项目       | 区别                                                         | 备注                         |
| ---------- | ------------------------------------------------------------ | ---------------------------- |
| Client规则 | 对于限制客户端接入，服务端有选择地拒绝建立TCP                | 工作在TCP层，优先于Socks规则 |
| Socks规则  | 对于已经accept connection的连接，服务端有选择的拒绝转发Socket | 工作在Socks层                |

所以Client规则是在TCP的accept阶段进行控制，Socks规则是满足Client规则后且建立TCP连接后的Sock层控制。有顺序之分。

# 代码参考

- [proxy to smtp](https://www.lunaplus.net/posts/2021/06/proxy-for-smtp/)
- https://github.com/631068264/pyemailtool

```python
from_addr = 'xxxx@163.com'
mail = EmailUtil(host='smtp.163.com', passwd='xxxx', port=465, from_addr=from_addr,
                 proxy_url='http://localhost:3128')
mail.send_email(from_addr, 'test', 'msg')
```





