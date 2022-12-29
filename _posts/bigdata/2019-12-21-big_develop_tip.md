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
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301030355.jpg)

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

## 清理 hadoop文件

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

## find the failed log in mapreduce.Job

```
2019-12-31 06:02:10,611 INFO  [main] mapreduce.JobSubmitter: Submitting tokens for job: job_1577781650355_0001
2019-12-31 06:02:10,611 INFO  [main] mapreduce.JobSubmitter: Executing with tokens: []
2019-12-31 06:02:10,982 INFO  [main] conf.Configuration: found resource resource-types.xml at file:/etc/hadoop/3.1.4.0-315/0/resource-types.xml
2019-12-31 06:02:11,532 INFO  [main] impl.YarnClientImpl: Submitted application application_1577781650355_0001
2019-12-31 06:02:11,668 INFO  [main] mapreduce.Job: The url to track the job: http://node1:8088/proxy/application_1577781650355_0001/
2019-12-31 06:02:11,670 INFO  [main] mapreduce.Job: Running job: job_1577781650355_0001
2019-12-31 06:02:34,983 INFO  [main] mapreduce.Job: Job job_1577781650355_0001 running in uber mode : false
2019-12-31 06:02:34,985 INFO  [main] mapreduce.Job:  map 0% reduce 0%
2019-12-31 06:02:47,134 INFO  [main] mapreduce.Job:  map 4% reduce 0%
2019-12-31 06:02:50,161 INFO  [main] mapreduce.Job:  map 7% reduce 0%
2019-12-31 06:02:53,184 INFO  [main] mapreduce.Job:  map 10% reduce 0%
2019-12-31 06:02:56,212 INFO  [main] mapreduce.Job:  map 14% reduce 0%
2019-12-31 06:02:59,247 INFO  [main] mapreduce.Job:  map 17% reduce 0%
2019-12-31 06:03:02,275 INFO  [main] mapreduce.Job:  map 20% reduce 0%
2019-12-31 06:03:05,296 INFO  [main] mapreduce.Job:  map 24% reduce 0%
2019-12-31 06:03:08,323 INFO  [main] mapreduce.Job:  map 27% reduce 0%
2019-12-31 06:03:11,349 INFO  [main] mapreduce.Job:  map 29% reduce 0%
2019-12-31 06:03:14,370 INFO  [main] mapreduce.Job:  map 33% reduce 0%
2019-12-31 06:03:18,402 INFO  [main] mapreduce.Job:  map 35% reduce 0%
2019-12-31 06:03:21,427 INFO  [main] mapreduce.Job:  map 39% reduce 0%
2019-12-31 06:03:24,455 INFO  [main] mapreduce.Job:  map 42% reduce 0%
2019-12-31 06:03:27,480 INFO  [main] mapreduce.Job:  map 45% reduce 0%
2019-12-31 06:03:30,502 INFO  [main] mapreduce.Job:  map 48% reduce 0%
2019-12-31 06:03:30,542 INFO  [main] mapreduce.Job: Task Id : attempt_1577781650355_0001_m_000000_0, Status : FAILED
2019-12-31 06:03:32,651 INFO  [main] mapreduce.Job:  map 50% reduce 0%
2019-12-31 06:04:25,000 INFO  [main] mapreduce.Job: Task Id : attempt_1577781650355_0001_m_000000_1, Status : FAILED
2019-12-31 06:05:18,339 INFO  [main] mapreduce.Job: Task Id : attempt_1577781650355_0001_m_000000_2, Status : FAILED
2019-12-31 06:05:31,481 INFO  [main] mapreduce.Job:  map 52% reduce 0%
2019-12-31 06:05:34,495 INFO  [main] mapreduce.Job:  map 54% reduce 0%
2019-12-31 06:05:37,515 INFO  [main] mapreduce.Job:  map 56% reduce 0%
2019-12-31 06:05:40,559 INFO  [main] mapreduce.Job:  map 57% reduce 0%
2019-12-31 06:05:43,580 INFO  [main] mapreduce.Job:  map 58% reduce 0%
2019-12-31 06:05:46,626 INFO  [main] mapreduce.Job:  map 59% reduce 0%
2019-12-31 06:05:49,657 INFO  [main] mapreduce.Job:  map 60% reduce 0%
2019-12-31 06:05:52,703 INFO  [main] mapreduce.Job:  map 62% reduce 0%
2019-12-31 06:05:55,732 INFO  [main] mapreduce.Job:  map 63% reduce 0%
2019-12-31 06:05:58,763 INFO  [main] mapreduce.Job:  map 65% reduce 0%
2019-12-31 06:06:01,783 INFO  [main] mapreduce.Job:  map 67% reduce 0%
2019-12-31 06:06:04,804 INFO  [main] mapreduce.Job:  map 68% reduce 0%
2019-12-31 06:06:07,827 INFO  [main] mapreduce.Job:  map 70% reduce 0%
2019-12-31 06:06:10,843 INFO  [main] mapreduce.Job:  map 71% reduce 0%
2019-12-31 06:06:13,861 INFO  [main] mapreduce.Job:  map 73% reduce 0%
2019-12-31 06:06:16,892 INFO  [main] mapreduce.Job:  map 74% reduce 0%
2019-12-31 06:06:25,977 INFO  [main] mapreduce.Job:  map 76% reduce 0%
2019-12-31 06:06:28,994 INFO  [main] mapreduce.Job:  map 77% reduce 0%
2019-12-31 06:06:32,027 INFO  [main] mapreduce.Job:  map 78% reduce 0%
2019-12-31 06:06:35,045 INFO  [main] mapreduce.Job:  map 79% reduce 0%
2019-12-31 06:06:38,071 INFO  [main] mapreduce.Job:  map 81% reduce 0%
2019-12-31 06:06:41,088 INFO  [main] mapreduce.Job:  map 82% reduce 0%
2019-12-31 06:06:44,112 INFO  [main] mapreduce.Job:  map 84% reduce 0%
2019-12-31 06:06:47,130 INFO  [main] mapreduce.Job:  map 86% reduce 0%
2019-12-31 06:06:50,147 INFO  [main] mapreduce.Job:  map 87% reduce 0%
2019-12-31 06:06:53,181 INFO  [main] mapreduce.Job:  map 89% reduce 0%
2019-12-31 06:06:56,201 INFO  [main] mapreduce.Job:  map 91% reduce 0%
2019-12-31 06:06:59,215 INFO  [main] mapreduce.Job:  map 93% reduce 0%
2019-12-31 06:07:02,230 INFO  [main] mapreduce.Job:  map 95% reduce 0%
2019-12-31 06:07:05,246 INFO  [main] mapreduce.Job:  map 97% reduce 0%
2019-12-31 06:07:08,267 INFO  [main] mapreduce.Job:  map 98% reduce 0%
2019-12-31 06:07:11,296 INFO  [main] mapreduce.Job:  map 100% reduce 0%
2019-12-31 06:07:11,306 INFO  [main] mapreduce.Job: Job job_1577781650355_0001 completed successfully
2019-12-31 06:07:11,565 INFO  [main] mapreduce.Job: Counters: 35
```

