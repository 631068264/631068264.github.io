###  部署思路
分两部分来看：
一、部署HDFS集群，搭建分布式文件系统HDFS
- NameNode节点对外暴露端口9000、50070
- 根据pod容器env定义不同的节点类型，编写startnode.sh启动脚本
- HDFS master节点Pod定义
- HDFS DataNode节点Pod定义
二、部署Yarn资源调度集群

### HDFS集群
HDFS集群是由NameNode（Master节点）和Datanode（数据节点）等两类节点所组成，其中，客户端程序（Client）以及DataNode节点会访问NameNode，因此，NameNode节点需要建模为Kubernetes Service以提供服务
其中， NameNode节点暴露2个服务端口：
- 9000端口用于内部IPC通讯，主要用于获取文件的元数据
- 50070端口用于HTTP服务，为Hadoop的Web管理使用

另外，为了减少镜像打包次数，可以在HDFS的NameNode节点、DataNode节点、Yarn的NodeManager节点配置容器的环境变量HADOOP_NODE_TYPE，以便区分不同的节点类型，共享一个镜像结合下面镜像里面的启动脚本startnode.sh来执行

#### 打包镜像文件分布情况
打包文件分布：
```
[root@node-1 hadoop_docker_package]# pwd
/home/hadoop_docker_package
[root@node-1 hadoop_docker_package]# ls
Dockerfile  etc  etc-xml  hadoop-3.2.4  hadoop-3.2.4.tar.gz  jdk-8u311-linux-x64.tar.gz  startnode.sh
```

#### Dockerfile文件
```
#hadoop 3.2.4 容器化配置
FROM centos:7

ARG workdir=/home/hp

RUN mkdir -p $workdir
COPY jdk-8u311-linux-x64.tar.gz $workdir

RUN cd $workdir && \
    tar -zxf ./jdk-8u311-linux-x64.tar.gz

COPY hadoop-3.2.4.tar.gz $workdir
RUN cd $workdir && \
    tar -zxf ./hadoop-3.2.4.tar.gz

# java
ENV JAVA_HOME=/home/hp/jdk1.8.0_311
ENV CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV PATH=$PATH:$JAVA_HOME/bin

# hadoop
ENV HADOOP_HOME=$workdir/hadoop-3.2.4
ENV HADOOP_MAPRED_HOME=$HADOOP_HOME
ENV HADOOP_COMMON_HOME=$HADOOP_HOME
ENV HADOOP_HDFS_HOME=$HADOOP_HOME
ENV YARN_HOME=$HADOOP_HOME
ENV HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
ENV PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
ENV HADOOP_INSTALL=$HADOOP_HOME
ENV JAVA_LIBRARY_PATH=/home/hp/hadoop-3.2.4/lib/native

# hdfs-site.xml配置需要路径
# dfs.namenode.name.dir = file:/home/hp/hadoop-data/name
# dfs.datanode.data.dir = file:/home/hp/hadoop-data/data
RUN mkdir -p $HADOOP_HOME/hdfs-data/name && \
    mkdir -p $HADOOP_HOME/hdfs-data/data && \
    mkdir -p $HADOOP_HOME/logs

# 用完压缩包可删除
RUN cd $workdir && rm -rf ./jdk-8u311-linux-x64.tar.gz && rm -rf ./hadoop-3.2.4.tar.gz

RUN chown -R root:root $HADOOP_HOME
COPY startnode.sh /
RUN chmod +x /startnode.sh
EXPOSE 22 50070 9000

RUN hadoop version
RUN java -version

ENTRYPOINT ["/startnode.sh"]
```

