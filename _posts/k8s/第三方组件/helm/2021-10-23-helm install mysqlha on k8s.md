---
layout:     post
rewards: false
title:   使用helm部署mysqlha到k8s集群排错
categories:
    - k8s
---

# 排错步骤

用`helm install mysql .`为例

```shell
# 获取所有resource关于状态
kubectl get pods,svc,statefulsets -o wide|grep mysqlha

pod/mysql-mysqlha-master-0   2/2     Running   0          84s   10.1.0.17   docker-desktop   <none>           <none>
pod/mysql-mysqlha-salve-0    2/2     Running   0          84s   10.1.0.18   docker-desktop   <none>           <none>
pod/mysql-mysqlha-salve-1    1/2     Running   0          12s   10.1.0.19   docker-desktop   <none>           <none>
service/mysql-mysqlha-master     NodePort    10.106.41.243   <none>        3306:30008/TCP   84s   app=mysql-mysqlha-master
service/mysql-mysqlha-readonly   NodePort    10.98.164.96    <none>        3306:30009/TCP   84s   app=mysql-mysqlha-salve
statefulset.apps/mysql-mysqlha-master   1/1     84s   mysql,xtrabackup   mysql:5.7,gcr-xtrabackup
statefulset.apps/mysql-mysqlha-salve    1/2     84s   mysql,xtrabackup   mysql:5.7,gcr-xtrabackup
```



遇到pod status 要问题

```shell
➜  helm-charts kubectl get pods,svc,statefulsets -o wide
NAME                         READY   STATUS                  RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES
pod/mysql-mysqlha-master-0   2/2     Running                 0          70m   10.1.0.21   docker-desktop   <none>           <none>
pod/mysql-mysqlha-salve-0    0/2     Init:CrashLoopBackOff   18         70m   10.1.0.20   docker-desktop   <none>           <none>

NAME                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE   SELECTOR
service/kubernetes               ClusterIP   10.96.0.1       <none>        443/TCP          27h   <none>
service/mysql-mysqlha-master     NodePort    10.102.237.60   <none>        3306:30008/TCP   70m   app=mysql-mysqlha-master
service/mysql-mysqlha-readonly   NodePort    10.99.213.3     <none>        3306:30009/TCP   70m   app=mysql-mysqlha-salve

NAME                                    READY   AGE   CONTAINERS         IMAGES
statefulset.apps/mysql-mysqlha-master   1/1     70m   mysql,xtrabackup   mysql:5.7,gcr-xtrabackup
statefulset.apps/mysql-mysqlha-salve    0/2     70m   mysql,xtrabackup   mysql:5.7,gcr-xtrabackup
```

可以通过**describe**看pod状态，pod有时候还没有初始化。

```shell
 kubectl describe pod/mysql-mysqlha-salve-0
 
 。。。。
     Restart Count:  0
    Requests:
      cpu:     100m
      memory:  100Mi
    Environment:
      MYSQL_PWD:                   <set to the key 'mysql-root-password' in secret 'mysql-mysqlha'>  Optional: false
      MYSQL_REPLICATION_USER:      repl
      MYSQL_REPLICATION_PASSWORD:  <set to the key 'mysql-replication-password' in secret 'mysql-mysqlha'>  Optional: false
    Mounts:
      /etc/mysql/conf.d from conf (rw)
      /mnt/scripts from scripts (rw)
      /var/lib/mysql from data (rw,path="mysql")
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-h948v (ro)
Conditions:
  Type              Status
  Initialized       False
  Ready             False
  ContainersReady   False
  PodScheduled      True
Volumes:
  data:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  data-mysql-mysqlha-salve-0
    ReadOnly:   false
  conf:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:  <unset>
  config-map:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      mysql-mysqlha
    Optional:  false
  scripts:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:  <unset>
  kube-api-access-h948v:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason   Age                   From     Message
  ----     ------   ----                  ----     -------
  Warning  BackOff  103s (x328 over 71m)  kubelet  Back-off restarting failed container
```

主要是看**Events（可以知道出错的的地方）**和一些描述

