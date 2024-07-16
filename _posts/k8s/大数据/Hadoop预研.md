#### Hadoop是什么
Hadoop是用java语言编写的，在分布式服务器集群上存储海量数据并运行分布式分析应用的开源框架，其核心部分是HDFS、MapReduce与Yarn

HDFS是分布式文件系统，引入存放文件元数据的服务器NameNode和实际存放数据的服务器DataNode，对数据进行分布式存储和读取

MapReduce是分布式计算框架，MapRuduce的核心思想是把计算任务分配给集群内的服务器执行，通过对计算任务的拆分（Map计算/Reduce计算）再根据任务调度器（JobTracher）对任务进行分布式计算。

Yarn是分布式资源框架，管理整个集群的资源（内存、CPU核数）分配调度集群的资源

把HDFS理解为一个分布式的，有冗余备份的，可以动态扩展的用来存储大规模数据的大硬盘
把MapRuduce理解成一个计算引擎，按照MapReduce的规则编写Map计算/Reduce计算的程序，可以完成计算任务。

#### 怎么使用Hadoop
①Hadoop集群的搭建
无论是在windows上装几台虚拟机玩Hadoop，还是真实的服务器来玩，说简单点就是把Hadoop的安装包放在每一台服务器上，改改配置，启动就完成了Hadoop集群的搭建。

②上传文件到Hadoop集群，实现文件存储
Hadoop集群搭建好以后，可以通过web页面查看集群的情况，还可以通过Hadoop命令来上传文件到hdfs集群，通过Hadoop命令在hdfs集群上建立目录，通过Hadoop命令删除集群上的文件等等。

③编写map/reduce程序，完成计算任务
通过集成开发工具（例如eclipse）导入Hadoop相关的jar包，编写map/reduce程序，将程序打成jar包扔在集群上执行，运行后出计算结果。

### Hadoop 架构

#### HDFS读取流程：
![](./img/hadoop/hdfs%E8%AF%BB%E5%8F%96%E6%B5%81%E7%A8%8B.png)
（1）客户端通过Distributed FileSystem模块向NameNode请求上传文件，NameNode检查目标文件是否已存在，父目录是否存在。
（2）NameNode返回是否可以上传。
（3）客户端请求第一个 Block上传到哪几个DataNode服务器上。
（4）NameNode返回3个DataNode节点，分别为dn1、dn2、dn3。
（5）客户端通过FSDataOutputStream模块请求dn1上传数据，dn1收到请求会继续调用dn2，然后dn2调用dn3，将这个通信管道建立完成。
（6）dn1、dn2、dn3逐级应答客户端。
（7）客户端开始往dn1上传第一个Block（先从磁盘读取数据放到一个本地内存缓存），以Packet为单位，dn1收到一个Packet就会传给dn2，dn2传给dn3；dn1每传一个packet会放入一个应答队列等待应答。
（8）当一个Block传输完成之后，客户端再次请求NameNode上传第二个Block的服务器。（重复执行3-7步）。

#### MapReduce工作流程：
![](./img/hadoop/mapReduce%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B.png)

#### Yarn调度相关：
![](./img/hadoop/yarn%E8%B0%83%E5%BA%A6%E6%B5%81%E7%A8%8B.png)
Yarn详细调度流程：
![](./img/hadoop/yarn%E8%B0%83%E5%BA%A6%E6%B5%81%E7%A8%8B2.png)

1.client向yarn提交job，首先找ResourceManager分配资源，
2.ResourceManager开启一个Container,在Container中运行一个Application manager
3.Application manager找一台nodemanager启动Application master，计算任务所需的计算
4.Application master向Application manager（Yarn）申请运行任务所需的资源
5.Resource scheduler将资源封装发给Application master
6.Application master将获取到的资源分配给各个nodemanager
7.各个nodemanager得到任务和资源开始执行map task
8.map task执行结束后，开始执行reduce task
9.map task和 reduce task将执行结果反馈给Application master
10.Application master将任务执行的结果反馈pplication manager。

### Hadoop 部署安装流程
- 创建一个用于管理hadood的用户（可新建或者使用已有的用户）
- 安装并配置ssh免密码登陆
- 安装Java环境
- 下载hadoop并配置环境变量
- 配置相关的Hadoop配置
- 验证hadoop安装并启动