#### startnode.sh文件
```
#!/bin/bash
set -e
# service ssh start
hdfs_dir=$HADOOP_HOME/hdfs-data/
if [ $HADOOP_NODE_TYPE = "datanode" ]; then
  echo -e "\033[32m start datanode \033[0m"
  $HADOOP_HOME/bin/hdfs datanode -regular
fi
if [ $HADOOP_NODE_TYPE = "namenode" ]; then
  if [ -z $(ls -A ${hdfs_dir}) ]; then
    echo -e "\033[32m start hdfs namenode format \033[0m"
    $HADOOP_HOME/bin/hdfs namenode -format
  fi
  echo -e "\033[32m start hdfs namenode \033[0m"
  $HADOOP_HOME/bin/hdfs namenode
fi

if [ $HADOOP_NODE_TYPE = "yarn-master" ]; then
  echo "Start yarn master(resourceManager) ...."
  yarn resourcemanager
  mapred --daemon start historyserver
else
   if [ $HADOOP_NODE_TYPE = "yarn-nodemanager" ]; then
      echo "Start Yarn nodeManager ...."
      yarn nodemanager
   fi
fi
```

#### 镜像打包
打包命令：
```
docker build -t hadoop_k8s_3.2.4:v1 .
docker tag hadoop_k8s_3.2.4:v1 chq3272991/spark:hadoop_3.2.4_v1
docker push chq3272991/spark:hadoop_3.2.4_v1

# 测试容器，查看文件情况
docker run -itd --name test-hdfs hadoop_k8s_3.2.4:v1
[root@node-1 hadoop_k8s_yaml]# docker ps -a | grep hdfs
41241116ec33   hadoop_k8s_3.2.4:v1                   "/startnode.sh"          4 seconds ago   Exited (0) 3 seconds ago             test-hdfs
# 查看容器日志，方便调试
docker logs test-hdfs
```

### K8s部署相关yaml文件
#### ConfigMap文件（hdfs\yarn集群共用）
configmap.yaml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: hadoop
  namespace: big-data
  labels:
    app: hadoop
data:
  hadoop-env.sh: |
    export HDFS_DATANODE_USER=root
    export HDFS_NAMENODE_USER=root
    export HDFS_SECONDARYNAMENODE_USER=root
    export JAVA_HOME=/home/hp/jdk1.8.0_311
    export HADOOP_HOME=/home/hp/hadoop-3.2.4
    export HADOOP_OS_TYPE=${HADOOP_OS_TYPE:-$(uname -s)}
    export HADOOP_OPTS="-Djava.library.path=${HADOOP_HOME}/lib/native"
  core-site.xml: |
    <?xml version="1.0" encoding="UTF-8"?>
    <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
    <configuration>
      <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hadoop-master:9000</value>
      </property>
      <property>
        <name>hadoop.tmp.dir</name>
        <value>file:/home/hp/hadoop-3.2.4/hdfs-data/tmp</value>
      </property>
      <property>
        <name>dfs.namenode.rpc-bind-host</name>
        <value>0.0.0.0</value>
      </property>
    </configuration>
  hdfs-site.xml: |
    <?xml version="1.0" encoding="UTF-8"?>
    <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
    <configuration>
      <property>
         <name>dfs.namenode.http-address</name>
         <value>0.0.0.0:50070</value>
      </property>
      <property>
        <name>dfs.replication</name>
        <value>1</value>
      </property>
      <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/home/hp/hadoop-3.2.4/hdfs-data/name</value>
     </property>
     <property>
      <name>dfs.datanode.data.dir</name>
      <value>file:/home/hp/hadoop-3.2.4/hdfs-data/data</value>
     </property>
     <property>
        <name>dfs.namenode.datanode.registration.ip-hostname-check</name>
        <value>false</value>
      </property>
    </configuration>
  mapred-site.xml: |
    <?xml version="1.0" encoding="UTF-8"?>
    <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
    <configuration>
        <property>
            <name>mapreduce.framework.name</name>
            <value>yarn</value>
        </property>
        <property>
            <name>mapreduce.jobhistory.address</name>
            <value>yarn-master:10020</value>
        </property>
        <property>
            <name>mapreduce.jobhistory.webapp.address</name>
            <value>yarn-master:19888</value>
        </property>
        <property>
            <name>yarn.app.mapreduce.am.env</name>
            <value>HADOOP_MAPRED_HOME=/home/hp/hadoop-3.2.4</value>
        </property>
        <property>
            <name>mapreduce.map.env</name>
            <value>HADOOP_MAPRED_HOME=/home/hp/hadoop-3.2.4</value>
        </property>
        <property>
            <name>mapreduce.reduce.env</name>
            <value>HADOOP_MAPRED_HOME=/home/hp/hadoop-3.2.4</value>
        </property>
    </configuration>
  yarn-site.xml: |
    <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
    <configuration>
        <property>
            <name>yarn.resourcemanager.address</name>
            <value>yarn-master:8032</value>
        </property>
        <property>
            <name>yarn.resourcemanager.scheduler.address</name>
            <value>yarn-master:8030</value>
        </property>
        <property>
            <name>yarn.resourcemanager.resource-tracker.address</name>
            <value>yarn-master:8031</value>
        </property>
        <property>
            <name>yarn.resourcemanager.admin.address</name>
            <value>yarn-master:8033</value>
        </property>
        <property>
            <name>yarn.nodemanager.aux-services</name>
            <value>mapreduce_shuffle</value>
            <final>true</final>
        </property>
    </configuration>
