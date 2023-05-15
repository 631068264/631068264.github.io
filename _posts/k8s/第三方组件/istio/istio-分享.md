# 由来

**[传统微服务](https://www.cnblogs.com/wzh2010/p/14940280.html)**

- 微服务框架出现前：服务需要自己处理网络通信所面临的丢包、错误、乱序、重试等一系列流控问题，因此服务实现中，除了业务逻辑外，还包含对网络传输问题的处理逻辑。
- 微服务框架出现后：框架实现分布式系统通信需要的各种通用语义功能：如负载均衡和服务发现等，因此一定程度上屏蔽了这些通信细节，使得开发人员使用较少的框架代码就能开发出健壮的分布式系统。

- **缺点**：
  - **侵入性强。**想要集成SDK的能力，除了需要添加相关依赖，业务层中入侵的代码、注解、配置，与治理层界限不清晰
  - **升级成本高。**每次升级都需要业务应用修改SDK版本，重新进行功能回归测试，并对每一台服务进行部署上线，与快速迭代开发相悖。
  - **版本碎片化严重。**由于升级成本高，而中间件版本更新快，导致线上不同服务引用的SDK版本不统一、能力参差不齐，造成很难统一治理。
  - **治理功能不全。**不同于RPC框架，SpringCloud作为治理全家桶的典型，也不是万能的，诸如协议转换支持、多重授权机制、动态请求路由、故障注入、灰度发布等高级功能并没有覆盖到。
  - **无法实现真正意义上的语言无关性。**提供的框架一般只支持一种或几种语言，要将框架不支持的语言研发的服务也纳入微服务架构中，是比较有难度的。

**什么是服务网格**

- 应用程序间通讯的中间层
- 轻量级网络代理
- 应用程序无感知
- 解耦应用程序的重试/超时、监控、追踪和服务发现

[istio和其他热门服务网格比较](https://www.helight.cn/blog/2020/comparison-of-service-mesh/)

|                                                    | Istio                                             | Linkerd v2           | Consul                                                       |
| :------------------------------------------------- | :------------------------------------------------ | :------------------- | ------------------------------------------------------------ |
| **Supported Workloads**  是否支持 VM 和 Kubernetes | Kubernetes + VMs                                  | Kubernetes only      | Kubernetes + VMs                                             |
|                                                    |                                                   |                      |                                                              |
| **Architecture 架构**                              | Istio                                             | Linkerd v2           | Consul                                                       |
| Single point of failure 单点故障                   | No – 每个 pod 上都有 sidecar                      | No                   | No. 但增加了管理HA的复杂性， 因为必须按指定数量安装Consul服务， 而非用本地K8S master |
| Sidecar Proxy                                      | Yes (Envoy)                                       | Yes                  | Yes (Envoy)                                                  |
| Per-node agent                                     | No                                                | No                   | Yes                                                          |
|                                                    |                                                   |                      |                                                              |
| **Secure Communication 安全通信**                  | Istio                                             | Linkerd v2           | Consul                                                       |
| mTLS                                               | Yes                                               | Yes                  | Yes                                                          |
| Certificate Management                             | Yes                                               | Yes                  | Yes                                                          |
| Authentication and Authorization                   | Yes                                               | Yes                  | Yes                                                          |
|                                                    |                                                   |                      |                                                              |
| **Communication Protocols 通信协议**               | Istio                                             | Linkerd v2           | Consul                                                       |
| TCP                                                | Yes                                               | Yes                  | Yes                                                          |
| HTTP/1.x                                           | Yes                                               | Yes                  | Yes                                                          |
| HTTP/2                                             | Yes                                               | Yes                  | Yes                                                          |
| gRPC                                               | Yes                                               | Yes                  | Yes                                                          |
|                                                    |                                                   |                      |                                                              |
| **Traffic Management 流量管控**                    | Istio                                             | Linkerd v2           | Consul                                                       |
| Blue/Green Deployments 蓝绿发布                    | Yes                                               | Yes                  | Yes                                                          |
| Circuit Breaking 熔断                              | Yes                                               | No                   | Yes                                                          |
| Fault Injection 故障注入                           | Yes                                               | Yes                  | Yes                                                          |
| Rate Limiting 限频                                 | Yes                                               | No                   | Yes                                                          |
|                                                    |                                                   |                      |                                                              |
| **Chaos Monkey-style Testing: 混沌测试**           | Istio                                             | Linkerd v2           | Consul                                                       |
| Testing                                            | Yes- 可以配置服务延时响应或者按请求百分比返回失败 | Limited              | No                                                           |
|                                                    |                                                   |                      |                                                              |
| **Observability 可观测性**                         | Istio                                             | Linkerd v2           | Consul                                                       |
| Monitoring 监控                                    | Yes, with Prometheus                              | Yes, with Prometheus | Yes, with Prometheus                                         |
| Distributed Tracing 分布式追踪                     | Yes                                               | Some                 | Yes                                                          |
|                                                    |                                                   |                      |                                                              |
| **Multicluster Support 多集群支持**                | Istio                                             | Linkerd v2           | Consul                                                       |
| Multicluster                                       | Yes                                               | No                   | Yes                                                          |
|                                                    |                                                   |                      |                                                              |
| **Installation 安装支持**                          | Istio                                             | Linkerd v2           | Consul                                                       |
| Deployment                                         | 通过 Helm 和 istioctl 安装                        | Helm                 | Helm                                                         |
|                                                    |                                                   |                      |                                                              |
| **Operations Complexity 操作复杂度**               | Istio                                             | Linkerd v2           | Consul                                                       |
| Complexity                                         | High                                              | Low                  | Medium                                                       |

Istio 是 3 个技术方案中拥有最多的特性和灵活性的一个，但是要记住灵活性就意味着复杂性，所以你的团队要明白这一点，要为此做好准备。如果只是支持 Kubernetes，那么 Linkerd 或许是最好的选择。如果你想支持多种环境（包括了 Kubernetes 和 VM 环境）但是又不需要 Istio 的复杂性，那么 Consul 可能是最好的选择。







# 功能

- 流量控制功能：通过丰富的 HTTP、gRPC、WebSocket 和 TCP 流量路由规则来执行细粒度的流量控制。
- 网络弹性特性：重试设置、故障转移、熔断器和故障注入。
- 安全性和身份认证特性：执行安全性策略，并强制实行通过配置 API 定义的访问控制和速率限制。
- 基于 WebAssembly 的可插拔扩展模型，允许通过自定义策略执行和生成网格流量的遥测。

# 架构

管理模式：安装到集群，指定namespace， Istio 在部署应用的时候，自动注入 Sidecar，管理Service



Istio 服务网格从逻辑上分为**数据平面**和**控制平面**。

- **数据平面** 由一组智能代理（[Envoy](https://www.envoyproxy.io/)）组成，被部署为 Sidecar。这些代理负责协调和控制微服务之间的所有网络通信。它们还收集和报告所有网格流量的遥测数据。
- **控制平面** 管理并配置代理来进行流量路由。

![The overall architecture of an Istio-based application.](https://cdn.jsdelivr.net/gh/631068264/img/202304201741041.svg)

## Envoy

Istio 使用 [Envoy](https://www.envoyproxy.io/) 代理的扩展版本。Envoy 是用 C++ 开发的高性能代理，用于协调服务网格中所有服务的入站和出站流量。Envoy 代理是唯一与数据平面流量交互的 Istio 组件。

Envoy 代理被部署为服务的 Sidecar，在逻辑上为服务增加了 Envoy 的许多内置特性，例如：

- 动态服务发现
- 负载均衡
- TLS 终端
- HTTP/2 与 gRPC 代理
- 熔断器
- 健康检查
- 基于百分比流量分割的分阶段发布
- 故障注入
- 丰富的指标

Sidecar 代理模型还允许您向现有的部署添加 Istio 功能，而不需要重新设计架构或重写代码。

由 Envoy 代理启用的一些 Istio 的功能和任务包括：

- 流量控制功能：通过丰富的 HTTP、gRPC、WebSocket 和 TCP 流量路由规则来执行细粒度的流量控制。
- 网络弹性特性：重试设置、故障转移、熔断器和故障注入。
- 安全性和身份认证特性：执行安全性策略，并强制实行通过配置 API 定义的访问控制和速率限制。
- 基于 WebAssembly 的可插拔扩展模型，允许通过自定义策略执行和生成网格流量的遥测。

### Sidecar 注入

简单来说，Sidecar 注入会将额外容器的配置添加到 Pod 模板中。Istio 服务网格目前所需的容器有：

`istio-init` [init 容器](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)用于设置 iptables 规则，以便将入站/出站流量通过 sidecar 代理。初始化容器与应用程序容器在以下方面有所不同：

- 它在启动应用容器之前运行，并一直运行直至完成。
- 如果有多个初始化容器，则每个容器都应在启动下一个容器之前成功完成。

因此，您可以看到，对于不需要成为实际应用容器一部分的设置或初始化作业来说，这种容器是多么的完美。在这种情况下，`istio-init` 就是这样做并设置了 `iptables` 规则。

`istio-proxy` 这个容器是真正的 sidecar 代理（基于 Envoy）。

## istiod

在 Istio 1.5 中，更改了 Istio 的打包方式，将控制平面功能合并为[一个被称为 **istiod** 的二进制文件](https://istio.io/latest/zh/blog/2020/istiod/)

Istiod 提供服务发现、配置和证书管理。

Istiod 将控制流量行为的高级路由规则转换为 Envoy 特定的配置，并在运行时将其传播给 Sidecar。Pilot 提取了特定平台的服务发现机制，并将其综合为一种标准格式，任何符合 [Envoy API](https://www.envoyproxy.io/docs/envoy/latest/api/api) 的 Sidecar 都可以使用。

Istio 可以支持发现多种环境，如 Kubernetes 或 VM。

您可以使用 Istio [流量管理 API](https://istio.io/latest/zh/docs/concepts/traffic-management/#introducing-istio-traffic-management) 让 Istiod 重新构造 Envoy 的配置，以便对服务网格中的流量进行更精细的控制。

Istiod [安全](https://istio.io/latest/zh/docs/concepts/security/)通过内置的身份和凭证管理，实现了强大的服务对服务和终端用户认证。您可以使用 Istio 来升级服务网格中未加密的流量。使用 Istio，运营商可以基于服务身份而不是相对不稳定的第 3 层或第 4 层网络标识符来执行策略。此外，您可以使用 [Istio 的授权功能](https://istio.io/latest/zh/docs/concepts/security/#authorization)控制谁可以访问您的服务。

Istiod 充当证书授权（CA），并生成证书以允许在数据平面中进行安全的 mTLS 通信。

## 扩展WASM

[istio 扩展发展](https://cloudnative.to/blog/importance-of-wasm-in-istio/)

| Istio 1.4 之前                       | Istio 1.5                                                   | Istio 1.12 和未来         |
| :----------------------------------- | :---------------------------------------------------------- | :------------------------ |
| 用 C++ 扩展维护自己的 Envoy 代理构建 | 使用 EnvoyFilter 资源引入新的 Wasm 可扩展性模型（仍然复杂） | 引入专用的 WasmPlugin API |
| 使用 Mixer（效率低）                 | 仅支持本地或 HTTP 位置                                      | 包括对 OCI 注册表的支持   |

在 Istio 1.4（2019 年 11 月发布）之前，没有良好的机制来运行插件。当时，Istio 维护了他们自己的 Envoy 代理的分支，以运行自定义插件，如用 C++ 编写并与 Envoy 代理一起构建的 RBAC 和 JWT 过滤器。

Istio 1.5 版本包括一个使用 WebAssembly 的可扩展性新模型。随着 WebAssembly 的引入，不再需要运行单独的 Mixer 组件，这也导致了 Istio 部署的简化 —— 少了一件部署的东西，也少了一件需要担心的东西。

最近在 Istio 1.12 中引入了最重要的突破性功能。为 Wasm 插件引入了一个专门的 API，称为 WasmPlugin API，它使用一种新的方法从符合 OCI 的注册表中获取 Wasm 二进制文件。

新 API 的引入消除了使用 EnvoyFilter 来部署扩展的需要。扩展开发者现在可以使用一个名为 WasmPlugin 的资源来指定要部署插件的工作负载。

Wasm 的特性让它充满无限可能:

- **标准** —— Wasm 被设计成无版本、特性可测试、向后兼容的, 主流浏览器均已实现初版 Wasm 规范。
- **快速** —— 它可以通过大多数运行时的 JIT/AOT 能力提供类似原生的速度。 与启动 VM 或启动容器不同的是, 它没有冷启动。
- **安全** —— 默认情况下, Wasm 运行时是沙箱化的, 允许安全访问内存。基于能力的模型确保 Wasm 应用程序只能访问得到明确允许的内容。软件供应链更加安全。
- **可移植** —— Wasm 的二进制格式是被设计成可在不同操作系统(目前支持 Linux、Windows、macOS、Android、甚至是嵌入式设备)与指令集（目前支持 x86、ARM、RISC-V等）上高效执行的。
- **高性能** —— Wasm 只需极小的内存占用和超低的 CPU 门槛就能运行。
- ️**支持多语言** —— [多种编程语言 (opens new window)](https://github.com/appcypher/awesome-wasm-langs)可以编译成 Wasm。





# 常用CRD

## VirtualService

通过对客户端请求的目标地址与真实响应请求的目标工作负载进行解耦来实现。虚拟服务同时提供了丰富的方式，为发送至这些工作负载的流量指定不同的路由规则。

一个典型的用例是将流量发送到被指定为服务子集的服务的不同版本。客户端将虚拟服务视为一个单一实体，将请求发送至虚拟服务主机，然后 Envoy 根据虚拟服务规则把流量路由到不同的版本。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"  # 客户端向服务发送请求时使用的一个或多个地址。 https://istio.io/latest/zh/docs/concepts/traffic-management/#the-hosts-field
  gateways:
  - bookinfo-gateway
  http: # 路由规则
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination: # 指定了符合此条件的流量的实际目标地址。与虚拟服务的 hosts 不同，destination 的 host 必须是存在于 Istio 服务注册中心的实际目标地址
        host: productpage
        port:
          number: 9080
```

集群内访问reviews服务由sidecar 

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
  namespace: istio-example
spec:
  hosts:
  - reviews
  http:
  - match: # jason用户 流向v2子集
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route: # 其余v1子集
    - destination:
        host: reviews
        subset: v1
```

配合DestinationRule

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
  namespace: istio-example
spec:
  host: reviews
  subsets:
  - labels:
      version: v1
    name: v1
  - labels:
      version: v2
    name: v2
  - labels:
      version: v3
    name: v3
```









## DestinationRule

将虚拟服务视为将流量如何路由到给定目标地址，然后使用**目标规则来配置该目标的流量**。

特别是，您可以使用目标规则来指定命名的服务子集，例如按版本为所有给定服务的实例分组。然后可以在虚拟服务的路由规则中使用这些服务子集来控制到服务不同实例的流量。

默认情况下，Istio 使用轮询的负载均衡策略，实例池中的每个实例依次获取请求。Istio 同时支持如下的负载均衡模型，可以在 `DestinationRule` 中为流向某个特定服务或服务子集的流量指定这些模型。

- 随机：请求以随机的方式转发到池中的实例。
- 权重：请求根据指定的百分比转发到池中的实例。
- 最少请求：请求被转发到最少被访问的实例。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: my-svc
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1
    labels:
      version: v1 # 根据应用label区分不同版本
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v3
    labels:
      version: v3

```

定义在 `subsets` 上的默认策略，为 `v1` 和 `v3` 子集设置了一个简单的随机负载均衡器。在 `v2` 策略中，轮询负载均衡器被指定在相应的子集字段上。

### Gateway

使用[网关](https://istio.io/latest/zh/docs/reference/config/networking/gateway/#Gateway)来管理网格的入站和出站流量，可以让您指定要进入或离开网格的流量。网关配置被用于运行在网格边缘的独立 Envoy 代理，而不是与服务工作负载一起运行的 sidecar Envoy 代理。

网关主要用于管理进入的流量，但您也可以配置出口网关。出口网关让您为离开网格的流量配置一个专用的出口节点，这可以限制哪些服务可以或应该访问外部网络，或者启用[出口流量安全控制](https://istio.io/latest/zh/blog/2019/egress-traffic-control-in-istio-part-1/)为您的网格添加安全性。您也可以使用网关配置一个纯粹的内部代理。

展示了一个外部 HTTPS 入口流量的网关配置：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ext-host-gwy
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - ext-host.example.com
    tls:
      mode: SIMPLE
      serverCertificate: /tmp/tls.crt
      privateKey: /tmp/tls.key
```

http

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "httpbin.example.com"

```





# Jaeger 链路追踪

https://istio.io/latest/zh/docs/tasks/observability/distributed-tracing/jaeger/

Jaeger是一个开源的分布式追踪系统，支持多种语言。Jaeger的trace是由一系列的span组成，每个span代表了一次请求的处理过程。

获取traces

```sh
GET https://rancherIP:rancherPort/k8s/clusters/<cluster_id>/api/v1/namespaces/istio-system/services/http:tracing:16686/proxy/jaeger/api/traces?end=1680512849164000&limit=20&lookback=3h&maxDuration=1.2s&minDuration=0ms&service=productpage.istio-example&start=1680502049164000
```



```json
{
    "data":[
        {
            "traceID":"fe6a2c9dc7a8fd410fb5ed260d63bb84",
            "spans":[...],
            "processes":{...},
            "warnings":null
        }
    ],
    "total":0,
    "limit":0,
    "offset":0,
    "errors":null
}
```





get trace by id

```sh
GET https://rancherIP:rancherPort/k8s/clusters/<cluster_id>/api/v1/namespaces/istio-system/services/http:tracing:16686/proxy/jaeger/api/traces/<traceID>
```



```json
{
    "data":[
        {
            "traceID":"fe6a2c9dc7a8fd410fb5ed260d63bb84",
            "spans":[
                {
                    "traceID":"fe6a2c9dc7a8fd410fb5ed260d63bb84" , # 整个trace的唯一标识
                    "spanID":"0fb5ed260d63bb84", # 当前span的唯一标识
                    "operationName":"productpage.istio-example.svc.cluster.local:9080/productpage", # 当前span的操作名称
                    "references":[  # 描述span之间父子关系
                        {
                            "refType": "CHILD_OF",
                            "traceID": "fe6a2c9dc7a8fd410fb5ed260d63bb84",
                            "spanID": "cc128ec14790dc18"
                        }
                    ],
                    "startTime":1680511825018273, # 当前span开始时间 微秒 (µs)
                    "duration":58397, # 当前span持续时间 微秒 (µs)
                    "tags":[     # 于存储一些键值对信息，如请求的URL、请求的参数等
                         {
                            "key": "istio.canonical_service",
                            "type": "string",
                            "value": "productpage"
                        },
                        .....
  										],  
                    "logs":Array[0],
                    "processID":"p1",
                    "warnings":null
                },
               ....
            ],
            "processes":{
                "p1":{
                    "serviceName":"istio-ingressgateway.istio-system",
                    "tags":[
                        {
                            "key":"ip",
                            "type":"string",
                            "value":"xxx.xx.xx.133"
                        }
                    ]
                },
                ....
            },
            "warnings":null
        }
    ],
    "total":0,
    "limit":0,
    "offset":0,
    "errors":null
}
```



# 示例

Bookinfo 应用分为四个单独的微服务：

- `productpage`. 这个微服务会调用 `details` 和 `reviews` 两个微服务，用来生成页面。
- `details`. 这个微服务中包含了书籍的信息。
- `reviews`. 这个微服务中包含了书籍相关的评论。它还会调用 `ratings` 微服务。
- `ratings`. 这个微服务中包含了由书籍评价组成的评级信息。

`reviews` 微服务有 3 个版本：

- v1 版本不会调用 `ratings` 服务。
- v2 版本会调用 `ratings` 服务，并使用 1 到 5 个黑色星形图标来显示评分信息。
- v3 版本会调用 `ratings` 服务，并使用 1 到 5 个红色星形图标来显示评分信息。

![Bookinfo Application without Istio](https://cdn.jsdelivr.net/gh/631068264/img/202304181117064.svg)



**流量管理**

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productpage
spec:
  hosts:
  - productpage
  http:
  - route:
    - destination:
        host: productpage
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    fault:
      abort:
        percentage:
          value: 100.0
        httpStatus: 500
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1

---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: details
spec:
  hosts:
  - details
  http:
  - route:
    - destination:
        host: details
        subset: v1
---
```

![image-20230418151707216](https://cdn.jsdelivr.net/gh/631068264/img/202304181517372.png)



![image-20230418152059536](https://cdn.jsdelivr.net/gh/631068264/img/202304201741244.png)

**通讯加密**

默认明文、mTLS加密

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "default"
  namespace: "istio-example"
spec:
  mtls:
    mode: STRICT # 只能双向加密
```





```sh
kubectl label namespace istio-curl istio-injection=enabled
kubectl label namespace istio-curl istio-injection-

curl http://httpbin.istio-example:8000/headers -s | grep X-Forwarded-Client-Cert

```

**https gateway**

```sh
openssl req -newkey rsa:2048 -nodes -keyout ca.key -x509 -days 36500 -out ca.crt -subj "/C=xx/ST=x/L=x/O=x/OU=x/CN=ca/emailAddress=x/"

openssl genrsa -out tls.key 2048
openssl req -new -key tls.key -out tls.csr -subj "/CN=httpbin.example.com/O=httpbin organization"
openssl x509 -req -days 36500 -in tls.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out tls.crt


kubectl create -n istio-example secret tls httpbin-credential   --key=tls.key   --cert=tls.crt



kubectl get svc -owide -n istio-system

# 测试https
curl -kv -HHost:httpbin.example.com --resolve "httpbin.example.com:31390:10.19.64.202" "https://httpbin.example.com:31390/status/418"
```





**wasm**

```sh
curl -v -H "Authorization: Basic YWRtaW4zOmFkbWluMw==" http://10.19.64.202:31380/productpage
```

- [go-wasm-sdk](https://github.com/tetratelabs/proxy-wasm-go-sdk/tree/main)
- https://istio.io/latest/zh/docs/tasks/extensibility/wasm-module-distribution/
- https://github.com/istio-ecosystem/wasm-extensions



```sh
go mod tidy
tinygo build -o plugin.wasm -scheduler=none -target=wasi  main.go

docker build -t xx/istio/wasm_demo -f wasm-image.Dockerfile .

docker push xx/istio/wasm_demo
```



```dockerfile
# Dockerfile for building "compat" variant of Wasm Image Specification.
# https://github.com/solo-io/wasm/blob/master/spec/spec-compat.md
FROM scratch

COPY plugin.wasm ./plugin.wasm

```





**dex+ istio**

[Kubeflow: Authentication with Istio + Dex](https://journal.arrikto.com/kubeflow-authentication-with-istio-dex-5eafdfac4782)

kubeflow使用EnvoyFilter实现传入的 HTTP 请求是否被授权，外部授权服务通过调用外部 gRPC 或者 HTTP 服务来检查传入的 HTTP 请求是否被授权。 如果该请求被视为未授权，则通常会以 403 （禁止）响应拒绝该请求。 注意，从授权服务向上游、下游或者授权服务发送其他自定义元数据也是被允许的。在 [HTTP 过滤器](https://cloudnative.to/envoy/api-v3/extensions/filters/http/ext_authz/v3/ext_authz.proto.html#envoy-v3-api-msg-extensions-filters-http-ext-authz-v3-extauthz) 中有更多详细的解释。

[ext_authz](https://cloudnative.to/envoy/configuration/http/http_filters/ext_authz_filter.html)

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: authn-filter
spec:
  workloadSelector:
    labels:
      istio: ingressgateway
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: GATEWAY
        listener:
          filterChain:
            filter:
              name: "envoy.http_connection_manager"
      patch:
        # For some reason, INSERT_FIRST doesn't work
        operation: INSERT_BEFORE
        value:
          # See: https://www.envoyproxy.io/docs/envoy/v1.17.0/configuration/http/http_filters/ext_authz_filter#config-http-filters-ext-authz
          name: "envoy.filters.http.ext_authz"
          typed_config:
            '@type': type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz
            http_service:
              server_uri:  # 授权检查地址
                uri: http://$(AUTHSERVICE_SERVICE).$(AUTHSERVICE_NAMESPACE).svc.cluster.local
                cluster: outbound|8080||$(AUTHSERVICE_SERVICE).$(AUTHSERVICE_NAMESPACE).svc.cluster.local
                timeout: 10s
              authorization_request:
                allowed_headers:
                  patterns:
                    # XXX: MUST be lowercase!
                    - exact: "authorization"
                    - exact: "cookie"
                    - exact: "x-auth-token"
              authorization_response:
                allowed_upstream_headers:
                  patterns:
                    - exact: "kubeflow-userid"

```

用户登录通过dex给的授权JWT，会重定向到kubeflow服务，同时带上JWT在cookie，经过istio时候会通过AuthService拦截

![img](https://cdn.jsdelivr.net/gh/631068264/img/202304210935754.png)

