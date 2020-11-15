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