```

### HDFS集群部署
#### namenode的service文件
hdfs-service.yaml
```
# namenode svc
apiVersion: v1
kind: Service
metadata:
  name: hadoop-master
  namespace: big-data
spec:
  selector:
    app: hadoop-namenode
  type: NodePort
  ports:
    - name: rpc
      port: 9000
      targetPort: 9000
    - name: http
      port: 50070
      targetPort: 50070
      nodePort: 32393
```
集群内部可以通过9000访问namenode，并对外公开端口50070（通过32393访问）

#### namenode的deployment文件
namenode-deployment-emptydir.yaml 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hadoop-namenode
  namespace: big-data
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: hadoop-namenode
  template:
    metadata:
      labels:
        app: hadoop-namenode
    spec:
      volumes:
        - name: hadoop-env
          configMap:
            name: hadoop
            items:
              - key: hadoop-env.sh
                path: hadoop-env.sh
        - name: core-site
          configMap:
            name: hadoop
            items:
              - key: core-site.xml
                path: core-site.xml
        - name: hdfs-site
          configMap:
            name: hadoop
            items:
              - key: hdfs-site.xml
                path: hdfs-site.xml
        - name: hadoop-data
         # persistentVolumeClaim:
         #   claimName: data-hadoop-namenode
          emptyDir: {}
      containers:
        - name: hadoop
          image: hadoop_k8s_3.2.4:v1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 22
            - containerPort: 9000
            - containerPort: 50070
          volumeMounts:
            - name: hadoop-env
              mountPath: /home/hp/hadoop-3.2.4/etc/hadoop/hadoop-env.sh
              subPath: hadoop-env.sh
            - name: core-site
              mountPath: /home/hp/hadoop-3.2.4/etc/hadoop/core-site.xml
              subPath: core-site.xml
            - name: hdfs-site
              mountPath: /home/hp/hadoop-3.2.4/etc/hadoop/hdfs-site.xml
              subPath: hdfs-site.xml
            - name: hadoop-data
              mountPath: /home/hp/hadoop-3.2.4/hdfs-data/ 
              subPath: hdfs-data
            - name: hadoop-data
              mountPath: /home/hp/hadoop-3.2.4/logs/
              subPath: logs
          env:
            - name: HADOOP_NODE_TYPE
              value: namenode
```

#### datanode配置
datanode-DaemonSet.yaml
```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: hadoop-datanode
  namespace: big-data
spec:
  selector:
    matchLabels:
      app: hadoop-datanode
  template:
    metadata:
      labels:
        app: hadoop-datanode
    spec:
      volumes:
        - name: hadoop-env
          configMap:
            name: hadoop
            items:
              - key: hadoop-env.sh
                path: hadoop-env.sh
        - name: core-site
          configMap:
            name: hadoop
            items:
              - key: core-site.xml
                path: core-site.xml
        - name: hdfs-site
          configMap:
            name: hadoop
            items:
              - key: hdfs-site.xml
                path: hdfs-site.xml
        - name: hadoop-data
         # persistentVolumeClaim:
         #   claimName: data-hadoop-namenode
          emptyDir: {}
      containers:
        - name: hadoop
          image: chq3272991/spark:hadoop_3.2.4_v1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 22
            - containerPort: 9000
            - containerPort: 50070
          volumeMounts:
            - name: hadoop-env
              mountPath: /home/hp/hadoop-3.2.4/etc/hadoop/hadoop-env.sh
              subPath: hadoop-env.sh
            - name: core-site
              mountPath: /home/hp/hadoop-3.2.4/etc/hadoop/core-site.xml
              subPath: core-site.xml
            - name: hdfs-site
              mountPath: /home/hp/hadoop-3.2.4/etc/hadoop/hdfs-site.xml
              subPath: hdfs-site.xml
            - name: hadoop-data
              mountPath: /home/hp/hadoop-3.2.4/hdfs-data/ 
              subPath: hdfs-data
            - name: hadoop-data
              mountPath: /home/hp/hadoop-3.2.4/logs/
              subPath: logs
          env:
            - name: HADOOP_NODE_TYPE
              value: datanode
```



