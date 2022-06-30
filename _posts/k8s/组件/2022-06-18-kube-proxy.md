---
ewards: false
title:      Kube Proxy 原理
categories:
    - k8s


---

Kubernetes 抽象出了 Service 的概念 。 Service 是对一组 Pod 的抽象，它会根据访问策略(如负载均衡策略)来访问这组 Pod。



Kubernetes 在创建服务时会为服务分配一个虚拟 IP 地址，客户端通过访问这个虚拟 IP 地址来访问服务，服务则负责将请求转发到后端的 Pod 上 。这其实就是一个反 向代理， 但与普通的反向代理有一些不同:它的 IP地址是虚拟，若想从外面访问，则还需要一些 技巧;它的部署和启停是由 Kubernetes 统一自动管理的。



# 初始Proxy

起初， kube-proxy 进程是 一个 直实的 TCP/UDP 代理，类似 HA Proxy, 负责转发从 Service 到 Pod 的访问流摆，这被称为 userspace (用户空间代理)模式 。 当某个客户端 Pod 以 ClusterIP 地址访问某个 Service 时，这个流 立就 被 Pod 所在 Node 的 iptables 转发给 kube-proxy 进程，然后由 kube-proxy 建立起到后端 Pod 的 TCP/UDP 连接， 再将请求转发到某个后端 Pod 上，并在这个过程中实现负载均衡功能。

![image-20220618135935445](https://tva1.sinaimg.cn/large/e6c9d24egy1h3ccxc12i8j213u0tkwh5.jpg)

此外， **Service 的 ClusterIP 与 NodePort 等概念是 kube-proxy 服务通过 iptables 的 NAT 转换实现的， kube-proxy在运行过程中动态创建与 Service相关的 iptables规则，这些规则 实现了将访问服务 (ClusterIP或 NodePort) 的请求负载分发到后端 Pod 的功能。**由于 iptables 机制针对的是本地的 kube-proxy 端口，所以在每个 Node 上都要运行 kube-proxy 组件，这 样一来，在 Kubernetes集群内部，我们可以在任意 Node上发起对 Service的访问请求。 

# 二代Proxy

从 1.2 版本开始， Kubernetes 将 iptables 作为 kube-proxy 的默认模式， iptables 模式下 的第二代 kube-proxy 进程不再起到数据层面的 Proxy 的作用， Client 向 Service 的请 求**流量通过 iptables 的 NAT 机制 直接发送到目标 Pod**, 不 经过kube-proxy 进程的转发， kube-proxy 进程只承担了控制层面的功能，即**通过 API Server 的 Watch 接口实时跟踪 Service 与 Endpoint 的变更信息，并更新 Node 节点上相应的 iptables 规则 。**

一个 Node 上的 Pod 与 其他 Node 上的 Pod 应该能够直 接建立双向的 TCP/IP 通信通道，所以如果直接修改 iptables 规则，则也可以实现 kube-proxy 的功能，只不过后者更加高端，因为是全自动模式的 。 **与第一代的 userspace 模式相比， iptables 模式完全工作在内核态，不用再经过用户态的 kube-proxy 中转，因而性能更强 。**



# 三代Proxy

在集 群中的 Service 和 Pod 大址增加以后，每个 Node 节点上 iptables 中的规则会急速膨胀，导 致网络性能显著下降，在某些极端清况下甚至会出现规则丢失的情况，并且这种故席难以重现与排查。千是Kubernetes从 1.8版本开始引入第三代的IPVS(IPVirtualServer)模式，

**iptables 与 IPVS 虽然都是基于 Netfilter 实现的，但因为定 位不 同，二者有 着本质的差 别: iptables 是为防火墙设计的; IPVS 专门用于高性能负载均衡，并使用更高效的数据结 构 I: 哈希表)，允许几乎无限的规模扩张，因此被 kube-proxy 采纳为第三代模式 。**

与 iptables 相比 ， IPVS 拥有以下明显优势:

- 为大型集群提供了更好的可扩展性和性能
- 支待比iptables更复杂的复制均衡算法
- 支持服务器健康检查和连接重试等功能;
- 可以动态修改 ipset 的集合，即使 iptables 的规则正在使用这个集合 

由千 IPVS 无法提供包过滤、 airpin-masquerade tricks (地址伪装) 、 SNAT 等功能 ，因 此在某些场景(如 NodePort的实现)下还要与 iptables搭配使用 。**在 IPVS模式下， kube-proxy 又攸了重要的 升级，即使用 iptables 的扩展 ipset, 而不是直接调用 iptables 来生成规则链。**

**iptables 规则链是一个线性数据结构， ipset 则引入了带索引的数据结构，因此当规则 很多时，也可以高效地查找和匹配 。 我们可以将 ipset 简单理解为一个 IP (段)的集合， 这个集合的内容可以是 IP 地址、 IP 网段 、 端口等， iptables 可以直接添加规则对这个“可 变的集合”进行操作，这样做的好处在于大大减少了 iptables 规则的数量，从而减少了性 能损耗 。**

 假设要禁止上万个 IP 访问我们的服务器，则用 iptables 的话，就需要一条一条地 添加规则，会在 iptables 中生成大批的规则;但是用 ipset 的话，只需将相关的 IP 地址(网 段)加入 ipset 集合中即可，这样只需设置少盈的 iptables 规则即可 实现目标。
