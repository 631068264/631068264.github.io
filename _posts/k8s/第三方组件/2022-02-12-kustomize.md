---
layout:     post
rewards: false
title:   Kustomize
categories:
    - k8s

---

> Kustomize 允许用户以一个应用描述文件 （YAML 文件）为基础（Base YAML），然后通过 Overlay 的方式生成最终部署应用所需的描述文件，而不是像 Helm 那样只提供应用描述文件模板，然后通过字符替换（Templating）的方式来进行定制化。

根据环境的不同，又配置了多套的应用描述文件。随着服务越部越多，应用描述文件更是呈爆炸式的增长。

使用了 `helm` ，但是其只提供应用描述文件模板，在不同环境拉起一整套服务会节省很多时间，而像我们这种在指定环境快速迭代的服务，并不会减少很多时间。





不同环境相同的配置都放在 `base` 中，而差异就可以在 `overlays` 中实现。

```
deploy
    ├── base
    │   ├── deployment.yaml
    │   ├── kustomization.yaml
    │   └── service.yaml
    └── overlays
        ├── dev
        │   ├── healthcheck_patch.yaml
        │   ├── kustomization.yaml
        │   └── memorylimit_patch.yaml
        └── prod
            ├── healthcheck_patch.yaml
            ├── kustomization.yaml
            └── memorylimit_patch.yaml
```

`base` 中维护了项目共同的基础配置，如果有镜像版本等基础配置需要修改，可以使用 `kustomize edit set image ...` 来直接修改基础配置，而真正不同环境，或者不同使用情况的配置则在 `overlays` 中 以 patch 的形式添加配置。

```sh
kustomize build 配置目录 | kubectl apply -f -

kustomize build deploy/base | kubectl apply -f -

# kustomize build deploy/overlays/dev | kubectl apply -f -
kubectl apply -k deploy/overlays/dev

```

配置目录里面有**kustomization.yaml**里面包含合成的yaml一起安装

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
# Cert-Manager
- ../common/cert-manager/cert-manager/base
- ../common/cert-manager/kubeflow-issuer/base
# Istio
- ../common/istio-1-9/istio-crds/base
- ../common/istio-1-9/istio-namespace/base
- ../common/istio-1-9/istio-install/base
# OIDC Authservice
- ../common/oidc-authservice/base
```

# kustomization.yaml 的作用

这里我们看一下官方示例 `helloWorld` 中的 `kustomization.yaml`：

```yaml
commonLabels:
  app: hello

resources:
- deployment.yaml
- service.yaml
- configMap.yaml
```

可以看到该项目中包含3个 resources ， `deployment.yaml`、`service.yaml` 、 `configMap.yaml`。

```
.
└── helloWorld
    ├── configMap.yaml
    ├── deployment.yaml
    ├── kustomization.yaml
    └── service.yaml
```

直接执行命令：

```sh
kustomize build helloWorld
```

kustomize 通过 `kustomization.yaml` 将3个 resources 进行了处理，给三个 resources 添加了共同的 labels `app: hello` 。这个示例展示了 `kustomization.yaml` 的作用：**将不同的 resources 进行整合，同时为他们加上相同的配置**。

# 根据环境生成不同配置

使用最多的就是为不同的环境配置不同的 `deploy.yaml`，而使用 kustomize 可以把配置拆分为多个小的 patch ，然后通过 kustomize 来进行组合。而根据环境的不同，每个 patch 都可能不同，包括分配的资源、访问的方式、部署的节点都可以自由的定制。

```
.
├── flask-env
│   ├── README.md
│   ├── base
│   │   ├── deployment.yaml
│   │   ├── kustomization.yaml
│   │   └── service.yaml
│   └── overlays
│       ├── dev
│       │   ├── healthcheck_patch.yaml
│       │   ├── kustomization.yaml
│       │   └── memorylimit_patch.yaml
│       └── prod
│           ├── healthcheck_patch.yaml
│           ├── kustomization.yaml
│           └── memorylimit_patch.yaml
```

这里可以看到配置分为了 `base` 和 `overlays`， `overlays` 则是继承了 `base` 的配置，同时添加了诸如 healthcheck 和 memorylimit 等不同的配置，那么我们分别看一下 `base` 和 `overlays` 中 `kustomization.yaml` 的内容

- base

```
commonLabels:
  app: test-cicd

resources:
- service.yaml
- deployment.yaml
```

`base` 中的 `kustomization.yaml` 中定义了一些基础配置

- overlays

```yaml
bases:
- ../../base
patchesStrategicMerge:
- healthcheck_patch.yaml
- memorylimit_patch.yaml
namespace: devops-dev
```

`overlays` 中的 `kustomization.yaml` 则是基于 `base` 新增了一些个性化的配置，来达到生成不同环境的目的。

执行命令

```
kustomize build flask-env/overlays/dev
```

可以看到包括 `replicas`、`limits`、`requests`、`env` 等 dev 中个性的配置都已经出现在了生成的 yaml 中。由于篇幅有限，这里没有把所有的配置有罗列出来，需要的可以去 [GitHub](https://github.com/sunny0826/kustomize-lab) 上自取。