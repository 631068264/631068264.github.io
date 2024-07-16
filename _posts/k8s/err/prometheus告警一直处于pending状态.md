# prometheus告警一直处于pending状态

### 问题描述

当加了label:`<value>: {{ value }}`的告警，如果触发等待时间设置得稍长一点，该告警一直处于`pending`状态，永远不会变成`firing`状态。

### 原因剖析

该问题是由两个原因的综合因素导致的：

1. 评估持续时间比触发等待时间短

	prometheus的[rule_group配置](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/#rule_group)有个`interval`字段，指的是每次rule评估的持续时间（刚开始我看字段名还误以为是评估间隔时间），如果`interval`设置的时间比`for`(触发等待时间)短，而每次评估时间只会持续`interval`所配置的时间，所以导致每次告警的持续时间都比触发等待时间短，从而告警不会从`pending`状态变为`firing`状态。

2. label的值若改变则视为新的告警

	之前为了解决显示当前告警值的需求，就把告警值作为label添加进告警触发配置。但是prometheus的告警触发逻辑是：把告警ID和label都作为判断是否相同告警的条件，当下一次评估时，若告警ID和label都是相同的，则视为同一个告警，告警的持续时间会在上次评估的基础上续上；而改变了label的值后，则视为不同告警(如下图)，这样做的结果就是每次告警的时间都只持续了`interval`所配置的时间，从而告警不会从`pending`状态变为`firing`状态。社区上也有人提过同样的[issue](https://github.com/prometheus/prometheus/issues/4836)。

	![截屏2022-07-29 16.59.58](./images/截屏2022-07-29 16.59.58.png)

### 解决方案

把当前告警值放到`annotations`中，`annotations`的改变不会视为新的告警。
