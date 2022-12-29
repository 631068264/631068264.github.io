---
layout:     post
rewards: false
title:      Bigdata for Mac Install
categories:
    - big data
tags:
    - big data
---

# Hadoop
- Local (Standalone) Mode 本地（独立）模式  （默认情况）
- Pseudo-Distributed Mode 伪分布式模式
- Fully-Distributed Mode  全分布式模式

**伪分布式模式**

## 准备工作

### .ssh
检查〜/ .ssh / id_rsa和〜/ .ssh / id_rsa.pub文件是否存在，以验证是否存在**ssh localhost**密钥。 如果这些存在继续前进，如果不存在
```
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 0600 ~/.ssh/authorized_keys
```

### 系统配置
System Preferences -> Sharing, change Allow access for: All Users

### ssh login
`ssh: connect to host localhost port 22: Connection refused` 远程登录已关闭

```
$ sudo systemsetup -getremotelogin
Remote Login: off
```
open port 22 
```
$ sudo systemsetup -setremotelogin on
$ ssh localhost
Last login: ...
```

## install hadoop
3.1.1 version
不同版本的配置看 [官方文档 Single Node Setup](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SingleCluster.html)

```
brew install hadoop
```

## hadoop配置

go to `/usr/local/Cellar/hadoop/3.1.1/libexec/etc/hadoop`

### hadoop-env.sh

获取JAVA_HOME
```
➜  hadoop echo $JAVA_HOME
/Library/Java/JavaVirtualMachines/jdk1.8.0_77.jdk/Contents/Home
```

加入hadoop-env.sh
```
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_77.jdk/Contents/Home
```

### core-site.xml
```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```

### hdfs-site.xml
```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

## yarn 配置

### mapred-site.xml
```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.application.classpath</name>
        <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
    </property>
</configuration>
```

### yarn-site.xml
```xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
    </property>
</configuration>
```

## 启动关闭

格式化
```
hdfs namenode -format
```

hadoop
```
start-dfs.sh
stop-dfs.sh
```
[http://localhost:9870](http://localhost:9870)

`Permission denied: user=dr.who, access=READ_EXECUTE, inode="/tmp"`

`core-site.xml`
```
<property>
    <name>hadoop.http.staticuser.user</name>
    <value><user></value>
</property>
```
或者

`hdfs-default.xml`

`dfs.permissions.enabled=true #是否在HDFS中开启权限检查,默认为true`


yarn
```
start-yarn.sh
stop-yarn.sh
```
[http://localhost:8088](http://localhost:8088)

## 源码编译
```
brew install gcc autoconf automake libtool cmake snappy gzip bzip2 protobuf@2.5 zlib openssl maven
```

> 必须是`protobuf@2.5` 安装后按照**说明配置好PATH** `protoc version is 'libprotoc 3.6.1', expected version is '2.5.0'`

`hadoop checknative -a`

`mvn package -Pdist,native -DskipTests -Dtar -e`

### error

[Native Libraries Guide](https://hadoop.apache.org/docs/r3.1.1/hadoop-project-dist/hadoop-common/NativeLibraries.html#Download)

[Native build fails on macos due to getgrouplist not found](https://stackoverflow.com/questions/54801924/build-hadoop-3-1-1-in-osx-to-get-native-libraries)

OPENSSL_ROOT_DIR
```
export OPENSSL_ROOT_DIR="/usr/local/opt/openssl"
export LDFLAGS="-L${OPENSSL_ROOT_DIR}/lib"
export CPPFLAGS="-I${OPENSSL_ROOT_DIR}/include"
export PKG_CONFIG_PATH="${OPENSSL_ROOT_DIR}/lib/pkgconfig"
export OPENSSL_INCLUDE_DIR="${OPENSSL_ROOT_DIR}/include"
```

#### check native
经过好久终于可以了，配置好上面的话会更快。时间主要花在下载maven上面，老是ssl error
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g0hst1jktnj30vj0u0diq.jpg)

复制native libraries到
```
cp -R hadoop/hadoop-dist/target/hadoop-3.1.1/lib /usr/local/Cellar/hadoop/3.1.1/libexec
```
check build
```
➜  libexec hadoop checknative -a
2019-02-24 21:19:16,068 WARN bzip2.Bzip2Factory: Failed to load/initialize native-bzip2 library system-native, will use pure-Java version
2019-02-24 21:19:16,072 INFO zlib.ZlibFactory: Successfully loaded & initialized native-zlib library
2019-02-24 21:19:16,078 WARN erasurecode.ErasureCodeNative: ISA-L support is not available in your platform... using builtin-java codec where applicable
Native library checking:
hadoop:  true /usr/local/Cellar/hadoop/3.1.1/libexec/lib/native/libhadoop.dylib
zlib:    true /usr/lib/libz.1.dylib
zstd  :  false
snappy:  true /usr/local/lib/libsnappy.1.dylib
lz4:     true revision:10301
bzip2:   false
openssl: false build does not support openssl.
ISA-L:   false libhadoop was built without ISA-L support
2019-02-24 21:19:16,106 INFO util.ExitUtil: Exiting with status 1: ExitException
```

# Spark
`brew install apache-spark scala`
2.4.0 version

[spark-doc](https://spark.apache.org/docs/latest/cluster-overview.html)

在环境配置上有`HADOOP_CONF_DIR`才能用`spark-shell --master yarn`
```
export HADOOP_CONF_DIR=/usr/local/Cellar/hadoop/3.1.1/libexec/etc/hadoop
```
不要忘记`source`

# Hive
`brew install hive`
3.1.1 version

- Embedded mode (Derby) 内嵌模式 (实验)

> 默认此模式下，Metastore使用Derby数据库，数据库和Metastore服务都嵌入在主HiveServer进程中。当您启动HiveServer进程时，两者都是为您启动的。此模式需要最少的配置工作量，但它一次只能支持一个活动用户，并且未经过生产使用认证。
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g0g6b2hg8fj309p03bwed.jpg)

- Local mode 本地元存储
> 在本地模式下，Hive Metastore服务在与主HiveServer进程相同的进程中运行，但Metastore数据库在单独的进程中运行，并且可以位于单独的主机上。嵌入式Metastore服务通过JDBC与Metastore数据库通信。
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g0g6bb3u9nj309t06ajrd.jpg)

- Remote mode  远程元存储(推荐)

> 通过**Thrift**连接clinet和远程metastore服务`hive.metastore.uris`,metastore服务通过jdbc连接db'javax.jdo.option.ConnectionURL'  远程元存储需要单独起metastore服务，然后每个客户端都在配置文件里配置连接到该metastore服务。远程元存储的metastore服务和hive运行在不同的进程里。单独的主机上运行HiveServer进程可提供更好的可用性和可伸缩性。
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g0g6engvt5j30970ar74b.jpg)