- [How to find the failed log in mapreduce.Job](https://stackoverflow.com/questions/59543408/how-to-find-the-failed-log-in-mapreduce-job/59544530#59544530)

> The url to track the job:
> http://node1:8088/proxy/application_1577781650355_0001/

根据log找对应的task和**attempt_***

<span class='gp-3'>
    <img src='https://cdn.jsdelivr.net/gh/631068264/img/202212301030357.jpg' />
    <img src='https://cdn.jsdelivr.net/gh/631068264/img/202212301030358.jpg' />
    <img src='https://cdn.jsdelivr.net/gh/631068264/img/202212301030359.jpg' />
</span>

# hadoop namenode -format

格式化后重启遇到各种问题

##  Failed to add storage directory [DISK]file:/tmp/hadoop-hadoop/dfs/data/

-[Hadoop 数据节点DataNode异常](https://blog.csdn.net/gis_101/article/details/52679914)

```
WARN org.apache.hadoop.hdfs.server.common.Storage: Failed to add storage directory [DISK]file:/tmp/hadoop-hadoop/dfs/data/
java.io.IOException: Incompatible clusterIDs in /tmp/hadoop-hadoop/dfs/data: namenode clusterID = CID-1ac4e49a-ff06-4a34-bfa2-4e9d7248855b; datanode clusterID = CID-3ae02e74-742f-4915-92e7-0625fa8afcc5
at org.apache.hadoop.hdfs.server.datanode.DataStorage.doTransition(DataStorage.java:775)
at org.apache.hadoop.hdfs.server.datanode.DataStorage.loadStorageDirectory(DataStorage.java:300)
at org.apache.hadoop.hdfs.server.datanode.DataStorage.loadDataStorage(DataStorage.java:416)
at org.apache.hadoop.hdfs.server.datanode.DataStorage.addStorageLocations(DataStorage.java:395)
at org.apache.hadoop.hdfs.server.datanode.DataStorage.recoverTransitionRead(DataStorage.java:573)
at org.apache.hadoop.hdfs.server.datanode.DataNode.initStorage(DataNode.java:1362)
at org.apache.hadoop.hdfs.server.datanode.DataNode.initBlockPool(DataNode.java:1327)
at org.apache.hadoop.hdfs.server.datanode.BPOfferService.verifyAndSetNamespaceInfo(BPOfferService.java:317)
at org.apache.hadoop.hdfs.server.datanode.BPServiceActor.connectToNNAndHandshake(BPServiceActor.java:223)
at org.apache.hadoop.hdfs.server.datanode.BPServiceActor.run(BPServiceActor.java:802)
at java.lang.Thread.run(Thread.java:745)
2016-09-26 16:38:56,124 FATAL org.apache.hadoop.hdfs.server.datanode.DataNode: Initialization failed for Block pool (Datanode Uuid unassigned) service to s0/192.168.48.134:8020. Exiting.

```

**namenode和datanode的clusterID不一致**,删掉各个DataNode节点`/tmp/hadoop-hadoop/dfs/data/current`重启hadoop

## /hadoop/hdfs/namenode/current/VERSION (Permission denied)

- [Permission denied error during NameNode start](https://community.cloudera.com/t5/Support-Questions/Permission-denied-error-during-NameNode-start/td-p/132827)

```
WARN namenode.FSNamesystem (FSNamesystem.java:loadFromDisk(683)) - Encountered exception loading fsimage java.io.FileNotFoundException: /hadoop/hdfs/namenode/current/VERSION (Permission denied)
```

```
chown -R hdfs:hdfs /hadoop/hdfs/namenode
```

## log 目录

- `/var/log`
- `/var/log/hadoop/hdfs/hadoop-hdfs-namenode-<hostname>.log`

## YARN Timeline Service can not start due to HBase

```
2018-12-08 12:59:18,852 INFO  [main] client.RpcRetryingCallerImpl: Call exception, tries=6, retries=6, started=4859 ms ago, cancelled=false, msg=Call to examples.foodscience-01.de/163.49.39.115:17020 failed on connection exception: org.apache.hbase.thirdparty.io.netty.channel.AbstractChannel$AnnotatedConnectException: Connection refused: examples.foodscience-01.de/163.49.39.115:17020, details=row 'prod.timelineservice.entity' on table 'hbase:meta' at region=hbase:meta,,1.1588230740, hostname=examples.foodscience-01.de,17020,1543619998977, seqNum=-1
```

```
NoNode for /atsv2-hbase-secure/master, details=row 'prod.timelineservice.entity' on table 'hbase:meta' at null
```


```
hbase zkcli
rmr 和hbase有关的  重启
```

# Kylin

- [kylin-3.0.0在集群hadoop 3.1.1部署问题汇总](https://www.jianshu.com/p/ef383f6b91cc)


## install kylin on ambari

- [ambari-kylin-service](https://github.com/cas-packone/ambari-kylin-service/)

## Failed to create /kylin. Please make sure the user has right to access /kylin

```
KYLIN_HOME is set to /opt/kylin/latest
WARNING: log4j.properties is not found. HADOOP_CONF_DIR may be incomplete.
ERROR: JAVA_HOME /usr/jdk64/default does not exist.
Failed to create /kylin. Please make sure the user has right to access /kylin
```

- [Failed to create /kylin](https://stackoverflow.com/questions/50618154/i-get-error-failed-to-create-kylin-please-make-sure-the-user-has-right-to-acc)
- [JAVA _Home is not set in Hadoop](https://stackoverflow.com/questions/20628093/java-home-is-not-set-in-hadoop/29387495)


`kylin.env.hdfs-working-dir` in `$KYLIN_HOME/conf/kylin.properties`

```
cd $KYLIN_HOME/bin
vim check-env.sh
```

```
hadoop ${hadoop_conf_param} fs -mkdir -p $WORKING_DIR
hadoop ${hadoop_conf_param} fs -mkdir -p $WORKING_DIR/spark-history
```
替换成
```
hadoop  fs -mkdir -p $WORKING_DIR
hadoop  fs -mkdir -p $WORKING_DIR/spark-history
```

## Exception in thread "main" java.lang.IllegalArgumentException: Failed to find metadata store by url: kylin_metadata@hbase

```
Exception in thread "main" java.lang.IllegalArgumentException: Failed to find metadata store by url: kylin_metadata@hbase
	at org.apache.kylin.common.persistence.ResourceStore.createResourceStore(ResourceStore.java:89)
	at org.apache.kylin.common.persistence.ResourceStore.getStore(ResourceStore.java:101)
	at org.apache.kylin.rest.service.AclTableMigrationTool.checkIfNeedMigrate(AclTableMigrationTool.java:94)
	at org.apache.kylin.tool.AclTableMigrationCLI.main(AclTableMigrationCLI.java:41)
Caused by: java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at org.apache.kylin.common.persistence.ResourceStore.createResourceStore(ResourceStore.java:83)
	... 3 more
Caused by: java.lang.NoSuchMethodError: org.apache.curator.framework.api.CreateBuilder.creatingParentsIfNeeded()Lorg/apache/curator/framework/api/ProtectACLCreateModePathAndBytesable;
	at org.apache.kylin.storage.hbase.util.ZookeeperDistributedLock.lock(ZookeeperDistributedLock.java:145)
	at org.apache.kylin.storage.hbase.util.ZookeeperDistributedLock.lock(ZookeeperDistributedLock.java:166)
	at org.apache.kylin.storage.hbase.HBaseConnection.createHTableIfNeeded(HBaseConnection.java:305)
	at org.apache.kylin.storage.hbase.HBaseResourceStore.createHTableIfNeeded(HBaseResourceStore.java:110)
	at org.apache.kylin.storage.hbase.HBaseResourceStore.<init>(HBaseResourceStore.java:91)
	... 8 more
```

kylin 版本问题

## Something wrong with Hive CLI or Beeline, please execute Hive CLI or Beeline CLI in terminal to find the root cause.

```
vim bin/find-hive-dependency.sh

替换
hive_env=`hive ${hive_conf_properties} -e set 2>&1 | grep 'env:CLASSPATH'`
```
```
hive -e set >/tmp/hive_env.txt 2>&1
hive_env=`grep 'env:CLASSPATH' /tmp/hive_env.txt`
hive_env=`echo ${hive_env#*env:CLASSPATH}`
hive_env="env:CLASSPATH"${hive_env}
```

## Couldn't find hive executable jar. Please check if hive executable jar exists in HIVE_LIB folder.

因为find-hive-dependency.sh缺了设置`hive_exec_path`

```
hive_exec_path=

加入

if [ -n "$HCAT_HOME" ]
then
    verbose "HCAT_HOME is set to: $HCAT_HOME, use it to locate hive configurations."
    hive_exec_path=$HCAT_HOME
fi
```

## HIVE_LIB not found, please check hive installation or export HIVE_LIB='YOUR_LOCAL_HIVE_LIB'.

配置HIVE_LIB

## spark not found, set SPARK_HOME, or run bin/download-spark.sh

```
export HIVE_CONF=/usr/hdp/current/hive-client/conf
export HCAT_HOME=/usr/hdp/current/hive-webhcat
export HIVE_LIB=/usr/hdp/current/hive-client/lib
export SPARK_HOME=/usr/hdp/current/spark2-client
```

## UI 访问不了

log `$KYLIN_HOME/logs`目录的kylin.log和kylin.out

```
2020-01-13 06:04:16,747 ERROR [localhost-startStop-1] context.ContextLoader:350 : Context initialization failed
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping': Invocation of init method failed; nested exception is java.lang.NoClassDefFoundError: org/apache/commons/configuration/ConfigurationException
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1628)
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:555)
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:483)
        at org.springframework.beans.factory.support.AbstractBeanFactory$1.getObject(AbstractBeanFactory.java:306)
        at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:230)
        at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:302)
        at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:197)
        at org.springframework.beans.factory.support.DefaultListableBeanFactory.preInstantiateSingletons(DefaultListableBeanFactory.java:761)
        at org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization(AbstractApplicationContext.java:867)
        at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:543)
        at org.springframework.web.context.ContextLoader.configureAndRefreshWebApplicationContext(ContextLoader.java:443)
        at org.springframework.web.context.ContextLoader.initWebApplicationContext(ContextLoader.java:325)
        at org.springframework.web.context.ContextLoaderListener.contextInitialized(ContextLoaderListener.java:107)
        at org.apache.catalina.core.StandardContext.listenerStart(StandardContext.java:4792)
        at org.apache.catalina.core.StandardContext.startInternal(StandardContext.java:5256)
        at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:150)
        at org.apache.catalina.core.ContainerBase.addChildInternal(ContainerBase.java:754)
        at org.apache.catalina.core.ContainerBase.addChild(ContainerBase.java:730)
        at org.apache.catalina.core.StandardHost.addChild(StandardHost.java:734)
        at org.apache.catalina.startup.HostConfig.deployWAR(HostConfig.java:985)
        at org.apache.catalina.startup.HostConfig$DeployWar.run(HostConfig.java:1857)
        at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
        at java.util.concurrent.FutureTask.run(FutureTask.java:266)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)
