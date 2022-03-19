---
layout:     post
rewards: false
title:   k8s网络模型1
categories:
 - k8s
---

# 基本网络模型

评价一个容器网络或者设计容器网络的时候，它的准入条件。它需要满足哪三条？ 才能认为它是一个合格的网络方案。

- 任意两个 pod 之间其实是可以直接通信的，无需经过显式地使用 NAT 来接收数据和地址的转换；
- node 与 pod 之间是可以直接通信的，无需使用明显的地址转换；
- pod 看到自己的 IP 跟别人看见它所用的IP是一样的，中间不能经过转换。

设计一个 K8s 的系统为外部世界提供服务的时候，从网络的角度要想清楚，外部世界如何一步一步连接到容器内部的应用

- **外部世界和 service 之间是怎么通信的？**就是有一个互联网或者是公司外部的一个用户，怎么用到 service？service 特指 K8s 里面的服务概念。
- **service 如何与它后端的 pod 通讯？**
- **pod 和 pod 之间调用是怎么做到通信的？**
- **容器与容器之间的通信？**

# 基本约束的解释

因为容器的网络发展复杂性就在于它其实是寄生在 Host 网络之上的。从这个角度讲，可以把容器网络方案大体分为 **Underlay/Overlay** 两大派别：

- Underlay 的标准是它与 Host 网络是同层的，从外在可见的一个特征就是它是不是使用了 Host 网络同样的网段、输入输出基础设备、容器的 IP 地址是不是需要与 Host 网络取得协同（来自同一个中心分配或统一划分）。这就是 Underlay；
- Overlay 不一样的地方就在于它并不需要从 Host 网络的 IPM 的管理的组件去申请IP，一般来说，它只需要跟 Host 网络不冲突，这个 IP 可以自由分配的。

