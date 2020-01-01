---
layout:     post
rewards: false
title:  开发大数据坑
description: 记录使用spark(Structured Streaming)解析来自kafka的json数据保存到hbase 
categories:
    - big data
tags:
    - big data
---

# 安装

## HDP安装

避免组件之间不同版本的冲突

要设置好hostname `hostname -f`
- [ambari install](https://cwiki.apache.org/confluence/display/AMBARI/Installation+Guide+for+Ambari+2.7.5)
- [ambari install2](https://docs.cloudera.com/HDPDocuments/Ambari-2.7.4.0/bk_ambari-installation/content/name_your_cluster.html)

- [集群1](https://zhuanlan.zhihu.com/p/44270177)
- [集群2](https://www.cnblogs.com/zlslch/p/6629251.html)

- [dfs.namenode.http-bind-host](https://community.cloudera.com/t5/Support-Questions/Ambari-Namenode-UI-unable-to-acess-from-outside-of-cluster/m-p/112734/highlight/true#M75553)

[hive mysql-connector-java.jar 404 not found ](https://community.cloudera.com/t5/Community-Articles/Hive-start-failed-because-of-ambari-error-mysql-connector/ta-p/247743)


### add new hosts 一直都是preparing

`tail -f /var/log/ambari-server/ambari-server.log`


[Error executing bootstrap Cannot create /var/run/ambari-server/bootstrap](https://community.cloudera.com/t5/Support-Questions/Ambari-Status-quot-Preparing-quot-in-confirmation-of-hosts/m-p/184097/highlight/true#M146247)



## pom scope

编译，测试，运行

- 默认**compile** 打包的时候通常需要包含进去
- **test** 仅参与测试相关的工作，包括测试代码的编译,执行
- **provided** 不会将包打入本项目中，只是依赖过来，编译和测试时有效
  (一般运行环境已经存在提供对应jar,项目不用)
  
引入依赖时注意也**安装的组件的版本对应**，根据官方选择对应的scala version

```xml
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
    <spark.version>2.3.2</spark.version>
    <scala.version>2.11.12</scala.version>
</properties>
    
<dependencies>
    <dependency>
        <groupId>org.scala-lang</groupId>
        <artifactId>scala-library</artifactId>
        <version>${scala.version}</version>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-core_2.11</artifactId>
        <version>${spark.version}</version>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-streaming_2.11</artifactId>
        <version>${spark.version}</version>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-sql_2.11</artifactId>
        <version>${spark.version}</version>
    </dependency>
    <!--kafka-->
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-sql-kafka-0-10_2.11</artifactId>
        <version>${spark.version}</version>
    </dependency>
    <!--hbase-->
    <dependency>
        <groupId>org.apache.hbase</groupId>
        <artifactId>hbase-client</artifactId>
        <version>2.0.2</version>
    </dependency>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>RELEASE</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

# 代码

## kafka SSL connect

- [Structured Streaming spark](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)
- [kafka for Structured Streaming](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html)
- [spark example](https://sparkbyexamples.com/)
- [kafka ssl](http://kafka.apache.org/documentation.html#security_ssl)
  [gen-ssl-certs.sh](https://github.com/edenhill/librdkafka/blob/master/tests/gen-ssl-certs.sh)

```java
SparkSession spark = SparkSession
                .builder()
                .appName("kafka-hbase-test")
//                .master("local[*]")
                .getOrCreate();

spark.readStream().format("kafka")
                .option("kafka.bootstrap.servers", prop.getProperty("kafka.broker.list"))
                .option("kafka.ssl.truststore.location", prop.getProperty("kafka.jks.path"))
                .option("kafka.ssl.truststore.password", prop.getProperty("kafka.jks.passwd"))
                .option("kafka.security.protocol", "SSL")
                .option("kafka.ssl.endpoint.identification.algorithm", "")
                .option("startingOffsets", "latest")
                .option("subscribe", "xx,xx")
                .load()
                .selectExpr("CAST(topic AS STRING)", "CAST(value AS STRING)");
    }
```

## json 解析

```java
StructType trafficSchema = new StructType()
                .add("guid", DataTypes.LongType)
                .add("srcport", DataTypes.IntegerType)
                .add("destip", DataTypes.StringType)
                ........
                .add("downsize", DataTypes.LongType);

Dataset<Row> ds = df.select(functions.from_json(df.col("value").cast(DataTypes.StringType), trafficSchema).as("data")).select("data.*");
```

## implements Serializable

`org.apache.spark.SparkException: Task not serializable`

java bean 和 spark job 都要 `implements Serializable`
，或者使用`Kyro`但是没看懂怎么用

## java bean

JavaBean 实现 **getter and setter**, 不然会**foreach**接收到对象是为**null**

```java
// 转换成java bean
Dataset<Traffic> dTraffic = ds.as(ExpressionEncoder.javaBean(Traffic.class));
StreamingQuery query = dTraffic.writeStream().foreach(new HBaseForeachWriter<Traffic>() {
    @Override
    public String setTableName() {
        return "loh:traffic";
    }

    @Override
    public List<Put> toPut(Traffic record) {
        List<Put> putList = new LinkedList<>();
        Long rowKey = record.getGuid();
        putList.add(HBaseUtils.put(rowKey, "src", "srcip", record.getSrcip()));
        putList.add(HBaseUtils.put(rowKey, "src", "srcmac", record.getSrcmac()));
        putList.add(HBaseUtils.put(rowKey, "src", "srcport", record.getSrcport()));
       ......
        return putList;
    }
}).start();
```

## shc库不支持spark stream

[垃圾库](https://github.com/hortonworks-spark/shc/issues/205)，还要自己编译，依赖也不一定是你想要的，还不如自己用基础库撸

```java
public abstract class HBaseForeachWriter<T> extends ForeachWriter<T> {
    String tableName;
    private Table table;


    @Override
    public boolean open(long partitionId, long version) {
        this.tableName = setTableName();
        try {
            this.table = HBaseUtils.getTable(this.tableName);
            return true;
        } catch (IOException e) {
            e.printStackTrace();
            return false;
        }
    }

    @Override
    public void process(T value) {
        List<Put> put = toPut(value);
        try {
            this.table.put(put);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }


    @Override
    public void close(Throwable errorOrNull) {
        try {
            this.table.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public abstract String setTableName();

    public abstract List<Put> toPut(T record);

}
```

```java
public class HBaseUtils {
    private static Connection conn;
    private static Admin admin;

    static {
        try {
            Configuration conf = HBaseConfiguration.create();
            Properties prop = Config.getProp();
            conf.set("hbase.zookeeper.quorum", prop.getProperty("hbase.zookeeper.quorum"));
            conf.set("hbase.zookeeper.property.clientPort", prop.getProperty("hbase.zookeeper.property.clientPort"));
            conf.set("zookeeper.znode.parent", prop.getProperty("zookeeper.znode.parent"));

            HBaseUtils.conn = ConnectionFactory.createConnection(conf);
            HBaseUtils.admin = HBaseUtils.conn.getAdmin();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static Table getTable(String tableName) throws IOException {
        return conn.getTable(TableName.valueOf(tableName));
    }

    public static TableName[] listTableNames() throws IOException {
        return admin.listTableNames();
    }

    public static Put put(Object rowKey, String family, String column, Object value) {
        Put put = new Put(Bytes.toBytes(String.valueOf(rowKey)));
        put.addColumn(Bytes.toBytes(family), Bytes.toBytes(column), Bytes.toBytes(String.valueOf(value)));
        return put;
    }

}
```
## The node /hbase is not in ZooKeeper

查看 **conf/hbase-site.xml** 文件，找到配置项：**zookeeper.znode.parent**

它就是表示HBase 在 ZooKeeper 中的管理目录，里面存储着关于 HBase
集群的各项重要信息：

```xml
<property>
  <name>zookeeper.znode.parent</name>
  <value>/hbase-unsecure</value>
</property>
```

查看 **conf/hbase-env.sh** 里面的配置信息：HBASE_MANAGES_ZK，
- 这个参数是告诉 HBase 是否使用自带的 ZooKeeper 管理 HBase 集群。如果为 true，
- 则使用自带的 ZooKeeper ,如果为 false，则使用外部的 ZooKeeper。

`export HBASE_MANAGES_ZK=false`


hbase分布式配置

```xml
<property>
    <name>hbase.cluser.distributed</name>
    <value>true</value>
</property>
```


## 运行代码找不到provided包

`java.lang.NoClassDefFoundError: org/apache/spark/internal/Logging`

**pom**已经有spark的依赖，但是run跑不起来,是因为maven里面配置了`<scope>provided</scope>`，配置了这个就可以跑了
![](https://tva1.sinaimg.cn/large/006tNbRwgy1ga4st1zhjqj31cn0u0jwh.jpg)

[spark no class found](https://stackoverflow.com/questions/58899517/structured-streaming-kafka-spark-java-lang-noclassdeffounderror-org-apache-spar)

# spark submit

## java.lang.ClassNotFoundException: Failed to find data source: kafka.

- [maven-shade-plugin](https://maven.apache.org/plugins/maven-shade-plugin/examples/resource-transformers.html#ServicesResourceTransformer)
    - ManifestResourceTransformer 指定入口函数 
    - ServicesResourceTransformer META-INF/service

- [Failed to find data source kafka](https://stackoverflow.com/a/57656062/5360312)

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <configuration>
        <createDependencyReducedPom>false</createDependencyReducedPom>
        <filters>
            <filter>
                <artifact>*:*</artifact>
                <excludes>
                    <exclude>META-INF/*.SF</exclude>
                    <exclude>META-INF/*.DSA</exclude>
                    <exclude>META-INF/*.RSA</exclude>
                </excludes>
            </filter>
        </filters>
    </configuration>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>shade</goal>
            </goals>
            <configuration>
                <transformers>
                    <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                    <transformer
                            implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                        <mainClass>com.yizhisec.bigdata.KafkaStructStream</mainClass>
                    </transformer>
                </transformers>
            </configuration>
        </execution>
    </executions>
</plugin>
```

## 配置路径问题

- [在IDE里面可以获取`resources`下的配置文件路径，但是打包成jar后就不行](https://stackoverflow.com/questions/59402669/java-get-file-path-in-resources-fail-in-jar)
- [How to get a path to a resource in a Java JAR file](https://stackoverflow.com/questions/941754/how-to-get-a-path-to-a-resource-in-a-java-jar-file)

```java
public class Config {
    public static final String CONFIG_FILE = "config.properties";
    public static final String KAFKA_JKS = "kafka.truststore.jks";

    public static Properties getProp() throws IOException {
        Properties prop = new Properties();
        InputStream inputStream = Config.class.getClassLoader().getResourceAsStream(CONFIG_FILE);
        if (inputStream != null) {
            prop.load(inputStream);
            return prop;
        } else {
            throw new FileNotFoundException("property file '" + CONFIG_FILE + "' not found in the classpath");
        }
    }

    public static String getPath(String fileName) throws FileNotFoundException {
        URL fileURL = Config.class.getClassLoader().getResource(fileName);
        if (fileURL != null) {
            return fileURL.getPath();
        } else {
            throw new FileNotFoundException(fileName + "' not found in the classpath");
        }
    }
}
```

只能在使用服务器固定路径


## yarn.ApplicationMaster: Final app status: FAILED, exitCode: 13

- [spark run on yarn](https://stackoverflow.com/a/36605869/5360312)

```shell
spark-submit --class com.yizhisec.bigdata.KafkaStructStream --master yarn --deploy-mode cluster --supervise bigdata-1.0.jar
```

去掉代码里面的关于`setMaster`

## Spark Job Keep on Running

使用 `deploy-mode`
[spark job 后台运行](https://stackoverflow.com/questions/37201918/spark-job-keep-on-running)

## hadoop Permission denied

`org.apache.hadoop.security.AccessControlException: Permission denied: 
user=root, access=WRITE, inode="/":root:supergroup:drwxr-xr-x`

- [submit spark by root ](https://github.com/sequenceiq/docker-spark/issues/30)
- [Permission-denied](https://community.cloudera.com/t5/Support-Questions/Permission-denied-user-root-access-WRITE-inode-quot-user/td-p/4943)

```
sudo -u hdfs hadoop fs -mkdir /user/root
sudo -u hdfs hadoop fs -chown root /user/root
sudo -u hdfs hadoop fs -chmod 755 /user/root

export HADOOP_USER_NAME=root
```

- [stop spark job](https://community.cloudera.com/t5/Support-Questions/What-is-the-correct-way-to-start-stop-spark-streaming-jobs/td-p/30183)

```
yarn application -list

yarn application -kill application_id
```

# Phoenix

## migrate hbase to phoenix
 
`./sqlline.py node1:2181`

`./psql.py node1:2181 -m "loh"."traffic"`

- [namespace mapping](https://phoenix.apache.org/namspace_mapping.html)
- [语法](https://phoenix.apache.org/language/)
- [functions](https://phoenix.apache.org/language/functions.html)
- [hbase namespace](https://blog.csdn.net/opensure/article/details/46470969)

配置`phoenix.schema.isNamespaceMappingEnabled=true`在`hbase-site.xml`自动映射

```sql
create 'loh:traffic',"src","dest","time","info"

create schema "loh";

view只读 VARCHAR ,schema table_name family_name , column 大小写一致 才能映射
create view "loh"."traffic"(
            "id" VARCHAR PRIMARY KEY,
            "src"."srcip" VARCHAR,
            "src"."srcmac" VARCHAR,
            ...
            "info"."downsize" VARCHAR
            );
```
- [关于 phoenix 的 引号](https://stackoverflow.com/questions/59465253/phoenix-select-view-undefined-column)

命令行中字符串用单引号，表，列用双引号区分大小写(默认是大写)

```sql
select * from "traffic" where TO_NUMBER("downsize") > 0;
select * from "traffic" where "guid"='6773891525274486511'
```


## `Error: Operation timed out. (state=TIM01,code=6000)`

phoenix.query.timeoutMs


## 开启Phoenix Query Server

- [phoenix server](http://phoenix.apache.org/server.html)
- [jdbc url](https://phoenix.apache.org/faq.html#What_is_the_Phoenix_JDBC_URL_syntax)

[HBase2.0中的Benchmark工具](https://www.cnblogs.com/felixzh/p/10246335.html)
[读写测试](https://blog.csdn.net/qq_24651739/article/details/81188053)
```
hbase pe --oneCon=true --valueSize=100 --rows=1000000 --autoFlush=true --presplit=1 sequentialWrite 100
```

# 性能测试对比clickhouse


数据导入导出

```

clickhouse client --port xxx --password="xxxxx" -d database --query="INSERT INTO table FORMAT CSVWithNames" < data.csv

clickhouse client --port xxx --password="xxxxx" -d database --query="SELECT * FROM table  FORMAT CSVWithNames" > data.csv

```


# hadoop

## 复制文件到hadoop报错

`java.io.IOException: Failed to replace a bad datanode on the existing
pipeline due to no more good datanodes being available to try.`

- [写入出错](https://www.cnblogs.com/codeOfLife/p/5940613.html)

> hdfs-site
- dfs.client.block.write.replace-datanode-on-failure.enable true
- dfs.client.block.write.replace-datanode-on-failure.policy NEVER

## 导入csv到hbase

- [Import CSV data into HBase using importtsv](https://community.cloudera.com/t5/Community-Articles/Import-CSV-data-into-HBase-using-importtsv/ta-p/244842)

```
hadoop fs -copyFromLocal hbase.csv /tmp
hbase org.apache.hadoop.hbase.mapreduce.ImportTsv -Dimporttsv.separator=',' -Dimporttsv.columns="HBASE_ROW_KEY,TIME:date,INFO:guid,TIME:time,TIME:end_time,SRC:srcip,SRC:src_ip_range_id,SRC:srcport,SRC:srcmac,SRC:srccountry,DEST:destip,INFO:host,DEST:dest_ip_range_id,DEST:destport,DEST:destmac,DEST:destcountry,INFO:sensorid,INFO:proto,INFO:appproto,INFO:status,INFO:upsize,INFO:downsize,INFO:detail" TRAFFIC /tmp/hbase-data.csv
```

- [大文件读取](https://itbilu.com/linux/man/Nkz2hoeNm.html)

```
split -C 100M large_file.txt stxt
cat stxt* > new_file.txt
```

## 清理 hadoop

- [hadoop 命令](https://segmentfault.com/a/1190000002672666#item-1-4)

```
hadoop fs

hdfs dfs -rm -skipTrash /path_to_directory
hdfs dfs -expunge
```

## yarn log

```
yarn logs -applicationId <application ID>
```

## YARN Registry DNS Start failed

YARN `hadoop.registry.dns.bind-port` default value = 53