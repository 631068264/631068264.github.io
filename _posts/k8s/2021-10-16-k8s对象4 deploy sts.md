---
layout:     post
rewards: false
title:      k8s对象 deployment statefulSet
categories:
    - k8s
---

# Deployment

Deployment 为 Pod 和 ReplicaSet 提供了一个声明式定义的方法，用来替代以前的 RC 来方便的管理应用。而定义方式分为，命令式(RS)和声明式(Deployment)两种，前者侧重于考虑如何实现，而后者侧重于定义想要什么。

- 应用场景
- 定义 Deployment 来创建 Pod 和 RS
    - 滚动升级和回滚应用
    - 扩容和缩容
    - 暂停和继续 Deployment
- RS 和 Deployment 的关联

![image-20211016214611265](https://tva1.sinaimg.cn/large/008i3skNgy1gvhhla38dxj61860jgta702.jpg)

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
DESIRED: 显示应用程序的期望状态的所需副本数
CURRENT: 显示当前正在运行的副本数
UP-TO-DATE: 显示已更新以实现期望状态的副本数
AVAILABLE: 显示应用程序可供用户使用的副本数
AGE: 显示应用程序运行的时间量

# 查看Deployment展开状态
$ kubectl rollout status deployment/nginx-deployment

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

## 更新

让我们更新 nginx 的 Pods，使用 nginx:1.9.1 镜像来代替之前的旧镜像。

```
# 更新nginx服务
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
deployment/nginx-deployment image updated
```

当然，我们也可以使用 edit 命令来编辑 Deployment 配置文件，达到同样的效果。

```
# 编辑Deployment文件
$ kubectl edit deployment/nginx-deployment
deployment/nginx-deployment edited
```

使用 rollout 命令来，查看其展开的状态。

```
# 查看展开状态
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

- Deployment 可确保在更新时仅关闭一定数量的 Pods，默认情况下，它确保至少 75% 所需 Pods 是运行的，即有 25% 的最大不可用。Deployment 还确保仅创建一定数量的 Pods 高于期望的 Pods 数，默认情况下，它可确保最多增加 25% 期望 Pods 数。出现上述情况的时候，多是在进行扩容、缩容、滚动升级和回滚应用时出现。
- 可以看到，当第一次创建 Deployment 的时候，它创建了一个 ReplicaSet 并将其直接扩展至 3 个副本。更新 Deployment 时，它创建了一个新的 ReplicaSet ，并将其扩展为 1，然后将旧 ReplicaSet 缩小到 2，以便至少有 2 个 Pod 可用，并且最多创建 4 个 Pod。然后，它继续向上和向下扩展新的和旧的 ReplicaSet ，具有相同的滚动更新策略。最后，将有 3 个可用的副本在新的 ReplicaSet 中，旧 ReplicaSet 将缩小到 0。

```shell
# 获取Deployment的更多信息
$ kubectl describe deployments
```

## 回滚

- 当 Deployment 不稳定或遇到故障的的时候，例如循环崩溃，这时可能就需要回滚 Deployment 了。在默认情况下，所有 Deployment 历史记录都保留在系统中，以便可以随时回滚。当然，可以通过修改历史记录限制来更改该限制。当回滚到较早的修改版时，只有 Deployment Pod 模板部分会回滚。这是因为当 Deployment Pod 模板(.spec.template)发生更改时，才会创建新修改版本。例如，如果更新模板的标签或容器镜像，其他更新，如扩展 Deployment。
- 假设在更新 Deployment 时犯了一个拼写错误，将镜像名称命名为 nginx:1.91 而不是 nginx:1.9.1，这样我们更新 Pod 的时候就会出现错误。

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


# StatefulSet

在 k8s 中，ReplicaSet 和 Deployment 主要是用于处理无状态的服务，无状态服务的需求往往非常简单并且轻量，每一个无状态节点存储的数据在重启之后就会被删除。但是如果我们需要保留，那该怎么办呢？所以为了满足有状态的服务这一特殊需求，StatefulSet 就是 Kubernetes 为了运行有状态服务引入的资源，例如 MySQL 等。

产生 StatefulSet 的用途主要是用于管理有状态应用的工作负载对象，与 ReplicaSet 和 Deployment 这两个对象不同，StatefulSet 不仅能管理 Pod 的对象，还它能够保证这些 Pod 的顺序性和唯一性。以及，其会为每个 Pod 设置一个单独的持久标识 ID 号，这些用于标识序列的标识符在发生调度时也不会丢失，即无论怎么调度，每个 Pod 都有一个永久不变的 ID。



- 功能特点

StatefulSets 最为重要的功能就是稳定，稳定意味着 Pod 调度或重调度的整个过程是有持久性的。如果应用程序不需要任何稳定的标识符或有序的部署、删除或伸缩，则应该使用由一组无状态的副本控制器提供的工作负载来部署应用程序，比如 Deployment 或者 ReplicaSet 可能更适用于您的无状态应用部署需要。

- 稳定的、唯一的网络标识符
- 稳定的、持久的存储
- 有序的、优雅的部署和缩放
- 有序的、自动的滚动更新

- 访问方式

我们在下面的示例中是使用 StatefulSet 和对应的无头服务来做演示的，当 StatefulSet Pod 创建之后其具有唯一的标识，该标识包括顺序标识、稳定的网络标识和稳定的存储。该标识和 Pod 是绑定的，不管它被调度在哪个节点上。

- 有序索引

对于具有 N 个副本的 StatefulSet，StatefulSet 中的每个 Pod 将被分配一个整数序号，从 0 到 N-1，该序号在 StatefulSet 上是唯一的。

- 稳定的网络 ID

  StatefulSet 中的每个 Pod 根据 StatefulSet 的名称和 Pod 的序号派生出它的主机名。 组合主机名的格式为`$(StatefulSet 名称)-$(序号)`。 上例将会创建三个名称分别为 `web-0、web-1、web-2` 的 Pod。 

  

  StatefulSet 可以使用 [无头服务](https://kubernetes.io/zh/docs/concepts/services-networking/service/#headless-services) 控制它的 Pod 的网络域。管理域的这个服务的格式为： `$(服务名称).$(命名空间).svc.cluster.local`，其中 `cluster.local` 是集群域。

  

   一旦每个 Pod 创建成功，就会得到一个匹配的 **DNS 子域**，格式为： `$(pod 名称).$(所属服务的 DNS 域名)`，其中所属服务由 StatefulSet 的 `serviceName` 域来设定。

  

​       **集群内部pod之间通过service匹配到的DNS子域互相访问**

- 稳定的存储

在 Kubernetes 中 StatefulSet 模式会为每个 VolumeClaimTemplate 创建一个 PersistentVolumes。请注意，当 Pod 或者 StatefulSet 被删除时，与 PersistentVolumeClaims 相关联的 PersistentVolume 并不会被删除。要删除它必须通过手动方式来完成。

![图片](https://tva1.sinaimg.cn/large/008i3skNgy1gvctka1wiij60qc05xgme02.jpg)

- 管理策略

对于某些分布式系统来说，StatefulSet 的顺序性保证是不必要和/或者不应该的。这些系统仅仅要求唯一性和身份标志。为了解决这个问题，在 Kubernetes 1.7 中引入了 **.spec.podManagementPolicy**。

**OrderedReady 的 Pod 管理策略**：是 StatefulSets 的默认选项，它告诉 StatefulSet 控制器遵循上文展示的顺序性保证。

**Parallel 的 Pod 管理策略**：是告诉 StatefulSet 控制器并行的终止所有 Pod，在启动或终止另一个 Pod 前，不必等待这些 Pod 变成 Running 和 Ready 或者完全终止状态。

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

#### 示例运行

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

创建和查看 - StatefulSet

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

查看绑定的特点 - StatefulSet

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

删除 StatefulSet 中所有的 Pod 之后，我们使用上述查看网络身份标识的命令再次查看，发现 Pod 的序号、主机名、SRV 条目和记录名称没有改变，但和 Pod 相关联的 IP 地址可能发生了改变。因为我们使用的是 StatefulSet 的模式，其他模式不会有这个问题。这就是为什么不要在其他应用中使用 StatefulSet 中的 Pod 的 IP 地址进行连接，这点很重要。

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

更新 - StatefulSet

StatefulSet 里的 Pod 采用和序号相反的顺序更新。在更新下一个 Pod 前，StatefulSet 控制器终止每个 Pod 并等待它们变成 Running 和 Ready。请注意，虽然在顺序后继者变成 unning 和 Ready 之前 StatefulSet 控制器不会更新下一个 Pod，但它仍然会重建任何在更新过程中发生故障的 Pod，使用的是它们当前的版本。已经接收到更新请求的 Pod 将会被恢复为更新的版本，没有收到请求的 Pod 则会被恢复为之前的版本。像这样，控制器尝试继续使应用保持健康并在出现间歇性故障时保持更新的一致性。

```shell
# Patch web StatefulSet 来执行 RollingUpdate 更新策略
$ sudo kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate"}}}'
statefulset.apps/web patched

# 在一个终端窗口中 patch web StatefulSet 来再次的改变容器镜像
$ sudo kubectl patch statefulset web --type='json' \
    -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", \
    "value":"gcr.io/google_containers/nginx-slim:0.8"}]'
statefulset.apps/web patched
```

删除 - StatefulSet

```shell
# 删除StatefulSet
kubectl delete statefulset web
statefulset.apps "web" deleted

# 删除Pod
kubectl delete pod web-0
pod "web-0" deleted
```


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

- 综上选择总结

- - 如果是不需额外数据依赖或者状态维护的部署，或者 replicas 是 1，优先考虑使用 Deployment；
- 如果单纯的要做数据持久化，防止 pod 宕掉重启数据丢失，那么使用 pv/pvc 就可以了；
- 如果要打通 app 之间的通信，而又不需要对外暴露，使用 headlessService 即可；
- 如果需要使用 service 的负载均衡，不要使用 StatefulSet，尽量使用 clusterIP 类型，用 serviceName 做转发；
- 如果是有多 replicas，且需要挂载多个 pv 且每个 pv 的数据是不同的，因为 pod 和 pv 之间是一一对应的，如果某个 pod 挂掉再重启，还需要连接之前的 pv，不能连到别的 pv 上，考虑使用 StatefulSet；
- 能不用 StatefulSet，就不要用；