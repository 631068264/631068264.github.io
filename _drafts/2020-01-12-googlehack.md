---
layout:     post
rewards: false
title:      tool
categories:
    - hack
tags:
    - kali
---

# google hack

- [GHDB](https://www.exploit-db.com/google-hacking-database)
- [shodan](https://www.shodan.io/)
- [netcraft](https://www.netcraft.com/)

当然可以用google的**高级搜索**

## common
```
""  必须包含且不能拆开的关键词
intitle:xxx 页面标题关键字
ext:xxx 查找指定文件类型 xlsx
site:domain 域名中所有的 URL 地址
inurl:xxx  在url中查找关键字
intext:xxx 网页内容关键字
link:xx 查找与关键字做了链接的 URL 地址
Index of 主页服务器所进行操作的一个索引目录
```

## example

```
intitle:"webview login" alcatel lucent
site:*/cgi/domadmin.cgi
intitle:"Zabbix" intext:"username" intext:"password" inurl:"/zabbix/index.php"
"MailChimp API error:" ext:log

Index of /passwd
Index of /password
Index of /mail
"Index of /" +passwd
"Index of /" +password.txt
"Index of /" +.htaccess
"Index of /secret"
"Index of /confidential"
"Index of /root"
"Index of /cgi-bin"
"Index of /credit-card"
"Index of /logs"
"Index of /config"


site:xx.com-site:www.xx.com  差集
```

# theharvester

search email 

`theHarvester -d baidu.com -b baidu`


# nmap

## 基础

nmap -p 端口 IP(域名)
```
nmap -p 8080 192.168.31.13
nmap  -p 22,21,80 192.168.31.13
nmap  -p 22,21,80 192.168.31.1-253
nmap  -p 22,21,80 192.168.31.1/24

nmap 10.0.1.161,162
nmap 10.0.1.161  10.0.1.162
```

## 常用
```
-sV 端口扫描的时候检测服务端软件的版本信息
-O 操作系统检测
-Pn 禁用主机检测 主机屏蔽了ping请求，Nmap可能会认为该主机没有开机。 指定这个选项之后，Nmap会认为目标主机已经开机并会进行全套的检测工作
-sC 脚本扫描
–traceroute 路由跟踪

-A 相当于 -sV -O  -sC –traceroute
```

## TCP/UDP

```
-sT  默认目标主机的每个端口都进行完整的三次握手 扫描方式比较慢，而且扫描行为很可能被目标主机记录下来  
-sS  发送SYN包 目标主机回复了SYN/ACK包，则说明该端口处 于开放状态 如果回复的是RST/ACK包，则说明这个端口处于关闭状态；如果没有任何响应 或者发送了ICMP unreachable信息，则可认为这个端口被屏蔽了

-sU  UDP扫描  加限制会耗时 --top-ports 200
```

## 端口

```
-p 22 22,21 1-1024  扫描1〜65535端口时，可使用-p-选项
-F 快速扫描 将仅扫描100个常用端口。
--top-ports 扫描nmap-services里排名前N的端口。
```

状态

```
open 工作于开放端口的服务器端的应用程序可以受理TCP	连接、接收UDP数据包或者响 应SCTP（流控制传输协议）请求。

closed 虽然我们确实可以访问有关的端口，但是没有应用程序工作于该端口上。

filtered Nmap不能确定该端口是否开放。包过滤设备屏蔽了我们向目标发送的探测包。

未过滤：虽然可以访问到指定端口，但Nmap不能确定该端口是否处于开放状态。 

open|filtered：Nmap认为指定端口处于开放状态或过滤状态，但是不能确定处于两者之中的 哪种状态。在遇到没有响应的开放端口时，Nmap会作出这种判断。这可以是由于防火墙丢 弃数据包造成的。

closed｜过滤：Nmap	认为指定端口处于关闭状态或过滤状态，但是不能确定处于两者之中的 哪种状态。
```

## 输出

```
-oN 正常输出 不显示runtime信息和警告信息。
-oX XML格式 可被Nmap的图形界面解析
```

## 时间
- T

0~5 数字越大扫描速度越快
```
3 默认 同时向多个目标发送多个数据包 扫描时间和网络负载之间进行平衡
```

## 改进

```
-f 使用小数据包 使用8字节甚至更小数据体的数据包 可避免对方识别出我们探测的数据包
-sC --script=all  /usr/share/nmap/scripts
```

