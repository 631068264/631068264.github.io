---
layout:     post
rewards: false
title:     Web Hack 工具
categories:
    - hack
tags:
    - web hack
---

# nmap
[nmap](https://nmap.org/download.html)

- 通过对设备或者防火墙的探测来审计其安全性。
- 探测目标主机的开放端口。
- 网络存储、网络映射、维护和资产管理。（这个有待深入）
- 通过识别新的服务器审计网络的安全性。
- 探测网络上的主机。
## script
NSE 的脚本引擎 `--script=<类型>`

- auth: 负责处理鉴权证书（绕开鉴权）的脚本
- broadcast: 在局域网内探查更多服务开启状况，如dhcp/dns/sqlserver等服务
- brute: 提供暴力破解方式，针对常见的应用如http/snmp等
- default: 使用-sC或-A选项扫描时候默认的脚本，提供基本脚本扫描能力
- discovery: 对网络进行更多的信息，如SMB枚举、SNMP查询等
- dos: 用于进行拒绝服务攻击
- exploit: 利用已知的漏洞入侵系统
- external: 利用第三方的数据库或资源，例如进行whois解析
- fuzzer: 模糊测试的脚本，发送异常的包到目标机，探测出潜在漏洞
- intrusive: 入侵性的脚本，此类脚本可能引发对方的IDS/IPS的记录或屏蔽
- malware: 探测目标机是否感染了病毒、开启了后门等信息
- safe: 此类与intrusive相反，属于安全性脚本
- version: 负责增强服务与版本扫描（Version Detection）功能的脚本
- vuln: 负责检查目标机是否有常见的漏洞（Vulnerability），如是否有MS08_067

## scan
要扫描1 ~ 1024 端口，详细输出，并且探测操作系统。
`nmap 192.168.1.1 -p 1-1024 -vv -O`

# sqlmap

`pip install sqlmap`

直接使用`-u`命令把 URL 给 SqlMap 会判断注入点

## 判断所有的动态参数

`sqlmap -u http://localhost/sql.php?id=`
## 指定某个参数，使用`-p`

`sqlmap -u http://localhost/sql.php?id= -p id`


## 获取数据库及用户名称

--dbs用于获取所有数据库名称，--current-db用于获取当前数据库，--current-user获取当前用户

`sqlmap -u http://localhost/sql.php?id= -p id --current-db`

## 获取表名

-D用于指定数据库名称，如果未指定则获取所有数据库下的表名。--tables用于获取表名。

`sqlmap -u http://localhost/sql.php?id= -p id -D test --tables`

## 获取列名

-T用于指定表名，--columns用于获取列名。

`sqlmap -u http://localhost/sql.php?id= -p id -D test -T email --columns`

## 获取记录

--dump用于获取记录，使用-C指定列名的话是获取某一列的记录，不指定就是获取整个表。

`sqlmap -u http://localhost/sql.php?id= -p id -D test -T email --dump`