```shell
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  21s               default-scheduler  Successfully assigned default/mysql-mysqlha-salve-0 to docker-desktop
  Normal   Pulled     5s (x3 over 21s)  kubelet            Container image "gcr-xtrabackup" already present on machine
  Normal   Created    5s (x3 over 21s)  kubelet            Created container clone-mysql
  Normal   Started    5s (x3 over 21s)  kubelet            Started container clone-mysql
  Warning  BackOff    4s (x3 over 20s)  kubelet            Back-off restarting failed container

```

然后通常**initContainers**会按序执行，根据**Events**可以知道应该该看哪个**container（因为pod通常包含好几个container）**

```shell
# 选择合适的container get log
➜  helm-charts kubectl logs -f pod/mysql-mysqlha-salve-0
error: a container name must be specified for pod mysql-mysqlha-salve-0, choose one of: [mysql xtrabackup] or one of the init containers: [clone-mysql init-mysql]


➜  helm-charts kubectl logs -f pod/mysql-mysqlha-salve-0 init-mysql
Error from server (BadRequest): container "init-mysql" in pod "mysql-mysqlha-salve-0" is waiting to start: PodInitializing

# get log from error
➜  helm-charts kubectl logs -f pod/mysql-mysqlha-salve-0 clone-mysql
++ hostname
+ [[ mysql-mysqlha-salve-0 =~ -([0-9]+)$ ]]
+ ordinal=0
+ [[ -d /var/lib/mysql/mysql ]]
+ ncat --recv-only mysql-mysqlha-master-0.mysql-mysqlha-master 3307
+ xbstream -x -C /var/lib/mysql
Ncat: Connection refused.
+ xtrabackup --prepare --user=repl --password=mfRnFiobcZdR --target-dir=/var/lib/mysql
xtrabackup version 2.4.4 based on MySQL server 5.7.13 Linux (x86_64) (revision id: df58cf2)
xtrabackup: cd to /var/lib/mysql
xtrabackup: Error: cannot open ./xtrabackup_checkpoints
xtrabackup: error: xtrabackup_read_metadata()
xtrabackup: This target seems not to have correct metadata...
InnoDB: Number of pools: 1
InnoDB: Operating system error number 2 in a file operation.
InnoDB: The error means the system cannot find the path specified.
xtrabackup: Warning: cannot open ./xtrabackup_logfile. will try to find.
InnoDB: Operating system error number 2 in a file operation.
InnoDB: The error means the system cannot find the path specified.
  xtrabackup: Fatal error: cannot find ./xtrabackup_logfile.
xtrabackup: Error: xtrabackup_init_temp_log() failed.

```

 根据log检查错误，检查对应的错误，排查**deployment.yaml**或者**statefulset.yaml**相关的配置。



# 安装过程 分析原因

修改helm repo

```shell
helm repo add ali https://apphub.aliyuncs.com
helm repo update


helm search repo mysql
NAME                         	CHART VERSION	APP VERSION	DESCRIPTION
ali/mysql                    	6.8.0        	8.0.19     	Chart to create a Highly available MySQL cluster
ali/mysqldump                	2.6.0        	2.4.1      	A Helm chart to help backup MySQL databases usi...
ali/mysqlha                  	1.0.0        	5.7.13     	MySQL cluster with a single master and zero or ...
ali/prometheus-mysql-exporter	0.5.2        	v0.11.0    	A Helm chart for prometheus mysql exporter with...
bitnami/mysql                	8.8.11       	8.0.27     	Chart to create a Highly available MySQL cluster
incubator/mysqlha            	2.0.2        	5.7.13     	DEPRECATED MySQL cluster with a single master a...
ali/percona                  	1.2.0        	5.7.17     	free, fully compatible, enhanced, open source d...
ali/percona-xtradb-cluster   	1.0.3        	5.7.19     	free, fully compatible, enhanced, open source d...
ali/phpmyadmin               	4.2.12       	5.0.1      	phpMyAdmin is an mysql administration frontend
bitnami/phpmyadmin           	8.2.17       	5.1.1      	phpMyAdmin is an mysql administration frontend
ali/mariadb                  	7.3.9        	10.3.22    	Fast, reliable, scalable, and easy to use open-...
ali/mariadb-galera           	0.8.1        	10.4.12    	MariaDB Galera is a multi-master database clust...
bitnami/mariadb              	9.6.3        	10.5.12    	Fast, reliable, scalable, and easy to use open-...
bitnami/mariadb-cluster      	1.0.2        	10.2.14    	DEPRECATED Chart to create a Highly available M...
bitnami/mariadb-galera       	6.0.1        	10.6.4     	MariaDB Galera is a multi-master database clust...

# 下载helm chart压缩包
helm pull ali/mysqlha

# 使用helm uninstall卸载不会删除pvc pv
kubectl delete $(kubectl get pv,pvc |grep mysqlha|awk '{print $1}')
```

