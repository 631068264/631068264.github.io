### Kuberbetes Native模式
Native模式简而言之就是将Driver和Executor Pod化，用户将之前向Yarn提交Spark作业的方式提交给Kubernetes的apiServer，提交命令如下：
```
$ bin/spark-submit \
    --master k8s://https://<k8s-apiserver-host>:<k8s-apiserver-port> \
    --deploy-mode cluster \
    --name spark-pi \
    --class org.apache.spark.examples.SparkPi \
    --conf spark.executor.instances=5 \
    --conf spark.kubernetes.container.image=<spark-image> \
    local:///path/to/examples.jar
```

### Native模式工作原理
![image](https://pic1.imgdb.cn/item/63316e9f16f2c2beb1b6b9d5.png)
- spark-submit可直接用于将Spark应用程序提交到Kubernetes集群。提交机制的工作原理如下：
- Spark创建在Kubernetes pod中运行的Spark驱动程序。
- 驱动程序创建也在Kubernetes pod中运行的执行程序并连接到它们，并执行应用程序代码。
- 当应用程序完成时，executor pod终止并被清理，但驱动程序pod 保留日志并在Kubernetes API中保持“已完成”状态，直到最终被垃圾收集或手动清理。

### 测试环境机器分布情况：
k8s集群情况：
```
$ kubectl get nodes
NAME     STATUS   ROLES                      AGE   VERSION
node-1   Ready    controlplane,etcd,worker   49d   v1.23.7
node-2   Ready    etcd,worker                49d   v1.23.7
```
hadoop集群情况：
| 主机名 | IP | 用户 | HDFS | Yarn |
| --- | --- | --- | --- | --- |
| node-4 | 192.168.208.132 | hp | NameNode、DataNode | NodeManager、ResourceManager |
| node-5 | 192.168.208.133 | hp | DataNode、SecondaryNameNode | NodeManager |
| node-6 | 192.168.208.134 | hp | DataNode | NodeManager |

### 基于spark 提供的pi运算案例实践
1. 获取当前k8s集群master信息：
```
kubectl cluster-info
Kubernetes control plane is running at https://192.168.208.136:8443/k8s/clusters/c-6j7vn
CoreDNS is running at https://192.168.208.136:8443/k8s/clusters/c-6j7vn/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

2. 下载spark 3.2.2安装包，基于安装包构建docker镜像
```
$ ./bin/docker-image-tool.sh -r <repo> -t my-tag build
$ ./bin/docker-image-tool.sh -r <repo> -t my-tag push

本地实践:dodo
./bin/docker-image-tool.sh -r chq3272991 -t my_spark3.2.2 build
./bin/docker-image-tool.sh -r chq3272991 -t my_spark3.2.2 push
生成路径： chq3272991/spark:my_spark3.2.2的镜像
[root@node-1 spark-3.2.2-bin-hadoop3.2]# docker images
REPOSITORY                                            TAG                    IMAGE ID       CREATED         SIZE
chq3272991/spark                                      my_spark3.2.2          46ccc8223dd3   23 hours ago    618MB
```

3. 创建Spark用户并赋予权限
```
$ kubectl create serviceaccount spark
$ kubectl create clusterrolebinding spark-role --clusterrole=edit --serviceaccount=default:spark --namespace=default
```

4. 以集群模式启动Spark Pi
```
./bin/spark-submit \
--master k8s://https://192.168.208.136:8443/k8s/clusters/c-6j7vn \
--deploy-mode cluster \
--name spark-pi \
--class org.apache.spark.examples.SparkPi \
--conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \
--conf spark.executor.instances=1 \
--conf spark.kubernetes.container.image=chq3272991/spark:my_spark3.2.2 \
local:///opt/spark/examples/jars/spark-examples_2.12-3.2.2.jar
```
这里针对以上常用参数配置解析：
```
--master k8s://<api_server_host>:<k8s-apiserver-port>  指定spark-submit使用特定的 Kubernetes 集群
--name 指定的 Spark 应用程序名称来命名创建的 Kubernetes 资源，如驱动程序和执行程序
--class <package name>  指定spark需要执行的应用程序jar包入口包名
--conf spark.kubernetes.authenticate.driver.serviceAccountName 配置属性指定驱动程序 pod 使用的自定义服务帐户（默认default）
--conf spark.executor.instances 该参数用于设置Spark作业总共要用多少个Executor进程来执行（传统Yarn模式，建议50~100）
--conf spark.kubernetes.container.image pod容器镜像
local://path 指定对应Docker映像中的示例jar的位置
```

5. 查看spark应用在kubernete上的运行结果
```
$ kubectl get pod
NAME                                READY   STATUS      RESTARTS      AGE

spark-pi-0d859683406935da-driver    0/1     Completed   0             94s

$ kubectl logs spark-pi-0d859683406935da-driver

22/09/15 09:09:56 INFO TaskSchedulerImpl: Killing all running tasks in stage 0: Stage finished
22/09/15 09:09:56 INFO DAGScheduler: Job 0 finished: reduce at SparkPi.scala:38, took 1.238436 s
Pi is roughly 3.1474157370786853
22/09/15 09:09:56 INFO SparkUI: Stopped Spark web UI at http://spark-pi-0d859683406935da-driver-svc.default.svc:4040
22/09/15 09:09:56 INFO KubernetesClusterSchedulerBackend: Shutting down all executors
```

### 测试wordcount java应用案例实践
新增调用远程hdfs wordcount案例测试， 目的结合pv卷、pvc持久卷使用， 并调用远程HDFS分布式文件存储，以贴合实践工程项目实践
1. 测试案例需要结合之前的hadoop集群搭建，和新增的java案例
- hadoop集群(https://www.cnblogs.com/chq3272991/p/16650000.html) ， 记录core-site.xml 的fs.defaultFS配置NameNode地址和端口，以提供java访问远程hdfs
- wordcount案例（https://www.cnblogs.com/chq3272991/p/16707175.html）， 生成jar应用之后，重新放置到spark解压包的example文件夹，打包镜像：spark:my_spark3.2.2_v2

2. 因为需要调用到远程HDFS，通过node-4 主机名访问，因此配置coredns
```
kubectl edit configmaps -n kube-system coredns

# 添加
hosts {
  192.168.208.132 node-4
  fallthrough
}
```

3. pv、pvc创建
3.1 创建PV（hostPath作存储卷）
```
apiVersion: v1
kind: PersistentVolume     # 指定为PV类型
metadata:
  name:  spark-pv  # 指定PV的名称
  labels:                  # 指定PV的标签
    release: spark-pv
spec:
  capacity: 
    storage: 1Gi  # 指定PV的容量
  accessModes:
  - ReadWriteMany    # 指定PV的访问模式
  persistentVolumeReclaimPolicy: Delete   # 指定PV的回收策略，Recycle表示支持回收，回收完成后支持再次利用
  hostPath: # 指定PV的存储类型，本文是以hostPath为例
    path: /home/hp/test   # 文件路径
```
部署pv，查看pv状态，Available说明卷是一个空闲资源，尚未绑定到任何pvc:
```
$ kubectl get pv
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
spark-pv   1Gi        RWX            Delete           Available                                   9m23s
```
3.2 创建pvc
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: spark-pvc2          # 指定PVC的名称
spec:
  resources:
    requests:
      storage: 1Gi           # 指定PVC 容量
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany          
```
部署pvc，查看pvc状态，Bound说明已绑定到pv
```
$ kubectl get pv,pvc
NAME                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                STORAGECLASS   REASON   AGE
persistentvolume/spark-pv   1Gi        RWX            Delete           Bound    default/spark-pvc2                           36m

NAME                               STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/spark-pvc2   Bound    spark-pv   1Gi        RWX                           16m
```

3.3 执行spark-submit
```
./bin/spark-submit \
--master k8s://https://192.168.208.136:8443/k8s/clusters/c-6j7vn \
--deploy-mode cluster \
--name WordCount_Java \
--class com.example.App \
--conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \
--conf spark.executor.instances=1 \
--conf spark.kubernetes.container.image=chq3272991/spark:my_spark3.2.2_v2 \
--conf spark.kubernetes.driver.volumes.persistentVolumeClaim.data.options.claimName=spark-pvc2 \
--conf spark.kubernetes.driver.volumes.persistentVolumeClaim.data.mount.path=/home/hp/test/  \
local:///opt/spark/examples/jars/demo-1.0-SNAPSHOT.jar
```
执行spark-submit之后，实际上可以查看到启动的pod配置：
```
kubectl get pod wordcount-java-c90731835e071111-driver -o yaml
apiVersion: v1
kind: Pod
metadata:
.....
spec:
  containers:
      ..... 
    volumeMounts:                     # spark.kubernetes.driver.volumes.[VolumeType].[VolumeName].mount.path=<mount path>
    - mountPath: /home/hp/test/        # 对应参数： <mount path>
      name: data                        # 对应参数： [VolumeName]
    - mountPath: /var/data/spark-9d9b67f8-d7e6-4e61-8678-6e5db236f48a # 创建pod 并为列出的每个目录或环境变量安装一个emptyDir卷
      name: spark-local-dir-1  # 没有卷设置为本地存储，Spark 会在 shuffle 和其他操作期间使用临时暂存空间将数据溢出到磁盘，
    - mountPath: /opt/spark/conf
      name: spark-conf-volume-driver
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-ffmb5
      readOnly: true
     ..... 
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: spark-pvc2  
```

3.4 查看spark执行结果：
这里主要有两种持久化日志查看，一种是程序编码中集成日志输出（如写log文件，或elk），一种是kubectl logs pod查看pod日志
```
# 切换到pod工作节点查看输出目录
$ cat /home/hp/test/job.log
[(of,18), (to,16), (the,14), (your,10), (I,9), (and,9), (my,5), (so,5), (it,5), (which,5), (not,5), (by,5), (have,4), (be,4), (on,4), (were,4),
....

# 查看pod的log日志
.....
22/09/21 03:11:18 INFO SparkContext: Running Spark version 3.2.2
22/09/21 03:11:18 INFO SparkContext: Submitted application: WordCount_Java        # 开始提交任务（jar包）
....
22/09/21 03:11:19 INFO Utils: Successfully started service 'sparkDriver' on port 7078.  # 启动driver
22/09/21 03:11:19 INFO SparkEnv: Registering MapOutputTracker                           
22/09/21 03:11:19 INFO SparkEnv: Registering BlockManagerMaster
22/09/21 03:11:19 INFO BlockManagerMasterEndpoint: Using org.apache.spark.storage.DefaultTopologyMapper for getting topology information
22/09/21 03:11:19 INFO BlockManagerMasterEndpoint: BlockManagerMasterEndpoint up
22/09/21 03:11:19 INFO SparkEnv: Registering BlockManagerMasterHeartbeat
22/09/21 03:11:19 INFO DiskBlockManager: Created local directory at /var/data/spark-9d9b67f8-d7e6-4e61-8678-6e5db236f48a/blockmgr-93c2ab05-78de-40cf-aa70-c8e83243f72c
22/09/21 03:11:20 INFO MemoryStore: MemoryStore started with capacity 413.9 MiB
22/09/21 03:11:20 INFO SparkEnv: Registering OutputCommitCoordinator
22/09/21 03:11:20 INFO Utils: Successfully started service 'SparkUI' on port 4040.
22/09/21 03:11:20 INFO SparkUI: Bound SparkUI to 0.0.0.0, and started at http://wordcount-java-c90731835e071111-driver-svc.default.svc:4040
22/09/21 03:11:20 INFO SparkContext: Added JAR local:///opt/spark/examples/jars/demo-1.0-SNAPSHOT.jar at file:/opt/spark/examples/jars/demo-1.0-SNAPSHOT.jar with timestamp 1663729878519
22/09/21 03:11:20 INFO SparkContext: The JAR local:///opt/spark/examples/jars/demo-1.0-SNAPSHOT.jar at file:/opt/spark/examples/jars/demo-1.0-SNAPSHOT.jar has been added already. Overwriting of added jar is not supported in the current version.
22/09/21 03:11:21 INFO Executor: Starting executor ID driver on host wordcount-java-c90731835e071111-driver
22/09/21 03:11:21 INFO Executor: Fetching file:/opt/spark/examples/jars/demo-1.0-SNAPSHOT.jar with timestamp 1663729878519
22/09/21 03:11:21 INFO Utils: Copying /opt/spark/examples/jars/demo-1.0-SNAPSHOT.jar to /var/data/spark-9d9b67f8-d7e6-4e61-8678-6e5db236f48a/spark-e378032a-b3aa-4859-9f84-3fc2894cfb1a/userFiles-9d9b3d34-0125-46ad-8bdf-ff41c38f0386/demo-1.0-SNAPSHOT.jar
22/09/21 03:11:21 INFO Executor: Adding file:/var/data/spark-9d9b67f8-d7e6-4e61-8678-6e5db236f48a/spark-e378032a-b3aa-4859-9f84-3fc2894cfb1a/userFiles-9d9b3d34-0125-46ad-8bdf-ff41c38f0386/demo-1.0-SNAPSHOT.jar to class loader
22/09/21 03:11:21 INFO Utils: Successfully started service 'org.apache.spark.network.netty.NettyBlockTransferService' on port 7079.
....
22/09/21 03:11:25 INFO Executor: Running task 1.0 in stage 0.0 (TID 1)
22/09/21 03:11:25 INFO Executor: Running task 0.0 in stage 0.0 (TID 0)
22/09/21 03:11:26 INFO HadoopRDD: Input split: hdfs://node-4:9000/home/hp/word.txt:1083+1083        # 读取hdfs文件
22/09/21 03:11:26 INFO HadoopRDD: Input split: hdfs://node-4:9000/home/hp/word.txt:0+1083
22/09/21 03:11:26 INFO Executor: Finished task 0.0 in stage 0.0 (TID 0). 1376 bytes result sent to driver   
22/09/21 03:11:26 INFO Executor: Finished task 1.0 in stage 0.0 (TID 1). 1376 bytes result sent to driver
....
22/09/21 03:11:27 INFO TaskSchedulerImpl: Killing all running tasks in stage 4: Stage finished
22/09/21 03:11:27 INFO DAGScheduler: Job 1 finished: collect at App.java:90, took 0.424357 s        # 运行结果打印
[(of,18), (to,16), (the,14), (your,10), (I,9), (and,9), (my,5), (so,5), (it,5), (which,5), (not,5), (by,5), (have,4), (be,4), (on,4), (were,4),
....
```
此次，说明spark native结合pv、pvc和远程hdfs调用成功

### 报错汇总
`镜像打包错误：`
```
[root@node-1 spark-3.2.2-bin-hadoop3.2]# ./bin/docker-image-tool.sh -r chq3272991 -t my_spark3.2.2 build
Sending build context to Docker daemon  339.1MB
Step 1/18 : ARG java_image_tag=11-jre-slim
Step 2/18 : FROM openjdk:${java_image_tag}
11-jre-slim: Pulling from library/openjdk
a2abf6c4d29d: Already exists 
.....
+ apt-get update
Get:1 http://security.debian.org/debian-security bullseye-security InRelease [48.4 kB]
Get:2 http://security.debian.org/debian-security bullseye-security/main amd64 Packages [183 kB]
Err:3 https://deb.debian.org/debian bullseye InRelease
  Could not connect to deb.debian.org:443 (151.101.110.132). - connect (111: Connection refused)
Get:4 https://deb.debian.org/debian bullseye-updates InRelease [44.1 kB]
Get:5 https://deb.debian.org/debian bullseye-updates/main amd64 Packages [2596 B]
Fetched 278 kB in 32s (8733 B/s)
Reading package lists...
W: Failed to fetch https://deb.debian.org/debian/dists/bullseye/InRelease  Could not connect to deb.debian.org:443 (151.101.110.132). - connect (111: Connection refused)
W: Some index files failed to download. They have been ignored, or old ones used instead.
....

The command '/bin/sh -c set -ex &&     sed -i 's/http:\/\/deb.\(.*\)/https:\/\/deb.\1/g' /etc/apt/sources.list &&     apt-get update &&     ln -s /lib /lib64 &&     apt install -y bash tini libc6 libpam-modules krb5-user libnss3 procps &&     mkdir -p /opt/spark &&     mkdir -p /opt/spark/examples &&     mkdir -p /opt/spark/work-dir &&     touch /opt/spark/RELEASE &&     rm /bin/sh &&     ln -sv /bin/bash /bin/sh &&     echo "auth required pam_wheel.so use_uid" >> /etc/pam.d/su &&     chgrp root /etc/passwd && chmod ug+rw /etc/passwd &&     rm -rf /var/cache/apt/*' returned a non-zero code: 100
Failed to build Spark JVM Docker image, please refer to Docker build output for details.
```
解决办法：
centos 7.6中，官方spark提供的Dockerfile，涉及一些debian（ubuntu）更新，网络链接出现问题。可以通过翻墙配置代理，解决以上问题。另一种方法是修改Dockerfile文件，对镜像包在联网环境执行成功之后，重新打包新的镜像包。

`Jar应用路径错误：`
```
22/09/15 02:18:03 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Using Spark's default log4j profile: org/apache/spark/log4j-defaults.properties
22/09/15 02:18:03 WARN DependencyUtils: Local jar /opt/spark/examples/spark-examples_2.12-3.2.2.jar does not exist, skipping.
22/09/15 02:18:03 WARN DependencyUtils: Local jar /opt/spark/examples/spark-examples_2.12-3.2.2.jar does not exist, skipping.
Error: Failed to load class org.apache.spark.examples.SparkPi.
22/09/15 02:18:03 INFO ShutdownHookManager: Shutdown hook called
```
解决办法：
查看打包时的DockFile文件确认相关Jar应用路径，或者运行镜像，切换到容器里面查看：
```
docker run -it chq3272991/spark:my_spark3.2.2 /bin/bash
185@33a193caa61d:/opt/spark/examples/jars$ pwd
/opt/spark/examples/jars
185@33a193caa61d:/opt/spark/examples/jars$ ls
scopt_2.12-3.7.1.jar  spark-examples_2.12-3.2.2.jar
```

`账号权限相关：`
```
Caused by: io.fabric8.kubernetes.client.KubernetesClientException: Failure executing: GET at: https://kubernetes.default.svc/api/v1/namespaces/default/pods/spark-pi-1c8a42833eeab069-driver. Message: Forbidden!Configured service account doesn't have access. Service account may have been revoked. pods "spark-pi-1c8a42833eeab069-driver" is forbidden: User "system:serviceaccount:default:default" cannot get resource "pods" in API group "" in the namespace "default".
```
解决办法：
创建相关账号，绑定角色对应命名空间的编辑权限

`hostPath路径映射权限相关：`
```
22/09/21 03:07:02 INFO SparkContext: Successfully stopped SparkContext
Exception in thread "main" java.io.FileNotFoundException: /home/hp/test/job.log (Permission denied)
at java.base/java.io.FileOutputStream.open0(Native Method)
at java.base/java.io.FileOutputStream.open(Unknown Source)
```
解决办法：
修改/home/hp/test 权限(或其他方法，如chown spark:spark -R /home/hp/test/ )
```
chmod 777 -R /home/hp/test/
```

`spark在kubernetes执行任务完成之后，再暴露web UI端口`
```
[root@node-2 test]# kubectl port-forward --address 0.0.0.0 wordcount-java-c90731835e071111-driver 4040:4040
error: unable to forward port because pod is not running. Current status=Succeeded
```
修改项目大数据应用，工作完成之后，保持链接，这样spark启动的pod就会处于running状态，最后确认工作结束了，再通过kill来结束pod

参考：
https://spark.apache.org/docs/3.2.2/running-on-kubernetes.html
https://www.helloworld.net/p/9534492240
https://blog.csdn.net/zjjcchina/article/details/126036228
https://developer.aliyun.com/article/768355
https://blog.csdn.net/aaronhadoop/article/details/121586848


