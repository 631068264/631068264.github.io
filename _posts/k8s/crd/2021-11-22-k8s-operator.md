---
layout:     post
rewards: false
title:   k8s operator 开发
categories:
    - k8s
tags:
    - crd
---

# CR定制资源

*资源（Resource）* 是 [Kubernetes API](https://kubernetes.io/zh/docs/concepts/overview/kubernetes-api/) 中的一个端点， 其中存储的是某个类别的 [API 对象](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/kubernetes-objects/) 的一个集合。 例如内置的 *pods* 资源包含一组 Pod 对象。

*定制资源（Custom Resource）* 是对 Kubernetes API 的扩展，不一定在默认的 Kubernetes 安装中就可用。定制资源所代表的是对特定 Kubernetes 安装的一种定制。 不过，很多 Kubernetes 核心功能现在都用定制资源来实现，这使得 Kubernetes 更加模块化。

定制资源可以通过动态注册的方式在运行中的集群内或出现或消失，集群管理员可以独立于集群 更新定制资源。一旦某定制资源被安装，用户可以使用 [kubectl](https://kubernetes.io/zh/docs/reference/kubectl/overview/) 来创建和访问其中的对象，就像他们为 *pods* 这种内置资源所做的一样。

## 定制控制器

就定制资源本身而言，它只能用来存取结构化的数据。 当你将定制资源与 *定制控制器（Custom Controller）* 相结合时，定制资源就能够 提供真正的 *声明式 API（Declarative API）*。

使用[声明式 API](https://kubernetes.io/zh/docs/concepts/overview/kubernetes-api/)， 你可以 *声明* 或者设定你的资源的期望状态，并尝试让 Kubernetes 对象的当前状态 同步到其期望状态。控制器负责将结构化的数据解释为用户所期望状态的记录，并 持续地维护该状态。

你可以在一个运行中的集群上部署和更新定制控制器，这类操作与集群的生命周期无关。 定制控制器可以用于任何类别的资源，不过它们与定制资源结合起来时最为有效。 [Operator 模式](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/operator/)就是将定制资源 与定制控制器相结合的。你可以使用定制控制器来将特定于某应用的领域知识组织 起来，以编码的形式构造对 Kubernetes API 的扩展。

## 我应该使用一个 ConfigMap 还是一个定制资源？[ ](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/api-extension/custom-resources/#我应该使用一个-configmap-还是一个定制资源)

如果满足以下条件之一，应该使用 ConfigMap：

- 存在一个已有的、文档完备的配置文件格式约定，例如 `mysql.cnf` 或 `pom.xml`。
- 你希望将整个配置文件放到某 configMap 中的一个主键下面。
- 配置文件的主要用途是针对运行在集群中 Pod 内的程序，供后者依据文件数据配置自身行为。
- 文件的使用者期望以 Pod 内文件或者 Pod 内环境变量的形式来使用文件数据， 而不是通过 Kubernetes API。
- 你希望当文件被更新时通过类似 Deployment 之类的资源完成滚动更新操作。

**说明：** 请使用 [Secret](https://kubernetes.io/zh/docs/concepts/configuration/secret/) 来保存敏感数据。 Secret 类似于 configMap，但更为安全。

如果以下条件中大多数都被满足，你应该使用定制资源（CRD 或者 聚合 API）：

- 你希望使用 Kubernetes 客户端库和 CLI 来创建和更改新的资源。
- 你希望 `kubectl` 能够直接支持你的资源；例如，`kubectl get my-object object-name`。
- 你希望构造新的自动化机制，监测新对象上的更新事件，并对其他对象执行 CRUD 操作，或者监测后者更新前者。
- 你希望编写自动化组件来处理对对象的更新。
- 你希望使用 Kubernetes API 对诸如 `.spec`、`.status` 和 `.metadata` 等字段的约定。
- 你希望对象是对一组受控资源的抽象，或者对其他资源的归纳提炼

## 添加定制资源 

Kubernetes 提供了两种方式供你向集群中添加定制资源：

- CRD 相对简单，创建 CRD 可以不必编程。
- [API 聚合](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/) 需要编程，但支持对 API 行为进行更多的控制，例如数据如何存储以及在不同 API 版本间如何转换等。

Kubernetes 提供这两种选项以满足不同用户的需求，这样就既不会牺牲易用性也不会牺牲灵活性。

聚合 API 指的是一些下位的 API 服务器，运行在主 API 服务器后面；主 API 服务器以代理的方式工作。这种组织形式称作 [API 聚合（API Aggregation，AA）](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/) 。 对用户而言，看起来仅仅是 Kubernetes API 被扩展了。

CRD 允许用户创建新的资源类别同时又不必添加新的 API 服务器。 使用 CRD 时，你并不需要理解 API 聚合。



### 比较易用性 

CRD 比聚合 API 更容易创建

| CRDs                                                         | 聚合 API                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 无需编程。用户可选择任何语言来实现 CRD 控制器。              | 需要使用 Go 来编程，并构建可执行文件和镜像。                 |
| 无需额外运行服务；CRD 由 API 服务器处理。                    | 需要额外创建服务，且该服务可能失效。                         |
| 一旦 CRD 被创建，不需要持续提供支持。Kubernetes 主控节点升级过程中自动会带入缺陷修复。 | 可能需要周期性地从上游提取缺陷修复并更新聚合 API 服务器。    |
| 无需处理 API 的多个版本；例如，当你控制资源的客户端时，你可以更新它使之与 API 同步。 | 你需要处理 API 的多个版本；例如，在开发打算与很多人共享的扩展时。 |

# Operator

Kubernetes 为自动化而生。无需任何修改，你即可以从 Kubernetes 核心中获得许多内置的自动化功能。 你可以使用 Kubernetes 自动化部署和运行工作负载， *甚至* 可以自动化 Kubernetes 自身。

Kubernetes 的 [Operator 模式](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/operator/)概念 使你无需修改 Kubernetes 自身的代码，通过把定制控制器关联到一个以上的定制资源上，即可以扩展集群的行为。 Operator 是 Kubernetes API 的客户端，充当 [定制资源](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/api-extension/custom-resources/) 的控制器。



## Operator 示例

使用 Operator 可以自动化的事情包括：

- 按需部署应用
- 获取/还原应用状态的备份
- 处理应用代码的升级以及相关改动。例如，数据库 schema 或额外的配置设置
- 发布一个 service，要求不支持 Kubernetes API 的应用也能发现它
- 模拟整个或部分集群中的故障以测试其稳定性
- 在没有内部成员选举程序的情况下，为分布式应用选择首领角色

想要更详细的了解 Operator？下面是一个示例：

1. 有一个名为 SampleDB 的自定义资源，你可以将其配置到集群中。
2. 一个包含 Operator 控制器部分的 Deployment，用来确保 Pod 处于运行状态。
3. Operator 代码的容器镜像。
4. 控制器代码，负责查询控制平面以找出已配置的 SampleDB 资源。
5. Operator 的核心是告诉 API 服务器，如何使现实与代码里配置的资源匹配。
   - 如果添加新的 SampleDB，Operator 将设置 PersistentVolumeClaims 以提供 持久化的数据库存储，设置 StatefulSet 以运行 SampleDB，并设置 Job 来处理初始配置。
   - 如果你删除它，Operator 将建立快照，然后确保 StatefulSet 和 Volume 已被删除。
6. Operator 也可以管理常规数据库的备份。对于每个 SampleDB 资源，Operator 会确定何时创建（可以连接到数据库并进行备份的）Pod。这些 Pod 将依赖于 ConfigMap 和/或具有数据库连接详细信息和凭据的 Secret。
7. 由于 Operator 旨在为其管理的资源提供强大的自动化功能，因此它还需要一些 额外的支持性代码。在这个示例中，代码将检查数据库是否正运行在旧版本上， 如果是，则创建 Job 对象为你升级数据库。

```bash
kubectl get SampleDB                   # 查找所配置的数据库

kubectl edit SampleDB/example-database # 手动修改某些配置

```

# 使用 CustomResourceDefinition 扩展 Kubernetes API

resourcedefinition.yaml

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # 名字必需与下面的 spec 字段匹配，并且格式为 '<名称的复数形式>.<组名>'
  name: crontabs.stable.example.com
spec:
  # 组名称，用于 REST API: /apis/<组>/<版本>
  group: stable.example.com
  # 列举此 CustomResourceDefinition 所支持的版本
  versions:
    - name: v1
      # 每个版本都可以通过 served 标志来独立启用或禁止
      served: true
      # 其中一个且只有一个版本必需被标记为存储版本
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  # 可以是 Namespaced 或 Cluster
  scope: Namespaced
  names:
    # 名称的复数形式，用于 URL：/apis/<组>/<版本>/<名称的复数形式>
    # 新的受名字空间约束的 RESTful API 端点会被创建在：
    # /apis/stable.example.com/v1/namespaces/*/crontabs/...
    plural: crontabs
    # 名称的单数形式，作为命令行使用时和显示时的别名
    singular: crontab
    # kind 通常是单数形式的帕斯卡编码（PascalCased）形式。你的资源清单会使用这一形式。
    kind: CronTab
    # shortNames 允许你在命令行使用较短的字符串来匹配资源
    shortNames:
    - ct
```

```bash
# 创建它
kubectl apply -f resourcedefinition.yaml
```

my-crontab.yaml

```yaml
# 新的受名字空间约束的 RESTful API 端点会被创建在：
# /apis/stable.example.com/v1/namespaces/*/crontabs/...

apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
```

```bash
# 创建
kubectl apply -f my-crontab.yaml
# kubectl 来管理你的 CronTab 对象
kubectl get crontab

NAME                 AGE
my-new-cron-object   6s

```

## openAPIV3Schema

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        # openAPIV3Schema 是用来检查定制对象的模式定义
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                  pattern: '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
                  default: "5 0 * * *"
                image:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
                  default: 1
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
```

# 自定义k8s api 扩展 apiserver

[Configure the Aggregation Layer](https://kubernetes.io/zh/docs/tasks/extend-kubernetes/configure-aggregation-layer/)

Kubernetes apiserver 会与扩展 apiserver通信，使用 x509 证书向扩展 apiserver 认证

大致流程如下：

1. Kubernetes apiserver：对发出请求的用户身份认证，并对请求的 API 路径执行鉴权。
2. Kubernetes apiserver：将请求转发到扩展 apiserver
3. 扩展 apiserver：认证来自 Kubernetes apiserver 的请求
4. 扩展 apiserver：对来自原始用户的请求鉴权
5. 扩展 apiserver：执行

![](https://tva1.sinaimg.cn/large/008i3skNly1gwsjko4jcuj30sg16kdhs.jpg)