![image-20211225204954013](https://tva1.sinaimg.cn/large/008i3skNgy1gxqdcbqhnej310o0cw74w.jpg)

# Network namespace

**Network Namespace 里面能网络实现的内核基础**。狭义上来说 runC 容器技术是不依赖于任何硬件的，**它的执行基础就是它的内核里面，进程的内核代表就是 task，它如果不需要隔离，那么用的是主机的空间（ namespace）**，并不需要特别设置的空间隔离数据结构（ nsproxy-namespace proxy）。

![image-20211225205332165](https://tva1.sinaimg.cn/large/008i3skNgy1gxqdg2rz9aj316s0rmq6d.jpg)

从感官上来看一个隔离的网络空间，它会拥有自己的网卡或者说是网络设备。网卡可能是虚拟的，也可能是物理网卡，它会拥有自己的 IP 地址、IP 表和路由表、拥有自己的协议栈状态。这里面特指就是 TCP/Ip协议栈，它会有自己的status，会有自己的 iptables、ipvs。**这就相当于拥有了一个完全独立的网络，它与主机网络是隔离的。当然协议栈的代码还是公用的，只是数据结构不相同。**

![image-20211225210241163](https://tva1.sinaimg.cn/large/008i3skNgy1gxqdpl06iuj31ak0pg419.jpg)

这张图可以清晰表明 pod 里 Netns 的关系，每个 pod 都有着独立的网络空间，pod net container 会共享这个网络空间。一般 K8s 会推荐选用 Loopback 接口，在 pod net container 之间进行通信，而所有的 container 通过 pod 的 IP 对外提供服务。另外对于宿主机上的 Root Netns，可以把它看做一个特殊的网络空间，只不过它的 Pid 是1。

# 网络解决方案

## 典型的容器网络实现方案

容器网络方案可能是 K8s 里最为百花齐放的一个领域，它有着各种各样的实现。容器网络的复杂性，其实在于它需要跟底层 Iass 层的网络做协调、需要在性能跟 IP 分配的灵活性上做一些选择，这个方案是多种多样的。

![image-20211225211107681](https://tva1.sinaimg.cn/large/008i3skNgy1gxqdydfaocj31dc0nsjw6.jpg)

- **Flannel** 是一个比较大一统的方案，它提供了多种的网络 backend。不同的 backend 实现了不同的拓扑，它可以覆盖多种场景；
- **Calico** 主要是采用了策略路由，节点之间采用 BGP 的协议，去进行路由的同步。它的特点是功能比较丰富，尤其是对 Network Point 支持比较好，大家都知道 Calico 对底层网络的要求，一般是需要 mac 地址能够直通，不能跨二层域；
- 当然也有一些社区的同学会把 Flannel 的优点和 Calico 的优点做一些集成。我们称之为嫁接型的创新项目 **Cilium**；
- 最后讲一下 **WeaveNet**，如果大家在使用中需要对数据做一些加密，可以选择用 WeaveNet，它的动态方案可以实现比较好的加密。

## Flannel

![image-20211225212155431](https://tva1.sinaimg.cn/large/008i3skNgy1gxqe9m0xj0j31aw0regow.jpg)

Flannel 方案是目前使用最为普遍的。如上图所示，可以看到一个典型的容器网方案。它首先要解决的是 container 的包如何到达 Host，这里采用的是加一个 Bridge 的方式。它的 backend 其实是独立的，也就是说这个包如何离开 Host，是采用哪种封装方式，还是不需要封装，都是可选择的。 

现在来介绍三种主要的 backend：

- 一种是用户态的 udp，这种是最早期的实现；
- 然后是内核的 Vxlan，这两种都算是 overlay 的方案。Vxlan 的性能会比较好一点，但是它对内核的版本是有要求的，需要内核支持 Vxlan 的特性功能；
- 如果你的集群规模不够大，又处于同一个二层域，也可以选择采用 host-gw 的方式。这种方式的 backend 基本上是由一段广播路由规则来启动的，性能比较高。

 

# Network Policy 基本概念

下面介绍一下 Network Policy 的概念。

 

![img](https://tva1.sinaimg.cn/large/008i3skNgy1gxqeww1hl9j31ab0b00uw.jpg)

 

刚才提到了 Kubernetes 网络的基本模型是需要 pod 之间全互联，这个将带来一些问题：可能在一个 K8s 集群里，有一些调用链之间是不会直接调用的。比如说两个部门之间，那么我希望 A 部门不要去探视到 B 部门的服务，这个时候就可以用到策略的概念。

基本上它的想法是这样的：**它采用各种选择器（标签或 namespace），找到一组 pod，或者找到相当于通讯的两端，然后通过流的特征描述来决定它们之间是不是可以联通，可以理解为一个白名单的机制。**

在使用 Network Policy 之前，如上图所示要注意 apiserver 需要打开一下这几个开关。另一个更重要的是我们选用的网络插件需要支持 Network Policy 的落地。大家要知道，Network Policy 只是 K8s 提供的一种对象，并没有内置组件做落地实施，需要取决于你选择的容器网络方案对这个标准的支持与否及完备程度，如果你选择 Flannel 之类，它并没有真正去落地这个 Policy，那么你试了这个也没有什么用。



## 默认情况

默认情况下，Pod 是非隔离的，它们接受任何来源的流量。

**Pod 在被某 NetworkPolicy 选中时进入被隔离状态。** 一旦名字空间中有 NetworkPolicy 选择了特定的 Pod，该 Pod 会拒绝该 NetworkPolicy 所不允许的连接。 （名字空间下其他未被 NetworkPolicy 所选择的 Pod 会继续接受所有的流量）

**网络策略**不会冲突，它们**是累积的**。 如果任何一个或多个策略选择了一个 Pod, 则该 Pod 受限于**这些策略的 入站（Ingress）/出站（Egress）规则的并集。**

**podSelector**：每个 NetworkPolicy 都包括一个 `podSelector`，它对该策略所 适用的一组 Pod 进行选择。示例中的策略选择带有 "role=db" 标签的 Pod。 空的 `podSelector` 选择名字空间下的所有 Pod。

```yaml
# 创建networkPolicy，针对namespace internal下的pod，只允许同样namespace下的pod访问，并且可访问pod的9000端口。

# 不允许不是来自这个namespace的pod访问。

# 不允许不是监听9000端口的pod访问。


apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: all-port-from-namespace
  namespace: internal
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
    ports:
    - port: 9000
```



![image-20211225215619028](https://tva1.sinaimg.cn/large/008i3skNgy1gxqf9e0l73j316w0s0jur.jpg)

- 第一件事是控制对象，就像这个实例里面 spec 的部分。spec 里面通过 podSelector 或者 namespace 的 selector，可以选择做特定的一组 pod 来接受我们的控制；
- 第二个就是对流向考虑清楚，需要控制入方向还是出方向？还是两个方向都要控制？
- 最重要的就是第三部分，如果要对选择出来的方向加上控制对象来对它流进行描述，具体哪一些 stream 可以放进来，或者放出去？类比这个流特征的五元组，可以通过一些选择器来决定哪一些可以作为我的远端，这是对象的选择；也可以通过 IPBlock 这种机制来得到对哪些 IP 是可以放行的；最后就是哪些协议或哪些端口。其实流特征综合起来就是一个五元组，会把特定的能够接受的流选择出来 。

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
  # 每个 NetworkPolicy 都包括一个 podSelector，它对该策略所 适用的一组 Pod 进行选择。示例中的策略选择带有 "role=db" 标签的 Pod。 空的 podSelector 选择名字空间下的所有 Pod。
    matchLabels:
      role: db
  policyTypes: # 每个 NetworkPolicy 都包含一个 policyTypes 列表，其中包含 Ingress 或 Egress 或两者兼具。policyTypes 字段表示给定的策略是应用于 进入所选 Pod 的入站流量还是来自所选 Pod 的出站流量，或两者兼有。 如果 NetworkPolicy 未指定 policyTypes 则默认情况下始终设置 Ingress； 如果 NetworkPolicy 有任何出口规则的话则设置 Egress。
  - Ingress
  - Egress
  ingress: # 每个 NetworkPolicy 可包含一个 ingress 规则的白名单列表。 每个规则都允许同时匹配 from 和 ports 部分的流量。示例策略中包含一条 简单的规则： 它匹配某个特定端口，来自三个来源中的一个，第一个通过 ipBlock 指定，第二个通过 namespaceSelector 指定，第三个通过 podSelector 指定。
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress: #每个 NetworkPolicy 可包含一个 egress 规则的白名单列表。 每个规则都允许匹配 to 和 port 部分的流量。该示例策略包含一条规则， 该规则将指定端口上的流量匹配到 10.0.0.0/24 中的任何目的地。
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```



# 总结

- 在 pod 的容器网络中核心概念就是 IP，IP 就是每个 pod 对外通讯的地址基础，必须内外一致，符合 K8s 的模型特征；

- 那么在介绍网络方案的时候，**影响容器网络性能最关键的就是拓扑**。要能够理解你的包端到端是怎么联通的，中间怎么从 container 到达 Host，Host 出了 container 是要封装还是解封装？还是通过策略路由？最终到达对端是怎么解出来的？

- 容器网络选择和设计选择。如果你并不清楚你的外部网络，或者你需要一个**普适性最强的方案**，假设说你对 mac 是否直连不太清楚、对外部路由器的路由表能否控制也不太清楚，那么你可以选择 **Flannel 利用 Vxlan 作为 backend 的这种方案**。如果你确信你的网络是 2 层可直连的，你可以进行选用 Calico 或者 Flannel-Hostgw 作为一个 backend；

- 最后就是对 Network Policy，在运维和使用的时候，它是一个很强大的工具，可以实现对进出流的精确控制。实现的方法我们也介绍了，要想清楚你要控制谁，然后你的流要怎么去定义。