metastore 支持常用的数据库
- Derby
- MySQL
- MS SQL Server
- Oracle
- Postgres

## mysql 配置

### jdbc
[下载mysql-connector-java.jar](https://mvnrepository.com/artifact/mysql/mysql-connector-java)
cp jar `/usr/local/Cellar/hive/3.1.1/libexec/lib`
```
cp mysql-connector-java-8.0.15.jar /usr/local/Cellar/hive/3.1.1/libexec/lib
```

### set up mysql
[Configuring the Hive Metastore](https://www.cloudera.com/documentation/enterprise/5-6-x/topics/cdh_ig_hive_metastore_configure.html)

```
mysql -u root -p
CREATE DATABASE metastore;
USE metastore;
CREATE USER 'hive'@'metastorehost' IDENTIFIED BY 'mypassword';

REVOKE ALL PRIVILEGES, GRANT OPTION FROM 'hive'@'metastorehost';
GRANT ALL PRIVILEGES ON metastore.* TO 'hive'@'metastorehost';
FLUSH PRIVILEGES;
quit;
```

## hive conf
```
cd /usr/local/Cellar/hive/3.1.1/libexec/conf
cp hive-default.xml.template hive-site.xml
```

先看一下`hive-site.xml`有没有报错<span class='heimu'>艹尼玛 apache !!!!!</span>再修改

### 错误处理

- `:3210`定位
```
Exception in thread "main" java.lang.RuntimeException: com.ctc.wstx.exc.WstxParsingException: Illegal character entity: expansion character (code 0x8
 at [row,col,system-id]: [3210,96,"file:/usr/local/Cellar/hive/3.1.1/libexec/conf/hive-site.xml"]
	at org.apache.hadoop.conf.Configuration.loadResource(Configuration.java:3003)
	at org.apache.hadoop.conf.Configuration.loadResources(Configuration.java:2931)
	at org.apache.hadoop.conf.Configuration.getProps(Configuration.java:2806)
	at org.apache.hadoop.conf.Configuration.get(Configuration.java:1460)
	at org.apache.hadoop.hive.conf.HiveConf.getVar(HiveConf.java:4990)
	at org.apache.hadoop.hive.conf.HiveConf.getVar(HiveConf.java:5063)
	at org.apache.hadoop.hive.conf.HiveConf.initialize(HiveConf.java:5150)
	at org.apache.hadoop.hive.conf.HiveConf.<init>(HiveConf.java:5093)
	at org.apache.hadoop.hive.common.LogUtils.initHiveLog4jCommon(LogUtils.java:97)
	at org.apache.hadoop.hive.common.LogUtils.initHiveLog4j(LogUtils.java:81)
	at org.apache.hadoop.hive.cli.CliDriver.run(CliDriver.java:699)
	at org.apache.hadoop.hive.cli.CliDriver.main(CliDriver.java:683)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.apache.hadoop.util.RunJar.run(RunJar.java:318)
	at org.apache.hadoop.util.RunJar.main(RunJar.java:232)
Caused by: com.ctc.wstx.exc.WstxParsingException: Illegal character entity: expansion character (code 0x8
 at [row,col,system-id]: [3210,96,"file:/usr/local/Cellar/hive/3.1.1/libexec/conf/hive-site.xml"]
	at com.ctc.wstx.sr.StreamScanner.constructWfcException(StreamScanner.java:621)
	at com.ctc.wstx.sr.StreamScanner.throwParseError(StreamScanner.java:491)
	at com.ctc.wstx.sr.StreamScanner.reportIllegalChar(StreamScanner.java:2456)
	at com.ctc.wstx.sr.StreamScanner.validateChar(StreamScanner.java:2403)
	at com.ctc.wstx.sr.StreamScanner.resolveCharEnt(StreamScanner.java:2369)
	at com.ctc.wstx.sr.StreamScanner.fullyResolveEntity(StreamScanner.java:1515)
	at com.ctc.wstx.sr.BasicStreamReader.nextFromTree(BasicStreamReader.java:2828)
	at com.ctc.wstx.sr.BasicStreamReader.next(BasicStreamReader.java:1123)
	at org.apache.hadoop.conf.Configuration$Parser.parseNext(Configuration.java:3257)
	at org.apache.hadoop.conf.Configuration$Parser.parse(Configuration.java:3063)
	at org.apache.hadoop.conf.Configuration.loadResource(Configuration.java:2986)
	... 17 more
```

- `java.net.URISyntaxException: Relative path in absolute URI: ${system:java.io.tmpdir%7D/$%7Bsystem:user.name%7D`

[启动HIVE时java.net.URISyntaxException](https://stackoverflow.com/questions/27099898/java-net-urisyntaxexception-when-starting-hive)

将以下内容放在hive-site.xml的开头
```xml
<property>
    <name>system:java.io.tmpdir</name>
    <value>/tmp/hive/java</value>
</property>
<property>
    <name>system:user.name</name>
    <value>${user.name}</value>
</property>
```


hive-site.xml
```xml
<property>
  <name>javax.jdo.option.ConnectionURL</name>
  <value>jdbc:mysql://localhost:3306/metastore?useSSL=false</value>
  <description>the URL of the MySQL database</description>
</property>

<property>
  <name>javax.jdo.option.ConnectionDriverName</name>
  <value>com.mysql.cj.jdbc.Driver</value>
</property>

<property>
  <name>javax.jdo.option.ConnectionUserName</name>
  <value>hive</value>
</property>

<property>
  <name>javax.jdo.option.ConnectionPassword</name>
  <value>mypassword</value>
</property>

<property>
  <name>hive.metastore.uris</name>
  <value>thrift://localhost:9083</value>
  <description>IP address (or fully-qualified domain name) and port of the metastore host</description>
</property>
```

初始化metastore & 测试
```
schematool -dbType mysql -initSchema

hive
hive> show tables;
```
![](https://cdn.jsdelivr.net/gh/631068264/img/006tKfTcgy1g0ghzk78a6j31ch0u0gqo.jpg)

## ERROR

### `schematool -dbType mysql -initSchema`

- [https://cwiki.apache.org/confluence/display/Hive/Hive+Schema+Tool](https://cwiki.apache.org/confluence/display/Hive/Hive+Schema+Tool)
- [https://stackoverflow.com/questions/42209875/hive-2-1-1-metaexceptionmessageversion-information-not-found-in-metastore](https://stackoverflow.com/questions/42209875/hive-2-1-1-metaexceptionmessageversion-information-not-found-in-metastore)


```
FAILED: HiveException java.lang.RuntimeException: Unable to instantiate org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient
```
因为没有正常启动Hive 的 Metastore Server服务进程。
```
hive --service metastore
```

```
MetaException(message:Version information not found in metastore.)
```

# Hive on Spark

> Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.

[Hive on Spark doc](https://cwiki.apache.org/confluence/display/Hive/Configuration+Properties#ConfigurationProperties-Spark)
```
<property>
  <name>hive.execution.engine</name>
  <value>spark</value>
</property>
```

版本问题
- [Hive on Spark: Getting Started](https://cwiki.apache.org/confluence/display/Hive/Hive+on+Spark%3A+Getting+Started)

```
Failed to execute spark task, with exception 'org.apache.hadoop.hive.ql.metadata.HiveException(Failed to create Spark client for Spark session 54d3f345-ca78-42c5-b3f4-5e7f07f77ae0)
FAILED: Execution Error, return code 30041 from org.apache.hadoop.hive.ql.exec.spark.SparkTask. Failed to create Spark client for Spark session 54d3f345-ca78-42c5-b3f4-5e7f07f77ae0
```
spark 通过`hive.metastore.uris` 连接hive

`cp hive-site.xml /usr/local/Cellar/apache-spark/2.4.0/libexec/conf`

启动Hive Metastore Server
`hive --service metastore`


```python
from pyspark.sql import SparkSession

spark = SparkSession.builder\
    .config("hive.metastore.uris", "thrift://localhost:9083/")\
    .enableHiveSupport() \
    .getOrCreate()
    
spark.sql('....')
```