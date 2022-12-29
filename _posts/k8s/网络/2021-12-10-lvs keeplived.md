---
layout:     post
rewards: false
title:   lvs keeplived nginx ha
categories:
    - k8s


---

# 基本概念

## VS/NAT,VS/TUN,VS/DR

- Virtual Server via Network Address Translation（VS/NAT）

  通过网络地址转换，**调度器重写请求报文的目标地址，根据预设的调度算法，将请求分派给后端的真实服务器**；真实服务器的响应报文通过调度器时，报文的源地址被重写，再返回给客户，完成整个负载调度过程。

- Virtual Server via IP Tunneling（VS/TUN）

  **采用 NAT 技术时，由于请求和响应报文都必须经过调度器地址重写，**当客户请求越来越多时，调度器的处理能力将成为瓶颈。为了解决这个问题，**调度器把请求报文通过 IP 隧道转发至真实服务器，而真实服务器将响应直接返回给客户，**所以调度器只处理请求报文。由于一般网络服务应答比请求报文大许多，采用 VS/TUN 技术后，集群系统的最大吞吐量可以提高 10 倍。

- Virtual Server via Direct Routing（VS/DR）

  **VS/DR 通过改写请求报文的 MAC 地址，将请求发送到真实服务器，而真实服务器将响应直接返回给客户**。同 VS/TUN 技术一样，VS/DR 技术可极大地 提高集群系统的伸缩性。这种方法没有 IP 隧道的开销，对集群中的真实服务器也没有必须支持 IP 隧道协议的要求，但是**要求调度器与真实服务器都有一块网卡连 在同一物理网段上**。

| 模式与特点 | NAT 模式         | IPIP 模式                                 | DR 模式                                                      |                                                              
| :--------------- | :---------------------------------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| 对服务器的要求   | 服务节点可以使任何操作系统                | 必须支持 IP 隧道，目前只有 Linux 系统支持                    | 服务器节点支持虚拟网卡设备，能够禁用设备的 ARP 响应          |
| 网络要求         | 拥有私有 IP 地址的局域网络                | 拥有合法 IP 地址的局域，网或广域网                           | 拥有合法 IP 地址的局域，服务器节点与负载均衡器必须在同一个网段 |
| 通常支持节点数量 | 10 到 20 个，根据负载均衡器的处理能力而定 | 较高，可以支持 100 个服务节点                                | 较高，可以支持 100 个服务节点                                |
| 网关             | 负载均衡器为服务器节点网关                | 服务器的节点同自己的网关或者路由器连接，不经过负载均衡器     | 服务节点同自己的网关或者路由器连接，不经过负载均衡器         |
| 服务节点安全性   | 较好，采用内部 IP，服务节点隐蔽           | 较差，采用公用 IP 地址，节点安全暴露                         | 较差，采用公用 IP 地址，节点安全暴露                         |
| IP 要求          | 仅需要一个合法的 IP 地址作为 VIP 地址     | 除了 VIPO 地址外，每个服务器界定啊需要拥有合法的 IP 地址，可以直接从路由到客户端 | 除了 VIP 外，每个服务节点需拥有合法的 IP 地址，可以直接从路由到客户端 |
| 特点             | 地址转换                                  | 封装 IP                                                      | 修改 MAC 地址                                                |
| 配置复杂度       | 简单                                      | 复杂                                                         | 复杂                                                         |

## 相关术语

- LB (Load Balancer 负载均衡)
- HA (High Available 高可用)
- Failover (失败切换)
- Cluster (集群)
- LVS (Linux Virtual Server Linux 虚拟服务器)
- DS (Director Server)，指的是前端负载均衡器节点
- RS (Real Server)，后端真实的工作服务器
- VIP (Virtual IP)，虚拟的 IP 地址，向外部直接面向用户请求，作为用户请求的目标的 IP 地址
- DIP (Director IP)，主要用于和内部主机通讯的 IP 地址
- RIP (Real Server IP)，后端服务器的 IP 地址
- CIP (Client IP)，访问客户端的 IP 地址



# LVS是什么

