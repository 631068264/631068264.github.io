> 将Spark运行在K8S集群上可以采用Spark官方原生的作业运行方式 https://spark.apache.org/docs/3.0.0/running-on-kubernetes.html ，在该模式下提交Spark作业仍然延用了spark-submit命令，并且通过指定K8S集群的ApiServer地址作为master来提交Spark作业，该方式目前对于Spark作业的管理功能较为简单，并且缺乏统一的资源调度和管理能力。

### 介绍
在 Spark 2.3 中，除了独立的调度程序、Mesos 和 Yarn 之外，Kubernetes 成为 Spark 的官方调度程序后端。与在 Kubernetes 上部署独立 Spark 集群并提交应用程序以在独立集群上运行的替代方法相比，将 Kubernetes 作为本地调度程序后端提供了一些重要的好处，如 [ SPARK-18278]（https://issues.apache.org/jira/browse/SPARK-18278），这是一个巨大的飞跃。然而，管理 Spark 应用程序生命周期的方式，例如如何提交应用程序以在 Kubernetes 上运行以及如何跟踪应用程序状态，与 Kubernetes 上的其他类型的工作负载（例如 Deployments、DaemonSets 和 StatefulSets）有很大不同. Apache Spark 的 Kubernetes Operator 缩小了差距，并允许在 Kubernetes 上以惯用的方式指定、运行和监控 Spark 应用程序。