#### 怎么使用Hadoop
①Hadoop集群的搭建
无论是在windows上装几台虚拟机玩Hadoop，还是真实的服务器来玩，说简单点就是把Hadoop的安装包放在每一台服务器上，改改配置，启动就完成了Hadoop集群的搭建。

②上传文件到Hadoop集群，实现文件存储
Hadoop集群搭建好以后，可以通过web页面查看集群的情况，还可以通过Hadoop命令来上传文件到hdfs集群，通过Hadoop命令在hdfs集群上建立目录，通过Hadoop命令删除集群上的文件等等。

③编写map/reduce程序，完成计算任务
通过集成开发工具（例如eclipse）导入Hadoop相关的jar包，编写map/reduce程序，将程序打成jar包扔在集群上执行，运行后出计算结果。

### Hadoop 单机部署安装
- 创建一个用于管理hadood的用户（可新建或者使用已有的用户）
- 安装并配置ssh免密码登陆
- 安装Java环境
- 下载hadoop并配置环境变量
- 配置相关的Hadoop配置
- 验证hadoop安装并启动

### Hadoop standalone集群部署
#### 基础系统环境准备
在VMware中创建3台centos7.6主机，空间50G：
- 配置/etc/hostname
- 修改/etc/sysconfig/network-scripts/ifcfg-ens33静态ip、网关（192.168.208.1）、DNS（8.8.8.8）
- 配置/etc/hosts , 测试是否相互ping通

| 主机名 | IP | 用户 | HDFS | Yarn |
| --- | --- | --- | --- | --- |
| node-4 | 192.168.208.132 | hp | NameNode、DataNode | NodeManager、ResourceManager |
| node-5 | 192.168.208.133 | hp | DataNode、SecondaryNameNode | NodeManager |
| node-6 | 192.168.208.134 | hp | DataNode | NodeManager |

#### 配置服务器之间的ssh免密码登录（master-> node）
```
ssh localhost # 会提示输入密码
cd ~/.ssh/
ssh-keygen -t rsa	# 会有提示，都按回车就行
cat ./id_rsa.pub >> ./authorized_keys	# 加入授权
```
现在再使用"ssh localhost"，就可以不用输入密码登录ssh

将master节点(node-4)的公钥传给各个Slave节点
```
scp ~/.ssh/id_rsa.pub node-5:/home/hp
scp ~/.ssh/id_rsa.pub node-6:/home/hp
```

#### Java1.8、Hodoop 3.2.4下载\安装
配置 ~\.bashrc
```
# java
export JAVA_HOME=/home/hp/jdk1.8.0_311
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin

# hadoop
export HADOOP_HOME=/home/hp/hadoop-3.2.4
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
export HADOOP_INSTALL=$HADOOP_HOME
```
安装情况:
java版本
```
[hp@node-5 .ssh]$ java -version
java version "1.8.0_311"
Java(TM) SE Runtime Environment (build 1.8.0_311-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.311-b11, mixed mode)

```
Hadoop版本
```
[hp@node-4 ~]$ hadoop version
Hadoop 3.2.4
Source code repository Unknown -r 7e5d9983b388e372fe640f21f048f2f2ae6e9eba
Compiled by ubuntu on 2022-07-12T11:58Z
Compiled with protoc 2.5.0
From source with checksum ee031c16fe785bbb35252c749418712
This command was run using /home/hp/hadoop-3.2.4/share/hadoop/common/hadoop-common-3.2.4.jar
```

#### 修改Hadoop配置
配置集群模式时，需要修改Hadoop解压包下配置文件，包括workers、core-site.xml、hdfs-site.xml、mapred-site.xml、yarn-site.xml

修改workers文件
```
node-4
node-5
node-6
```

修改core-site.xml文件
```
<configuration>
  <!-- 指定NameNode的内部通信地址 -->
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://node-4:9000</value>
  </property>
  <!-- 指定hadoop集群在工作时存储的一些临时文件存放的目录 -->
  <property>
    <name>hadoop.tmp.dir</name>
    <value>file:/home/hp/hadoop-data/tmp</value>
  </property>

</configuration>
```

修改hdfs-site.xml