1. 首先它是基于 4 层的网络协议的，抗负载能力强，对于服务器的硬件要求除了网卡外，其他没有太多要求；
2. 配置性比较低，这是一个缺点也是一个优点，因为没有可太多配置的东西，大大减少了人为出错的几率；
3. 应用范围比较广，不仅仅对 web 服务做负载均衡，还可以对其他应用（mysql）做负载均衡；
4. LVS 架构中存在一个虚拟 IP 的概念，需要向 IDC 多申请一个 IP 来做虚拟 IP。

LVS 是一个开源的软件，可以实现 LINUX 平台下的简单负载均衡。LVS 是 [Linux Virtual Server](http://www.linuxvirtualserver.org/) 的缩写，意思是 Linux 虚拟服务器。

LB 集群的架构和原理很简单，就是当用户的请求过来时，会直接分发到 Director Server 上，然后它把用户的请求根据设置好的调度算法，智能均衡地分发到后端真正服务器 (real server) 上。为了避免不同机器上用户请求得到的数据不一样，需要用到了共享存储，这样保证所有用户请求的数据是一样的。

LVS 已经是 Linux 内核标准的一部分。使用 LVS 可以达到的技术目标是：通过 LVS 达到的负载均衡技术和 Linux 操作系统实现一个高性能高可用的 Linux 服务器集群，它具有良好的可靠性、可扩展性和可操作性。从而以低廉的成本实现最优的性能。LVS 是一个实现负载均衡集群的开源软件项目，**LVS 架构从逻辑上可分为调度层、Server 集群层和共享存储。**



# LVS 原理

![](https://cdn.jsdelivr.net/gh/631068264/img/008i3skNgy1gx94hi9k98j30o70brdg7.jpg)

- 当用户向负载均衡调度器（Director Server）发起请求，调度器将请求发往至内核空间
- PREROUTING 链首先会接收到用户请求，判断**目标 IP 确定是本机 IP，将数据包发往 INPUT 链**
- IPVS 是工作在 INPUT 链上的，当用户请求到达 INPUT 时，IPVS 会将用户请求和自己已定义好的集群服务进行比对，如果**用户请求的就是定义的集群服务，那么此时 IPVS 会强行修改数据包里的目标 IP 地址及端口，并将新的数据包发往 POSTROUTING 链**
- POSTROUTING 链接收数据包后**发现目标 IP 地址刚好是自己的后端服务器，那么此时通过选路，将数据包最终发送给后端的服务器**

# 工作模式 IP 负载均衡

LVS 是四层（传输层）负载均衡之上，传输层上有我们熟悉的 TCP/UDP，LVS 支持 TCP/UDP 的负载均衡。相对于其它高层负载均衡的解决办法，比如 DNS 域名轮流解析、应用层负载的调度、客户端的调度等，它的效率是非常高的。

LVS 的 **IP 负载均衡技术**是通过 IPVS 模块来实现的。

IPVS 是 LVS 集群系统的核心软件，它的主要作用是：

- 安装在 Director Server 上，同时在 Director Server 上虚拟出一个 IP 地址，用户必须通过这个虚拟的 IP 地址访问服务。这个虚拟 IP 一般称为 LVS 的 VIP，即 Virtual IP。
- 访问的请求首先经过 VIP 到达负载调度器，然后由负载调度器从 Real Server 列表中选取一个服务节点响应用户的请求。 
- 当用户的请求到达负载调度器后，**调度器如何将请求发送到提供服务的 Real Server 节点，而 Real Server 节点如何返回数据给用户，是 IPVS 实现的重点技术**，IPVS 实现负载均衡机制有几种，分别是 NAT、DR、TUN 及 FULLNAT。

## LVS/NAT

![](https://cdn.jsdelivr.net/gh/631068264/img/008i3skNgy1gx95ex3rgdj30or0dsgma.jpg)

> NAT 方式的实现原理和数据包的改变。

- 当用户请求到达 Director Server，此时请求的数据报文会先到内核空间的 PREROUTING 链。 此时报文的源 IP 为 CIP，目标 IP 为 VIP
- PREROUTING 检查发现数据包的目标 IP 是本机，将数据包送至 INPUT 链
- IPVS 比对数据包请求的服务是否为集群服务，若是，修改数据包的目标 IP 地址为后端服务器 IP，然后将数据包发至 POSTROUTING 链。 此时报文的源 IP 为 CIP，目标 IP 为 RIP
- POSTROUTING 链通过选路，将数据包发送给 Real Server
- Real Server 比对发现目标为自己的 IP，开始构建响应报文发回给 Director Server。 此时报文的源 IP 为 RIP，目标 IP 为 CIP
- Director Server 在响应客户端前，此时会将源 IP 地址修改为自己的 VIP 地址，然后响应给客户端。 此时报文的源 IP 为 VIP，目标 IP 为 CIP

**特性**

NAT（Network Address Translation 网络地址转换）是一种外网和内外地址映射的技术，内网可以是私有网址，外网可以使用 NAT 方法修改数据报头，让外网与内网能够互相通信。

- RS 应该使用私有地址，RS 的网关必须指向 DIP
- DIP 和 RIP 必须在同一个网段内，且应该使用私网地址
- 请求和响应报文都需要经过 Director Server，高负载场景中，Director Server 易成为性能瓶颈
- 支持端口映射，可修改请求报文的目标 PORT
- RS 可以使用任意操作系统

**缺陷：**
所有输入输出的流量都要经过 LVS 调度服务器。显然，LVS 调度服务器的网络 I/O 压力将会非常大，因此很容易成为瓶颈

**优点：**

NAT 模式的优点在于配置及管理简单，由于了使用 NAT 技术，LVS 调度器及应用服务器可以在不同网段中，网络架构更灵活，应用服务器只需要进行简单的网络设定即可加入集群

## LVS/DR

![](https://cdn.jsdelivr.net/gh/631068264/img/008i3skNgy1gx9nm66uhgj30p10e0t9r.jpg)

> 重点将请求报文的目标 MAC 地址设定为挑选出的 RS 的 MAC 地址

- 当用户请求到达 Director Server，此时请求的数据报文会先到内核空间的 PREROUTING 链。 此时报文的源 IP 为 CIP，目标 IP 为 VIP
- PREROUTING 检查发现数据包的目标 IP 是本机，将数据包送至 INPUT 链
- IPVS 比对数据包请求的服务是否为集群服务，若是，将请求报文中的源 MAC 地址修改为 DIP 的 MAC 地址，将目标 MAC 地址修改 RIP 的 MAC 地址，然后将数据包发至 POSTROUTING 链。 此时的源 IP 和目的 IP 均未修改，仅修改了源 MAC 地址为 DIP 的 MAC 地址，目标 MAC 地址为 RIP 的 MAC 地址
- 由于 DS 和 RS 在同一个网络中，所以是通过二层来传输。POSTROUTING 链检查目标 MAC 地址为 RIP 的 MAC 地址，那么此时数据包将会发至 Real Server。
- RS 发现请求报文的 MAC 地址是自己的 MAC 地址，就接收此报文。处理完成之后，将响应报文通过 lo 接口传送给 eth0 网卡然后向外发出。 此时的源 IP 地址为 VIP，目标 IP 为 CIP
- 响应报文最终送达至客户端

**特性**

DR（Direct Routing 直接路由模式）**此模式时 LVS 调度器只接收客户发来的请求并将请求转发给后端服务器，后端服务器处理请求后直接把内容直接响应给客户，而不用再次经过 LVS 调度器。**LVS 只需要将网络帧的 MAC 地址修改为某一台后端服务器 RS 的 MAC，该包就会被转发到相应的 RS 处理，注意此时的源 IP 和目标 IP 都没变。RS 收到 LVS 转发来的包时，链路层发现 MAC 是自己的，到上面的网络层，发现 IP 也是自己的，于是这个包被合法地接受，RS 感知不到前面有 LVS 的存在。而当 RS 返回响应时，只要直接向源 IP（即用户的 IP）返回即可，不再经过 LVS。

- 保证前端路由将目标地址为 VIP 报文统统发给 Director Server，而不是 RS
- RS 可以使用私有地址；也可以是公网地址，如果使用公网地址，此时可以通过互联网对 RIP 进行直接访问
- RS 跟 Director Server 必须在同一个物理网络中
- 所有的请求报文经由 Director Server，但响应报文必须不能进过 Director Server
- 不支持地址转换，也不支持端口映射
- RS 可以是大多数常见的操作系统
- RS 的网关绝不允许指向 DIP(因为我们不允许他经过 director)
- RS 上的 lo 接口配置 VIP 的 IP 地址

**缺点**：唯一的缺陷在于它要求 LVS 调度器及所有应用服务器在同一个网段中，因此不能实现集群的跨网段应用。

**优点**：可见在处理过程中 LVS Route 只处理请求的直接路由转发，所有响应结果由各个应用服务器自行处理，并对用户进行回复，网络流量将集中在 LVS 调度器之上。

## LVS/TUN

![](https://cdn.jsdelivr.net/gh/631068264/img/008i3skNgy1gx9pifs1n8j30p30e53zi.jpg)

> 在原有的 IP 报文外再次封装多一层 IP 首部，内部 IP 首部(源地址为 CIP，目标 IIP 为 VIP)，外层 IP 首部(源地址为 DIP，目标 IP 为 RIP)

- 当用户请求到达 Director Server，此时请求的数据报文会先到内核空间的 PREROUTING 链。 此时报文的源 IP 为 CIP，目标 IP 为 VIP 。
- PREROUTING 检查发现数据包的目标 IP 是本机，将数据包送至 INPUT 链
- IPVS 比对数据包请求的服务是否为集群服务，若是，**在请求报文的首部再次封装一层 IP 报文，封装源 IP 为 DIP，目标 IP 为 RIP。然后发至 POSTROUTING 链。 此时源 IP 为 DIP，目标 IP 为 RIP**
- POSTROUTING 链根据最新封装的 IP 报文，将数据包发至 RS（**因为在外层封装多了一层 IP 首部，所以可以理解为此时通过隧道传输**）。 此时源 IP 为 DIP，目标 IP 为 RIP
- RS 接收到报文后发现是自己的 IP 地址，**就将报文接收下来，拆除掉最外层的 IP 后，会发现里面还有一层 IP 首部，而且目标是自己的 lo 接口 VIP，**那么此时 RS 开始处理此请求，处理完成之后，通过 lo 接口送给 eth0 网卡，然后向外传递。 此时的源 IP 地址为 VIP，目标 IP 为 CIP
- 响应报文最终送达至客户端

**特性**

- RIP、VIP、DIP 全是公网地址
- RS 的网关不会也不可能指向 DIP
- 所有的请求报文经由 Director Server，但响应报文必须不能进过 Director Server
- 不支持端口映射
- RS 的系统必须支持隧道

> 其实企业中最常用的是 DR 实现方式，而 NAT 配置上比较简单和方便，后边实践中会总结 DR 和 NAT 具体使用配置过程。

TUN（virtual server via ip tunneling　IP 隧道）调度器把请求的报文通过 IP 隧道转发到真实的服务器。真实的服务器将响应处理后的数据直接返回给客户端。这样调度器就只处理请求入站报文。此转发方式不修改请求报文的 IP 首部（源 IP 为 CIP，目标 IP 为 VIP），而在原 IP 报文之外再封装一个 IP 首部（源 IP 是 DIP，目标 IP 是 RIP），将报文发往挑选出的目标 RS；RS 直接响应给客户端（源 IP 是 VIP，目标 IP 是 CIP），由于一般网络服务应答数据比请求报文大很多，采用 lvs-tun 模式后，集群系统的最大吞吐量可以提高 10 倍。

**缺点**: 由于后端服务器 RS 处理数据后响应发送给用户，此时需要租借大量 IP（特别是后端服务器使用较多的情况下）。

**优点**: 实现 lvs-tun 模式时，LVS 调度器将 TCP/IP 请求进行重新封装并转发给后端服务器，由目标应用服务器直接回复用户。应用服务器之间是通过 IP 隧道来进行转发，故两者可以存在于不同的网段中。

# LVS组成

LVS 由 2 部分程序组成，包括 ipvs 和 ipvsadm。

- ipvs(ip virtual server)：一段代码工作在内核空间，叫 ipvs，是真正生效实现调度的代码。
- ipvsadm：另外一段是工作在用户空间，叫 ipvsadm，负责为 ipvs 内核框架编写规则，定义谁是集群服务，而谁是后端真实的服务器(Real Server)