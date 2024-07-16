### 简介
**架构图：**
![image](https://pic1.imgdb.cn/item/634e0b2216f2c2beb1ad7d36.png)

由上图可以看到，整个HBase组件是建立在HDFS的基础上，HBase集群中存在一个HMater和多个HRegionServer，HMaster是通过Zookeeper来管理整个HRegionServer的负载均衡，调整HRegin的分配。如果说当前的HMaster死掉之后，剩余的机器会进行选举，会有机会成为下一个HMaster。

- **Client:** 指HBase开放给开发人员使用HBase的入口，例如, HBase shell控制台入口，用JAVA API编写的程序
- **Zookeeper:** 为分布式系统提供协同服务的软件，主要作用有：
1. 配置同步、时间同步；
2. 相互感知（集群中各主机的上下线消息及各种状态），选举HMaster
3. 配置信息存储，master与slaver机器的信息、hbase:meta表所在的位置、HBase系统中所有表信息
- **HMaster:**
1. 分发Region， 如果HBase中一张表所包含的内容超过了设定的上限，也就是一张表太多的话，就会被分成两部分，被切分的两部分就成为Region，这里就有两个Region
2. 监控HRegionServer的故障转移
3. 负责HRegionMaster的负载均衡
4. 管理元数据，主要是HBase:meta表，元数据表是负责记载你想要查询的数据是在哪台HRegionServer上保存着的信息的，起到加快查询的作用
- **HRegionServer**
一个HRegionServer就代表一台计算机，其实也就是一台计算机上就运行一个HRegionServer进程，主要有以下组件：
1. HLog: 每个HRegionServer可以看到里面维护着一个HLog, HLog的作用就是说将一系列的写操作进行保存，如果某一时刻服务器宕机，我们可以再次读取HLog中的操作进行数据还原，重新将数据写回HDFS。
2. HRegion：一个HRegionServer中有0~n个HRegion，类似n个HRegionServer构成整个HBase集群实现分布式组成的重要部分，n个Region同样也是实现大表存储的重要部分
3. Store: 一个Store就代表一个列族（列的集合），类似Region，Store是更细分的存储部分，而MenStore和0~n个StoreFile构成了最终的文件存储

因此，结合上面HRegionServer组件的理解，可以看出HRegionServer的工作职责：
1. 托管数据，HMaster只负责作决策，实际的数据存储的工作则全部由HRegionServer来存储、完成。
2. 维护HLog，负责更新或删除HLog中的内容，HLog是直接按行存储的，只要客户端发送一次存储请求过来，HLog的末尾都会按一定格式添加进去，直到这些数据被持久化到HDFS中以后，才会删除HLog中对应的信息。
3. 大、小合并，就是负责HBase系统中小文件太多，就将它合并成一个大-点的文件。
4. 监控Region，如果一个Region容量过大，会影响检索速度

### HBase列式存储理解
下面的示例是Boogtable纸的第2页上的一种稍微修改的形式。有一个名为WebTable的表，其中包含两个行（com.cnn.www and com.example.www），三个列家族称为内容，锚和人。在此示例中，对于第一行（com.cnn.www），Anchor包含两个列（锚：cssnsi.com，锚点：my.look.ca），并且内容包含一个列（内容：HTML）。此示例包含使用行键com.cnn.www的5个版本，以及带有行键com.example.www的行的一个版本。内容：HTML列预选赛包含给定网站的整个HTML。锚柱家族的预选赛每个都包含外部站点，该站点链接到该行代表的站点，以及其在其链接的锚点中使用的文本。 People Compon家族代表与该网站相关的人。
按照惯例，列名由其列族前缀和限定符组成。例如，列内容：html由列族内容和html限定符组成。冒号字符（：）将列族与列族限定符分隔开。
Tabke webtable

| Row Key | Time Stamp | ColumnFamily contents | ColumnFamily anchor | ColumnFamily people |
| --- | --- | --- | --- | --- |
| "com.cnn.www" | t9 |  | anchor:cnnsi.com = "CNN" |  |
| "com.cnn.www" | t8 |  | anchor:my.look.ca = "CNN.com" |  |
| "com.cnn.www" | t6 | contents:html = "<html>…​" |  |  |
| "com.cnn.www" | t5 | contents:html = "<html>…​" |  |  |
| "com.cnn.www" | t3 | contents:html = "<html>…​" |  |  |
| "com.example.www" | t5 | contents:html = "<html>…​" |  | people:author = "John Doe" |

此表中显示为空的单元格不占用空间，或者实际上在HBase中存在。这就是HBase“稀疏”的原因。表格视图不是查看HBase中数据的唯一可能方式，甚至不是最准确的方式。以下所示信息与多维地图相同。这只是一个用于说明目的的模型，可能并不精确。
```
{
  "com.cnn.www": {
    contents: {
      t6: contents:html: "<html>..."
      t5: contents:html: "<html>..."
      t3: contents:html: "<html>..."
    }
    anchor: {
      t9: anchor:cnnsi.com = "CNN"
      t8: anchor:my.look.ca = "CNN.com"
    }
    people: {}
  }
  "com.example.www": {
    contents: {
      t5: contents:html: "<html>..."
    }
    anchor: {}
    people: {
      t5: people:author: "John Doe"
    }
  }
}
```

#### 物理视图
虽然在概念级别上，表可能被视为一组稀疏的行，但它们是按列族进行物理存储的。可以随时将新的列限定符（column_family:column_限定符）添加到现有列族中。
Table ColumnFamily anchor
| Row Key | Time Stamp | Column Family anchor |
| --- | --- | --- |
| "com.cnn.www" | t9 | anchor:cnnsi.com = "CNN" |
| "com.cnn.www" | t8 | anchor:my.look.ca = "CNN.com" |

Table ColumnFamily contents

| Row Key | Time Stamp | ColumnFamily contents |
| --- | --- | --- |
| "com.cnn.www" | t6 | contents:html = "<html>…​" |
| "com.cnn.www" | t5 | contents:html = "<html>…​" |
| "com.cnn.www" | t3 | contents:html = "<html>…​" |

概念视图中显示的空单元根本不会存储。因此，在时间戳t8处请求contents:html列的值将不会返回任何值。类似地，对锚的请求：my.look。时间戳t9处的ca值将不返回值。但是，如果没有提供时间戳，则会返回特定列的最新值。给定多个版本，最新的也是第一个找到的版本，因为时间戳是按降序存储的。因此，请求com.cnn行中所有列的值。如果没有指定时间戳，www将是：timestamp t6中的contents:html的值，anchor:cnnsi的值。com来自时间戳t9，锚的值为：my.look。时间戳t8的ca。

有关Apache HBase如何存储数据的内部机制的更多信息，请参阅[regions.arch](https://hbase.apache.org/book.html#regions.arch)。

### 快速入门-独立HBase
一个独立示例具有所有HBase守护进程--Master、RegionServers和ZooKeeper--在一个持久化到本地文件系统的JVM中运行。主要涉及两方面介绍：
- ./bin/conf
- hbase shell cli
#### 安装单机HBase
1. [从这个Apache网站下载镜像](https://www.apache.org/dyn/closer.lua/hbase/)， 下载HBase stable的版本，以bin.tar.gz结尾的镜像， 解压
```
$ tar zxvf hbase-2.5.0-bin.tar.gz
$ cd ./hbase-2.5.0
```

2. 修改config/hbase-env.sh的JAVA_HOME配置
```
$ whereis java
java: /home/hp/jdk1.8.0_311/bin/java
$ vi ./config/hbase-env.sh
export JAVA_HOME=/home/hp/jdk1.8.0_311/bin/java
```

3. 启动bin/start-hbase.sh脚本，查看控制台输出日志和jps查看进程
```
$ ./bin/start-hbase.sh 
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/home/hp/hadoop-3.2.4/share/hadoop/common/lib/slf4j-reload4j-1.7.35.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/home/hp/hbase-2.5.0/lib/client-facing-thirdparty/log4j-slf4j-impl-2.17.2.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.slf4j.impl.Reload4jLoggerFactory]
running master, logging to /home/hp/hbase-2.5.0/bin/../logs/hbase-hp-master-node-6.out
$ jps
37139 Jps
36743 HMaster
```
此时，可以通过http://node-6:16010/ 访问HBase Web UI
要停止HBase，可以使用以下命令
```
$ ./bin/stop-hbase.sh
stopping hbase....................
```

#### HBase shell cli使用
1. 连接到HBase
```
$ ./bin/hbase shell
.... # 省略部分启动信息
hbase:002:0>
```
2. 显示HBase Shell 帮助文本
键入**help**并按 Enter，以显示 HBase Shell 的一些基本使用信息以及几个示例命令。请注意，表名、行、列都必须用引号字符括起来。

3. 创建一个表
使用create命令创建一个新表。您必须指定表名称和 ColumnFamily 名称。
```
hbase(main):001:0> create 'test', 'cf'
0 row(s) in 0.4170 seconds

=> Hbase::Table - test
```

4. 列出有关您的表的信息
使用list命令确认您的表存在
```
hbase(main):002:0> list 'test'
TABLE
test
1 row(s) in 0.0180 seconds

=> ["test"]
```
现在使用describe命令查看详细信息，包括配置默认值
```
hbase(main):003:0> describe 'test'
Table test is ENABLED
test
COLUMN FAMILIES DESCRIPTION
{NAME => 'cf', VERSIONS => '1', EVICT_BLOCKS_ON_CLOSE => 'false', NEW_VERSION_BEHAVIOR => 'false', KEEP_DELETED_CELLS => 'FALSE', CACHE_DATA_ON_WRITE =>
'false', DATA_BLOCK_ENCODING => 'NONE', TTL => 'FOREVER', MIN_VERSIONS => '0', REPLICATION_SCOPE => '0', BLOOMFILTER => 'ROW', CACHE_INDEX_ON_WRITE => 'f
alse', IN_MEMORY => 'false', CACHE_BLOOMS_ON_WRITE => 'false', PREFETCH_BLOCKS_ON_OPEN => 'false', COMPRESSION => 'NONE', BLOCKCACHE => 'true', BLOCKSIZE
 => '65536'}
1 row(s)
Took 0.9998 seconds
```

5. 将数据放入表中。
要将数据放入表中，请使用put命令。
```
hbase(main):003:0> put 'test', 'row1', 'cf:a', 'value1'
0 row(s) in 0.0850 seconds

hbase(main):004:0> put 'test', 'row2', 'cf:b', 'value2'
0 row(s) in 0.0110 seconds

hbase(main):005:0> put 'test', 'row3', 'cf:c', 'value3'
0 row(s) in 0.0100 seconds
```
在这里，我们一次插入三个值。第一个插入位于row1列cf:a，值为value1。HBase 中的列由一个列族前缀组成，cf在本例中，后跟一个冒号，然后是一个列限定符后缀，a在本例中。

6. 一次扫描表中的所有数据。
从 HBase 获取数据的方法之一是扫描。使用scan命令扫描表中的数据。您可以限制您的扫描，但目前，所有数据都已获取。
```
hbase(main):006:0> scan 'test'
ROW                                      COLUMN+CELL
 row1                                    column=cf:a, timestamp=1421762485768, value=value1
 row2                                    column=cf:b, timestamp=1421762491785, value=value2
 row3                                    column=cf:c, timestamp=1421762496210, value=value3
3 row(s) in 0.0230 seconds
```

7. 获取单行数据。
要一次获取一行数据，请使用该get命令。
```
hbase(main):007:0> get 'test', 'row1'
COLUMN                                   CELL
 cf:a                                    timestamp=1421762485768, value=value1
1 row(s) in 0.0350 seconds
```

8. 禁用表。
如果要删除表或更改其设置，以及在某些其他情况下，您需要先禁用该表，使用该disable命令。enable您可以使用命令重新启用它。
```
hbase(main):008:0> disable 'test'
0 row(s) in 1.1820 seconds

hbase(main):009:0> enable 'test'
0 row(s) in 0.1770 seconds
```

9. 删除table
要删除（删除）表，请使用drop命令。
```
hbase(main):011:0> drop 'test'
0 row(s) in 0.1370 seconds
```

10. 退出HBase Shell
要退出HBase Shell 并断开与集群的连接，请使用该quit命令。HBase 仍在后台运行。

### Zookeeper集群安装
1. 上传apache-zookeeper-3.7.1-bin.tar.gz安装包到node-4节点安装
```
cd /home/hp
tar -zxvf apache-zookeeper-3.7.1-bin.tar.gz
mv apache-zookeeper-3.7.1-bin zookeeper-3.7.1
```
添加环境配置：
```
# zookeeper
export ZK_HOME=/home/hp/zookeeper-3.7.1
export PATH=$PATH:$ZK_HOME/bin
```

2. 进入zookeeper目录，创建zkData，创建myid文件
```
cd zookeeper-3.7.1
mkdir zkData
touch myid
echo "1" > ./myid
```

3. 进入conf文件夹，修改zoo_sample.cfg名字为zoo.cfg
```
cp zoo_sample.cfg zoo.cfg
```

4. 修改zoo.cfg文件
在文件最后添加以下内容：
```
server.1=node-4:2888:3888
server.2=node-5:2888:3888
server.3=node-6:2888:3888
```

5. 同步文件到其他节点
```
scp -r /home/hp/zookeeper-3.7.1 node-5:/home/hp/zookeeper-3.7.1
scp -r /home/hp/zookeeper-3.7.1 node-6:/home/hp/zookeeper-3.7.1
```

6. 分别将node-5和node-6节点的/home/hp/zookeeper-3.7.1/zkData/myid修改为2,3
```
[hp@node-5 zookeeper-3.7.1]$ cat /home/hp/zookeeper-3.7.1/zkData/myid
2
[hp@node-6 zookeeper-3.7.1]$ cat /home/hp/zookeeper-3.7.1/zkData/myid 
3
```

7. 完成以上步骤之后，分布在node-4、node-5、node-6所有节点上启动bin/zkServer.sh start
```
cd /home/hp/zookeeper-3.7.1
./bin/zkServer.sh start
```


8.启动之后，状态检查
node-4节点(leader)：
```
[hp@node-4 zookeeper-3.5.10]$ ./bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /home/hp/zookeeper-3.5.10/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost. Client SSL: false.
Mode: leader
[hp@node-4 zookeeper-3.5.10]$ jps
19637 Jps
19199 QuorumPeerMain
```
node-5节点(follower)：
```
[hp@node-5 zookeeper-3.5.10]$ ./bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /home/hp/zookeeper-3.5.10/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost. Client SSL: false.
Mode: follower
[hp@node-5 zookeeper-3.5.10]$ jps
19281 QuorumPeerMain
19378 Jps
```
node-6节点(follower):
```
[hp@node-6 zookeeper-3.5.10]$ ./bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /home/hp/zookeeper-3.5.10/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost. Client SSL: false.
Mode: follower
[hp@node-6 zookeeper-3.5.10]$ jps
19394 Jps
19259 QuorumPeerMain
```
配置完成

#### 集群搭建
**前提准备：**
1. 三个虚拟机节点：
192.168.208.132 node-4
192.168.208.133 node-5
192.168.208.134 node-6

2. 三台虚拟机之间设置了ssh免密码登录，可以相互访问
 参考：[ssh-keygen -t rsa部分](https://hbase.apache.org/book.html#quickstart_fully_distributed)
3. 安装JDK、Hadoop集群并启动HDFS服务
4. 安装zookeeper集群并启动

**实践安装：**
1. 解压安装，参考上面单机安装部分
2. 修改配置hbase-env.sh
```
export JAVA_HOME=/home/hp/jdk1.8.0_311/bin/java
export HBASE_MANAGES_ZK=false
```
3. 修改hbase-site.xml
```
  <property>
    <name>hbase.rootdir</name>
    <value>hdfs://node-4:9000/hbase</value>
  </property>

  <property>
    <name>hbase.cluster.distributed</name>
    <value>true</value>
  </property>
  
 <property>
    <name>hbase.zookeeper.quorum</name>
    <value>node-4,node-5,node-6</value>
  </property>
  
<!-- 解决启动HMaster无法初始化WAL的问题 -->
  <property>
    <name>hbase.unsafe.stream.capability.enforce</name>
    <value>false</value>
  </property>
  
    <!-- Phoenix 支持HBase 命名空间映射 -->
  <property>
    <name>phoenix.schema.isNamespaceMappingEnabled</name>
    <value>true</value>
  </property>

  <property>
   <name>phoenix.schema.mapSystemTablesToNamespace</name>
    <value>true</value>
  </property>
  
  <property>
    <name>hbase.tmp.dir</name>
    <value>./tmp</value>
  </property>
```

4. 启动HBase集群
```
[hp@node-4 hbase-2.5.0]$ ./bin/start-hbase.sh
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/home/hp/hadoop-3.2.4/share/hadoop/common/lib/slf4j-reload4j-1.7.35.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/home/hp/hbase-2.5.0/lib/client-facing-thirdparty/log4j-slf4j-impl-2.17.2.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.slf4j.impl.Reload4jLoggerFactory]
running master, logging to /home/hp/hbase-2.5.0/bin/../logs/hbase-hp-master-node-4.out
node-5: running regionserver, logging to /home/hp/hbase-2.5.0/bin/../logs/hbase-hp-regionserver-node-5.out
node-6: running regionserver, logging to /home/hp/hbase-2.5.0/bin/../logs/hbase-hp-regionserver-node-6.out
node-4: running regionserver, logging to /home/hp/hbase-2.5.0/bin/../logs/hbase-hp-regionserver-node-4.out
```

### 问题汇总
1. 关闭HBase时，输入stop-hbase.sh命令一直处于等待状态。
stopping hbase…
解决办法是：
先输入hbase-daemon.sh stop master命令
再输入stop-hbase.sh命令。
这样hbase就可以成功关闭。

参考：
https://blog.csdn.net/m0_47256162/article/details/118117750
https://hbase.apache.org/book.html#quickstart
https://blog.csdn.net/qq_44665283/article/details/125759217
https://blog.csdn.net/foreveyking/article/details/121695043
