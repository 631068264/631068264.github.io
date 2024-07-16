# k8s部署的prometheus alert manager不会发送告警消息

### 问题描述

通过CRD AlertmanagerConfig创建的告警路由匹配不到告警，导致告警到了alert manager这里没有发到接收者。

### 原因剖析

用户创建CR`AlertmanagerConfig`之后，prometheus operator会把`AlertmanagerConfig`转化为alert manager的配置文件，但根据[CRD AlertmanagerConfig规范](https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/crds/crd-alertmanagerconfigs.yaml#L41)，Alertmanager配置仅适用于具有“命名空间”标签的警报且等于AlertmanagerConfig资源的命名空间，在[prometheus operator的api文档](https://prometheus-operator.dev/docs/operator/api/#monitoring.coreos.com/v1alpha1.Route)也有说明。所以prometheus operator会在将AlertmanagerConfig转化为alert manager的配置文件的时候，删除关于`namespace`的`matches`，再添加label:`namespace: <object namespace>`，详见[prometheus operator源码](https://github.com/prometheus-operator/prometheus-operator/blob/dab2194da371aee19ce251bcad34f8debb91ea9f/pkg/alertmanager/amcfg.go#L143-L146)。

### 解决方案

目前prometheus operator社区也在讨论这个[issue](https://github.com/prometheus-operator/prometheus-operator/issues/3954)，但还没有得到作者的回应，我们会持续跟进这个问题。

现在采取临时的解决方案是，通过配置`PrometheusRule`，对所有生成的`alert`都加上相同的`namesapce` label，和`AlertmanagerConfig`所在的`namespace`一致，这样alert manager就会根据配置路由到告警。