#### pv和pvc配置(遇到subPath挂载目录问题)
pv配置
```
apiVersion: v1
kind: PersistentVolume     # 指定为PV类型
metadata:
  name: hadoop-pv  # 指定PV的名称
  labels:                  # 指定PV的标签
    release: hadoop-pv
  namespace: big-data
spec:
  capacity: 
    storage: 10Gi  # 指定PV的容量
  accessModes:
  - ReadWriteMany                                # 指定PV的访问模式
  persistentVolumeReclaimPolicy: Delete                                         # 指定PV的回收策略，Recycle表示支持回收，回收完成后支持再次利用
  hostPath: # 指定PV的存储类型，本文是以hostPath为例
    path: /home/hostpath-dir                # 文件路径
```
pvc配置
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-hadoop-namenode
  namespace: big-data
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```

#### workers(忽略，不影响)
配置工作节点(这里pod单独负责namenode功能)
```
#localhost
```

### 其他参考（测试使用）
##### NameNode节点
启动NameNode命令：
```
# 格式化NameNode

# $HADOOP_PREFIX/bin/hdfs namenode -format <cluster_name>
hdfs namenode -format

#  启动NameNode
hdfs --daemon start namenode

```
参考svc暴露端口：
```
[root@node-1 ~]# kubectl get svc | grep hadoop
k8s-hadoop-master-svc   NodePort    10.43.152.208   <none>        9000:32393/TCP,50070:31446/TCP   10d
```
访问网页打开namenode ui
http://192.168.208.130:31446/

##### DataNode节点
启动DataNode命令
```
hdfs --daemon start datanode
```
日志查看：
```
tail -n 200 /home/hp/hadoop-3.2.4/logs/hadoop-root-datanode-k8s-hadoop-datanode1.log

# 报错：
2022-10-10 03:07:43,403 INFO org.apache.hadoop.ipc.Client: Retrying connect to server: k8s-hadoop-master:9000. Already tried 9 time(s); retry policy is RetryUpToMaximumCountWithFixedSleep(maxRetries=10, sleepTime=1000 MILLISECONDS)
2022-10-10 03:07:43,424 WARN org.apache.hadoop.hdfs.server.datanode.DataNode: Problem connecting to server: k8s-hadoop-master:9000
```
解决办法，新增svc，配置网络，之后在重新重启datanode
```
kubectl expose pod k8s-hadoop-master --port=9000 --target-port=9000

