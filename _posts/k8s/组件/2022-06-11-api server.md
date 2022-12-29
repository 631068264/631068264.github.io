---
layout:     post
rewards: false
title:      Kubernetes API Server概述
categories:
    - k8s

---

# Kubernetes API Server 原理解析

Kubernetes API Server 的核心功能是提供 Kubernetes 各类资源对象(如 Pod、 RC、 Service 等)的增 、 删 、 改、查及 Watch 等 HTTP REST 接口， 成为集群内各个功能模 块之间数据交互和通信的中心枢纽，是整个系统的数据总线和数据中心 。 除此之外，它还 是集群管理的 API 入口，是资源配额控制的入口，提供了完备的集群安全机制 。

# 概述

Kubernetes API Server 通过一个名为 kube-apiserver 的进程 提供服务 ， 该进程运行在 Master上。在默认情况下，kube-apiserver进程在本机的 8080端口(对应参数--insecure-port) 提供 REST 服务 。 我们可以同时启动 HTTPS 安全端口 (--secure-port=6443) 来启动安全机 制，加强 REST API 访问的安全性 。

我们通常通过命令行工具 kubectl 与 Kubernetes API Server 交互，它们之间的接口是 RESTful API。 为了测试和学习 Kubernetes API Server 提供的接口，我 们也可以 使用 curl 命令行工具进行快速验证 。



由千 API Server 是 Kubernetes 集群数据的唯一访问入口，因此安全性与高性能成为 API Server设计 和实现的两 大核心目标。通过采用 HTTPS 安全传输通道与 CA 签名数字证 书强制双 向认证的方式， API Server 的安 全性得 以保障 。 此外 ，为了更 细粒度地控制用 户 或应用对 Kubernetes 资 源对象的访问权限， Kubernetes 启用了 RBAC 访问 控制策略，之后 会深入讲解这一安 全策略 。



**API Server 的性能是决定 Kubemetes集群整体性能的关键因素，因此 Kubemetes 的设计者综合运用以下方式来最大程度地保证 API Server 的性能 。**

- APIServer拥有大量高性能的底层代码。在APIServer源码中**使用协程 (Coroutine)+ 队列 (Queue) 这种轻量级的高性能并发代码**，使得单进程的 API Server 具备超强的多核 处理能力，从而以很快的速度并发处理大盘的请求 。
- 普通 List接口结合异步 Watch接口，不但完美解决了 Kubemetes 中各种资源对象 的高性能同步问题，也极大提升了 Kubemetes 集群实时响应各种事件的灵敏度 。
- 采用了高性能的 etcd 数据库而非传统的关系数据库，不仅解决了数据的可靠性问 题，也极大提升了 API Server 数据访问层的性能 。 在常见的公有云环境中，一个 3 节点的 etcd 集群在轻负载环境中处理一个请求的时间可以少于 lms, 在重负载环境中可以每秒处 理超过 30000个请求。

目前 Kubernetes 1.19 版本的集群可支持的最大规模如下:

- 最多支持 5000 个 Node;
-  最多支待 150000 个 Pod;
- 每个Node最多支持100个Pod; 0 最多支持300000个容器。

# API Server 架构解析

- API 层:主要以 REST 方式提供各种 API 接口，除了有 Kubernetes 资源对象 的 CRUD 和 Watch 等主要 API, 还有健康检查、 UI、日志、性能指标等运维监控相关的 API。 Kubernetes 从 1.11 版本开始废弃 Heapster 监控组件，转而使用 Metrics Server 提供 Metrics API 接口，进一步完善了自身的监控能力 。
- 访问控制层 :当客户端访问 API 接口时，访问控制层负责对用 户身份鉴权，验明 用户身份，核准用户对 Kubernetes 资源对象的访问权限，然后根据配置的各种资源访问许 可逻辑 (Admission Control)，判断是否允许访问 。
- 注册表层 : Kubernetes 把所有资源对象都保存在注册表 (Registry) 中，针对注册 表中的各种资源对象都定义了资源对象的类型、如何创建资源对象、如何转换资源的不同版本，以及如何将资源编码和解码为 JSON 或 ProtoBuf格式进行存储 。
- etcd 数据库:用千持久化存储 Kubernetes 资源对象的 KV 数据库 。 etcd 的 Watch API 接口对千 API Server 来说至关重要，因为通过这个接口， API Server 创新性地设计了 List-Watch 这种高性能的 资源对象实时同步机制，使 Kubernetes 可以 管理超大规模的集群， 及时响应和快速处理集群中的各种事件 。