dfs.namenode.name.dir：namenode数据的存放位置，元数据存放位置
dfs.datanode.data.dir：datanode数据的存放位置，block块存放的位置
dfs.repliction：hdfs的副本数设置，默认为3
dfs.secondary.http.address：secondarynamenode运行节点的信息，应该和namenode存放在不同节点

```
<configuration>
  <property>
  <!-- namenode web端访问的地址 -->
     <name>dfs.namenode.http-address</name>
     <value>node-4:9870</value>
  </property>
  <property>
   <!-- secondarynamenode(简称2nn) web端访问的地址 -->
     <name>dfs.namenode.secondary.http-address</name>
     <value>node-5:50090</value>
  </property>
  <property>
    <name>dfs.replication</name>
    <value>3</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:/home/hp/hadoop-data/name</value>
 </property>
 <property>
  <name>dfs.datanode.data.dir</name> 
  <value>file:/home/hp/hadoop-data/data</value> 
</property>

```

修改mapred-site.xml

mapreduce.framework.name：指定mapreduce框架为yarn方式
mapreduce.jobhistory.address：指定历史服务器的地址和端口
mapreduce.jobhistory.webapp.address：查看历史服务器已经运行完的Mapreduce作业记录的web地址，需要启动该服务才行
```
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>node-4:10020</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>node-4:19888</value>
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

```

修改yarn-site文件
```
<configuration>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>node-4</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
```

修改完上面几个配置文件之后，同步这些文件到其他节点的hadoop配置上

#### HDFS初始化只能在主节点（node-4）进行
```
cd /home/hp/hadoop-3.2.4
./bin/hdfs namenode -format
```
执行部分结果：
```
2022-09-02 11:45:22,618 INFO snapshot.SnapshotManager: SkipList is disabled
2022-09-02 11:45:22,627 INFO util.GSet: Computing capacity for map cachedBlocks
2022-09-02 11:45:22,627 INFO util.GSet: VM type       = 64-bit
2022-09-02 11:45:22,627 INFO util.GSet: 0.25% max memory 828.5 MB = 2.1 MB
2022-09-02 11:45:22,627 INFO util.GSet: capacity      = 2^18 = 262144 entries
2022-09-02 11:45:22,656 INFO metrics.TopMetrics: NNTop conf: dfs.namenode.top.window.num.buckets = 10
2022-09-02 11:45:22,657 INFO metrics.TopMetrics: NNTop conf: dfs.namenode.top.num.users = 10
2022-09-02 11:45:22,657 INFO metrics.TopMetrics: NNTop conf: dfs.namenode.top.windows.minutes = 1,5,25
2022-09-02 11:45:22,681 INFO namenode.FSNamesystem: Retry cache on namenode is enabled
2022-09-02 11:45:22,681 INFO namenode.FSNamesystem: Retry cache will use 0.03 of total heap and retry cache entry expiry time is 600000 millis
2022-09-02 11:45:22,684 INFO util.GSet: Computing capacity for map NameNodeRetryCache
2022-09-02 11:45:22,684 INFO util.GSet: VM type       = 64-bit
2022-09-02 11:45:22,684 INFO util.GSet: 0.029999999329447746% max memory 828.5 MB = 254.5 KB
2022-09-02 11:45:22,684 INFO util.GSet: capacity      = 2^15 = 32768 entries
2022-09-02 11:45:22,729 INFO namenode.FSImage: Allocated new BlockPoolId: BP-354792982-192.168.208.132-1662090322715
2022-09-02 11:45:22,743 INFO common.Storage: Storage directory /home/hp/hadoop-data/name has been successfully formatted.
2022-09-02 11:45:22,794 INFO namenode.FSImageFormatProtobuf: Saving image file /home/hp/hadoop-data/name/current/fsimage.ckpt_0000000000000000000 using no compression
2022-09-02 11:45:22,985 INFO namenode.FSImageFormatProtobuf: Image file /home/hp/hadoop-data/name/current/fsimage.ckpt_0000000000000000000 of size 397 bytes saved in 0 seconds .
2022-09-02 11:45:22,995 INFO namenode.NNStorageRetentionManager: Going to retain 1 images with txid >= 0
2022-09-02 11:45:23,023 INFO namenode.FSNamesystem: Stopping services started for active state
2022-09-02 11:45:23,024 INFO namenode.FSNamesystem: Stopping services started for standby state
2022-09-02 11:45:23,032 INFO namenode.FSImage: FSImageSaver clean checkpoint: txid=0 when meet shutdown.
2022-09-02 11:45:23,033 INFO namenode.NameNode: SHUTDOWN_MSG: 
/************************************************************
SHUTDOWN_MSG: Shutting down NameNode at node-4/192.168.208.132
************************************************************/

```
看到以下一句提示，说明初始化成功：
```
INFO common.Storage: Storage directory /home/hp/hadoop-data/name has been successfully formatted.
```

