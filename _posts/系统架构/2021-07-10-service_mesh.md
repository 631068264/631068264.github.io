---
layout:     post
rewards: false
title: Service Mesh
categories:
    - 系统架构
---

https://levelup.gitconnected.com/deciphering-the-difference-between-a-service-mesh-and-api-gateway-c57e4abec302

# what Service Mesh

服务网格是一个专用的基础设施层，它的目标是 “[在微服务架构中实现可靠、快速和安全的服务间调用](https://link.zhihu.com/?target=https%3A//buoyant.io/2017/04/25/whats-a-service-mesh-and-why-do-i-need-one/)”。

服务网格是一种工具，通过在平台层而不是应用程序层插入这些功能，为应用程序添加**可观察性、安全性和可靠性功能**。

服务网格通常被实现为与应用程序代码一起部署的一组可扩展的网络代理（这种模式有时称为**sidecar**）。这些**代理**处理微服务之间的通信，并充当可以引入服务网格功能的点。**代理包括服务网格的数据平面，并由其控制平面作为一个整体进行控制。**

**这些代理作为sidecar(边车)注入到每个服务部署中。服务不直接通过网络调用服务，而是调用它们的本地sidecar代理，后者代表服务管理请求，从而封装了服务间调用的复杂性。相互连接的sidecar代理实现了所谓的数据平面。**

服务网格的兴起与“云原生”应用程序的兴起息息相关。在云原生世界中，一个应用程序可能包含数百个服务；每个服务可能有数千个实例；并且这些实例中的每一个都可能处于不断变化的状态，因为它们是由 Kubernetes 等编排器动态调度的。在这个世界中，服务到服务的通信不仅极其复杂，而且还是应用程序运行时行为的基本部分。管理它对于确保端到端性能、可靠性和安全性至关重要。

# service mesh 结构

服务网格通常由两层实现：数据平面和控制平面。

- 数据平面

  充当连接的客户端和服务器端点的代理，执行从控制平面接收到的策略并将运行时指标报告给控制平面的监控工具。

- 控制平面

  管理数据平面的服务策略和编排。

![](https://cdn.jsdelivr.net/gh/631068264/img/008i3skNgy1gsc3gx6b4zj30dw05rgli.jpg)

![img](https://cdn.jsdelivr.net/gh/631068264/img/008i3skNgy1gsc6dbzq7zj312o0m6t9u.jpg)

## 数据平面

由整个网格内的 [sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 代理组成，这些代理以 [sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 的形式和应用服务一起部署。每一个 [sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 会接管进入和离开服务的流量，并配合控制平面完成流量控制等方面的功能。可以把数据平面看做是网格内 [sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 代理的网络拓扑集合。

## 控制平面

控制和管理数据平面中的 [sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 代理，完成配置的分发、服务发现、和授权鉴权等功能。



## 核心组件

下面我们简单的介绍一下 [Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio) 架构中几个核心组件的主要功能。

### Envoy

[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio) 的数据平面默认使用 [Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy) 作为 [sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 代理，在未来也将支持使用 [MOSN](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#mosn) 作为数据平面。[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy) 将自己定位于高性能的 [sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 代理，也可以认为它是第一代 [Service Mesh](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#service-mesh) 产品。可以说，流量控制相关的绝大部分功能都是由 [Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy) 提供的，这主要包括三个部分：

- 路由、流量转移。
- 弹性能力：如超时重试、熔断等。
- 调试功能：如故障注入、流量镜像。

### Pilot

[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot) 主要功能就是管理和配置部署在特定 [Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio) 服务网格中的所有 [sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 代理实例。它管理 [sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 代理之间的路由流量规则，并配置故障恢复功能，如超时、重试和熔断。

### Citadel

Citadel 是 [Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio) 中专门负责安全的组件，内置有身份和证书管理功能，可以实现较为强大的授权和认证等操作。

### Galley

Galley 是 [Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio) 1.1 版本中新增加的组件，其目的是将 [Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot) 和底层平台（如 Kubernetes）进行解耦。它分担了原本 [Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot) 的一部分功能，主要负责配置的验证、提取和处理等功能。

## sidecar

[sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 模式允许您在应用程序旁边添加更多功能，而无需额外第三方组件配置或修改应用程序代码。在软件架构中， [Sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 连接到父应用并且为其添加扩展或者增强功能。[Sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 应用与主应用程序松散耦合。它可以屏蔽不同编程语言的差异，统一实现微服务的可观察性、监控、日志记录、配置、断路器等功能。



在 Kubernetes 的 [Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod) 中，在原有的应用容器旁边注入一个 [Sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 容器，两个容器共享存储、网络等资源，可以广义的将这个包含了 [sidecar](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#sidecar) 容器的 [Pod](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pod) 理解为一台主机，两个容器共享主机资源。





# service mesh 功能

基础功能

- Traffic Routing

  根据策略或配置将请求路由到服务实例，流量可能会被选择性地路由到不同版本的服务。

  - 金丝雀发布（Canary release）。

    根据权重把 5% 的流量路由给新版本，如果服务正常，再逐渐转移更多的流量到新版本。

    ![image-20210920095305671](https://cdn.jsdelivr.net/gh/631068264/img/008i3skNgy1gumuvaqfbdj61ay0fsabg02.jpg)

  - AB testing。

  - 服务版本控制/向后兼容性。

- 服务监控追踪

  proxy记录客户端和服务端调用的log，开发者不需要把打log的工作做到每个客户端和服务端。可以根据日志分析系统性能和可用性，可以追踪调用链

  - 服务图和仪表板显示服务如何相互连接（无需更改代码）。
  - 发出信号和警报，以显示延迟、吞吐量和错误率（无需更改代码）。
  - 跟踪请求或业务交易是如何通过网格的（只需在代码标头中更改传递交易 ID）。

- 弹性

  服务调用短暂无法使用，代理可以尝试使用该服务的备用路径或故障转移到备份服务。代理可以尝试使用该服务的备用路径或故障转移到备份服务。

  我们可以配置和实施的弹性模式示例：

  - 重试策略。
  - 断路器模式。
  - 速率限制、节流。

- 安全策略

  **将单体应用拆分为许多独立的服务会大大增加其攻击面。**每个服务都是需要保护的入口。使用服务网格，客户端和服务器端点上的代理都可以应用策略来保护两者之间的通信。代理负责身份验证、授权和加密，这就是服务网格内的零信任安全性。

- 身份识别

  **服务网格可以管理和维护哪些身份能访问哪些服务，并维护访客访问服务的日志。**身份可以通过 JWT 进行验证，从而允许基于终端用户以及服务调用进行授权。

- 加密

  如上所述，服务之间的通信是加密的。**控制平面提供证书管理功能**，例如证书生成和证书轮换，它会将这些证书和相关的配置数据推送到数据平面。

  相互 TLS 身份验证的支持非常强大。相互 TLS 是指两个端点把证书加入连接的另一端点白名单，它能提供身份验证和加密。



## 流量控制

流量控制功能主要分为三个方面：

- 请求路由和流量转移
- 弹性功能，包括熔断、超时、重试
- 调试能力，包括故障注入和流量镜像



### 路由和流量转移

为了控制服务请求，引入了服务版本（version）的概念，可以通过版本这一标签将服务进行区分。版本的设置是非常灵活的，可以根据服务的迭代编号进行定义（如 v1、v2 版本）；也可以根据部署环境进行定义（比如 dev、staging、production）。**通过版本标签，Istio就可以定义灵活的路由规则来控制流量**



服务间流量控制

控制与网格边界交互的流量，在系统的入口和出口处部署 Sidecar代理，让所有流入和流出的流量都由代理进行转发。负责入和出的代理就叫做入口网关和出口网关，它们把守着进入和流出网格的流量。

![入口和出口网关（图片来自 Istio 官方网站）](https://cdn.jsdelivr.net/gh/631068264/img/008i3skNgy1gumvrqsa6gj61la0iwdi202.jpg)

还能设置流量策略。比如可以对连接池相关的属性进行设置，通过修改最大连接等参数，实现对请求负载的控制。还可以对负载均衡策略进行设置，在轮询、随机、最少访问等方式之间进行切换。还能设置异常探测策略，将满足异常条件的实例从负载均衡池中摘除，以保证服务的稳定性。

### 弹性功能

支持超时、重试和熔断

- 超时就是设置一个等待时间，当上游服务的响应时间超过这个时间上限，就不再等待直接返回，就是所谓的快速失败。超时主要的目的是控制故障的范围，避免故障进行扩散。
- 重试一般是用来解决网络抖动时通信失败的问题。因为网络的原因，或者上游服务临时出现问题时，可以通过重试来提高系统的可用性。

只需要在路由配置里添加 `timeout` 和 `retry` 这两个关键字就可以实现。

- 熔断，它是一种非常有用的过载保护手段，可以避免服务的级联失败。

  熔断一共有三个状态，

  当上游服务可以返回正常时，熔断开关处于关闭状态；

  一旦失败的请求数超过了失败计数器设定的上限，就切换到打开状态，让服务快速失败；

  熔断还有一个半开状态，通过一个超时时钟，在一定时间后切换到半开状态，让请求尝试去访问上游服务，看看服务是否已经恢复正常。如果服务恢复就关闭熔断，否则再次切换为打开状态。
  
  需要在自定义资源 `DestinationRule` 的 `TrafficPolicy` 里进行设置。

### 调试能力

对流量进行调试的能力，包括故障注入和流量镜像。对流量进行调试可以让系统具有更好的容错能力，也方便我们在问题排查时通过调试来快速定位原因所在。

#### 故障注入

故障注入就是在系统中人为的设置一些故障，来测试系统的稳定性和系统恢复的能力。比如给某个服务注入一个延迟，使其长时间无响应，然后检测调用方是否能处理这种超时而自身不受影响（比如及时的终止对故障发生方的调用，避免自己被拖慢、或让故障扩展）。

Isito 支持注入两种类型的故障：延迟和中断。

延迟是模拟网络延迟或服务过载的情况；

中断是模拟上游服务崩溃的情况，以 HTTP 的错误码和 TCP 连接失败来表现。Istio 里实现故障注入很方便，在路由配置中添加`fault`关键字即可。

#### 流量镜像

通过复制一份请求并把它发送到镜像服务，从而实现流量的复制功能。

最主要的就是进行线上问题排查。一般情况下，因为系统环境，特别是数据环境、用户使用习惯等问题，我们很难在开发环境中模拟出真实的生产环境中出现的棘手问题，同时生产环境也不能记录太过详细的日志，因此很难定位到问题。有了流量镜像，**我们就可以把真实的请求发送到镜像服务，再打开 debug 日志来查看详细的信息**。除此以外，还可以通过它来观察生产环境的请求处理能力，比如在**镜像服务进行压力测试。也可以将复制的请求信息用于数据分析**。只需要在路由配置中通添加`mirror`关键字即可。

## 安全

安全功能主要分为认证和授权两部分

[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio) 中的安全架构是由多个组件协同完成的。Citadel 是负责安全的主要组件，用于密钥和证书的管理；[Pilot](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#pilot) 会将安全策略配置分发给 [Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy) 代理；[Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy) 执行安全策略来实现访问控制。下图展示了 [Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio) 的安全架构和运作流程。

![安全架构（图片来自 Istio 官方网站）](https://cdn.jsdelivr.net/gh/631068264/img/008i3skNgy1guo2wkh97kj61pc0u079602.jpg)



### 认证

- 对等认证（Peer authentication）：用于服务到服务的认证。这种方式是通过双向 TLS（mTLS）来实现的，即客户端和服务端都要验证彼此的合法性。[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio) 中提供了内置的密钥和证书管理机制，可以自动进行密钥和证书的生成、分发和轮换，而无需修改业务代码。
- 请求认证（Request authentication）：也叫最终用户认证，验证终端用户或客户端。[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio) 使用目前业界流行的 JWT（JSON Web Token）作为实现方案。

Istio的 mTLS 提供了一种宽容模式（permissive mode）的配置方法，使得服务可以同时支持纯文本和 mTLS 流量。用户可以先用非加密的流量确保服务间的连通性，然后再逐渐迁移到 mTLS，这种方式极大的降低了迁移和调试的成本。



### 授权

[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio) 的授权策略可以为网格中的服务提供不同级别的访问控制，比如网格级别、命名空间级别和工作负载级别。授权策略支持 `ALLOW` 和 `DENY` 动作，每个 [Envoy](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#envoy) 代理都运行一个授权引擎，当请求到达代理时，授权引擎根据当前策略评估请求的上下文，并返回授权结果 `ALLOW` 或 `DENY`。授权功能没有显示的开关进行配置，默认就是启动状态，只需要将配置好的授权策略应用到对应的工作负载就可以进行访问控制了。

[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio) 中的授权策略通过自定义资源`AuthorizationPolicy`来配置。除了定义策略指定的目标（网格、命名空间、工作负载）和动作（容许、拒绝）外，[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio) 还提供了丰富的策略匹配规则，比如可以设置来源、目标、路径、请求头、方法等条件，甚至还支持自定义匹配条件，其灵活性可以极大的满足用户需求。

## 可观察性

对服务的运行时状态进行监控、上报、分析，以提高服务可靠性。具有可观察性的系统，可以在服务出现故障时大大降低问题定位的难度，甚至可以在出现问题之前及时发现问题以降低风险。具体来说，可观察性可以：

- 及时反馈异常或者风险使得开发人员可以及时关注、修复和解决问题（告警）；
- 出现问题时，能够帮助快速定位问题根源并解决问题，以减少服务损失（减损）；
- 收集并分析数据，以帮助开发人员不断调整和改善服务（持续优化）。

提供了三种不同类型的数据从不同的角度支撑起其可观察性

- 指标（Metrics）：指标本质上是时间序列上的一系列具有特定名称的计数器的组合，不同计数器用于表征系统中的不同状态并将之数值化。通过数据聚合之后，指标可以用于查看一段时间范围内系统状态的变化情况甚至预测未来一段时间系统的行为。举一个简单的例子，系统可以使用一个计数器来对所有请求进行计数，并且周期性（周期越短，实时性越好，开销越大）的将该数值输出到时间序列数据库（比如 Prometheus）中，由此得到的一组数值通过数学处理之后，可以直观的展示系统中单位时间内的请求数及其变化趋势，可以用于实时监控系统中流量大小并预测未来流量趋势。而具体到 [Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio) 中，它基于四类不同的监控标识（响应延迟、流量大小、错误数量、饱和度）生成了一系列观测不同服务的监控指标，用于记录和展示网格中服务状态。除此以外，它还提供了一组默认的基于上述指标的网格监控仪表板，对指标数据进行聚合和可视化。借助指标，开发人员可以快速的了解当前网格中流量大小、是否频繁的出现异常响应、性能是否符合预期等等关键状态。但是，如前所述，指标本质上是计数器的组合和系统状态的数值化表示，所以往往缺失细节内容，它是从一个相对宏观的角度来展现整个网格或者系统状态随时间发生的变化及趋势。在一些情况下，指标也可以辅助定位问题。

- 日志（Access Logs）：日志是软件系统中记录软件执行状态及内部事件最为常用也最为有效的工具。而在可观测性的语境之下，日志是具有相对固定结构的一段文本或者二进制数据（区别于运行时日志），并且和系统中需要关注的事件一一对应。当系统中发生一个新的事件，指标只会有几个相关的计数器自增，而日志则会记录下该事件具体的上下文。因此，日志包含了系统状态更多的细节部分。在分布式系统中，日志是定位复杂问题的关键手段；同时，由于每个事件都会产生一条对应的日志，所以日志也往往被用于计费系统，作为数据源。其相对固定的结构，也提供了日志解析和快速搜索的可能，对接 ELK 等日志分析系统后，可以快速的筛选出具有特定特征的日志以分析系统中某些特定的或者需要关注的事件。在 [Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio) 网格中，当请求流入到网格中任何一个服务时，[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio) 都会生成该请求的完整记录，包括请求源和请求目标以及请求本身的元数据等等。日志使网格开发人员可以在单个服务实例级别观察和审计流经该实例的所有流量。

- 分布式追踪（Distributed Traces）：尽管日志记录了各个事件的细节，可在分布式系统中，日志仍旧存在不足之处。日志记录的事件是孤立的，但是在实际的分布式系统中，不同组件中发生的事件往往存在因果关系。举例来说，组件 A 接收外部请求之后，会调用组件 B，而组件 B 会继续调用组件 C。在组件 A B C 中，分别有一个事件发生并各产生一条日志。但是三条日志没有将三个事件的因果关系记录下来。而分布式追踪正是为了解决该问题而存在。分布式追踪通过额外数据（Span ID等特殊标记）记录不同组件中事件之间的关联，并由外部数据分析系统重新构造出事件的完整事件链路以及因果关系。在服务网格的一次请求之中，[Istio](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#istio) 会为途径的所有服务生成分布式追踪数据并上报，通过 Zipkin 等追踪系统重构服务调用链，开发人员可以借此了解网格内服务的依赖和调用流程，构建整个网格的服务拓扑。在未发生故障时，可以借此分析网格性能瓶颈或热点服务；而在发生故障时，则可以通过分布式追踪快速定位故障点。


# why use/not service mesh

如果这些问题中的任何一个的答案是**“是”**，请评估服务网格的价值

- 网络拓扑是否会随着多个服务的扩展和缩减而频繁变化
- 代码变更频繁每周或者更频繁
- 10个或以上微服务，大量数据中心内的数据流量，会随时扩展5个或以上实例
- 安全：自动维护双向TLS
- 监控追踪服务间交互

什么情况下不使用

- 微服务小< 10，扩展实例小<5
- 不需要服务之间的细粒度跟踪。或者可观察性是您唯一的需要，也许可以使用 AppDynamics 等更简单的工具更轻松地解决。
- 几乎没有内部网络通信
- 固定网络拓扑
- 代码改动很少

几乎所有的服务网格都使用 Envoy side car 作为数据平面。**它们最明显的区别在于控制平面**。因此，对不同服务网格的评估应侧重于控制平面的特性以及哪些控制平面满足您的需求。

评估控制平面

- 控制平面是否在您的 Cl/CI 管道中提供最大价值
- 控制计划是否是配置形式的，最好不要手写script
- 操作容易，可以控制权限

# service mesh 和api 网关区别

微服务和 API 服务解决了两个不同的问题，前者是技术性问题，后者是业务问题。

- 微服务应在有界上下文（bounded context）中进行通信。它们的设计是由连接组成有界上下文的组件需求所驱动的，就像远程过程调用（RPC）一样。
- API（通常是 REST，但也包括事件流和其他协议，例如 SOAP、gRPC 和 GraphQL）应提供接口，将有界上下文暴露给外界。理想情况下，它们的接口设计是由业务价值驱动的，而不仅仅是 RPC。

换句话说，**API 是在外部将一个有界上下文的业务暴露给另一个有界上下文，而微服务是构成有界上下文内部的几个组件**。在传统体系结构中，这些组件可能是通过进程调用栈进行通信的类或 DLL。在微服务架构中，它们可能是跨网络通信的独立服务。



要了解服务网格和 API 网关之间的区别，首先我们要定义“定向流量”（directional traffic）。**东西流量通常是指数据中心内的数据，而南北方向是指进出数据中心的流量。**

在本文中，从有界上下文的角度来看：停留在有界上下文内的流量是东西流量，而越过有界上下文的流量是南北流量。

**服务网格旨在管理东西流量。**虽然 API 网关也可以管理东西流量，但服务网格更适合。这是因为服务网格在连接的两端都有代理。东西方都可以进行这种配置，因为这两个端点都由同一应用开发组织控制。

**服务网格同样可以管理南北流量，但 API 网关更适合**，因为它连接的一部分不在服务网格的控制范围之内。



此外，南北流量通常涉及业务合作伙伴，并且需要管理终端用户的体验。**API 网关更专注于管理终端用户体验。**它们通常是较大的 API 管理解决方案一部分，具有集成的 API 目录和开发人员门户，能将内部开发人员和外部业务合作伙伴加入其中。

相反，服务网格并不专注于管理服务客户端终端用户体验。由于**服务网格旨在管理组成应用程序、有界上下文的服务**，因此其所有的客户端通常由拥有服务的同一 IT 部门构建，团队可以轻松更改微服务的界面。

总而言之，**API 网关和服务网格两者是相辅相成的：API 网关处理外部流量和服务网格处理内部流量**，其拓扑结构如下图所示：

![图片](https://cdn.jsdelivr.net/gh/631068264/img/008i3skNgy1gsc6a3ctkbj30dw0g50ud.jpg)

