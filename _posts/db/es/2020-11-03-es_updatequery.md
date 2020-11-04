---
layout:     post
rewards: false
title:  es stat
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