继续在主节点上运行：
```
./sbin/start-dfs.sh
./sbin/start-yarn.sh
# 3.2之后可以用： mapred --daemon start
./sbin/mr-jobhistory-daemon.sh start historyserver
```
之后可以看到一些进程：
```
[hp@node-4 hadoop-3.2.4]$ jps
20522 JobHistoryServer
19563 NameNode
19676 DataNode
20173 NodeManager
20718 Jps
20047 ResourceManager
```
说明主节点启动成功

node-5节点进程：
```
[hp@node-5 hadoop-3.2.4]$ jps
20098 NodeManager
19348 SecondaryNameNode
19608 DataNode
21192 Jps
```
node-6节点进程：
```
[hp@node-6 hadoop-3.2.4]$ jps
19858 NodeManager
19417 DataNode
19997 Jps
```

最后，可以在node-4节点查看汇总报告：
```
[hp@node-4 hadoop-3.2.4]$ ./bin/hdfs dfsadmin -report
Configured Capacity: 86905466880 (80.94 GB)
Present Capacity: 55142129664 (51.36 GB)
DFS Remaining: 55141244928 (51.35 GB)
DFS Used: 884736 (864 KB)
DFS Used%: 0.00%
Replicated Blocks:
	Under replicated blocks: 0
	Blocks with corrupt replicas: 0
	Missing blocks: 0
	Missing blocks (with replication factor 1): 0
	Low redundancy blocks with highest priority to recover: 0
	Pending deletion blocks: 0
Erasure Coded Block Groups: 
	Low redundancy block groups: 0
	Block groups with corrupt internal blocks: 0
	Missing block groups: 0
	Low redundancy blocks with highest priority to recover: 0
	Pending deletion blocks: 0

-------------------------------------------------
Live datanodes (3):

Name: 192.168.208.132:9866 (node-4)
Hostname: node-4
Decommission Status : Normal
Configured Capacity: 28968488960 (26.98 GB)
DFS Used: 294912 (288 KB)
Non DFS Used: 10588753920 (9.86 GB)
DFS Remaining: 18379440128 (17.12 GB)
DFS Used%: 0.00%
DFS Remaining%: 63.45%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Fri Sep 02 14:45:58 CST 2022
Last Block Report: Fri Sep 02 11:46:30 CST 2022
Num of Blocks: 4


Name: 192.168.208.133:9866 (node-5)
Hostname: node-5
Decommission Status : Normal
Configured Capacity: 28968488960 (26.98 GB)
DFS Used: 294912 (288 KB)
Non DFS Used: 10587422720 (9.86 GB)
DFS Remaining: 18380771328 (17.12 GB)
DFS Used%: 0.00%
DFS Remaining%: 63.45%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Fri Sep 02 14:45:59 CST 2022
Last Block Report: Fri Sep 02 11:55:04 CST 2022
Num of Blocks: 4


Name: 192.168.208.134:9866 (node-6)
Hostname: node-6
Decommission Status : Normal
Configured Capacity: 28968488960 (26.98 GB)
DFS Used: 294912 (288 KB)
Non DFS Used: 10587160576 (9.86 GB)
DFS Remaining: 18381033472 (17.12 GB)
DFS Used%: 0.00%
DFS Remaining%: 63.45%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Fri Sep 02 14:45:59 CST 2022
Last Block Report: Fri Sep 02 13:30:26 CST 2022
Num of Blocks: 4
```

也可以通过浏览器访问：
HDFS: http://192.168.208.132:9870/dfshealth.html#tab-datanode
YARN: http://192.168.208.132:8088/cluster