具体来说，Apache Spark 的 Kubernetes Operator 遵循了利用 [ operator ](https://coreos.com/blog/introducing-operators.html) 模式管理 Kubernetes 集群上 Spark 应用程序生命周期的最新趋势。操作员允许以声明方式（例如，在 YAML 文件中）指定 Spark 应用程序并运行，而无需处理 spark 提交过程。它还支持跟踪和呈现 Spark 应用程序的状态，就像 Kubernetes 上的其他类型的工作负载一样。本文档讨论了运营商的设计和架构。有关 Spark 应用规范的 [ CustomResourceDefinition ](https://kubernetes.io/docs/concepts/api-extension/custom-resources/) 文档，请参考 [API定义](api-docs.md)      


具体来说，Apache Spark 的 Kubernetes Operator 遵循了利用Operator模式管理 Kubernetes 集群上 Spark 应用程序生命周期的最新趋势。操作员允许以声明方式（例如，在 YAML 文件中）指定 Spark 应用程序并运行，而无需处理 spark 提交过程。它还支持跟踪和呈现 Spark 应用程序的状态，就像 Kubernetes 上的其他类型的工作负载一样。本文档讨论了运营商的设计和架构。有关Spark 应用规范的CustomResourceDefinition文档，请参阅API 定义

### Spark Operator架构图
![image](https://files.imgdb.cn/static/images/c4/4f/633149e916f2c2beb18ec44f.png)

Operator应用包含：
* 一个`SparkApplication`控制器，用于监视创建、更新和删除的事件，`SparkApplication`对象并作用于监视事件，
* 一个*submission runner*运行`spark-submit`来处理从控制器收到的提交，
* 一个*Spark pod 监视器*，用于监视 Spark pod 并将 pod 状态更新发送到控制器，
* 一个 [ Mutating Admission Webhook ](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)，它根据添加的 pod 上的注释处理 Spark 驱动程序和执行程序 pod 的自定义由控制器，
* 还有一个名为`sparkctl`的命令行工具，用于与操作员一起工作。

具体来说，用户使用`sparkctl`（或`kubectl`）来创建`SparkApplication`对象。`SparkApplication`控制器通过观察者从 API 服务器接收对象，创建带有`spark-submit`参数的提交，并将提交发送到*submission runner*。提交运行程序提交要运行的应用程序并创建应用程序的驱动程序 pod。启动时，驱动程序 pod 会创建执行程序 pod。当应用程序运行时，*Spark pod 监视器*监视应用程序的 Pod 并将 Pod 的状态更新发送回控制器，然后控制器相应地更新应用程序的状态。

### Helm安装Spark Operator
```
$ helm repo add spark-operator https://googlecloudplatform.github.io/spark-on-k8s-operator

$ helm install my-release spark-operator/spark-operator --namespace spark-operator --create-namespace --set webhook.enable=true

NAME: my-release
LAST DEPLOYED: Tue Sep 20 07:36:49 2022
NAMESPACE: spark-operator
STATUS: deployed
REVISION: 1
TEST SUITE: None
```
如果是GKE类型（Google GKE）的集群的话，需要先授予自己自己集群管理员权限，然后才能在1.6及更高版本的GKE集群上创建自定义角色和角色绑定。在GKE上安装图表之前运行以下命令：
```
$ kubectl create clusterrolebinding <user>-cluster-admin-binding --clusterrole=cluster-admin --user=<user>@<domain>
```
现在，可以通过检查Helm版本的状态来查看集群中运行的操作员
```
$ helm status --namespace spark-operator my-release
```

### 运行示例
要运行Spark Pi示例，请运行以下命令：
```
$ kubectl apply -f examples/spark-pi.yaml
```
以上样例文件来自 [spark-on-k8s-operator
](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator)

参考上面样例，修改image路径和应用程序jar路径，新增crd文件：
```
$ cat spark-pi3.yaml 
#
# Copyright 2017 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     https://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

apiVersion: "sparkoperator.k8s.io/v1beta2"
kind: SparkApplication
metadata:
  name: spark-pi
  namespace: default
spec:
  type: Scala
  mode: cluster
  image: "chq3272991/spark:my_spark3.2.2_v2"
  imagePullPolicy: Always
  mainClass: org.apache.spark.examples.SparkPi
  mainApplicationFile: "local:///opt/spark/examples/jars/spark-examples_2.12-3.2.2.jar"
  sparkVersion: "3.1.1"
  restartPolicy:
    type: Never
  volumes:
    - name: "test-volume"
      hostPath:
        path: "/tmp"
        type: Directory
  driver:
    cores: 1
    coreLimit: "1200m"
    memory: "512m"
    labels:
      version: 3.1.1
    serviceAccount: spark
    volumeMounts:
      - name: "test-volume"
        mountPath: "/tmp"
  executor:
    cores: 1
    instances: 1
    memory: "512m"
    labels:
      version: 3.1.1
    volumeMounts:
      - name: "test-volume"
        mountPath: "/tmp"
```

### 修改pv配置
#### 新增PV配置
```
[root@node-2 test]# cat spark-operator-pv.yaml 
apiVersion: v1
kind: PersistentVolume     # 指定为PV类型
metadata:
  name:  spark-operator-pv  # 指定PV的名称
  labels:                  # 指定PV的标签
    release: spark-operator-pv
spec:
  capacity: 
    storage: 1Gi                                 # 指定PV的容量
  accessModes:
  - ReadWriteMany                                # 指定PV的访问模式
  persistentVolumeReclaimPolicy: Delete          # 指定PV的回收策略，Recycle表示支持回收，回收完成后支持再次利用
  hostPath:                                      # 指定PV的存储类型，本文是以hostPath为例
    path: /home/hp/test                          # 文件路径
    
# 执行apply
kubectl apply -f spark-operator-pv.yaml 
# 查看pv状态
[root@node-2 test]# kubectl get pv
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                STORAGECLASS   REASON   AGE
spark-operator-pv   1Gi        RWX            Delete           Available                                                7s
```

#### 新增PVC配置
```
[root@node-2 test]# cat spark-opertor-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: spark-operator-pvc                                                                               # 指定PVC的名称
spec:
  resources:
    requests:
      storage: 1Gi                                                                               # 指定PVC 容量
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany                                                                              # 指定PVC访问模式

[root@node-2 test]# kubectl apply -f spark-opertor-pvc.yaml
persistentvolumeclaim/spark-operator-pvc created
[root@node-2 test]# kubectl get pvc
NAME                 STATUS   VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS   AGE
spark-operator-pvc   Bound    spark-operator-pv   1Gi        RWX                           6s
```

#### 修改crd文件

主要以下地方需要修改:
- .spec.volumes.persistentVolumeClaim.claimName 
- .spec.driver.volumeMounts.mountPath
- .spec.executor.volumeMounts.mountPath
```
$ cat spark-pi3.yaml 
#
# Copyright 2017 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     https://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

apiVersion: "sparkoperator.k8s.io/v1beta2"
kind: SparkApplication
metadata:
  name: spark-operator-wordcount-with-pvc
  namespace: default
spec:
  type: Scala
  mode: cluster
  image: "chq3272991/spark:my_spark3.2.2_v2"
  imagePullPolicy: IfNotPresent
  mainClass: com.example.App
  mainApplicationFile: "local:///opt/spark/examples/jars/demo-1.0-SNAPSHOT.jar"
  sparkVersion: "3.2.2"
  restartPolicy:
    type: Never
  volumes:
    - name: "test-volume"
      persistentVolumeClaim:
        claimName: spark-operator-pvc
  driver:
    cores: 1
    coreLimit: "1200m"
    memory: "512m"
    labels:
      version: 3.2.2
    serviceAccount: spark
    volumeMounts:
      - name: "test-volume"
        mountPath: "/home/hp/test/"
  executor:
    cores: 1
    instances: 2
    memory: "512m"
    labels:
      version: 3.2.2
    volumeMounts:
      - name: "test-volume"
        mountPath: "/home/hp/test/"
```

#### 查看运行结果
```
[root@node-2 ~]# kubectl get pod -o wide | grep spark-operator-wordcount-with-pvc
spark-operator-wordcount-with-pvc-driver   0/1     Completed   0               3m34s   10.42.0.114   node-1   <none>           <none>
```
查看日志记录
```
22/09/26 03:16:42 INFO BlockManagerInfo: Removed broadcast_2_piece0 on spark-operator-wordcount-with-pvc-9deb768377cb9a3e-driver-svc.default.svc:7079 in memory (size: 3.7 KiB, free: 116.9 MiB)
[(of,18), (to,16), (the,14), (your,10), (I,9), (and,9), (my,5), (so,5), (it,5), (which,5), (not,5), (by,5), (have,4), (be,4), (on,4), (were,4), (this,3), (any,3), (as,3), (causes,3), (me.,3), (for,3), (though,3), (in,3), (those,3), (that,3), (what,2), (me,2), (want,2), (is,2), (with,2), (from,2), (last,2), (will,2), (night,2), (friend,2), (could,2), (you,,2), (required,2), (was,2), (an,2), (But,2), (had,2), (demand,2), (soon,2), (you,2), (which,,2), (or,2), (you.,2), (must,2), (farther,1), (letter,,1), (	Be,1), (its,1), (utmost,1), (apprehension,1), (equal,1), (still,1), (only,1), (been,1), (eldersister,,1), (evening,,1), (opinion,1), (evil,1), (without,1), (connection,1), (parties,1), (madam,,1), (immediately,1), (returning.,1), (too,1), (offend,1), (intention,1), (preserve,1), (Pardon,1), (concern,1), (read.,1), (sentiments,1), (aside,,1), (objections,1), (certain,,1), (nearest,1), (humbling,1), (there,1), (dwelling,1), (merely,1), (design,1), (propriety,1), (betrayed,1), (own,1), (myself,1), (connection.,1), (passed,1), (them,,1), (receiving,1), (bestow,1), (acknowledged,1), (freedom,1), (am,1), (even,1), (unwillingly,,1), (pardon,1), (day,1), (myself,,1), (letter,1), (Netherfield,1), (total,1), (sense,1), (defects,1), (amidst,1), (they,1), (instances,,1), (therefore,,1), (father.,1), (paining,1), (esteemed,1), (other,1), (family,,1), (degree,1), (praise,1), (perusal,1), (because,1), (effort,1), (marriage,1), (containing,1), (situation,1), (London,,1), (,1), (alarmed,,1), (three,1), (inducement,1), (honorable,1), (almost,1), (renewal,1), (These,1), (spared,,1), (unhappy,1), (censure,,1), (comparison,1), (bestowed,1), (existing,1), (than,1), (avoid,1), (The,1), (nothing,1), (force,1), (should,1), (great,1), (following,,1), (character,1), (heightened,1), (less,1), (	My,1), (repetition,1), (conducted,1), (existing,,1), (wishes,1), (put,1), (disgusting,1), (passion,1), (most,1), (attention;,1), (feelings,,1), (no,1), (all,1), (that,,1), (written,1), (younger,1), (like,1), (generally,1), (It,1), (disposition,1), (both,1), (forget,,1), (uniformly,1), (but,1), (justice.,1), (confirmed,,1), (happiness,1), (mother's,1), (repugnance;,1), (occasionally,1), (representation,1), (both,,1), (You,1), (relations,,1), (consider,1), (briefly.,1), (let,1), (before,1), (forgotten;,1), (at,1), (offers,1), (sisters,,1), (know,,1), (occasion,,1), (give,1), (endeavored,1), (left,1), (share,1), (He,1), (every,1), (must,,1), (cannot,1), (case;,1), (displeasure,1), (a,1), (say,1), (herself,,1), (stated,,1), (consolation,1), (yourselves,1), (pains,1), (frequently,,1), (before,,1), (both.,1), (objectionable,,1), (remember,,1), (write,1), (led,1), (formation,1)]
....
22/09/26 03:16:42 INFO Executor: Finished task 1.0 in stage 7.0 (TID 9). 5099 bytes result sent to driver
22/09/26 03:16:42 INFO TaskSetManager: Finished task 1.0 in stage 7.0 (TID 9) in 158 ms on spark-operator-wordcount-with-pvc-driver (executor driver) (2/2)
....
```
查看node-1节点的目录日志输出文件，说明pv、pvc持久卷工作正常
```
[root@node-1 ~]# cd /home/hp/test
[root@node-1 test]# ls
job.log
[root@node-1 test]# ls -l
total 4
-rw-r--r-- 1 185 root 2682 Sep 26 11:16 job.log
[root@node-1 test]# cat ./job.log 
[(of,18), (to,16), (the,14), (your,10), (I,9), (and,9), (my,5), (so,5), (it,5), (which,5), (not,5), (by,5), (have,4), (be,4), (on,4), (were,4), (this,3), (any,3), (as,3), (causes,3), (me.,3), (for,3), (though,3), (in,3), (those,3), (that,3), (what,2), (me,2), (want,2), (is,2), (with,2), (from,2), (last,2), (will,2), (night,2), (friend,2), (could,2), (you,,2), (required,2), (was,2), (an,2), (But,2), (had,2), (demand,2), (soon,2), (you,2), (which,,2), (or,2), (you.,2), (must,2), (farther,1), (letter,,1), (	Be,1), (its,1), (utmost,1), (apprehension,1), (equal,1), (still,1), (only,1), (been,1), (eldersister,,1), (evening,,1), (opinion,1), (evil,1), (without,1), (connection,1), (parties,1), (madam,,1), (immediately,1), (returning.,1), (too,1), (offend,1), (intention,1), (preserve,1), (Pardon,1), (concern,1), (read.,1), (sentiments,1), (aside,,1), (objections,1), (certain,,1), (nearest,1), (humbling,1), (there,1), (dwelling,1), (merely,1), (design,1), (propriety,1), (betrayed,1), (own,1), (myself,1), (connection.,1), (passed,1), (them,,1), (receiving,1), (bestow,1), (acknowledged,1), (freedom,1), (am,1), (even,1), (unwillingly,,1), (pardon,1), (day,1), (myself,,1), (letter,1), (Netherfield,1), (total,1), (sense,1), (defects,1), (amidst,1), (they,1), (instances,,1), (therefore,,1), (father.,1), (paining,1), (esteemed,1), (other,1), (family,,1), (degree,1), (praise,1), (perusal,1), (because,1), (effort,1), (marriage,1), (containing,1), (situation,1), (London,,1), (,1), (alarmed,,1), (three,1), (inducement,1), (honorable,1), (almost,1), (renewal,1), (These,1), (spared,,1), (unhappy,1), (censure,,1), (comparison,1), (bestowed,1), (existing,1), (than,1), (avoid,1), (The,1), (nothing,1), (force,1), (should,1), (great,1), (following,,1), (character,1), (heightened,1), (less,1), (	My,1), (repetition,1), (conducted,1), (existing,,1), (wishes,1), (put,1), (disgusting,1), (passion,1), (most,1), (attention;,1), (feelings,,1), (no,1), (all,1), (that,,1), (written,1), (younger,1), (like,1), (generally,1), (It,1), (disposition,1), (both,1), (forget,,1), (uniformly,1), (but,1), (justice.,1), (confirmed,,1), (happiness,1), (mother's,1), (repugnance;,1), (occasionally,1), (representation,1), (both,,1), (You,1), (relations,,1), (consider,1), (briefly.,1), (let,1), (before,1), (forgotten;,1), (at,1), (offers,1), (sisters,,1), (know,,1), (occasion,,1), (give,1), (endeavored,1), (left,1), (share,1), (He,1), (every,1), (must,,1), (cannot,1), (case;,1), (displeasure,1), (a,1), (say,1), (herself,,1), (stated,,1), (consolation,1), (yourselves,1), (pains,1), (frequently,,1), (before,,1), (both.,1), (objectionable,,1), (remember,,1), (write,1), (led,1), (formation,1)]
```

参考：
https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/
https://www.cnblogs.com/chq3272991/p/16707175.html