hdfs --daemon stop datanode
hdfs --daemon start datanode
```

### 测试hdfs可用性
1. 参考之前的spark 3.2.2 java编程案例
https://www.cnblogs.com/chq3272991/p/16707175.html
修改hdfs访问链接：
查看svc对应9000外网访问端口31651
```
[root@node-1 hadoop_docker_package]# kubectl get svc -n big-data
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                          AGE
hadoop-master   NodePort   10.43.48.235   <none>        9000:31651/TCP,50070:32393/TCP   28m
```

```
// todo:2、构建java版的sparkContext // 原本是node4:9000
JavaSparkContext sc = new JavaSparkContext(sparkConf);
sc.hadoopConfiguration().set("dfs.nameservice", "node-1");
sc.hadoopConfiguration().set("fs.defaultFS", "hdfs://node-1:31651");
```
并重新打包jar包, 输入 **mvn package**, jar包名称为 **demo-1.0-link-k8s-hdfs.jar**

2. 重新打包，上次spark运行Native模式（k8s）的镜像
拷贝**demo-1.0-link-k8s-hdfs.jar**到spark源文件./example/jars/目录内
```
root@chq:/home/spark-3.2.2-bin-hadoop3.2# ls -l ./examples/jars/
total 1624
-rw-r--r-- 1 root       root          7578 Sep 19 15:54 demo-1.0-SNAPSHOT.jar
-rw-r--r-- 1 root       root          7580 Oct 10 14:57 demo-1.0-link-k8s-hdfs.jar
-rw-r--r-- 1 chq-ubuntu chq-ubuntu   78803 Jul 12 00:01 scopt_2.12-3.7.1.jar
-rw-r--r-- 1 chq-ubuntu chq-ubuntu 1560864 Jul 12 00:01 spark-examples_2.12-3.2.2.jar
```
输入打包命令：
```
./bin/docker-image-tool.sh -r chq3272991 -t my_spark3.2.2_v5 build
./bin/docker-image-tool.sh -r chq3272991 -t my_spark3.2.2_v5 push
```

3. hdfs文件创建和准备
```
[root@node-1 hadoop_k8s_yaml]# kubectl get pods -n big-data
NAME                               READY   STATUS    RESTARTS   AGE
hadoop-datanode-d9dfd              1/1     Running   0          29m
hadoop-datanode-qslqc              1/1     Running   0          29m
hadoop-datanode-xxm46              1/1     Running   0          29m
hadoop-namenode-7dbf567495-v6v45   1/1     Running   0          92m
# 链接namenode的pod容器
kubectl exec -it hadoop-namenode-7dbf567495-v6v45 -n big-data -- bash
# 创建hdfs文件夹
hdfs dfs -mkdir -p /home/hp
vi word.txt
# 上传文件到hdfs
hdfs dfs -put ./word.txt /home/hp
# 查看文件上传情况
hdfs dfs -ls /home/hp
Found 1 items
-rw-r--r--   1 root supergroup       2166 2022-10-10 12:32 /home/hp/word.txt
```
也可以通过Hadoop UI创建目录和上传文件（涉及更多端口配置）
http://192.168.208.130:32393/explorer.html#/

4. spark访问hdfs，读取word文件，完成单词统计
切换node-1节点，/opt/spark-3.2.2-bin-hadoop3.2/目前执行以下命令：
```
./bin/spark-submit \
--master k8s://https://192.168.208.136:8443/k8s/clusters/c-6j7vn \
--deploy-mode cluster \
--name WordCount_Java \
--class com.example.App \
--conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \
--conf spark.executor.instances=1 \
--conf spark.kubernetes.container.image=chq3272991/spark:my_spark3.2.2_v5 \
local:///opt/spark/examples/jars/wordcount-link-k8s-hdfs-31651.jar
```
这个镜像包含的wordcount-link-k8s-hdfs-31651.jar对应31651端口
另一个jar文件demo-1.0-SNAPSHOT.jar对应30173端口（另一个案例测试使用）

5. 查看spark执行日志
```
kubectl logs wordcount-java-a176fe83c207f0f2-driver
...
22/10/10 13:14:13 INFO DAGScheduler: Job 1 finished: collect at App.java:90, took 0.154206 s
[(of,18), (to,16), (the,14), (your,10), (I,9), (and,9), (my,5), (so,5), (it,5), (which,5), (not,5), (by,5), (have,4), (be,4), (on,4), (were,4), (this,3), (any,3), (as,3), (causes,3), (me.,3), (for,3), (though,3), (in,3), (those,3),
```

### Yarn集群部署
#### yarn-master的service
yarn-master-service.yaml
```
# yarn-master svc
apiVersion: v1
kind: Service
metadata:
  name: yarn-master
  namespace: big-data
spec:
  selector:
    app: yarn-master
  clusterIP: None 
  ports:
    - name: asm
      port: 8032
    - name: scheduleripc
      port: 8030
    - name: rpc
      port: 8031
    - name: ipc
      port: 8033
    - name: http  # 查看UI端口
      port: 8088
    - name: locallizeripc
      port: 8040
    - name: httpnodemanager
      port: 8042
    - name: nmcontainer
      port: 8041
    - name: jobhistoryipc
      port: 10020
    - name: jobhistoryhttp
      port: 19888
```

#### yarn-master的pod
yarn-master-pod.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: yarn-master
  namespace: big-data
  labels:
    app: yarn-master
spec:
  nodeName: node-1
  volumes:
    - name: hadoop-env
      configMap:
        name: hadoop
        items:
          - key: hadoop-env.sh
            path: hadoop-env.sh
    - name: core-site
      configMap:
        name: hadoop
        items:
          - key: core-site.xml
            path: core-site.xml
    - name: hdfs-site
      configMap:
        name: hadoop
        items:
          - key: hdfs-site.xml
            path: hdfs-site.xml
    - name: mapred-site
      configMap:
        name: hadoop
        items:
          - key: mapred-site.xml
            path: mapred-site.xml
    - name: yarn-site
      configMap:
        name: hadoop
        items:
          - key: yarn-site.xml
            path: yarn-site.xml
    - name: hadoop-data
      emptyDir: {}
  containers:
    - name: hadoop
      image: chq3272991/spark:hadoop_3.2.4_v1
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 8032
        - containerPort: 8030
        - containerPort: 8031
        - containerPort: 8033
        - containerPort: 8088
        - containerPort: 8040
        - containerPort: 8042
        - containerPort: 8041
        - containerPort: 19888
      volumeMounts:
        - name: hadoop-env
          mountPath: /home/hp/hadoop-3.2.4/etc/hadoop/hadoop-env.sh
          subPath: hadoop-env.sh
        - name: core-site
          mountPath: /home/hp/hadoop-3.2.4/etc/hadoop/core-site.xml
          subPath: core-site.xml
        - name: hdfs-site
          mountPath: /home/hp/hadoop-3.2.4/etc/hadoop/hdfs-site.xml
          subPath: hdfs-site.xml
        - name: mapred-site
          mountPath: /home/hp/hadoop-3.2.4/etc/hadoop/mapred-site.xml
          subPath: mapred-site.xml
        - name: yarn-site
          mountPath: /home/hp/hadoop-3.2.4/etc/hadoop/yarn-site.xml
          subPath: yarn-site.xml
        - name: hadoop-data
          mountPath: /home/hp/hadoop-3.2.4/hdfs-data/ 
          subPath: hdfs-data
        - name: hadoop-data
          mountPath: /home/hp/hadoop-3.2.4/logs/
          subPath: logs
      env:
        - name: HADOOP_NODE_TYPE
          value: yarn-master
```

#### yarn-node-2/yarn-node-3的service（nodemanager节点）
yarn-node-2-service.yaml
```
# yarn-master svc
apiVersion: v1
kind: Service
metadata:
  name: yarn-node-2
  namespace: big-data
spec:
  selector:
    app: yarn-node-2
  clusterIP: None 
  ports:
    - name: httpnodemanager
      port: 8042
    - name: nmcontainer
      port: 8041
    - name: local
      port: 8040
```

yarn-node-2-nodeport-service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: yarn-node-2-nodeport
  namespace: big-data
spec:
  selector:
    app: yarn-node-2
  type: NodePort
  ports:
    - name: http
      port: 8042
```

#### yarn-node-2的pod
yarn-node-2-pod.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: yarn-node-2
  namespace: big-data
  labels:
    app: yarn-node-2
spec:
  nodeName: node-2
  volumes:
    - name: hadoop-env
      configMap:
        name: hadoop
        items:
          - key: hadoop-env.sh
            path: hadoop-env.sh
    - name: core-site
      configMap:
        name: hadoop
        items:
          - key: core-site.xml
            path: core-site.xml
    - name: hdfs-site
      configMap:
        name: hadoop
        items:
          - key: hdfs-site.xml
            path: hdfs-site.xml
    - name: mapred-site
      configMap:
        name: hadoop
        items:
          - key: mapred-site.xml
            path: mapred-site.xml
    - name: yarn-site
      configMap:
        name: hadoop
        items:
          - key: yarn-site.xml
            path: yarn-site.xml
    - name: hadoop-data
      emptyDir: {}
  containers:
    - name: hadoop
      image: chq3272991/spark:hadoop_3.2.4_v1
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 8032
        - containerPort: 8030
        - containerPort: 8031
        - containerPort: 8033
        - containerPort: 8088
        - containerPort: 8040
        - containerPort: 8042
        - containerPort: 8041
        - containerPort: 19888
      volumeMounts:
        - name: hadoop-env
          mountPath: /home/hp/hadoop-3.2.4/etc/hadoop/hadoop-env.sh
          subPath: hadoop-env.sh
        - name: core-site
          mountPath: /home/hp/hadoop-3.2.4/etc/hadoop/core-site.xml
          subPath: core-site.xml
        - name: hdfs-site
          mountPath: /home/hp/hadoop-3.2.4/etc/hadoop/hdfs-site.xml
          subPath: hdfs-site.xml
        - name: mapred-site
          mountPath: /home/hp/hadoop-3.2.4/etc/hadoop/mapred-site.xml
          subPath: mapred-site.xml
        - name: yarn-site
          mountPath: /home/hp/hadoop-3.2.4/etc/hadoop/yarn-site.xml
          subPath: yarn-site.xml
        - name: hadoop-data
          mountPath: /home/hp/hadoop-3.2.4/hdfs-data/ 
          subPath: hdfs-data
        - name: hadoop-data
          mountPath: /home/hp/hadoop-3.2.4/logs/
          subPath: logs
      env:
        - name: HADOOP_NODE_TYPE
          value: yarn-nodemanager
```
node-3节点类似node-2， 修改相关node-2字眼为node-3即可

### yarn集群mapred案例测试
```
[root@yarn-master hadoop-3.2.4]# ./bin/hadoop jar ./share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.4.jar pi 10 2

.....
	Shuffle Errors
		BAD_ID=0
		CONNECTION=0
		IO_ERROR=0
		WRONG_LENGTH=0
		WRONG_MAP=0
		WRONG_REDUCE=0
	File Input Format Counters 
		Bytes Read=1180
	File Output Format Counters 
		Bytes Written=97
Job Finished in 94.845 seconds
Estimated value of Pi is 3.80000000000000000000
```
看到运行结果Pi即说明运行成功

### 部分运行部署结果参考
```
[root@node-1 spark-3.2.2-bin-hadoop3.2]# kubectl get svc -n big-data
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                                                       AGE
hadoop-master          NodePort    10.43.48.235    <none>        9000:31651/TCP,50070:32393/TCP                                                                20h
yarn-master            ClusterIP   None            <none>        8032/TCP,8030/TCP,8031/TCP,8033/TCP,8088/TCP,8040/TCP,8042/TCP,8041/TCP,10020/TCP,19888/TCP   6h23m
yarn-master-nodeport   NodePort    10.43.154.204   <none>        19888:31187/TCP,8088:32683/TCP                                                                4h
yarn-node-2            ClusterIP   None            <none>        8042/TCP,8041/TCP,8040/TCP                                                                    4h36m
yarn-node-2-nodeport   NodePort    10.43.12.212    <none>        8042:30342/TCP                                                                                5s
yarn-node-3            ClusterIP   None            <none>        8042/TCP,8041/TCP,8040/TCP                                                                    4h12m
[root@node-1 spark-3.2.2-bin-hadoop3.2]# kubectl get all -n big-data
NAME                                   READY   STATUS    RESTARTS   AGE
pod/hadoop-datanode-d9dfd              1/1     Running   0          21h
pod/hadoop-datanode-qslqc              1/1     Running   0          21h
pod/hadoop-datanode-xxm46              1/1     Running   0          21h
pod/hadoop-namenode-7dbf567495-v6v45   1/1     Running   0          22h
pod/yarn-master                        1/1     Running   0          26m
pod/yarn-node-2                        1/1     Running   0          24m
pod/yarn-node-3                        1/1     Running   0          21m

NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                                                       AGE
service/hadoop-master          NodePort    10.43.48.235    <none>        9000:31651/TCP,50070:32393/TCP                                                                20h
service/yarn-master            ClusterIP   None            <none>        8032/TCP,8030/TCP,8031/TCP,8033/TCP,8088/TCP,8040/TCP,8042/TCP,8041/TCP,10020/TCP,19888/TCP   6h35m
service/yarn-master-nodeport   NodePort    10.43.154.204   <none>        19888:31187/TCP,8088:32683/TCP                                                                4h13m
service/yarn-node-2            ClusterIP   None            <none>        8042/TCP,8041/TCP,8040/TCP                                                                    4h48m
service/yarn-node-2-nodeport   NodePort    10.43.12.212    <none>        8042:30342/TCP                                                                                12m
service/yarn-node-3            ClusterIP   None            <none>        8042/TCP,8041/TCP,8040/TCP                                                                    4h24m
service/yarn-node-3-nodeport   NodePort    10.43.226.123   <none>        8042:31141/TCP                                                                                4m51s

NAME                             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/hadoop-datanode   3         3         3       3            3           <none>          21h

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hadoop-namenode   1/1     1            1           22h

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/hadoop-namenode-7dbf567495   1         1         1       22h
```

### 遇到的问题
#### 启动Yarn集群的history服务(startnode.sh文件)不生效 
```
mapred --daemon start historyserver
```
目前解决办法, pod起来之后，连接容器执行上面的命令

#### 检测Yarn集群可用性, 卡在跑job阶段
```
[root@yarn-master-8955d78fd-zns7f hadoop-3.2.4]# ./bin/hadoop jar ./share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.4.jar pi 10 2 
Number of Maps  = 10
Samples per Map = 2
Wrote input for Map #0
Wrote input for Map #1
Wrote input for Map #2
Wrote input for Map #3
Wrote input for Map #4
Wrote input for Map #5
Wrote input for Map #6
Wrote input for Map #7
Wrote input for Map #8
Wrote input for Map #9
Starting Job
2022-10-11 06:28:57,516 INFO client.RMProxy: Connecting to ResourceManager at yarn-master/10.42.0.219:8032
2022-10-11 06:28:57,954 INFO mapreduce.JobResourceUploader: Disabling Erasure Coding for path: /tmp/hadoop-yarn/staging/root/.staging/job_1665465445733_0003
2022-10-11 06:28:58,066 INFO input.FileInputFormat: Total input files to process : 10
2022-10-11 06:28:58,176 INFO mapreduce.JobSubmitter: number of splits:10
2022-10-11 06:28:58,742 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1665465445733_0003
2022-10-11 06:28:58,743 INFO mapreduce.JobSubmitter: Executing with tokens: []
2022-10-11 06:28:58,914 INFO conf.Configuration: resource-types.xml not found
2022-10-11 06:28:58,914 INFO resource.ResourceUtils: Unable to find 'resource-types.xml'.
2022-10-11 06:28:58,996 INFO impl.YarnClientImpl: Submitted application application_1665465445733_0003
2022-10-11 06:28:59,039 INFO mapreduce.Job: The url to track the job: http://yarn-master-8955d78fd-zns7f:8088/proxy/application_1665465445733_0003/
2022-10-11 06:28:59,040 INFO mapreduce.Job: Running job: job_1665465445733_0003
```
yarn集群内部运行依赖hdfs，需要当前yarn环境内的hdfs启动

参考连接： https://blog.csdn.net/zonelza3/article/details/115437460

hdfs-ui: http://192.168.208.129:32393/dfshealth.html#tab-datanode
yarn-ui: http://192.168.208.129:32683/cluster
jobhistory-ui: http://192.168.208.129:31187/jobhistory/app

### 建议：
hdfs datanode要考虑有状态的存储问题statefulSet + pvc （动态创建pvc的方式,rancher提供的一种storageClass、nfs次选）

### 参考安装：
https://github.com/helm/charts/tree/master/stable/hadoop （停更在2.9版本）
https://zhuanlan.zhihu.com/p/463682863?utm_id=0
https://www.jb51.net/article/243643.htm
https://blog.csdn.net/qq_46416934/article/details/124504584
https://www.jb51.net/article/243643.htm
hadoop常用端口： http://www.wjhsh.net/Komorebi-john-p-11725030.html
