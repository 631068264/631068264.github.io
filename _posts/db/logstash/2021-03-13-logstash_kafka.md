---
layout:     post
rewards: false
title:  logstash接收sslkafka数据存储到es
categories:
    - logstash
tags:
    - big data
---



# docker 配置

```yml
  logstash:
    image: hub.yizhisec.com/mirror/logstash
    network_mode: host
    privileged: true
    container_name: logstash
    depends_on:
      - elasticsearch
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    volumes:
      - /etc/localtime:/etc/localtime
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
      - ./logstash/config/pipelines.yml:/usr/share/logstash/config/pipelines.yml:ro
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
      - /yizhisec/communication/kafka/config/ssl/:/kafka/config/ssl/:ro
```

目录结构

```
logstash
├── Dockerfile
├── config
│   ├── logstash.yml
│   └── pipelines.yml
└── pipeline
    ├── xxx.conf
├── docker-compose.yml

```



# 配置文件

## logstash.yml



```yml
---
## Default Logstash configuration from Logstash base image.
## https://github.com/elastic/logstash/blob/master/docker/data/logstash/config/logstash-full.yml
#
http.host: "127.0.0.1"
xpack.monitoring.elasticsearch.hosts: [ "http://127.0.0.1:9200" ]

## X-Pack security credentials
#
#xpack.monitoring.enabled: true
#xpack.monitoring.elasticsearch.username: elastic
#xpack.monitoring.elasticsearch.password: "123456"

```



## pipelines.yml

```yml
# This file is where you define your pipelines. You can define multiple.
# For more information on multiple pipelines, see the documentation:
#   https://www.elastic.co/guide/en/logstash/current/multiple-pipelines.html

- pipeline.id: device
  path.config: "/usr/share/logstash/pipeline/device.conf"
  pipeline.workers: 4
- pipeline.id: user
  path.config: "/usr/share/logstash/pipeline/user.conf"
  pipeline.workers: 4

```



## pine.conf

```
input {
  kafka {
    type => "kafka"
    bootstrap_servers => "kafka_domain:9092"
    topics => "cert"
    group_id => "logstash"
    security_protocol=>"SSL"
    ssl_truststore_location=>"/kafka/config/ssl/client.truststore.jks"
    ssl_truststore_password=>"xxxxxx"
    ssl_endpoint_identification_algorithm=>""
    consumer_threads => 4
    codec => "json"
  }
}

output {
  elasticsearch {
    hosts => "127.0.0.1:9200"
    index => "tracer_cert"
  }
}

```