### 测试执行分布式实例
在HDFS上创建一个文件夹/test/input
```
cd /home/hp/hadoop-3.2.4
./bin/hdfs dfs -mkdir -p /test/input
# 查看文件夹
./bin/hdfs dfs -ls /
Found 2 items
drwxr-xr-x   - hp supergroup          0 2022-09-02 12:12 /test
drwxrwx---   - hp supergroup          0 2022-09-02 11:52 /tmp
```
创建一个~/word.txt测试文件
填充入一段英文文章
```
	Be not alarmed, madam, on receiving this letter, by the apprehension of its containing any repetition of those
sentiments or renewal of those offers which were last night so disgusting to you. I write without any intention of
paining you, or humbling myself, by dwelling on wishes which, for the happiness of both, cannot be too soon
forgotten; and the effort which the formation and the perusal of this letter must occasion, should have been spared,
had not my character required it to be written and read. You must, therefore, pardon the freedom with which I
demand your attention; your feelings, I know, will bestow it unwillingly, but I demand it of your justice.
	My objections to the marriage were not merely those which I last night acknowledged to have the utmost required
force of passion to put aside, in my own case; the want of connection could not be so great an evil to my friend as to
me. But there were other causes of repugnance; causes which, though still existing, and existing to an equal degree
in both instances, I had myself endeavored to forget, because they were not immediately before me. These causes
must be stated, though briefly. The situation of your mother's family, though objectionable, was nothing in
comparison to that total want of propriety so frequently, so almost uniformly betrayed by herself, by your three
younger sisters, and occasionally even by your father. Pardon me. It pains me to offend you. But amidst your
concern for the defects of your nearest relations, and your displeasure at this representation of them, let it give you
consolation to consider that, to have conducted yourselves so as to avoid any share of the like censure, is praise no
less generally bestowed on you and your eldersister, than it is honorable to the sense and disposition of both. I will
only say farther that from what passed that evening, my opinion of all parties was confirmed, and every inducement
heightened which could have led me before, to preserve my friend from what I esteemed a most unhappy
connection. He left Netherfield for London, on the day following, as you, I am certain, remember, with the design of
soon returning.
```
将word.txt上传到HDFS的/test/input文件夹中
```
./bin/hdfs dfs -put ~/word.txt /test/input
```
运行一个mapreduce的例子程序：wordcount
```
./bin/hadoop jar /home/hp/hadoop-3.2.4/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.4.jar wordcount /test/input /test/output
```
执行成功之后如下所示，输出相关信息：
```
2022-09-02 12:12:27,631 INFO client.RMProxy: Connecting to ResourceManager at node-4/192.168.208.132:8032
2022-09-02 12:12:28,232 INFO mapreduce.JobResourceUploader: Disabling Erasure Coding for path: /tmp/hadoop-yarn/staging/hp/.staging/job_1662090752653_0001
2022-09-02 12:12:28,534 INFO input.FileInputFormat: Total input files to process : 1
2022-09-02 12:12:28,669 INFO mapreduce.JobSubmitter: number of splits:1
2022-09-02 12:12:28,962 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1662090752653_0001
2022-09-02 12:12:28,963 INFO mapreduce.JobSubmitter: Executing with tokens: []
2022-09-02 12:12:29,352 INFO conf.Configuration: resource-types.xml not found
2022-09-02 12:12:29,352 INFO resource.ResourceUtils: Unable to find 'resource-types.xml'.
2022-09-02 12:12:29,785 INFO impl.YarnClientImpl: Submitted application application_1662090752653_0001
2022-09-02 12:12:29,819 INFO mapreduce.Job: The url to track the job: http://node-4:8088/proxy/application_1662090752653_0001/
2022-09-02 12:12:29,820 INFO mapreduce.Job: Running job: job_1662090752653_0001
2022-09-02 12:12:38,006 INFO mapreduce.Job: Job job_1662090752653_0001 running in uber mode : false
2022-09-02 12:12:38,007 INFO mapreduce.Job:  map 0% reduce 0%
2022-09-02 12:12:43,076 INFO mapreduce.Job:  map 100% reduce 0%
2022-09-02 12:12:50,115 INFO mapreduce.Job:  map 100% reduce 100%
2022-09-02 12:12:51,125 INFO mapreduce.Job: Job job_1662090752653_0001 completed successfully
2022-09-02 12:12:51,206 INFO mapreduce.Job: Counters: 54
	File System Counters
		FILE: Number of bytes read=2896
		FILE: Number of bytes written=481867
		FILE: Number of read operations=0
		FILE: Number of large read operations=0
		FILE: Number of write operations=0
		HDFS: Number of bytes read=2269
		HDFS: Number of bytes written=2014
		HDFS: Number of read operations=8
		HDFS: Number of large read operations=0
		HDFS: Number of write operations=2
		HDFS: Number of bytes read erasure-coded=0
	Job Counters 
		Launched map tasks=1
		Launched reduce tasks=1
		Data-local map tasks=1
		Total time spent by all maps in occupied slots (ms)=2884
		Total time spent by all reduces in occupied slots (ms)=3521
		Total time spent by all map tasks (ms)=2884
		Total time spent by all reduce tasks (ms)=3521
		Total vcore-milliseconds taken by all map tasks=2884
		Total vcore-milliseconds taken by all reduce tasks=3521
		Total megabyte-milliseconds taken by all map tasks=2953216
		Total megabyte-milliseconds taken by all reduce tasks=3605504
	Map-Reduce Framework
		Map input records=21
		Map output records=370
		Map output bytes=3643
		Map output materialized bytes=2896
		Input split bytes=103
		Combine input records=370
		Combine output records=220
		Reduce input groups=220
		Reduce shuffle bytes=2896
		Reduce input records=220
		Reduce output records=220
		Spilled Records=440
		Shuffled Maps =1
		Failed Shuffles=0
		Merged Map outputs=1
		GC time elapsed (ms)=317
		CPU time spent (ms)=1310
		Physical memory (bytes) snapshot=478359552
		Virtual memory (bytes) snapshot=5567832064
		Total committed heap usage (bytes)=409468928
		Peak Map Physical memory (bytes)=292597760
		Peak Map Virtual memory (bytes)=2781896704
		Peak Reduce Physical memory (bytes)=185761792
		Peak Reduce Virtual memory (bytes)=2785935360
	Shuffle Errors
		BAD_ID=0
		CONNECTION=0
		IO_ERROR=0
		WRONG_LENGTH=0
		WRONG_MAP=0
		WRONG_REDUCE=0
	File Input Format Counters 
		Bytes Read=2166
	File Output Format Counters 
		Bytes Written=2014

```
也可以在YARN Web界面，Applications栏目点击查看执行输出信息

