---
layout:     post
rewards: false
title:      计算机网络
categories:
    - network
tags:
    - network
---

# 体系结构

![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1k336rc92j313y0jawfq.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1k34vv5hxj30x50u0tbg.jpg)

# 物理层
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1k3ufx47zj314g0gudgh.jpg)

# 数据链路层
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1k3wby5zgj318y0s60v4.jpg)

一种叫做["以太网"](http://zh.wikipedia.org/wiki/以太网)（Ethernet）的协议，占据了主导地位。

以太网规定，一组电信号构成一个数据包，叫做"帧"（Frame）。每一帧分成两个部分：标头（Head）和数据（Data）



"标头"包含数据包的一些说明项，比如发送者、接受者、数据类型等等；"数据"则是数据包的具体内容。

"标头"的长度，固定为18字节。"数据"的长度，最短为46字节，最长为1500字节。因此，整个"帧"最短为64字节，最长为1518字节。如果数据很长，就必须分割成多个帧进行发送。

![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1k3wizt4ej318g0mwgnf.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1k3woue1qj31490u00v7.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1k3wvo8ovj316a0rcq73.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1k3x4tnw6j314s0j0n0r.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1k3yu44krj316s0nodh5.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1k3z4xz8fj30yc0u0wkf.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1k3zfvotaj315k0e2dgk.jpg)

# 网络层
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1k48kz2z9j31760siadc.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1k4ao0dbvj31160kydgw.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1k4au9uduj31940g4dh6.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1k4edxdiqj31660pc0v9.jpg)

![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1k4kmvhkoj31c20qun1o.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1k4kva67oj31cv0u0q6a.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1k4l5dfpoj311p0u0tbv.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1kkqpfwjvj31au0o2diw.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1kkqxllhhj31c60aw75h.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1kkr3sjsuj31b00nu77w.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1kkr9vl98j31ci0l4jug.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1kkrexfpzj31160l0myz.jpg)

# 运输层

![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1kl4cm13ij312y0u00xj.jpg)

三次握手
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1kl4m77guj31860meq3z.jpg)

四次挥手
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1kl4x2guzj314s0mwdh6.jpg)

## 超时重传 重传策略

![](https://cdn.jsdelivr.net/gh/631068264/img/008eGmZEgy1goroim2gy8j31gu0u0n1a.jpg)

![image-20210321180058626](https://cdn.jsdelivr.net/gh/631068264/img/008eGmZEgy1gorokjnjivj31ic0u01ed.jpg)

![](https://cdn.jsdelivr.net/gh/631068264/img/008eGmZEgy1goromfchllj31j00u0wik.jpg)



## 拥塞控制

![image-20210321181947219](https://cdn.jsdelivr.net/gh/631068264/img/008eGmZEgy1gorp445nsuj31ir0u0hdt.jpg)

![image-20210321182411996](https://cdn.jsdelivr.net/gh/631068264/img/008eGmZEgy1gorp8pd7l4j31g00u01kx.jpg)

![image-20210321183018930](https://cdn.jsdelivr.net/gh/631068264/img/008eGmZEgy1gorpf2lgayj31gn0u0ayx.jpg)



![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1kl54huhaj31ca0eytab.jpg)

# 应用层

![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1kleful8bj31az0u0gnp.jpg)
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g1klekxur9j311a0u0dj2.jpg)

# 通信过程



mac48位（link layer）Ethernet协议  以太网数据包 



ip 32位  IP & 子网掩码 子网划分  network layer

同一子网 ARP广播 通过IP获取对方mac

不同子网送到网关



Transport Layer  "传输层"的功能，就是建立"端口到端口"的通信。相比之下，"网络层"的功能是建立"主机到主机"的通信。只要确定主机和端口，我们就能实现程序之间的交流。

主机+端口，叫做"套接字"（socket）

UDP/TCP





dns 解析

**检查浏览器缓存中是否缓存过该域名对应的IP地址**

**如果在浏览器缓存中没有找到IP，那么将继续查找本机系统是否缓存过IP**

**向本地域名解析服务系统发起域名解析的请求**

**向根域名解析服务器发起域名解析请求**

**根域名服务器返回gTLD域名解析服务器地址**