![image-20220611210434101](https://cdn.jsdelivr.net/gh/631068264/img/e6c9d24egy1h34lvdr22aj20u00u90w9.jpg)

# API Server 中资源对象的 List-Watch 机制 

![image-20220611210827131](https://cdn.jsdelivr.net/gh/631068264/img/e6c9d24egy1h34lzd5urqj21ga0u0wjy.jpg)

首先，借助etcd提供的WatchAPI接口， **APIServer可以监听 (Watch) 在etcd上发 生的数据操作事件**，比如 Pod 创建事件、更新事件 、 删除事件等，在这些事件发生后， etcd 会及时通知 API Server。 图 5.3 中 API Server 与 etcd 之间的交互 箭头表明了这个过程:当 一个 ReplicaSet对象被创建并保存到 etcd 中后(图中的 2. Create RepliatSet箭头)， etcd会 立 即发送一个对应的 Create 事件给 API Server (图中的 3. Send RepliatSet Create Event 箭 头 )， 与其类似的 6、 7、 10、 11 箭头都针对 Pod 的创建 、更新事件 。

然后 ，为了 让 Kubernetes 中的其他组件在不访问底层 etcd 数据库的情况下，也能及时 获取资源对象的变化事件 ， API Server模仿 etcd 的 WatchAPI 接口提供了自己的 Watch 接 口，这样一来，这些组件就能近乎实时地获取自己感兴趣的任意资源对象的相关事件通知 了 。 图 5.3 中的 controller-manager、 scheduler、 kubelet 等组件与 API Server 之间的 3 个标记为 “List-Watch" 的虚线框表明了这个过程 。 同时，在监听自己感 兴趣 的资源时，客户 端可以增加过滤条件，以 List-Watch 3 为例， nodel 节点上的 kubelet 进程只对 自己节点上 的 Pod 事件感兴趣 。

最后， Kubernetes List-Watch 用于实现数据同步的代码逻 辑 。 客户端首先调用 API Server 的 List 接口获取相关资源对象的全量数据并将其缓存到内存中，然后启动对应资源 对象的 Watch 协程，在接收到 Watch 事件后，再根据事件的类型(比如新增、修改或删除) 对内存中的全量资源对象列表做出相应的同步修改。 从实现上来看，这是一种全益结合增 最的、高 性能的 、 近乎实时的数据同步方式 。

# Kubernetes 中的 CRD 在 API Server 中的设计和实现机制

- 资源对象的元数据 (Schema ) 的定义 :可以将其理解为数据 库 Table 的定义， 定 义了对应资源对象的数据结构，官方内建资源对象的元数据定义是固化在源码中的 。
- 资源对象的校验逻辑:确保用户提交的资源对象的属性的合法性 。
- 资源对象的 CRUD 操作代码:可以将其理解为数据库 表的 CRUD 代码，但比后 者更难，因为 API Server 对资源对象的 CRUD 操作都会保存到 etcd 数据库中，对处理性 能的要求也更高，还要考虑版本兼容性和版本转换等复杂问题 。
- 资源对象相关的“自动控制器”(如 RC 、 Deployment 等资源对象背后的控制器): 这是很重要的一个功能 。 Kubernetes 是 一 个以自动化为核心目标的平台，用户给出期望的 资源对象声明，运行过程中由资源背后的“自动控制楛“确保对应资源对象的数量、状态 、 行为等始终符合用户的预期。

# 集群功能模块之间的通信

Kubernetes API Server 作为集群的核心， 负责集群各功能模块之间 的通信 。 集群内的各个功能模块通过 API Server 将信息存入 etcd 中 ，当需要获取和操作这 些数据时，则通过APIServer提供的REST接口(用GET、 LIST或WATCH方法)来实 现，从而实现各模块之间的信息交互 。

kubelet 进程与 API Server 的交互 。

- 每个 Node 上的 kubelet 每 隔一个时间周期就会调用一次 API Server 的 REST 接口报告自 身状态， API Server 在接收 到这些信息后，会将节点状态信息更新到 etcd 中 。
- 此外， kubelet 也 通过 API Server 的 Watch 接口监听 Pod 信息，如果监听到新的 Pod 副本被调度绑定到本节点，则执行 Pod 对应的容器创建和启动逻辑 ;如 果监听到 Pod 对 象被删除，则删除本节点上相应的 Pod 容器;如果 监听到修改 Pod 的信 息， kubelet 就会相应地修改本节点的 Pod 容器 ，

kube-controller-manager 进程与 API Server 的交互 。 kube-controller­ manager中的NodeController模块通过AP! Server提供的Watch接口实时监控Node的信 息，并做相应的处理。

kube-scheduler与 APIServer的交互。 Scheduler在通 过 API Server 的 Watch 接口监听到新建 Pod 副本的信息后，会检索所有符合该 Pod 要求的 Node 列表，开始执行 Pod 调度逻辑，在调度成功后将 Pod 绑定到目标节点上 。

**为了缓解集群各模块对 API Server 的 访问压力，各功能模块都采用了缓存机制来缓存 数据** 。 各功能模块定时从 API Server 上获取指定的资源对象信息(通过 List-Watch 方法)， 然后将这些信息保存到本地缓存中，功能模块在某些情况下不直接访问 API Server, 而是 通过访问缓存数据来间接访问 API Server。

# API Server 网络隔离 

API Server Network Proxy 的 核心设计思想是将 API Server 放 翌在一个独立的网络中， 与 Node 节点的网络相互隔离，然后增加独立的 Network Proxy 进 程来解决这两个网络直 接的连通性 (Connectivity) 问题

![image-20220612171143296](https://cdn.jsdelivr.net/gh/631068264/img/e6c9d24egy1h35krcumczj219y0rywja.jpg)

具体实现方式是在 Master 节点的网络里部署 Konnectivity Server，在 Node节点的网络里部署 KonnectivityAgent, 两者之间建立起安全链接，对通信协议 可以采用标准的 HTTP 或者 gRPC, 此设计允许 Node 节点网络被划分为多个独立的分片， 这些分片都通过 Konnectivity Server/Agent建立的安全链接与 API Server实现点对点的连 通。

引入 API Server Network Proxy 机制以实现 Master 网络与 Node 网络的安全隔离的做 法 . 具有以下优势 。

- Connectivity proxy (Konnectivity Server/Agent) 可以独立扩展，不会影响到 API Server 的发展，集群管理员可以部署适合自己的各种 Connectivity proxy 的实现，具有更好 的自主性和灵活性 。
- 通过采用自定义的 Connectivity proxy, 也可以实现 VPN 网络的穿透等高级网络 代理特性，同时访问 API Server 的所有请求都可以方便地被 Connectivity proxy 记录并审 计分析，这进 一 步提升了系统的安全性 。
- 这种网络代理分离的设计将 Master 网络与 Node 网络之间的连通性问题从 API Server 中剥离出来，提升了 API Server 代码的内聚性 ，降低了 API Server 的代码复杂性， 也有利于进一步提升 API Server 的性能和稳定性 。 同时， Connectivity proxy 崩溃时也不影 响 A.PI Server 的正常运行， API Server仍然可以正常提供其主要服务能力，即资源对象的 CRUD 服务 。