helm反复部署，salve会报错

```shell
++ hostname
+ [[ mysql-mysqlha-salve-0 =~ -([0-9]+)$ ]]
+ ordinal=0
+ [[ -d /var/lib/mysql/mysql ]]
+ ncat --recv-only mysql-mysqlha-master-0.mysql-mysqlha-master 3307
+ xbstream -x -C /var/lib/mysql
Ncat: Connection refused.
+ xtrabackup --prepare --user=repl --password=mfRnFiobcZdR --target-dir=/var/lib/mysql
xtrabackup version 2.4.4 based on MySQL server 5.7.13 Linux (x86_64) (revision id: df58cf2)
xtrabackup: cd to /var/lib/mysql
xtrabackup: Error: cannot open ./xtrabackup_checkpoints
xtrabackup: error: xtrabackup_read_metadata()
xtrabackup: This target seems not to have correct metadata...
InnoDB: Number of pools: 1
InnoDB: Operating system error number 2 in a file operation.
InnoDB: The error means the system cannot find the path specified.
xtrabackup: Warning: cannot open ./xtrabackup_logfile. will try to find.
InnoDB: Operating system error number 2 in a file operation.
InnoDB: The error means the system cannot find the path specified.
  xtrabackup: Fatal error: cannot find ./xtrabackup_logfile.
xtrabackup: Error: xtrabackup_init_temp_log() failed.
```

这是因为master的**xtrabackup**没有初始化好

- **mysql-root-password**定下来
- helm uninstall
- 清理pvc
- helm install 

因为每次**install**默认**secret**都会随机生成**mysql-root-password**之类的，而**uninstall**是不会删除**statefulset**创建的**pvc，pv**，里面的密码还是旧的，用新的会报错。



# 修改结果

simple_mysql_ha 优化原来的service，没有做到读写分离，**statefulset**里面master和salve也混在一起，没有提供给nodePort



[statefulset_master.yaml ](https://github.com/631068264/helm_charts/blob/master/mysqlha/ali_mysqlha/templates/statefulset_master.yaml)

[statefulset_salve.yaml](https://github.com/631068264/helm_charts/blob/master/mysqlha/ali_mysqlha/templates/statefulset_salve.yaml)

<script src="https://gist.github.com/631068264/67877ff618fcfcd0894cbc7d74340ca1.js"></script>

svc.yaml

```yaml
# Headless service for stable DNS entries of StatefulSet members.
apiVersion: v1
kind: Service
metadata:
  name: {{ template "fullname" . }}-master
  labels:
    app: {{ template "fullname" . }}
    chart: "{{ template "mysqlha.chart" . }}"
    release: "{{ .Release.Name }}"
    heritage: "{{ .Release.Service }}"
spec:
  type: NodePort
  ports:
  - name: {{ template "fullname" . }}
    port: 3306
    nodePort: 30008
  selector:
    app: {{ template "fullname" . }}-master
---
# Client service for connecting to any MySQL instance for reads.
# For writes, you must instead connect to the master: mysql-0.mysql.
apiVersion: v1
kind: Service
metadata:
  name: {{ template "fullname" . }}-readonly
  labels:
    app: {{ template "fullname" . }}
    chart: "{{ template "mysqlha.chart" . }}"
    release: "{{ .Release.Name }}"
    heritage: "{{ .Release.Service }}"
  annotations:
{{- if and (.Values.metrics.enabled) (.Values.metrics.annotations) }}
{{ toYaml .Values.metrics.annotations | indent 4 }}
{{- end }}
spec:
  type: NodePort
  ports:
  - name: {{ template "fullname" . }}
    port: 3306
    nodePort: 30009
  {{- if .Values.metrics.enabled }}
  - name: metrics
    port: 9104
    targetPort: metrics
  {{- end }}
  selector:
    app: {{ template "fullname" . }}-salve

```









