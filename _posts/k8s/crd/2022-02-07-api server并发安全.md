---
layout:     post
rewards: false
title:   Kubernetes ApiServer 并发安全机制
categories:
    - k8s
---

当向 Kubernetes ApiServer 并发的发起多个请求，对同一个 API 资源对象进行更新时，Kubernetes ApiServer 是如何确保更新不丢失，以及如何做冲突检测的

[详细过程代码参考](https://yangxikun.com/kubernetes/2020/01/30/kubernetes-apiserver-concurrent-safe.html)

经常用于修改 API 资源对象的命令是 apply 和 edit。接下来以修改 Deployment 资源对象为例子，通过观察这2个命令发出的请求，发现它们都是使用 HTTP PATCH：

```shell
PATCH /apis/extensions/v1beta1/namespaces/default/deployments/nginx HTTP/1.1
Host: 127.0.0.1:8080
User-Agent: kubectl/v0.0.0 (linux/amd64) kubernetes/$Format
Content-Length: 246
Accept: application/json
Content-Type: application/strategic-merge-patch+json
Uber-Trace-Id: 5ca3fde0b9f9aaf1:0c0358897c8e0ef8:0f38135280523f87:1
Accept-Encoding: gzip

{"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"nginx"}],"containers":[{"$setElementOrder/env":[{"name":"DEMO_GREETING"}],"env":[{"name":"DEMO_GREETING","value":"Hello from the environment#kubectl edit"}],"name":"nginx"}]}}}}
```

**确保更新不丢失**

在将更新后的对象持久化到 Etcd 中时，通过事务保证的，当事务执行失败时会不断重试，事务的伪代码逻辑如下：

```
// oldObj = FromMemCache(key) or EtcdGet(key)
if inEtcd(key).rev == inMemory(oldObj).rev:
    EtcdSet(key) = newObj
    transaction = success
else:
    EtcdGet(key)
    transaction = fail
```

**并发更新是如何做冲突检测的**

有两处冲突检测的判断：

1. Preconditions
2. tryUpdate 中的 resourceVersion != version

对于 kubectl apply 和 edit （发送的都是 PATCH 请求），创建的 Preconditions 是零值，所以不会通过 Preconditions 进行冲突检测，而在 tryUpdate 中调用 objInfo.UpdatedObject(ctx, existing) 得到的 newObj.rv 始终是等于 existing.rv 的，所以也不会进行冲突检测。

那什么时候会进行冲突检测？其实 kubectl 还有个 replace 的命令，通过抓包发现 replace 命令发送的是 PUT 请求，并且请求中会带有 resourceVersion：

```
PUT /apis/extensions/v1beta1/namespaces/default/deployments/nginx HTTP/1.1
Host: localhost:8080
User-Agent: kubectl/v0.0.0 (linux/amd64) kubernetes/$Format
Content-Length: 866
Accept: application/json
Content-Type: application/json
Uber-Trace-Id: 6e685772cc06fc16:2514dc54a474fe88:4f488c05a7cef9c8:1
Accept-Encoding: gzip

{"apiVersion":"extensions/v1beta1","kind":"Deployment","metadata":{"labels":{"app":"nginx"},"name":"nginx","namespace":"default","resourceVersion":"744603"},"spec":{"progressDeadlineSeconds":600,"replicas":1,"revisionHistoryLimit":10,"selector":{"matchLabels":{"app":"nginx"}},"strategy":{"rollingUpdate":{"maxSurge":"25%","maxUnavailable":"25%"},"type":"RollingUpdate"},"template":{"metadata":{"creationTimestamp":null,"labels":{"app":"nginx"}},"spec":{"containers":[{"env":[{"name":"DEMO_GREETING","value":"Hello from the environment#kubectl replace"}],"image":"nginx","imagePullPolicy":"IfNotPresent","name":"nginx","resources":{},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File"}],"dnsPolicy":"ClusterFirst","restartPolicy":"Always","schedulerName":"default-scheduler","securityContext":{},"terminationGracePeriodSeconds":30}}}}
```

PUT 请求在 ApiServer 中的处理与 PATCH 请求类似，都是调用 k8s.io/apiserver/pkg/registry/generic/registry.(*Store).Update，创建的 rest.UpdatedObjectInfo 为 rest.DefaultUpdatedObjectInfo(obj, transformers…)，注意这里传了 obj 参数值（通过 decode 请求 body 得到），而不是 nil。

在 PUT 请求处理中，创建的 Preconditions 也是零值，不会通过 Preconditions 做检查。但是在 tryUpdate 中，`resourceVersion, err := e.Storage.Versioner().ObjectResourceVersion(obj)` 得到的 resourceVersion 是请求 body 中的值，而不像 PATCH 请求一样，来自 existing（看下 defaultUpdatedObjectInfo.UpdatedObject 方法就明白了）。

所以在 PUT 请求处理中，会通过 tryUpdate 中的 resourceVersion != version，检测是否发生了并发写冲突。

