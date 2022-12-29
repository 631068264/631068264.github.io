---
layout:     post
rewards: false
title:   Kubernetes endpoints分片 服务拓扑
categories:
 - k8s


---



Service的后端是一组Endpoint列表，为客户端应用提供了极大的便利。但是随着集群规模的扩大及Service数量的增加，特别是Service后端Endpoint数量的增加，kube-proxy需要维护的负载分发规则(例如iptables规则或)pvs规则)的数噩也会急剧增加，导致后续对Service后端Endpoint的添加、删除等更新操作的成本急剧上升。

举例来说，假设在Kubernetes集群中有10000个Endpoint运行在大约5000个Node上，则对单个Pod的更新将需要总计约5GB的数据传输，这不仅对集群内的网络带宽浪费巨大，而且对Master的冲击非常大，会影响Kubernetes集群的整体性能，在Deployment不断进行滚动升级操作的情况下尤为突出。

**Kubernetes从1.16版本开始引入端点分片(EndpointSlices)机制，包括一个新的EndpointSlice资源对象和一个新的EndpointSlice控制器，在1.17版本时达到Beta阶段。**

EndpointSlice通过对Endpoint进行分片管理来实现降低Master和各Node之间的网络传输数据堡及提高整体性能的目标。对于Deployment的滚动升级，可以实现仅更新部分Node上的Endpoint信息，Master与Node之间的数据传输量可以减少100倍左右，能够大大提高管理效率。

EndpointSlice根据Endpoint所在Node的拓扑信息进行分片管理

![image-20220605203821213](https://cdn.jsdelivr.net/gh/631068264/img/e6c9d24egy1h2xne8q4eqj21ls0rqq8a.jpg)

Endpoint Slices 要 实现的第 2 个目标是为基千 Node 拓扑的服务路由提供支持，这 需要 与服务拓扑 ( Service Topology ) 机制共同实现。





# 端点分片 (EndpointSlices)

![image-20220605205255691](https://cdn.jsdelivr.net/gh/631068264/img/e6c9d24egy1h2xntd1p7qj21ng0b240m.jpg)

![image-20220605205906790](https://cdn.jsdelivr.net/gh/631068264/img/e6c9d24egy1h2xnzsy0l5j21i00u00xc.jpg)

EndpointSlice 的复制功能和数据分布管理机制进行说明

![image-20220605212814486](https://cdn.jsdelivr.net/gh/631068264/img/e6c9d24egy1h2xou4epe0j21qy0qa105.jpg)

Kubernetes 控制平面对千 EndpointSlice 中数据的管理机制是尽可能填满，但不 会在多 个 EndpointSlice 数据不均衡的情况下主动执行重新平衡 (rebalance) 操作，其背后的逻辑 也很简单，步骤如下 。

![image-20220605213007173](https://cdn.jsdelivr.net/gh/631068264/img/e6c9d24egy1h2xow28gojj21b50u0tfy.jpg)

# Service Topology服务拓扑

默认情况下，发送到一个 Service 的流品会被均匀转发到每个后端 Endpoint, 但无 法根据更复杂的拓扑信息设置复杂的路由策略。服务拓扑机制的引入就是为了实现基于Node 拓扑的服务路由，允许 Service 创建者根据来源 Node 和目标 Node 的标签来定义流量路由策略 。

通过对来源 (source) Node 和目标 (destination) Node 标签的匹配，用户可以根据业 务需求对 Node 进行分组，设置有意义的指标值来标识”较近”或者”较远”的属性。例 如，对于公有云环境来说，通常有区域 (Zone 或 Region) 的划分，云平台倾向于把服务 流量限制在同 一 个区域内，这通常是因为跨区域网络流噩会收取额外的费用 。 另一个例子 是把流量路由到由 DaemonSet 管理的当前 Node 的 Pod 上 。 又如希望把流量保持在相同机 架内的 Node 上，以获得更低的网络延时 。



服务拓扑机制需要通过设置 kube-apiserver 和 kube-proxy 服务的启动参数

--feature-gates="ServiceTopology=true,EndpointSlice=true" 进行启用(需要同时启用 EndpointS!ice 功能)，然后就可以在 Service 资源对象上通过定义 topologyKeys 字段来控制 到 Service 的流益路由了 。

topologyKeys 字段设置的是一组 Node 标签列表，按顺序匹配 Node 完成流量的路由转 发，流量会被转发到标签匹配成功的 Node 上 。 如果按第 1 个标签找不到匹配的 Node, 就 尝试匹配第 2 个标签，以此类推 。 如果全部标签都没有匹配的 Node, 则请求将被拒绝， 就像 Service 没有后端 Endpoint 一样 。

将 topologyKeys 配置为“*”表示任意拓扑，它只能作为配置列表中的最后一个才有 效 。 如果完全不设置 topologyKeys 字段，或者将其值设置为空，就相当千没有启用服务拓 扑功能 。

对千需要使用服务拓扑机制的集群，管理员需要为 Node 设置相应的拓扑标签，包括 kubernetes.io/hostname、 topology.kubernetes.io/zone 和 topology.kubemetes.io/region。

然后为 Service 设置 topologyKeys 的值，就可以实现如下流呈路由策略 。