查看运行结果(部分)：
```
[hp@node-4 hadoop-3.2.4]$ ./bin/hdfs dfs -cat /test/output/*
Be	1
But	2
He	1
I	9
It	1
London,	1
My	1
Netherfield	1
Pardon	1
The	1
These	1
You	1
a	1
acknowledged	1
alarmed,	1
all	1
almost	1
am	1
amidst	1
an	2
and	9
any	3
apprehension	1
as	3
aside,	1
at	1
attention;	1
```
至此，可以看到单词统计结果有了，说明集群搭建起来了

关闭集群命令：
```
./sbin/stop-yarn.sh
./sbin/stop-dfs.sh
# 3.2之后可以用： mapred --daemon stop
./sbin/mr-jobhistory-daemon.sh stop historyserver
```

#### 部分错误处理:
##### warn library警告问题
```
WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform… using builtin-java classes where applicable警告问题
```
这个是因为hadoop程序内置的java类，需要读取/home/hp/hadoop-3.2.4/lib/native内的libhadoop.so，但系统变量没有设置，补充设置：
```
vi ~/.bashrc
export JAVA_LIBRARY_PATH=/home/hp/hadoop-3.2.4/lib/native
source ~/.bashrc
```

也可以通过配置core-site.xml来解决（没验证）
```
  <property>
    <name>hadoop.native.lib</name>
    <value>false</value>
    <description>Should native hadoop libraries, if present, be used.</description>
  </property>
```

##### 非root用户下，hadoop解压文件的权限问题
切换到root用户，chown修改用户组权限
```
chown -R hp:hp /home/hp/hadoop-3.2.4
```

参考资料：

[Hadoop: Setting up a Single Node Cluster.](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html)

[Hadoop Cluster Setup](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/ClusterSetup.html)

[HDFS简介和部分架构图](https://blog.csdn.net/weixin_40911578/article/details/125191930)

[Hadoop集群集群搭建](https://blog.csdn.net/qq_43650672/article/details/116485851)
