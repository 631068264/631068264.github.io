---
layout:     post
rewards: false
title:   CNI网络插件
categories:
   - k8s
tags:
  - rancher

---

# 背景

Kubernetes 网络模型定义了一个“扁平”网络，其中：

- 每个 pod 都有自己的 IP 地址。
- 任何节点上的 Pod 都可以在没有 NAT 的情况下与所有其他节点上的所有 Pod 通信。



CNI（容器网络接口）是一个标准的 API，它允许不同的网络实现插入到 Kubernetes 中。每当创建或销毁 pod 时，Kubernetes 都会调用 API。有两种类型的 CNI 插件：

- CNI 网络插件：负责向/从 Kubernetes pod 网络添加或删除 pod。这包括创建/删除每个 pod 的网络接口以及将其连接/断开与其余网络实现的连接。
- CNI IPAM 插件：负责在 Pod 创建或删除时分配和释放 IP 地址。根据插件的不同，这可能包括为每个节点分配一个或多个 IP 地址范围 (CIDR)，或者从底层公共云网络获取 IP 地址以分配给 pod。



**rke 版本： rancher/hyperkube:v1.23.6-rancher1  rke version v1.3.11**



# Calico

[Calico 3.25 配置k8s 1.23](https://docs.tigera.io/calico/3.25/getting-started/kubernetes/helm)

```sh
quay.io/tigera/operator:v1.29.0
calico/ctl:v3.25.0
calico/pod2daemon-flexvol:v3.25.0
calico/typha:v3.25.0
calico/kube-controllers:v3.25.0
calico/cni:v3.25.0
calico/node:v3.25.0
calico/csi:v3.25.0
calico/node-driver-registrar:v3.25.0
calico/apiserver:v3.25.0
```



## Overlay network

覆盖网络是一种在另一个网络之上构建的网络。在Kubernetes的上下文中，覆盖网络可以用于处理节点之间的Pod流量，覆盖在底层网络之上，该网络不知道Pod的IP地址或哪个Pod在哪个节点上运行。覆盖网络通过将底层网络不知道如何处理的网络数据包（例如使用Pod IP地址）封装在外部数据包中（例如使用节点IP地址），从而实现功能。常用的封装网络协议有VXLAN和IP-in-IP。

使用覆盖网络的主要优点是减少对底层网络的依赖。例如，您可以在几乎任何底层网络上运行VXLAN覆盖，而不需要与底层网络集成或对其进行任何更改。



- 轻微的性能影响。封装数据包的过程需要一定的CPU资源，并且由于需要添加VXLAN或IP-in-IP头来编码封装，因此会减少内部数据包的最大大小，这可能意味着需要发送更多的数据包才能传输相同数量的数据
- Pod IP地址无法在集群外路由



Calico支持两种封装方式不同：

VXLAN和IP in IP。在某些不支持 IP in IP 的环境（例如 Azure）中支持 VXLAN。VXLAN 的每个数据包开销略高，因为标头较大，但除非您运行的是网络密集型工作负载，否则您通常不会注意到这种差异。这两种封装的另一个小区别是 Calico 的 VXLAN 实现不使用 BGP，而 Calico 的 IP in IP 实现在 Calico 节点之间使用 BGP

IP in IP 支持IPV4

## Cross-subnet overlays

Calico 还支持 VXLAN 和 IP-in-IP 的“跨子网”模式。在这种模式下，在每个子网内，底层网络充当 L2 网络。在单个子网内发送的数据包未封装，因此您可以获得非覆盖网络的性能。跨子网发送的数据包被封装，就像普通的覆盖网络一样，减少了对底层网络的依赖（无需与底层网络集成或对其进行任何更改）。

## PodIP可路由性

不可路由时通过集群出口流量通过SNAT技术，入口流量通过Ingress。

实现可路由的话，不必通过SNAT和Ingress

可在集群外路由的 pod IP 地址的优点是：

- 避免出站连接的 SNAT 可能对于与现有更广泛的安全要求集成至关重要。它还可以简化操作日志的调试和理解。
- 如果您有专门的工作负载，这意味着某些 pod 需要直接访问而无需通过 Kubernetes 服务或 Kubernetes ingress，那么可路由的 pod IP 在操作上可能比使用主机网络 pod 的替代方法更简单。

可在集群外路由的 pod IP 地址的主要缺点是 pod IP 在更广泛的网络中必须是唯一的。因此，例如，如果运行多个集群，您将需要为每个集群中的 pod 使用不同的 IP 地址范围 (CIDR)。当大规模运行时，或者如果现有企业对 IP 地址空间有其他重要需求，这反过来会导致 IP 地址范围耗尽的挑战。



**BGP（边界网关协议）**是一个标准化的网络协议，用于在网络中共享路由。它是互联网的基本构建模块之一，具有出色的可扩展性特性。

Calico 内置支持 BGP。在本地部署中，这使得 Calico 可以与物理网络（通常是顶层交换机路由器）对等连接，交换路由，从而**创建一个非覆盖网络，使得 Pod IP 地址能够在更广泛的网络中路由，就像连接到网络的任何其他工作负载一样。**

## 安装calico

### rke安装calico(k8s v1.23.6, calico v3.21.1)

rke配置

```yaml
network:
  plugin: calico
```

rke 默认使用IPIP

```yaml
apiVersion: crd.projectcalico.org/v1
kind: IPPool
metadata:
 ...
spec:
  allowedUses:
  - Workload
  - Tunnel
  blockSize: 26
  cidr: 10.42.0.0/16
  ipipMode: Always
  natOutgoing: true
  nodeSelector: all()
  vxlanMode: Never
```



```sh
kubectl get pods -A |grep calico
kube-system     calico-kube-controllers-5f7cbbd955-hmxsn   1/1     Running     0          14m
kube-system     calico-node-f2vqq                          1/1     Running     0          14m
kube-system     calico-node-wb8pn                          1/1     Running     0          14m
```

#### 配置BGP

| 节点   | 角色   |
| ------ | ------ |
| xxx.57 | 全节点 |
| xx.155 | 全节点 |
| xx.53  | 全节点 |

**都是calico node**

- [birdcl命令参考](https://ost.51cto.com/posts/17511)

  ```sh
  # 查看bgp 路由
  birdcl show route
  10.0.0.0/24 via 192.168.1.2 on eth0 [kernel1 2023-03-16] * (10)
  10.0.1.0/24 via 192.168.1.3 on eth0 [bgp1 2023-03-16] (100) [AS65001i]
  
  其中：
  10.0.0.0/24：表示目标网络的地址和掩码。
  
  via 192.168.1.2：表示下一跳地址。
  
  on eth0：表示出口接口。
  
  [kernel1 2023-03-16]：表示协议来源和最后更新时间。
  
  *：表示最佳路由。
  
  (10)：表示度量值，越小越优先。
  
  [AS65001i]：表示BGP属性，包括AS号和路径类型（i表示内部）
  
  # 路由追踪
  show route for 网段/IP all
  
  # bird 配置路径
  cat /etc/calico/confd/config/bird.cfg
  ```

- [安装参考：Rancher容器集群跨主机网络模式切换](https://www.bladewan.com/2020/04/26/rancher_network/)

#### node-to-node mesh

```
https://github.com/projectcalico/calico/releases/download/v3.25.0/calicoctl-linux-amd64
```

**默认情况下calico采用node-to-node mesh方式** ，全网状结构非常适合 100 个或更少节点的中小型部署

```sh
# 在57上面运行，只能看到另外两个节点信息
calicoctl node status


Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| xx.53  | node-to-node mesh | up    | 02:00:46 | Established |
| xx.155 | node-to-node mesh | up    | 03:14:39 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.


# 查看BGP路由  在57上面的pod
kubectl exec -it pod/calico-node-zjdwd -n kube-system  sh

sh-4.4# birdcl show route
BIRD v0.3.3+birdv1.6.8 ready.
....
10.42.225.192/26   via xx.53 on eth0 [Mesh_xxx_53 03:10:21] * (100/0) [i]
10.42.221.128/26   via xx.155 on eth0 [Mesh_xxx_155 03:10:21] * (100/0) [i]
...

# 本机路由
ip route |grep bird

blackhole 10.42.139.128/26 proto bird 
10.42.221.128/26 via xxx.155 dev tunl0 proto bird onlink 
10.42.225.192/26 via xxx.53 dev tunl0 proto bird onlink 
```

默认情况节点间BGP路由

<img src="https://cdn.jsdelivr.net/gh/631068264/img/202303161442874.png" alt="image-20230316144232823" style="zoom:50%;" />

#### 配置路由反射器（Route reflector）

为了防止BGP路由环路，BGP协议规定在一个AS（自治系统）内部，IBGP路由器之间只能传一跳路由信息，所以在一个AS内部，IBGP路由器之间为了学习路由信息需要建立全互联的对等体关系，但是当一个AS规模很大的时候，这种全互联的对等体关系维护会大量消耗网络和CPU资源，所以这种情况下就需要建立**路由反射器**以减少IBGP路由器之间的对等体关系数量。



选定一个节点53作为路由反射器

```sh

kubectl drain <NODE>

# 配置Route Reflector节点  routeReflectorClusterID pod/cidr 没使用的IP
calicoctl patch node <NODE>  -p '{"spec": {"bgp": {"routeReflectorClusterID": "244.0.0.1"}}}'

kubectl label node <NODE>  route-reflector=true

cat <<EOF | calicoctl --allow-version-mismatch apply -f -
kind: BGPPeer
apiVersion: projectcalico.org/v3
metadata:
  name: peer-with-route-reflectors
spec:
  nodeSelector: all()
  peerSelector: route-reflector == 'true'
EOF

# 关闭nodeToNodeMeshEnabled
# 默认 https://docs.tigera.io/archive/v3.22/networking/bgp#change-the-default-global-as-number  asNumber: 64512
cat <<EOF | calicoctl --allow-version-mismatch apply -f -
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
 name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: false
  asNumber: 64512 # 查看ASnumber   calicoctl get node -o wide  --allow-version-mismatch
EOF



# BGP节点状态  这里用xx.53作为Route Reflector节点
calicoctl node status
Calico process is running.

IPv4 BGP status
+--------------+---------------+-------+----------+-------------+
| PEER ADDRESS |   PEER TYPE   | STATE |  SINCE   |    INFO     |
+--------------+---------------+-------+----------+-------------+
| xx.53  | node specific | up    | 06:32:50 | Established |
+--------------+---------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.


# 还原节点
kubectl uncordon <NODE>


# Route Reflector节点状态
calicoctl node status

Calico process is running.

IPv4 BGP status
+--------------+---------------+-------+----------+-------------+
| PEER ADDRESS |   PEER TYPE   | STATE |  SINCE   |    INFO     |
+--------------+---------------+-------+----------+-------------+
| xx.57  | node specific | up    | 06:32:40 | Established |
| xx.155 | node specific | up    | 06:32:40 | Established |
+--------------+---------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.


```

从对应节点所在的calico-node pod 查看
```sh
# 查看57 bgp 路由, 通往155路由来源已经改变，155的路由来自53这个BGP对等体
10.42.225.192/26   via xxx.53 on eth0 [Node_xx_53 03:36:15] * (100/0) [i]
10.42.221.128/26   via xxx.155 on eth0 [Node_xx_53 03:36:15 from xxx.53] * (100/0) [i]

# 53BGP 路由
10.42.139.128/26   via xx.57 on eth0 [Node_xx_57 03:36:15] * (100/0) [i]
10.42.221.128/26   via xx.155 on eth0 [Node_xx_155 03:36:15] * (100/0) [i]

# 155BGP路由  通往57的路由也是从53获取的
10.42.139.128/26   via xxx.57 on eth0 [Node_xxx_53 03:36:15 from xxx.53] * (100/0) [i]
10.42.225.192/26   via xxx.53 on eth0 [Node_xxx_53 03:36:15] * (100/0) [i]

# 由此看出155的BGP只连接了53
birdcl show protocols
...

Node_xxx_53 BGP      master   up     03:36:15    Established   
```

配置Route reflector后，BGP路由拓扑相当于这样

<img src="https://cdn.jsdelivr.net/gh/631068264/img/202303161454160.png" alt="image-20230316145413121" style="zoom:50%;" />



### 官网安装calico

修改rke配置

```yaml
network:
  plugin: none
```

helm 安装

```
helm repo add projectcalico https://docs.tigera.io/calico/charts

helm fetch projectcalico/calico

helm install calico tigera-operator-v3.25.0.tgz -n tigera-operator --create-namespace -f values.yaml
```



查看配置

```sh
kubectl get Installation/default -o yaml

kubectl get ippool/default-ipv4-ippool -oyaml


kubectl get pod -n kube-system |grep calico |awk '{print $1}' |xargs kubectl delete pod -n kube-system
```



values.yaml  

- BGP只能配合IPIP，IPIPCrossSubnet使用
- BGP不能和VXLAN，VXLANCrossSubnet配合

```yaml
tigeraOperator:
  registry: harbor.xxx.cn:20000/quay.io

# 基本默认值
installation:
  enabled: true
  kubernetesProvider: ""
  calicoNetwork:
    bgp: Enabled
    hostPorts: Enabled
    ipPools:
      - blockSize: 26
        cidr: 10.42.0.0/16 # cluster-cidr  通过ps -ef | grep "cluster-cidr" 获取
        disableBGPExport: false
        encapsulation: IPIP  # 必须配合bgp Enabled使用
        natOutgoing: Enabled
        nodeSelector: all()
```



```sh
kubectl get pods -A |grep calico
calico-apiserver   calico-apiserver-7ddb64fb46-584jg          1/1     Running     0          41s
calico-apiserver   calico-apiserver-7ddb64fb46-rknpb          1/1     Running     0          41s
calico-system      calico-kube-controllers-7dcb884587-gj225   1/1     Running     0          59s
calico-system      calico-node-f8wf8                          1/1     Running     0          60s
calico-system      calico-node-x2dkv                          1/1     Running     0          60s
calico-system      calico-typha-66c48ccb4-cl455               1/1     Running     0          60s
calico-system      csi-node-driver-fhnrx                      2/2     Running     0          56s
calico-system      csi-node-driver-w6st2                      2/2     Running     0          56s
```



## 遗留问题

[Can't ping another pod across the nodes via IPIPCrossSubnet](https://github.com/projectcalico/calico/issues/7462)



# Weave

```sh
weaveworks/weave-kube:2.8.1
weaveworks/weave-npc:2.8.1
```





rke配置

```yaml
network:
  plugin: weave
  weave_network_provider:
     password: 'N4xxxxWu'  # 16位
```

报错

```sh
kubectl logs -f pod/weave-net-4tw8g -n kube-system

Defaulted container "weave" out of: weave, weave-npc, weave-plugins, weave-init (init)
Network 10.42.0.0/16 overlaps with existing route 10.42.139.128/26 on host
```

- [Bird is adding a blackhole route to the service cluster CIDR that blocks access inside the cluster](https://github.com/projectcalico/calico/issues/2457)
- [Network 10.42.0.0/16 overlaps with existing route 10.42.139.128/26 on host](https://github.com/weaveworks/weave/issues/3987)

根据上面机器曾经安装过calico**会修改集群所有集群节点的路由**，所以在转换CNI插件的时候，要删除

```sh
ip route

....
blackhole 10.42.139.128/26 proto bird 
10.42.225.192/26 via xxx.53 dev tunl0 proto bird onlink 
169.254.169.254 via xxx.2 dev eth0 proto static 
....



ip -4 -o addr

1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
2: eth0    inet xxx.57/24 brd xxx.255 scope global dynamic eth0\       valid_lft 57457sec preferred_lft 57457sec
3: docker0    inet 172.18.0.1/16 brd 172.18.255.255 scope global docker0\       valid_lft forever preferred_lft forever
39: tunl0    inet 10.42.139.128/32 scope global tunl0\       valid_lft forever preferred_lft forever


# 清理残留网络配置

ip route del blackhole 10.42.139.128/26 proto bird 
ip route del 10.42.225.192/26 via xxx.53 dev tunl0 proto bird onlink

# ip addr del ip/netmask dev 接口
ip addr del 10.42.139.128/32 dev tunl0
```



# BGP相关概念

参考

- [32张图详解BGP路由协议：BGP基本概念、BGP对等体、BGP报文类型、BGP状态机等](https://cloud.tencent.com/developer/article/1922673)
- [最全BGP路由协议技术详解](https://zhuanlan.zhihu.com/p/126754314)

## 自治系统AS（Autonomous System）

**AS 是指在一个实体管辖下的拥有相同选路策略的 IP 网络**。BGP 网络中的每个 AS 都被分配一个唯一的 AS 号，用于区分不同的 AS。AS 号分为 2 字节 AS 号和 4 字节 AS 号，其中 2 字节 AS 号的范围为 1 至 65535， 4 字节 AS 号的范围为 1 至 4294967295。支持 4 字节 AS 号的设备能够与持2 字节 AS 号的设备兼容。

**默认情况下，所有 Calico 节点都使用 64512 作为自治域**

![img](https://cdn.jsdelivr.net/gh/631068264/img/202303161520472.png)

简单来说：就是你可以把一个网络中的不同的设备划分到不同的组（AS）中，或者都划分在一个组中，那么一个组中的这些设备具备相同的路由协议。

不同的AS可以运行不同的路由协议，那么不同AS的网络需要通信时，采用什么路由协议进行通信呢？答案就是BGP路由协议。

**BGP与IGP比较好处**

- 只传递路由信息，不计算路由，不会暴露AS内部的网络拓扑，并且具备丰富的路由策略
- 基于TCP的路由协议，只要能够建立TCP就能够建立BGP
- 路由更新是触发更新，不是周期性更新

## BGP协议

BGP是一种基于距离矢量的路由协议，用于实现不同AS之间的路由可达。

![img](https://cdn.jsdelivr.net/gh/631068264/img/202303161534271.png)

BGP协议的基本特点：

- BGP是一种外部网关协议，其着眼点不在于发现和计算路由，而在于控制路由的传播和选择最佳路由；
- BGP使用TCP作为其传输层协议（端口号179）,提高了协议的可靠性；
- BGP是一种距离矢量路由协议，在设计上就避免了环路的发生；
- BGP提供了丰富的路由策略，能够实现路由的灵活过滤和选择；
- BGP采用触发式增量更新，而不是周期性的更新；

## BGP对等体

BGP 报文交互中分为 Speaker 和 Peer 两种角色。

**Speaker：**发送 BGP 报文的设备（运行BGP路由协议的路由器）称为 BGP 发言者（Speaker），它接收或产生新的报文信息，并发布（Advertise）给其它 BGP Speaker。

**Peer：**相互交换报文的 Speaker 之间互称对等体（Peer）。若干相关的对等体可以构成对等体组（Peer Group）。**BGP对等体之间可以交换最优路由表**，邻居的意思是指运行BGP协议的两台路由器之间的关系

BGP对等体可以按照两个路由器是否AS相同，分为EBGP对等体（AS不同）和IBGP对等体（AS相同）

**能够建立对等体的条件:**

- 必须配置相同的AS号（对于IBGP）或相邻的AS号（对于EBGP）；
- TCP连接能够建立，通常使用TCP 179号端口；
- 必须配置相同的密码（如果启用了认证）和相同的参数（如hold time、keepalive time等）



BGP 对等体间通过以下 5 种报文进行交互，**其中 Keepalive 报文为周期性发送，其余报文为触发式发送**：

- Open 报文：用于建立 BGP 对等体连接。
- Update 报文：用于在对等体之间交换路由信息。
- Notification 报文：用于中断 BGP 连接。
- Keepalive 报文：用于保持 BGP 连接。
- Route-refresh 报文：用于在改变路由策略后请求对等体重新发送路由信息。只有支持路由刷新（Route-refresh）能力的 BGP 设备会发送和响应此报文。
