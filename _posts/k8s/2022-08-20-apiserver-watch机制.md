---
layout:     post
rewards: false
title:      K8s apiserver watch 机制浅析
categories:
    - k8s
---

K8s的apiserver是k8s所有组件的流量入口，其他的所有的组件包括**kube-controller-manager，kubelet，kube-scheduler等通过list-watch 机制向apiserver 发起list watch 请求**，根据收到的事件处理后续的请求。watch机制本质上是使客户端和服务端建立长连接，并将服务端的变化实时发送给客户端的方式减小服务端的压力。



k8s的apiserver实现了两种长连接方式：Chunked transfer encoding(分块传输编码)和 Websocket，其中基于chunked的方式是apiserver的默认配置。k8s的watch机制的实现依赖etcd v3的watch机制，etcd v3使用的是基于 HTTP/2 的 gRPC 协议，双向流的 Watch API 设计，实现了连接多路复用。etcd 里存储的key的任何变化都会发送给客户端。



# kube­apiserver对etcd的list­watch机制

为了减轻etcd的压力，kube-apiserver本身对etcd实现了list-watch机制，将所有对象的最新状态和最近的事件存放到cacher里，所有外部组件对资源的访问都经过cacher。我们看下**cacher**的数据结构

staging/src/k8s.io/apiserver/pkg/storage/cacher/cacher.go

```go
type Cacher struct {
// incoming 事件管道, 会被分发给所有的watchers
   incoming chan watchCacheEvent

//storage 的底层实现
   storage storage.Interface
// 对象类型
   objectType reflect.Type

// watchCache 滑动窗口，维护了当前kind的所有的资源，和一个基于滑动窗口的最近的事件数组
   watchCache *watchCache

// reflector list并watch etcd 并将事件和资源存到watchCache中
   reflector  *cache.Reflector

// watchersBuffer 代表着所有client-go客户端跟apiserver的连接
   watchersBuffer []*cacheWatcher
   ....
}
```

创建cacher的时候，也创建了

- watchCache（用于保存事件和所有资源）
- reflactor（执行对etcd的list-watch并更新watchCache）。

**同时开启了两个协程**

- cacher.dispatchEvents()
- cacher.startCaching(stopCh)

## CacheWatcher

**cacher.dispatchEvents()** 用于从cacher的incoming管道里获取事件，并放到cacheWatcher的input里



watchersBuffer 是一个数组，维护着所有client-go跟apiserver的watch连接产生的CacheWatcher。

因此**CacheWatcher跟发起watch请求的client-go的客户端是一对一的关系**。当apiserver收到一个etcd的事件之后，会将这个事件发送到所有的cacheWatcher的input channel里。

cacherWatcher的struct结构如下

```go
type cacheWatcher struct {
   input     chan *watchCacheEvent
   result    chan watch.Event
   done      chan struct{}
filter    filterWithAttrsFunc
   stopped   bool
   forget    func()
   versioner storage.Versioner
// The watcher will be closed by server after the deadline,
// save it here to send bookmark events before that.
   deadline            time.Time
   allowWatchBookmarks bool
// Object type of the cache watcher interests
   objectType reflect.Type


// human readable identifier that helps assigning cacheWatcher
// instance with request
   identifier string
}
```

cacherWatcher不用于存储数据，只是实现了watch接口，并且维护了两个channel

- input channel用于获取从cacher中的incoming通道中的事件
- result channel 用于跟**client-go的客户端交互**，客户端的informer发起watch请求后，会从这个chanel里获取事件进行后续的处理

## cacher.startCaching(stopCh) 

实际上调用了cacher的reflector的listAndWatch方法，这里的reflector跟informer的reflector一样，

list方法是获取etcd里的所有资源并对reflector的store做一次整体的replace替换，这里的store就是上面说的watchCache，watchCache实现了store接口，watch方法是watch etcd的资源，并从watcher的resultChan里拿到事件，根据事件的类型，调用watchCache的add，update，或delete方法。startCaching 执行对etcd的listAndWatch

reflector的list方法里的syncWith方法将list得到的结果替换放到watchCache里