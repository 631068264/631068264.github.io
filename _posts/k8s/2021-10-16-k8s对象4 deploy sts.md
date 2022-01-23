---
layout:     post
rewards: false
title:      k8s对象 deployment statefulSet
categories:
    - k8s
---

# Deployment

Deployment 为 Pod 和 ReplicaSet 提供了一个声明式定义的方法，用来替代以前的 RC 来方便的管理应用。而定义方式分为，命令式(RS)和声明式(Deployment)两种，前者侧重于考虑如何实现，而后者侧重于定义想要什么。

应用场景

- 定义 Deployment 来创建 Pod 和 RS
    - 滚动升级和回滚应用
    - 扩容和缩容
    - 暂停和继续 Deployment
- RS 和 Deployment 的关联

pod的owner是ReplicaSet，而不是Deployment。

![image-20211016214611265](https://tva1.sinaimg.cn/large/008i3skNgy1gvhhla38dxj61860jgta702.jpg)

![image-20211212200640567](https://tva1.sinaimg.cn/large/008i3skNgy1gxbb1c8fg0j31n70u0dix.jpg)

## Deployment 控制器

![image-20211212200851043](https://tva1.sinaimg.cn/large/008i3skNgy1gxbb3lq6p1j322z0u0gp8.jpg)

首先，我们所有的控制器都是通过 Informer 中的 Event 做一些 Handler 和 Watch。这个地方 Deployment 控制器，其实是关注 Deployment 和 ReplicaSet 中的 event，收到事件后会加入到队列中。而 Deployment controller 从队列中取出来之后，**它的逻辑会判断 Check Paused，这个 Paused 其实是 Deployment 是否需要新的发布，如果 Paused 设置为 true 的话，就表示这个 Deployment 只会做一个数量上的维持，不会做新的发布。**

如上图，可以看到如果 Check paused 为 Yes 也就是 true 的话，那么只会做 Sync replicas。也就是说把 replicas sync 同步到对应的 ReplicaSet 中，最后再 Update Deployment status，那么 controller 这一次的 ReplicaSet 就结束了。

 

那么如果 paused 为 false 的话，它就会做 Rollout，也就是通过 Create 或者是 Rolling 的方式来做更新，更新的方式其实也是通过 Create/Update/Delete 这种 ReplicaSet 来做实现的。

![image-20211212202547757](https://tva1.sinaimg.cn/large/008i3skNgy1gxbbl8l2guj31460jut9w.jpg)

当 Deployment 分配 ReplicaSet 之后，ReplicaSet 控制器本身也是从 Informer 中 watch 一些事件，这些事件包含了 ReplicaSet 和 Pod 的事件。从队列中取出之后，**ReplicaSet controller 的逻辑很简单，就只管理副本数。也就是说如果 controller 发现 replicas 比 Pod 数量大的话，就会扩容，而如果发现实际数量超过期望数量的话，就会删除 Pod。**

上面 Deployment 控制器的图中可以看到，**Deployment 控制器其实做了更复杂的事情，包含了版本管理，而它把每一个版本下的数量维持工作交给 ReplicaSet 来做。**

## rs 和 deploy 模拟

![image-20211212203520743](https://tva1.sinaimg.cn/large/008i3skNgy1gxbbv68rxaj31400jkabv.jpg)

![image-20211212203557514](https://tva1.sinaimg.cn/large/008i3skNgy1gxbbvt6hl4j31fo0qin0j.jpg)

Deployment水平缩放，是不会创建新的ReplicaSet的，但是涉及到Pod模板的更新后，比如更改容器的镜像，那么Deployment会用创建一个新版本的ReplicaSet用来替换旧版本。



![image-20211212203702555](https://tva1.sinaimg.cn/large/008i3skNgy1gxbbwxil9wj31gc0oatc4.jpg)

## spec

- MinReadySeconds：**判断pod available最少ready时间**。    Deployment 会根据 Pod ready 来看 Pod 是否可用，但是如果我们设置了 MinReadySeconds 之后，比如设置为 30 秒，那 Deployment 就一定会等到 Pod ready 超过 30 秒之后才认为 Pod 是 available 的。Pod available 的前提条件是 Pod ready，但是 ready 的 Pod 不一定是 available 的，它一定要超过 MinReadySeconds 之后，才会判断为 available；

- revisionHistoryLimit：**保留历史 revision，即保留历史 ReplicaSet 的数量**，默认值为 10 个。这里可以设置为一个或两个，如果回滚可能性比较大的话，可以设置数量超过 10；

- paused：paused 是标识，**Deployment 只做数量维持，不做新的发布**，这里在 Debug 场景可能会用到；

- progressDeadlineSeconds：前面提到当 Deployment 处于扩容或者发布状态时，它的 condition 会处于一个 processing 的状态，processing 可以设置一个超时时间。**如果超过超时时间还处于 processing，那么 controller 将认为这个 Pod 会进入 failed 的状态。**

 





##  部署

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: prod
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.7.9
          ports:
            - containerPort: 80
```

通过运行以下命令创建 Deployment。

```shell
# 部署服务
$ kubectl apply -f ./nginx-deployment.yaml

# 检查Deployments的状态
$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           18s

#上述输出字段含义解释
NAME: 列出了集群中Deployments的名称
DESIRED: 显示应用程序的期望状态的所需副本数  .spec.replicas
CURRENT: 显示当前正在运行的副本数
UP-TO-DATE: 显示已更新以实现期望状态的副本数
AVAILABLE: 显示应用程序可供用户使用的副本数
AGE: 显示应用程序运行的时间量

# 查看Deployment上线状态
$ kubectl rollout status deployment/nginx-deployment

Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment "nginx-deployment" successfully rolled out


# 检查RC的状态
$ kubectl describe rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-75675f5897   3         3         3       18s

# 查看每个Pod自动生成的标签
$ kubectl get pods --show-labels
NAME                                READY   STATUS   LABELS
nginx-deployment-75675f5897-7ci7o   1/1     Running  app=nginx,pod-template-hash=3123191453
nginx-deployment-75675f5897-kzszj   1/1     Running  app=nginx,pod-template-hash=3123191453
nginx-deployment-75675f5897-qqcnn   1/1     Running  app=nginx,pod-template-hash=3123191453
```

Deployment 控制器将 `pod-template-hash` 标签添加到 Deployment 所创建或收留的 每个 ReplicaSet ,**可确保 Deployment 的子 ReplicaSets 不重叠。**

## 更新

**更新策略**

`.spec.strategy` 策略指定用于用新 Pods 替换旧 Pods 的策略。 `.spec.strategy.type` 可以是 “Recreate” 或 “RollingUpdate”。“RollingUpdate” 是默认值。

- 如果 `.spec.strategy.type==Recreate`，在创建新 Pods 之前，所有现有的 Pods 会被杀死。
-  `.spec.strategy.type==RollingUpdate`时，采取 滚动更新的方式更新 Pods。你可以指定 `maxUnavailable` 和 `maxSurge` 来控制滚动更新 过程。
  - `.spec.strategy.rollingUpdate.maxUnavailable` 是一个可选字段，用来指定 **更新过程中不可用的 Pod 的个数上限**。该值可以是绝对数字（例如，5），也可以是 所需 Pods 的百分比（例如，10%）。百分比值会转换成绝对数并去除小数部分。**默认值为 25%。**
  - `.spec.strategy.rollingUpdate.maxSurge` 是一个可选字段，用来指定**可以创建的超出 期望 Pod 个数的 Pod 数量**。此值可以是绝对数（例如，5）或所需 Pods 的百分比（例如，10%）。**此字段的默认值为 25%。**
- **MaxSurge 和 MaxUnavailable 不能同时为 0**，当 MaxSurge 为 0 的时候，必须要删除 Pod，才能扩容 Pod；如果不删除 Pod 是不能新扩 Pod 的，因为新扩出来的话，总共的 Pod 数量就会超过期望数量。而两者同时为 0 的话，MaxSurge 保证不能新扩 Pod，而 MaxUnavailable 不能保证 ReplicaSet 中有 Pod 是 available 的，这样就会产生问题。所以说这两个值不能同时为 0。用户可以根据自己的实际场景来设置对应的、合适的值。

让我们更新 nginx 的 Pods，使用 nginx:1.9.1 镜像来代替之前的旧镜像。

![image-20211212173913626](https://tva1.sinaimg.cn/large/008i3skNgy1gxb6rvom63j31gy0rgq6l.jpg)

```bash
# 更新nginx服务
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
deployment/nginx-deployment image updated
# 查看上线状态
$ kubectl rollout status deployment/nginx-deployment
```

当然，我们也可以使用 edit 命令来编辑 Deployment 配置文件，达到同样的效果。

```bash
# 编辑Deployment文件
$ kubectl edit deployment/nginx-deployment
deployment/nginx-deployment edited
```

使用 rollout 命令来，查看其上线状态。

```bash
# 查看上线状态
$ kubectl rollout status deployment/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment/nginx-deployment successfully rolled out

# 检查RC的状态
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1564180365   3         3         3       6s
nginx-deployment-2035384211   0         0         0       36s

# 查看Pod的状态(生成了新的Pods)
$ kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-1564180365-khku8   1/1       Running   0          14s
nginx-deployment-1564180365-nacti   1/1       Running   0          14s
nginx-deployment-1564180365-z9gth   1/1       Running   0          14s
```

- Deployment 可确保在更新时仅关闭一定数量的 Pods，

  默认情况下，**它确保至少 75% 所需 Pods 是运行的，即有 25% 的最大不可用。**

- Deployment 还确保仅创建一定数量的 Pods 高于期望的 Pods 数，

  默认情况下，它可确保最多增加 25% 期望 Pods 数。出现上述情况的时候，多是在进行扩容、缩容、滚动升级和回滚应用时出现。

```bash
# 获取Deployment的更多信息
$ kubectl describe deployments

Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Thu, 30 Nov 2017 10:56:25 +0000
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision=2
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
   Containers:
    nginx:
      Image:        nginx:1.16.1
      Port:         80/TCP
      Environment:  <none>
      Mounts:       <none>
    Volumes:        <none>
  Conditions:
    Type           Status  Reason
    ----           ------  ------
    Available      True    MinimumReplicasAvailable
    Progressing    True    NewReplicaSetAvailable
  OldReplicaSets:  <none>
  NewReplicaSet:   nginx-deployment-1564180365 (3/3 replicas created)
  Events:
    Type    Reason             Age   From                   Message
    ----    ------             ----  ----                   -------
    Normal  ScalingReplicaSet  2m    deployment-controller  Scaled up replica set nginx-deployment-2035384211 to 3
    Normal  ScalingReplicaSet  24s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 1
    Normal  ScalingReplicaSet  22s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 2
    Normal  ScalingReplicaSet  22s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 2
    Normal  ScalingReplicaSet  19s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 1
    Normal  ScalingReplicaSet  19s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 3
    Normal  ScalingReplicaSet  14s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 0
```

**更新过程**

可以看到，当第一次创建 Deployment 的时候，它创建了一个 ReplicaSet 并将其直接扩展至 3 个副本。

更新 Deployment 时，它创建了一个新的 ReplicaSet ，并将其扩展为 1，然后将旧 ReplicaSet 缩小到 2，以便至少有 2 个 Pod 可用，并且最多创建 4 个 Pod。然后，它继续向上和向下扩展新的和旧的 ReplicaSet ，具有相同的滚动更新策略。最后，将有 3 个可用的副本在新的 ReplicaSet 中，旧 ReplicaSet 将缩小到 0。

## 回滚

- 当 Deployment 不稳定或遇到故障的的时候，例如循环崩溃，这时可能就需要回滚 Deployment 了。在默认情况下，所有 Deployment 历史记录都保留在系统中，以便可以随时回滚。当然，可以通过修改历史记录限制来更改该限制。
- Deployment 被触发上线时，系统就会创建 Deployment 的新的修订版本。 这意味着**仅当 Deployment 的 Pod 模板（`.spec.template`）发生更改时，才会创建新修订版本** -- 例如，模板的标签或容器镜像发生变化。 其他更新，如 Deployment 的**扩缩容操作不会创建 Deployment 修订版本**。 这是为了方便同时执行手动缩放或自动缩放。 换言之，**当你回滚到较早的修订版本时，只有 Deployment 的 Pod 模板部分会被回滚。**



假设在更新 Deployment 时犯了一个拼写错误，将镜像名称命名为 nginx:1.91 而不是 nginx:1.9.1，这样我们更新 Pod 的时候就会出现错误。

```shell
# 设置镜像来更新服务
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.91
deployment.apps/nginx-deployment image updated

# 遇到问题展开状态来验证
# 通过如下命令来查看Deployment是否完成
$ kubectl rollout status deployment/nginx-deployment
Waiting for rollout to finish: 1 out of 3 new replicas have been updated...
```

查看旧 ReplicaSets 和 Pod 的时候，会发现由新 ReplicaSet 创建的 1 个 Pod 卡在镜像拉取循环中，要解决此问题，需要回滚到以前稳定的 Deployment 版本。

```shell
# 查看RS状态
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1564180365   3         3         3       25s
nginx-deployment-2035384211   0         0         0       36s
nginx-deployment-3066724191   1         1         0       6s

# 查看Pod状态
$ kubectl get pods
NAME                                READY     STATUS             RESTARTS   AGE
nginx-deployment-1564180365-70iae   1/1       Running            0          25s
nginx-deployment-1564180365-jbqqo   1/1       Running            0          25s
nginx-deployment-1564180365-hysrc   1/1       Running            0          25s
nginx-deployment-3066724191-08mng   0/1       ImagePullBackOff   0          6s
```

**Deployment 控制器自动停止有问题的上线过程，并停止对新的 ReplicaSet 扩容。 这行为取决于所指定的 rollingUpdate 参数（具体为 `maxUnavailable`）。 默认情况下，Kubernetes 将此值设置为 25%。**



按照如下步骤，检查 Deployment 回滚历史。

```shell
# 检查Deployment修改历史
$ kubectl rollout history deployment/nginx-deployment
deployments "nginx-deployment"
REVISION    CHANGE-CAUSE
1           kubectl apply --filename=https://k8s.io/examples/controllers/nginx-deployment.yaml --record=true
2           kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.9.1 --record=true
3           kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.91 --record=true

# 查看修改历史的详细信息
$ kubectl rollout history deployment/nginx-deployment --revision=2
deployments "nginx-deployment" revision 2
  Labels:       app=nginx
          pod-template-hash=1159050644
......
```

按照下面给出的步骤将 Deployment 从当前版本回滚到以前的版本，即版本 2。现在，Deployment 将回滚到以前的稳定版本。如所见，Deployment 回滚事件回滚到修改版 2 是从 Deployment 控制器生成的。

```shell
# 现在已决定撤消当前展开并回滚到以前的版本
$ kubectl rollout undo deployment/nginx-deployment
deployment/nginx-deployment

# 通过下面命令来回滚到特定修改版本
$ kubectl rollout undo deployment/nginx-deployment --to-revision=2
deployment/nginx-deployment
```

检查回滚是否成功或者 Deployment 是否正在运行，可以通过下面命令查看。

```shell
# 查看deployment状态
$ kubectl get deployment nginx-deployment
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           30m

# 获取Deployment描述信息
$ kubectl describe deployment nginx-deployment
```

## 缩放

我们可以通过使用如下指令，缩放 Deployment。假设启用水平自动缩放 Pod 在集群中，可以为 Deployment 设置自动缩放器，并选择最小和最大 要基于现有 Pods 的 CPU 利用率运行的 Pods。

```shell
# 扩容(3->10)
$ kubectl scale deployment/nginx-deployment --replicas=10
deployment.apps/nginx-deployment scaled

# 水平缩放
$ kubectl autoscale deployment/nginx-deployment --min=10 --max=15 --cpu-percent=80
deployment/nginx-deployment scaled
```

## 暂停、恢复

这样做使得你能够在暂停和恢复执行之间应用多个修补程序，而**不会触发不必要的上线操作。(rolling histroy)**

```bash
# 暂停
kubectl rollout pause deployment.v1.apps/nginx-deployment

kubectl set resources deployment.v1.apps/nginx-deployment -c=nginx --limits=cpu=200m,memory=512Mi
kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1


# 恢复
kubectl rollout resume deployment.v1.apps/nginx-deployment
```

## 状态

Deployment 的生命周期中会有许多状态。上线新的 ReplicaSet 期间可能处于 [Progressing（进行中）](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/#progressing-deployment)，可能是 [Complete（已完成）](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/#complete-deployment)，也可能是 [Failed（失败）](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/#failed-deployment)以至于无法继续进行。

- *Progressing* 使用 `kubectl rollout status` 监视 Deployment 的进度，rs（创建，缩放，pod就绪）
- *Complete*更新都已完成，所有副本都可用
- [Failed原因](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/#failed-deployment)

![image-20211212200024087](https://tva1.sinaimg.cn/large/008i3skNgy1gxbaut4b85j31mg0u0whb.jpg)

# StatefulSet

在 k8s 中，ReplicaSet 和 Deployment 主要是用于处理无状态的服务，无状态服务的需求往往非常简单并且轻量，每一个无状态节点存储的数据在重启之后就会被删除。但是如果我们需要保留，那该怎么办呢？所以为了满足有状态的服务这一特殊需求，StatefulSet 就是 Kubernetes 为了运行有状态服务引入的资源，例如 MySQL 等。

产生 StatefulSet 的用途主要是用于管理有状态应用的工作负载对象，与 ReplicaSet 和 Deployment 这两个对象不同，StatefulSet 不仅能管理 Pod 的对象，还它能够保证这些 Pod 的顺序性和唯一性。以及，其会为每个 Pod 设置一个单独的持久标识 ID 号，这些用于标识序列的标识符在发生调度时也不会丢失，即无论怎么调度，每个 Pod 都有一个永久不变的 ID。

## 需求分析

比如下图所示的一些需求：

 

![img](https://tva1.sinaimg.cn/large/008i3skNgy1gynfp3tqkhj319a0hediw.jpg)

 

以上的这些需求都是 Deployment 无法满足的，因此 Kubernetes 社区为我们提供了一个叫 StatefluSet 的资源，用来管理有状态应用。

其实现在社区很多无状态应用也通过 StatefulSet 来管理，通过这节课程，大家也会明白为什么我们将部分无状态应用也通过 StatefulSet 来管理。 

![img](https://tva1.sinaimg.cn/large/008i3skNgy1gynfq9dc92j31e20petd2.jpg)

StatefulSet 中的 Pod 都是有序号的，从 0 开始一直到定义的 replica 数量减一。每个 Pod 都有独立的网络标识：一个 hostname、一块独立的 pvc 以及 pv 存储。这样的话，同一个 StatefulSet 下不同的 Pod，有不同的网络标识、有自己独享的存储盘，这就能很好地满足了绝大部分有状态应用的需求。

如上图右侧所示：

- 首先，每个 Pod 会有 Order 序号，会按照序号来创建，删除和更新 Pod；
- 其次，通过配置一个 headless Service，使每个 Pod 有一个唯一的网络标识 (hostname)；
- 第三，通过配置 pvc 模板，就是 pvc template，使每个 Pod 有一块或者多块 pv 存储盘；
- 最后，支持一定数量的灰度发布。比如现在有三个副本的 StatefulSet，**我们可以指定只升级其中的一个或者两个，更甚至是三个到新版本。通过这样的方式，来达到灰度升级的目的。**

##  功能特点

StatefulSets 最为重要的功能就是稳定，稳定意味着 Pod 调度或重调度的整个过程是有持久性的。如果应用程序不需要任何稳定的标识符或有序的部署、删除或伸缩，则应该使用由一组无状态的副本控制器提供的工作负载来部署应用程序，比如 Deployment 或者 ReplicaSet 可能更适用于您的无状态应用部署需要。

- 稳定的、唯一的网络标识符
- 稳定的、持久的存储
- 有序的、优雅的部署和缩放
- 有序的、自动的滚动更新
- 访问方式



## 特点详解

我们在下面的示例中是使用 StatefulSet 和对应的无头服务来做演示的，当 StatefulSet Pod 创建之后其具有唯一的标识，该标识包括顺序标识、稳定的网络标识和稳定的存储。该标识和 Pod 是绑定的，不管它被调度在哪个节点上。

- 有序索引

  对于具有 N 个副本的 StatefulSet，StatefulSet 中的每个 Pod 将被分配一个整数序号，从 0 到 N-1，该序号在 StatefulSet 上是唯一的。

- 稳定的pod ID

  StatefulSet 中的每个 Pod 根据 StatefulSet 的名称和 Pod 的序号派生出它的主机名。 组合主机名的格式为`$(StatefulSet 名称)-$(序号)`。 上例将会创**建三个名称分别为 `web-0、web-1、web-2` 的 Pod。** 


- 稳定网络ID

  StatefulSet 当前需要[无头服务](https://kubernetes.io/zh/docs/concepts/services-networking/service/#headless-services) 来负责 Pod 的网络标识。**你需要负责创建此服务。**管理域的这个服务的格式为： `$(服务名称).$(命名空间).svc.cluster.local`，其中 `cluster.local` 是集群域。

  **一旦每个 Pod 创建成功，就会得到一个匹配的 DNS 子域，格式为： `$(pod 名称).$(所属服务的 DNS 域名)`，其中所属服务由	StatefulSet 的 `serviceName` 域来设定。**

  **集群内部pod之间通过service匹配到的DNS子域互相访问**

![图片](https://tva1.sinaimg.cn/large/008i3skNgy1gvxgu65tffj30qc05xdgm.jpg)

- 稳定的存储

  在 Kubernetes 中 StatefulSet 模式会为每个 VolumeClaimTemplate 创建一个 PersistentVolumes。

  **请注意，当 Pod 或者 StatefulSet 被删除时，与 PersistentVolumeClaims 相关联的 PersistentVolume 并不会被删除。要删除它必须通过手动方式来完成。**





## pod 管理

Deployment 使用 ReplicaSet 来管理 Pod 的版本和所期望的 Pod 数量，但是在 StatefulSet 中，是由 **StatefulSet Controller 来管理下属的 Pod，因此 StatefulSet 通过 Pod 的 label 来标识这个 Pod 所属的版本，这里叫 controller-revision-hash**。这个 label 标识和 Deployment 以及 StatefulSet 在 Pod 中注入的 Pod template hash 是类似的。

![img](https://tva1.sinaimg.cn/large/008i3skNgy1gyng3nm8gwj31by0gkgoq.jpg)

通过 get pod 查看到 controller-revision-hash，这里的 hash 就是第一次创建 Pod 对应的 template 版本，可以看到后缀是 677759c9b8。这里先记录一下，接下来会做 Pod 升级，再来看一下 controller-revision-hash 会不会发生改变。



## 更新策略

在默认 [Pod 管理策略](https://kubernetes.io/zh/docs/concepts/workloads/controllers/statefulset/#pod-management-policies)(`OrderedReady`) 下使用 [滚动更新](https://kubernetes.io/zh/docs/concepts/workloads/controllers/statefulset/#rolling-updates) ，**可能进入需要人工干预才能修复的损坏状态**。

如果更新后 Pod 模板配置进入无法运行或就绪的状态（例如，由于错误的二进制文件 或应用程序级配置错误），StatefulSet 将停止回滚并等待。

在这种状态下，仅将 Pod 模板还原为正确的配置是不够的。由于 [已知问题](https://github.com/kubernetes/kubernetes/issues/67250)，StatefulSet 将继续等待损坏状态的 Pod 准备就绪（永远不会发生），然后再尝试将其恢复为正常工作配置。

**恢复模板后，还必须删除 StatefulSet 尝试使用错误的配置来运行的 Pod**。这样， StatefulSet 才会开始使用被还原的模板来重新创建 Pod。





## 示例运行

```yaml
sheapiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  clusterIP: None
  selector:
    app: nginx
  ports:
  - port: 80
    name: web

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  podManagementPolicy: "Parallel"
  replicas: 2
  ......
```



名为 nginx 的 Headless Service 用来控制网络域名。

名为 web 的 StatefulSet 有一个 Spec，它表明将在独立的 2 个 Pod 副本中启动 nginx 容器。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
    - port: 80
      name: web
  clusterIP: None
  selector:
    app: nginx

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: k8s.gcr.io/nginx-slim:0.8
          ports:
            - containerPort: 80
              name: web
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

### 创建和查看 - StatefulSet

```shell
# 创建定义在web.yaml中的Headless Service和StatefulSet
$ sudo kubectl apply -f web.yaml
service/nginx created
statefulset.apps/web created

# 验证是否成功创建
$ sudo kubectl get service nginx
NAME      TYPE         CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx     ClusterIP    None         <none>        80/TCP    12s

$ sudo kubectl get statefulset web
NAME      DESIRED   CURRENT   AGE
web       2         1         20s
# 动态查看创建状态，发现是顺序创建Pod的
# 我们可以动态的看到web-0的Pod先创建完毕之后，web-1才会被启动
$ sudo kubectl get pods -w -l app=nginx
NAME      READY     STATUS              RESTARTS   AGE
web-0     0/1       Pending             0          0s
web-0     0/1       Pending             0          0s
web-0     0/1       ContainerCreating   0          0s
web-0     1/1       Running             0          19s
web-1     0/1       Pending             0          0s
web-1     0/1       Pending             0          0s
web-1     0/1       ContainerCreating   0          0s
web-1     1/1       Running             0          18s

# 获取StatefulSet的Pod
$ sudo kubectl get pods -l app=nginx
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          1m
web-1     1/1       Running   0          1m

```

### 查看绑定的特点 - StatefulSet

```shell
# 使用稳定的网络身份标识
$ sudo kubectl exec web-0 -- bash -c \
    "for i in 0 1; do kubectl exec web-$i -- sh -c 'hostname'; done"
web-0

$ sudo kubectl exec web-1 -- bash -c \
    "for i in 0 1; do kubectl exec web-$i -- sh -c 'hostname'; done"
web-1
# 稳定的网络身份标识
$ sudo kubectl run -i --tty --image busybox:1.28 \
    dns-test --restart=Never --rm nslookup web-0.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.244.1.6

nslookup web-1.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 10.244.2.6
# 写入稳定的存储
$ sudo kubectl get pvc -l app=nginx
NAME        STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
www-web-0   Bound     pvc-15c268c7-b507-11e6-932f-42010a800002   1Gi        RWO           48s
www-web-1   Bound     pvc-15c79307-b507-11e6-932f-42010a800002   1Gi        RWO           48s
```

**删除 StatefulSet 中所有的 Pod 之后，我们使用上述查看网络身份标识的命令再次查看，发现 Pod 的序号、主机名、SRV 条目和记录名称没有改变，但和 Pod 相关联的 IP 地址可能发生了改变。**因为我们使用的是 StatefulSet 的模式，其他模式不会有这个问题。这就是为什么不要在其他应用中使用 StatefulSet 中的 Pod 的 IP 地址进行连接，这点很重要。

```shell
$ sudo kubectl delete pod -l app=nginx
pod "web-0" deleted
pod "web-1" deleted

# 对应的服务访问地址，我们可以间接访问
web-0.nginx.default.svc.cluster.local
web-1.nginx.default.svc.cluster.local
扩容/缩容 - StatefulSet
# 扩容
$ sudo kubectl scale sts web --replicas=5
statefulset.apps/web scaled

# 缩容
$ sudo kubectl patch sts web -p '{"spec":{"replicas":3}}'
statefulset.apps/web patched
# 五个存储仍然存在
$ sudo kubectl get pvc -l app=nginx
NAME        STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
www-web-0   Bound     pvc-15c268c7-b507-11e6-932f-42010a800002   1Gi        RWO           13h
www-web-1   Bound     pvc-15c79307-b507-11e6-932f-42010a800002   1Gi        RWO           13h
www-web-2   Bound     pvc-e1125b27-b508-11e6-932f-42010a800002   1Gi        RWO           13h
www-web-3   Bound     pvc-e1176df6-b508-11e6-932f-42010a800002   1Gi        RWO           13h
www-web-4   Bound     pvc-e11bb5f8-b508-11e6-932f-42010a800002   1Gi        RWO           13h
```

### 更新 - StatefulSet

**StatefulSet 里的 Pod 采用和序号相反的顺序更新。**在更新下一个 Pod 前，StatefulSet 控制器终止每个 Pod 并等待它们变成 Running 和 Ready。请注意，虽然在顺序后继者变成 unning 和 Ready 之前 StatefulSet 控制器不会更新下一个 Pod，但它仍然会重建任何在更新过程中发生故障的 Pod，使用的是它们当前的版本。已经接收到更新请求的 Pod 将会被恢复为更新的版本，没有收到请求的 Pod 则会被恢复为之前的版本。像这样，控制器尝试继续使应用保持健康并在出现间歇性故障时保持更新的一致性。



复用了之前的网络标识和存储盘呢？

**其实 headless Service 配置的 hostname 只是和 Pod name 挂钩的**，所以只要升级后的 Pod 名称和旧的 Pod 名称相同，那么就可以沿用之前 Pod 使用的网络标识。

关于存储盘，由上图可以看到 PVC 的状态，它们的创建时间一直没有改变，还是第一次创建 Pod 时的时间，所以现在升级后的 Pod 使用的还是旧 Pod 中使用的 PVC。

```shell
# Patch web StatefulSet 来执行 RollingUpdate 更新策略
$ sudo kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate"}}}'
statefulset.apps/web patched

# 在一个终端窗口中 patch web StatefulSet 来再次的改变容器镜像
$ sudo kubectl patch statefulset web --type='json' \
    -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", \
    "value":"gcr.io/google_containers/nginx-slim:0.8"}]'
statefulset.apps/web patched


# 更新
kubectl set iamge statefulset [statefulset name]  [container name]=new image
```

### 删除 - StatefulSet

```shell
# 删除StatefulSet
kubectl delete statefulset web
statefulset.apps "web" deleted

# 删除Pod
kubectl delete pod web-0
pod "web-0" deleted
```

## 注意事项

主要讨论关于 StatefulSet 的使用和注意细节！

- StatefulSet 的特性描述

  - 匹配 Pod name (网络标识) 的模式为：名称(序号)，比如上面的示例：web-0，web-1，web-2。
  - StatefulSet 为每个 Pod 副本创建了一个 DNS 域名，这个域名的格式为：**$(podname).(headless servername)**，也就意味着服务间是通过 Pod 域名来通信而非 Pod IP，因为当 Pod 所在 Node 发生故障时，Pod 会被飘移到其它 Node 上，Pod IP 会发生变化，但是 Pod 域名不会有变化。
  - StatefulSet 使用 Headless 服务来控制 Pod 的域名，这个域名的 FQDN 为：(namespace).svc.cluster.local，其中，“cluster.local” 指的是集群的域名。
  - 根据 volumeClaimTemplates，为每个 Pod 创建一个 pvc，pvc 的命名规则匹配模式：(volumeClaimTemplates.name)-(pod_name)，比如上面的 volumeMounts.name=www，Pod name=web-[0-2]，因此创建出来的 PVC 是 www-web-0、www-web-1、www-web-2。
  - **删除 Pod 不会删除其 pvc，手动删除 pvc 将自动释放 pv。**

- StatefulSet 的启停顺序

  - 当对 Pod 执行扩展操作时，与部署一样，它前面的 Pod 必须都处于 Running 和 Ready 状态。
  - 当 Pod 被删除时，它们被终止的顺序是从 N-1 到 0。
  - 部署 StatefulSet 时，如果有多个 Pod 副本，它们会被顺序地创建（从 0 到 N-1）并且，在下一个 Pod 运行之前所有之前的 Pod 必须都是 Running 和 Ready 状态。
  - 有序部署
  - 有序删除
  - 有序扩展

- StatefulSet 的使用场景

  - 有序收缩。
  - 有序部署，有序扩展，基于 init containers 来实现。
  - 稳定的网络标识符，即 Pod 重新调度后其 PodName 和 HostName 不变。
  - 稳定的持久化存储，即 Pod 重新调度后还是能访问到相同的持久化数据，基于 PVC 来实现。



## 架构设计

### 管理模式

- **第一种资源：ControllerRevision**

  **通过这个资源，StatefulSet 可以很方便地管理不同版本的 template 模板。**

  举个例子：比如上文中提到的 nginx，在创建之初拥有的第一个 template 版本，会创建一个对应的 ControllerRevision。而当修改了 image 版本之后，StatefulSet Controller 会创建一个新的 ControllerRevision，大家可以理解为**每一个 ControllerRevision 对应了每一个版本的 Template，也对应了每一个版本的 ControllerRevision hash。**其实在 **Pod label 中定义的 ControllerRevision hash，就是 ControllerRevision 的名字。**通过这个资源 StatefulSet Controller 来管理不同版本的 template 资源。

- **第二个资源：PVC**

  **如果在 StatefulSet 中定义了 volumeClaimTemplates，StatefulSet 会在创建 Pod 之前，先根据这个模板创建 PVC，并把 PVC 加到 Pod volume 中。**

  如果用户在 spec 的 pvc 模板中定义了 volumeClaimTemplates，StatefulSet 在创建 Pod 之前，根据模板创建 PVC，并加到 Pod 对应的 volume 中。当然也可以在 spec 中不定义 pvc template，那么所创建出来的 Pod 就不会挂载单独的一个 pv。

  **pvc name = volumeClaimTemplates.metadata.name-StatefulSet.metadata.name-orderIndex**

  **不设置volumeClaimTemplates**连pod都创建不出来

  > Deployment中的Pod template里定义的存储卷，是所有副本集共用一个存储卷，数据是相同的，因为是基于模板来的 ，而statefulset中每个Pod都要自已的专有存储卷，所以statefulset的存储卷就不能再用Pod模板来创建了，于是statefulSet使用volumeClaimTemplate，称为卷申请模板，它会为每个Pod生成不同的pvc，并绑定pv， 从而实现各pod有专用存储

- **第三个资源：Pod**

  **StatefulSet 按照顺序创建、删除、更新 Pod，每个 Pod 有唯一的序号。**

  这里不同的地方在于，当前版本的 **StatefulSet 只会在 ControllerRevision 和 Pod 中添加 OwnerReference**，而不会在 PVC 中添加 OwnerReference。之前的课程中提到过，**拥有 OwnerReference 的资源，在管理的这个资源进行删除的默认情况下，会关联级联删除下属资源。**因此默认情况下删除 StatefulSet 之后，StatefulSet 创建的 ControllerRevision 和 Pod 都会被删除，但是 PVC 因为没有写入 OwnerReference，PVC 并不会被级联删除。

### StatefulSet 控制器

![img](https://tva1.sinaimg.cn/large/008i3skNgy1gynh8g11pyj31fw0gywg9.jpg)

上图为 StatefulSet 控制器的工作流程，下面来简单介绍一下整个工作处理流程。 

首先通过注册 Informer 的 Event Handler(事件处理)，来处理 StatefulSet 和 Pod 的变化。在 Controller 逻辑中，每一次收到 StatefulSet 或者是 Pod 的变化，都会找到对应的 StatefulSet 放到队列。紧接着从队列取出来处理后，先做的操作是 Update Revision，也就是先查看当前拿到的 StatefulSet 中的 template，有没有对应的 ControllerRevision。如果没有，说明 **template 已经更新过，Controller 就会创建一个新版本的 Revision，也就有了一个新的 ControllerRevision hash 版本号。** 

然后 Controller 会把所有版本号拿出来，并且按照序号整理一遍。这个整理的过程中，如果发现有缺少的 Pod，它就会按照序号去创建，如果发现有多余的 Pod，就会按照序号去删除。当保证了 Pod 数量和 Pod 序号满足 Replica 数量之后，Controller 会去查看是否需要更新 Pod。也就是说这两步的区别在于，**Manger pods in order 去查看所有的 Pod 是否满足序号；而后者 Update in order 查看 Pod 期望的版本是否符合要求，并且通过序号来更新。** 

**Update in order 其更新过程如上图所示，其实这个过程比较简单，就是删除 Pod。**删除 Pod 之后，其实是在下一次触发事件，Controller 拿到这个 success 之后会发现缺少 Pod，然后再从前一个步骤 Manger pod in order 中把新的 Pod 创建出来。在这之后 Controller 会做一次 Update status，也就是之前通过命令行看到的 status 信息。

通过整个这样的一个流程，StatefulSet 达到了管理有状态应用的能力。

### 扩容模拟

![img](https://tva1.sinaimg.cn/large/008i3skNgy1gynhn75m5ej30m80fcgm7.jpg)

假设 StatefulSet 初始配置 replicas 为 1，有一个 Pod0。那么将 replicas 从 1 修改到 3 之后，其实我们是先创建 Pod1，默认情况是等待 Pod1 状态 READY 之后，再创建 Pod2。

通过上图可以看到每个 StatefulSet 下面的 Pod 都是从序号 0 开始创建的。因此一个 replicas 为 N 的 StatefulSet，它创建出来的 Pod 序号为 [0,N)，0 是开曲线，N 是闭曲线，也就是当 N>0 的时候，序号为 0 到 N-1。

### 扩缩容管理策略

![img](https://tva1.sinaimg.cn/large/008i3skNgy1gyni77hgv6j31dw0i6tbs.jpg)

对于某些分布式系统来说，StatefulSet 的顺序性保证是不必要和/或者不应该的。这些系统仅仅要求唯一性和身份标志。为了解决这个问题，在 Kubernetes 1.7 中引入了 **.spec.podManagementPolicy**。

#### OrderedReady 的 Pod 管理策略

是 StatefulSets 的默认选项，它告诉 StatefulSet 控制器遵循上文展示的顺序性保证。

  - 对于包含 N 个 副本的 StatefulSet，当部署 Pod 时，它们是依次创建的，顺序为 `0..N-1`。
  - 当删除 Pod 时，它们是**逆序终止的**，顺序为 `N-1..0`。
  - 在将缩放操作应用到 Pod 之前，它前面的所有 Pod 必须是 Running 和 Ready 状态。
  - 在 Pod 终止之前，所有的继任者必须完全关闭。
    StatefulSet 不应将 `pod.Spec.TerminationGracePeriodSeconds` 设置为 0。 这种做法是不安全的，要强烈阻止。更多的解释请参考 [强制删除 StatefulSet Pod](https://kubernetes.io/zh/docs/tasks/run-application/force-delete-stateful-set-pod/)。

>   在上面的 nginx 示例被创建后，会按照 web-0、web-1、web-2 的顺序部署三个 Pod。 在 web-0 进入 [Running 和 Ready](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-lifecycle/) 状态前不会部署 web-1。在 web-1 进入 Running 和 Ready 状态前不会部署 web-2。 如果 web-1 已经处于 Running 和 Ready 状态，而 web-2 尚未部署，在此期间发生了 web-0 运行失败，那么 web-2 将不会被部署，要等到 web-0 部署完成并进入 Running 和 Ready 状态后，才会部署 web-2。
>
>   如果用户想将示例中的 StatefulSet 收缩为 `replicas=1`，首先被终止的是 web-2。 在 web-2 没有被完全停止和删除前，web-1 不会被终止。 当 web-2 已被终止和删除、web-1 尚未被终止，如果在此期间发生 web-0 运行失败， 那么就不会终止 web-1，必须等到 web-0 进入 Running 和 Ready 状态后才会终止 web-1。
>

#### Parallel 的 Pod 管理策略

是告诉 StatefulSet 控制器并行的终止所有 Pod，在启动或终止另一个 Pod 前，不必等待这些 Pod 变成 Running 和 Ready 或者完全终止状态

`Parallel` Pod 管理让 StatefulSet 控制器并行的启动或终止所有的 Pod， 启动或者终止其他 Pod 前，无需等待 Pod 进入 Running 和 ready 或者完全停止状态。 这个选项只会影响伸缩操作的行为，更新则不会被影响。

**严格按照倒序串行升级Pod**，只是扩缩容不会。

### 发布模拟

![img](https://tva1.sinaimg.cn/large/008i3skNgy1gynqxnccovj31b40pw0vt.jpg) 

假设这里的 StatefulSet template1 对应逻辑上的 Revision1，这时 StatefulSet 下面的三个 Pod 都属于 Revision1 版本。在我们修改了 template，比如修改了镜像之后，**Controller 是通过倒序的方式逐一升级 Pod。**上图中可以看到 Controller 先创建了一个 Revision2，对应的就是创建了 ControllerRevision2 这么一个资源，并且将 ControllerRevision2 这个资源的 name 作为一个新的 Revision hash。在把 Pod2 升级为新版本后，逐一删除 Pod0、Pod1，再去创建 Pod0、Pod1。

它的逻辑其实很简单，在**升级过程中 Controller 会把序号最大并且符合条件的 Pod 删除掉**，那么删除之后在下一次 Controller 在做 reconcile 的时候，它会发现缺少这个序号的 Pod，然后再按照新版本把 Pod 创建出来。

### spec 字段解析

![img](https://tva1.sinaimg.cn/large/008i3skNgy1gynredb9zxj31g00qc0xk.jpg)

首先来看一下 spec 中前几个字段，Replica 和 Selector 都是我们比较熟悉的字段。

- Replica 主要是期望的数量；
- Selector 是事件选择器，必须匹配 spec.template.metadata.labels 中定义的条件；
- Template：Pod 模板，定义了所要创建的 Pod 的基础信息模板；
- VolumeClaimTemplates：PVC 模板列表，如果在 spec 中定义了这个，PVC 会先于 Pod 模板 Template 进行创建。在 PVC 创建完成后，把创建出来的 PVC name 作为一个 volume 注入到根据 Template 创建出来的 Pod 中。

![img](https://tva1.sinaimg.cn/large/008i3skNgy1gynrh0gc8dj31h40k2aej.jpg)

- ServiceName：对应 Headless Service 的名字。**通过headless service来为StatefulSet的每个Pod提供唯一hostname**，当然如果有人不需要这个功能的时候，会给 Service 定一个不存在的 value，Controller 也不会去做校验，所以可以写一个 fake 的 ServiceName。但是这里推荐每一个 Service 都要配置一个 Headless Service，不管 StatefulSet 下面的 Pod 是否需要网络标识；
- PodMangementPolicy：Pod 管理策略。前面提到过这个字段的可选策略为 OrderedReady 和 Parallel，默认情况下为前者；
- UpdataStrategy：Pod 升级策略。这是一个结构体，下面再详细介绍；
- RevisionHistoryLimit：**保留历史 ControllerRevision 的数量限制(默认为 10)**。需要注意的一点是，这里清楚的版本，必须没有相关的 Pod 对应这些版本，如果有 Pod 还在这个版本中，这个 ControllerRevision 是不能被删除的。

### 升级策略字段解析

![img](https://tva1.sinaimg.cn/large/008i3skNgy1gynrhqiwimj31fk0t6q98.jpg)

在上图右侧可以看到 StatefulSetUpdateStrategy 有个 type 字段，这个 type 定义了两个类型：一个是 RollingUpdate；一个是OnDelete。

- RollingUpdate 其实跟 Deployment 中的升级是有点类似的，就是根据滚动升级的方式来升级。 

- OnDelete 是在删除的时候升级，叫做禁止主动升级，Controller 并不会把存活的 Pod 做主动升级，而是通过 OnDelete 的方式。比如说当前有三个旧版本的 Pod，但是升级策略是 OnDelete，所以当更新 spec 中镜像的时候，Controller 并不会把三个 Pod 逐一升级为新版本，而是当我们缩小 Replica 的时候，Controller 会先把 Pod 删除掉，当我们下一次再进行扩容的时候，Controller 才会扩容出来新版本的 Pod。

在 RollingUpdateStatefulSetSetStrategy 中，可以看到有个字段叫 Partition。**这个 Partition 表示滚动升级时，保留旧版本 Pod 的数量。很多刚结束 StatefulSet 的同学可能会认为这个是灰度新版本的数量，这是错误的。** 

> 举个例子：假设当前有个 replicas 为 10 的 StatefulSet，当我们更新版本的时候，如果 Partition 是 8，并不是表示要把 8 个 Pod 更新为新版本，而是表示需要保留 8 个 Pod 为旧版本，只更新 2 个新版本作为灰度。当 Replica 为 10 的时候，下面的 Pod 序号为 [0,9)，因此当我们配置 Partition 为 8 的时候，其实还是保留 [0,7) 这 8个 Pod 为旧版本，只有 [8,9) 进入新版本。
>
> 总结一下，假设 replicas=N，Partition=M (M<N) ，则最终旧版本 Pod 为 [0，M) ,新版本 Pod 为 [M,N)。通过这样一个 Partition 的方式来达到灰度升级的目的，这是目前 Deployment 所不支持的。

 



# sts 和 deploy 区别

- 访问方式区别

![图片](https://tva1.sinaimg.cn/large/008i3skNgy1gvcu1xyjvoj60or0bqab802.jpg)

![图片](https://tva1.sinaimg.cn/large/008i3skNgy1gvcu1yahxoj60u00bo75n02.jpg)

- 综合对比区别

| 类型特性                        | **Deployment**                                               | **StatefulSet**                                              |
  | :------------------------------ | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 是否暴露到外网                  | 可以                                                         | 一般不                                                       |
| 请求面向的对象                  | serviceName                                                  | 指定 pod 的域名                                              |
| 灵活性                          | 只能通过 service/serviceIp 访问到 k8s 自动转发的 pod         | 可以访问任意一个自定义的 pod                                 |
| 易用性                          | 只需要关心 Service 的信息即可                                | 需要知道要访问的 pod 启动的名称、headlessService 名称        |
| PV/PVC 绑定关系的稳定性(多实例) | (pod 挂掉后重启)无法保证初始的绑定关系                       | 可以保证                                                     |
| pod 名称稳定性                  | 不稳定，因为是通过 template 创建，每次为了避免重复都会后缀一个随机数 | 稳定，每次都一样                                             |
| 启动顺序(多实例)                | 随机启动，如果 pod 宕掉重启，会自动分配一个 node 重新启动    | pod 按  app-0、app-1…app-（n-1），如果 pod 宕掉重启，还会在之前的 node 上重新启动 |
| 停止顺序(多实例)                | 随机停止                                                     | 倒序停止                                                     |
| 集群内部服务发现                | 只能通过 service 访问到随机的 pod                            | 可以打通 pod 之间的通信（主要是被发现）                      |
| 性能开销                        | 无需维护 pod 与 node、pod 与 PVC 等关系                      | 比 deployment 类型需要维护额外的关系信息                     |

综上选择总结

- 如果是不需额外数据依赖或者状态维护的部署，或者 replicas 是 1，优先考虑使用 Deployment；

- 如果单纯的要做数据持久化，防止 pod 宕掉重启数据丢失，那么使用 pv/pvc 就可以了；
- 如果要打通 app 之间的通信，而又不需要对外暴露，使用 headlessService 即可；
- 如果需要使用 service 的负载均衡，不要使用 StatefulSet，尽量使用 clusterIP 类型，用 serviceName 做转发；
- 如果是有多 replicas，且需要挂载多个 pv 且每个 pv 的数据是不同的，因为 pod 和 pv 之间是一一对应的，如果某个 pod 挂掉再重启，还需要连接之前的 pv，不能连到别的 pv 上，考虑使用 StatefulSet；
- 能不用 StatefulSet，就不要用；