Caused by: java.lang.NoClassDefFoundError: org/apache/commons/configuration/ConfigurationException
        at java.lang.Class.getDeclaredMethods0(Native Method)
        at java.lang.Class.privateGetDeclaredMethods(Class.java:2701)
        at java.lang.Class.getDeclaredMethods(Class.java:1975)
        at org.springframework.util.ReflectionUtils.getDeclaredMethods(ReflectionUtils.java:613)
        at org.springframework.util.ReflectionUtils.doWithMethods(ReflectionUtils.java:524)
        at org.springframework.core.MethodIntrospector.selectMethods(MethodIntrospector.java:68)
        at org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.detectHandlerMethods(AbstractHandlerMethodMapping.java:230)
        at org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.initHandlerMethods(AbstractHandlerMethodMapping.java:214)
        at org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.afterPropertiesSet(AbstractHandlerMethodMapping.java:184)
        at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping.afterPropertiesSet(RequestMappingHandlerMapping.java:127)
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.invokeInitMethods(AbstractAutowireCapableBeanFactory.java:1687)
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1624)
        ... 25 more
Caused by: java.lang.ClassNotFoundException: org.apache.commons.configuration.ConfigurationException
        at org.apache.catalina.loader.WebappClassLoaderBase.loadClass(WebappClassLoaderBase.java:1309)
        at org.apache.catalina.loader.WebappClassLoaderBase.loadClass(WebappClassLoaderBase.java:1137)
        ... 37 more
```

```

cp /usr/hdp/share/hst/hst-common/lib/commons-configuration-1.10.jar $KYLIN_HOME/tomcat/lib

````


# spark 批处理

- [Continuous trigger not found in Structured Streaming](https://stackoverflow.com/questions/50952042/continuous-trigger-not-found-in-structured-streaming)

kafka offset 要latest

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301030356.jpg)