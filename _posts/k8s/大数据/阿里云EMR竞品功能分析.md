# 大数据与云原生竞品功能分析

<aside>
💡 如何实现大数据Hadoop、Spark、Flink、Hbase等产品，在融云上一键部署，开箱即用的思路和实践

</aside>

# 1. 竞品参考（EMR）

阿里云上提供一些大数据相关的产品：

[**E-MapReduce**](https://www.alibabacloud.com/help/zh/e-mapreduce/latest/getting-started) （简称EMR）， 阿里云 [Elastic MapReduce](https://www.alibabacloud.com/zh/product/emapreduce)（E-MapReduce）是运行在阿里云平台上的一套大数据处理的系统解决方案。E-MapReduce 构建于阿里云云服务器 ECS 弹性虚拟机之上，基于开源的 Apache Hadoop 和 Apache Spark（k8s的Operator模式），您可以方便地使用Hadoop和Spark生态系统中的其他周边系统（如 Apache Hive、Apache Kafka、Flink、Druid、TensorFlow 等）来分析和处理自己的数据。您还可以通过E-MapReduce将数据非常方便地处理阿里云其他的云数据存储系统的数据，如OSS、SLS、RDS 等。主要步骤：

- 创建集群，集群名称、身份凭证、产品版本、服务高可用、可选服务(HDFS\YARN\Hive\Spark和TEZ，被选中的组件会默认启动相关的服务进程)
- 创建并执行作业，通过SSH方式链接集群，提交并运行作业：

```bash
spark-submit --class org.apache.spark.examples.SparkPi --master yarn --deploy-mode client --driver-memory 512m --num-executors 1 --executor-memory 1g --executor-cores 2 /usr/lib/spark-current/examples/jars/spark-examples_2.12-3.1.1.jar 10
```

- 查看作业运行记录，通过YARN UI、Hadoop控制台查看作业运行的详情
- (可选)释放集群，确认释放集群，会强制终止集群上的所有作业、终止并释放所有ECS实例

[**DataWorks on EMR**](https://www.alibabacloud.com/help/zh/e-mapreduce/latest/dataworks)， DataWorks支持基于EMR（E-MapReduce）计算引擎创建Hive、MR、Presto和Spark SQL等节点，实现EMR任务工作流的配置、定时调度和元数据管理等功能，帮助EMR用户更好地产出数据。

# 2. 试用体验EMR on ACK

[快速入门 - E-MapReduce - 阿里云](https://www.alibabacloud.com/help/zh/e-mapreduce/latest/start-ack)

通过阿里云账号登录E-MapReduce控制台，基于Kubernetes创建E-MapReduce（简称EMR）集群并执行作业。使用EMR之前，需要先执行以下步骤：

- k8s集群、oss管理等权限，角色授权
- 创建Kubernetes集群，主要是创建[Kubernetes专有版集群](https://www.alibabacloud.com/help/zh/container-service-for-kubernetes/latest/create-an-ack-dedicated-cluster?spm=a2c63.p38356.0.0.46e07148YmiR5M#task-skz-qwk-qfb)或者创建[Kubernetes托管版集群](https://www.alibabacloud.com/help/zh/container-service-for-kubernetes/latest/create-an-ack-managed-cluster?spm=a2c63.p38356.0.0.46e07148YmiR5M#task-skz-qwk-qfb)
- 创建[节点池](https://www.alibabacloud.com/help/zh/container-service-for-kubernetes/latest/manage-node-pools?spm=a2c63.p38356.0.0.46e07148YmiR5M#section-eq0-lmv-4a7)（管理集群节点数、单节点最大Pod数等等）
- 开通对象存储OSS

通过容器ack面板创建k8s集群之后，再登录[EMR on ACK控制台](https://emr-next.console.aliyun.com/#/region/cn-hangzhou/resource/all/ack/list)。

集群访问方式，**通过 kubectl 连接 Kubernetes 集群（内网或者SSH）**

![Untitled](./img/%E9%98%BF%E9%87%8C%E4%BA%91EMR%E7%AB%9E%E5%93%81/01.png)

## driverPotTemplate.yaml

```bash
apiversion: v1
kind: Pod
metadata:
  labels:
    product: emr
    spark/pod.monitor: "true"
spec:
  serviceAccountName: spark
  containers:
    - name: spark-kubernetes-driver
      env:
        - name: SPARKLOGENV
          value: spark-driver
        - name: SPARK_CONF_DIR
          value: /etc/spark/conf
        - name: SPARK_SUBMIT_OPTS
          value: -javaagent:/opt/spark/jars/jmx_prometheus_javaagent-0.16.1.jar=8090:/etc/metrics/conf/prometheus.yaml
      ports:
        - containerPort: 8090
          name: jmx-exporter
          protocol: TCP
  nodeSelector:
    emr-spark: emr-spark
  tolerations:
    - key: emr-node-taint
      effect: NoSchedule
      operator: Equal
      value: emr
```

## executorPodTemplate.yaml

```bash
apiversion: v1
kind: Pod
metadata:
  labels:
    product: emr
    spark/pod.monitor: "true"
spec:
  serviceAccountName: spark
  enableServiceLinks: false
  automountServiceAccountToken: false
  containers:
    - name: spark-kubernetes-executor
      env:
        - name: SPARKLOGENV
          value: spark-executor
      ports:
        - containerPort: 8090
          name: jmx-exporter
          protocol: TCP
  nodeSelector:
    emr-spark: emr-spark
  tolerations:
    - key: emr-node-taint
      effect: NoSchedule
      operator: Equal
      value: emr
```

## log4j.properties

```bash
log4j.rootCategory=INFO, console, file
log4j.appender.console=com.alibaba.emr.ack.AsyncConsoleAppender
log4j.appender.console.layoutPattern=%d{yy/MM/dd HH:mm:ss} %p [%t] %c{1}: %m%n

# Set the default spark-shell log level to WARN. When running the spark-shell, the
# log level for this class is used to overwrite the root logger's log level, so that
# the user can have different defaults for the shell and regular Spark apps.
log4j.logger.org.apache.spark.repl.Main=WARN

# Settings to quiet third party logs that are too verbose
log4j.logger.org.spark_project.jetty=WARN
log4j.logger.org.spark_project.jetty.util.component.AbstractLifeCycle=ERROR
log4j.logger.org.apache.spark.repl.SparkIMain$exprTyper=INFO
log4j.logger.org.apache.spark.repl.SparkILoop$SparkILoopInterpreter=INFO
log4j.logger.org.apache.parquet=ERROR
log4j.logger.parquet=ERROR

# SPARK-9183: Settings to avoid annoying messages when looking up nonexistent UDFs in SparkSQL with Hive support
log4j.logger.org.apache.hadoop.hive.metastore.RetryingHMSHandler=FATAL
log4j.logger.org.apache.hadoop.hive.ql.exec.FunctionRegistry=ERROR

log4j.appender.file=org.apache.log4j.FileAppender
log4j.appender.file.append=true
log4j.appender.file.file=/opt/spark/log/spark.log
log4j.appender.file.layout=org.apache.log4j.PatternLayout
log4j.appender.file.layout.ConversionPattern=%d{yy/MM/dd HH:mm:ss.SSS} %t %p %c{1}: %m%n
```

## spark-default.conf

spark.kubernetes.driver.podTemplateFile

```bash
/etc/spark/conf/driverPodTemplate.yaml
```

spark.kubernetes.executor.podTemplateFile

```bash
/etc/spark/conf/executorPodTemplate.yaml
```

spark.kubernetes.authenticate.driver.serviceAccountName

```bash
spark
```

ack.clusterid.for.rss.linked

```bash

```

spark.kubernetes.container.image.pullSecrets

```bash
emr-image-regsecret
```

spark.shuffle.manager

```bash
org.apache.spark.shuffle.sort.SortShuffleManager
```

spark.serializer

```bash
org.apache.spark.serializer.KryoSerializer
```

## oauth-config.conf

allowed-accounts

```bash
 1791744788253780
```

# 3. 配置kubectl访问集群、提交作业

![Untitled](./img/%E9%98%BF%E9%87%8C%E4%BA%91EMR%E7%AB%9E%E5%93%81/02.png)

```bash
[root@node-6 ~]# cd ~/.kube/
[root@node-6 .kube]# ls
[root@node-6 .kube]# vi config
[root@node-6 .kube]# kubectl get nodes
NAME                        STATUS   ROLES    AGE   VERSION
cn-hangzhou.192.168.0.143   Ready    <none>   32m   v1.22.15-aliyun.1
cn-hangzhou.192.168.0.144   Ready    <none>   32m   v1.22.15-aliyun.1

[root@node-6 .kube]# kubectl get pod  -A | grep operator
c-16504ffdf9e3e5d9   sparkoperator-fb6b678d6-jc9vm                              1/1     Running     0             16m
c-16504ffdf9e3e5d9   sparkoperator-webhook-init-8fw8x                           0/1     Completed   0             16m
kube-system          storage-operator-545bdc9dc7-bpf2b                          1/1     Running     0             37m
```

编辑spark-pi.yaml

```bash
apiVersion: "sparkoperator.k8s.io/v1beta2"
kind: SparkApplication
metadata:
  name: spark-pi-simple
spec:
  type: Scala
  sparkVersion: 3.2.1
  mainClass: org.apache.spark.examples.SparkPi
  mainApplicationFile: "local:///opt/spark/examples/spark-examples.jar"
  arguments:
    - "1000"
  driver:
    cores: 1
    coreLimit: 1000m
    memory: 1g
  executor:
    cores: 1
    coreLimit: 1000m
    memory: 1g
    memoryOverhead: 1g
    instances: 1
```

提交作业

```bash
kubectl apply -f spark-pi.yaml --namespace c-16504ffdf9e3e5d9
```

查看相关资源

```bash
[root@node-6 test-emr]# kubectl get SparkApplication -n c-16504ffdf9e3e5d9
NAME              STATUS      ATTEMPTS   START                  FINISH       AGE
spark-pi-simple   SUBMITTED   1          2023-02-02T09:20:33Z   <no value>   79s
```

![Untitled](./img/%E9%98%BF%E9%87%8C%E4%BA%91EMR%E7%AB%9E%E5%93%81/03.png)

![Untitled](./img/%E9%98%BF%E9%87%8C%E4%BA%91EMR%E7%AB%9E%E5%93%81/04.png)

# 4.提交Spark作业的方式汇总

[](https://help.aliyun.com/document_detail/212167.html?spm=a2cug.25178945.help.dexternal.736f12513A39VF)

# 5.EMR-ACK其他界面和功能

基于已有ack集群部署EMR-ACK

![Untitled](./img/%E9%98%BF%E9%87%8C%E4%BA%91EMR%E7%AB%9E%E5%93%81/05.png)

EMR-ACK集群创建进度

![Untitled](./img/%E9%98%BF%E9%87%8C%E4%BA%91EMR%E7%AB%9E%E5%93%81/06.png)

EMR-ACK集群详情

![Untitled](./img/%E9%98%BF%E9%87%8C%E4%BA%91EMR%E7%AB%9E%E5%93%81/07.png)

大数据spark组件配置

![Untitled](./img/%E9%98%BF%E9%87%8C%E4%BA%91EMR%E7%AB%9E%E5%93%81/08.png)

大数据spark相关访问UI链接

![Untitled](./img/%E9%98%BF%E9%87%8C%E4%BA%91EMR%E7%AB%9E%E5%93%81/09.png)

大数据组件服务详情

![Untitled](./img/%E9%98%BF%E9%87%8C%E4%BA%91EMR%E7%AB%9E%E5%93%81/10.png)