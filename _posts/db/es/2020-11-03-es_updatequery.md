---
layout:     post
rewards: false
title:  es 坑
categories:
    - es
tags:
    - big data
---

# Update By Query

## VersionConflictEngineException

由于**没有事务**更新存在version冲突，当多个update在**refresh_interval**内修改同一个doc就会发生error。

[Update By Query doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-update-by-query.html)

解决方式：

- **conflicts**

  发生version conflicts **abort**  or  **proceed**  默认**abort**，设置成**proceed**碰到error会继续

- **refresh**

  update后马上**refresh**一下

## Too many dynamic script compilations within, max: [75/5m]

这是关于[script的描述](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-scripting-using.html#prefer-params) 5min内最多**编译**75次，不是执行，触发circuit_breaking_exception，超过了[script-compilation-circuit-breaker](https://www.elastic.co/guide/en/elasticsearch/reference/current/circuit-breaker.html#script-compilation-circuit-breaker)配置

```yaml
script.max_compilations_rate: xxx/5m
```

编译script对于es来说非常重，编译好的**新**script会放到cache。

- 不要使用硬编码，每次修改参数的改变都会**重新编译**

  ```
  "source": "doc['my_field'] * 2"
  ```



- 使用**params**，只**编译一次**

  ```
    "source": "doc['my_field'] * multiplier",
    "params": {
      "multiplier": 2
    }
  ```



解决方法

- 调整`script.max_compilations_rate` 参数
- 使用params参数来传递参数
- stored script 预先存储script到es



# Painless script function return type



```
ctx._source.xx_time = Math.max(ctx._source.xx_time, params.xx_time)
```

传入的参数都是long类型，返回却是double

> Even though Java provides several overloaded `Math.max()` methods for `long`, `float` and `double`, Painless only provides the one for `double`, probably because all other types (i.e. `long` and `float`) can be upcast to `double`.

 [Painless Script Math.max change my data type](https://stackoverflow.com/questions/64690956/painless-script-math-max-change-my-data-type)



# ES 强转类型

[coerce](https://www.elastic.co/guide/en/elasticsearch/reference/current/coerce.html#coerce)

> “5”，5.0 强转成 5   "false" -> false

因为es这个特性mapping **设置了数据类型约等于没用**，脏数据进去了**照样可以保存，不报错**，**放进去后取出来还是原来脏数据**。go这种强类型的语言可以在测试时检查到异常，使用Python的话会是灾难。

要阻止这个傻逼特性

```console
"index.mapping.coerce": false
```

[Boolean 类型也要留意](https://www.elastic.co/guide/en/elasticsearch/reference/current/boolean.html) 



# must 和 should 同时使用 should 失效

要想must 和 should同时有用，不能同级。should用bool包起来放在must下级



**错误写法**

```json
{
    "query":{
        "bool":{
            "must":[
                {"term":{"a":"1"}}
            ],
            "should":[
                {
                    "match":{
                        "content":"xxxx"
                    }
                }
            ]
        }
    }
}
```

**正确写法**

```json
{
    "query":{
        "bool":{
            "must":[
                {"term":{"a":"1"}}
                {
                    "bool":{
                        "should":[
                            {
                                "match":{
                                    "content":"xxxx"
                                }
                            }
                        ]
                    }
                }
            ]
        }
    }
}
```

#  go uint8 数组 存放es short数组 报错

因为[go 里面byte是uint8的别名](https://golang.org/pkg/builtin/#byte)

`json.Marshal`把`[]uint8`当成`[]byte`然后会编码成**base64 string**，导致es报错

> failed to parse field [xxx] of type [short] in document with id 'xxx'. Preview of field's value: 'AQ=='

**用其他数据类型替代**

https://stackoverflow.com/questions/14177862/how-to-marshal-a-byte-uint8-array-as-json-array-in-go/14178407



# es mapping 不能直接update

```shell
ES_URL="127.0.0.1:9200/"
JSON_HEADER="Content-Type: application/json"

function es_config() {
    curl -X PUT $ES_URL$1 -H "$JSON_HEADER" -d "$2"
}
function es_del_index() {
    curl -X DELETE $ES_URL$1
}

function es_reindx() {
  	curl -X PUT $ES_URL_reindex -H "$JSON_HEADER" -d "$1"
}
es_config "xxx" '
{
  "mappings": {
    "properties": {
      "ip": { "type": "keyword"},
      "delete_time": { "type": "long" }
    }
  }
}'

# add field
es_config "xxx/_mapping" '
{
    "properties":{
        "detection_engine":{
            "type":"short"
        },
        "threat_tag":{
            "type":"keyword"
        }
    }
}
'
```

修改index 属性

```shell
# 改reindex

es_config "new_index" '
{
  "mappings": {
    "properties": {
      "priority": { "type": "keyword"}
    }
  }
}'

es_reindx '
{
  "source": {
    "index": "old_index"
  },
  "dest": {
    "index": "new_index"
  }
}
'

es_del_index "old_index"


es_reindx '
{
  "source": {
    "index": "old_index"
  },
  "dest": {
    "index": "new_index"
  }
}
'

# 重建index
es_config "old_index" '
{
  "mappings": {
    "properties": {
      "priority": { "type": "keyword"}
    }
  }
}'

es_reindx '
{
  "source": {
    "index": "new_index"
  },
  "dest": {
    "index": "old_index"
  }
}
'

es_del_index "new_index"

```

## 别名  alias

能非常优雅的解决两个索引无缝切换的问题 [es alias doc](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/indices-add-alias.html)

```
PUT /my-index-000001/_alias/alias1
```

- 在一个运行中的es集群中无缝的切换一个索引到另一个索引上

- 分组多个索引，比如按月创建的索引，我们可以通过别名构造出一个最近3个月的索引

- 查询一个索引里面的部分数据构成一个类似数据库的视图（views）可以用filter

  ```
  PUT /users/_alias/user_12
  {
    "routing" : "12",
    "filter" : {
      "term" : {
        "user_id" : 12
      }
    }
  }
  ```



### 索引别名切换

```
POST /_aliases
{
    "actions": [
        { "remove": { "index": "my_index_v1", "alias": "my_index" }},
        { "add":    { "index": "my_index_v2", "alias": "my_index" }}
    ]
}
```

顺序执行的，先解除my_index_v1的别名，然后给my_index_v2添加新别名，my_index_v1和my_index_v2现在想通过索引别名来实现无缝切换