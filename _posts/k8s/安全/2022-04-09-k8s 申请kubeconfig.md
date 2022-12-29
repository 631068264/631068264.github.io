---
layout:     post
rewards: false
title:   基于RBAC使用kubeconfig文件限制不同用户操作k8s集群的权限
categories:
 - k8s

---

需要限制用户在某个namespace里面的权限

-  在相应名称空间下创建一个SA （**创建SA后，会自动创建一个绑定的secret，在后面的kubeconfig文件中，会用到该secret中的token**）

- 创建一个ClusterRole或者Role（**限制相应名称空间级别资源权限**）创建
- 一个Rolebinding绑定到该SA
- 这样以该sa生成的kubeconfig文件就会具有ClusterRole或者Role的权限



role.yaml  可以任意修改Role权限

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: SA_NAME-sa
  namespace: NAMESPACE
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: SA_NAME-role
  namespace: NAMESPACE
rules:
  - apiGroups: [""] # "" 标明 core API 组
    resources:
      - configmaps
      - secrets
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - delete
      - patch
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: SA_NAME-rolebinding
  namespace: NAMESPACE
subjects:
  - kind: ServiceAccount
    name: SA_NAME-sa
roleRef:
  kind: Role
  name: SA_NAME-role
  apiGroup: rbac.authorization.k8s.io

```



创建脚本替换生成 sa2kubeconfig.sh sa_name namspace_name  生成xx_kubeconfig

```sh
#!/bin/bash

SA_NAME=$1
NAMESPACE=$2

rm -rf role4ns.yaml
cp role.yaml role_bak.yaml

sed -i "s/SA_NAME/${SA_NAME}/g" role_bak.yaml
sed -i "s/NAMESPACE/${NAMESPACE}/g" role_bak.yaml

mv role_bak.yaml role4ns.yaml

# 生成权限yaml
kubectl apply -f role4ns.yaml

# 获取
APISERVER=`kubectl config view --flatten --minify -o=jsonpath="{.clusters[0].cluster.server}"`
CERTIFICATE_AUTHORITY_DATA=`kubectl config view --flatten --minify -o=jsonpath="{.clusters[0].cluster.certificate-authority-data}"`

SA_TOKEN=`kubectl get secret -n develop -oname |grep ${SA_NAME}-sa |xargs kubectl describe -n ${NAMESPACE} |grep token: |awk '{print $2}'`

echo $APISERVER
echo $CERTIFICATE_AUTHORITY_DATA
echo $SA_TOKEN


echo -e 'apiVersion: v1
kind: Config
users:  #  API Server 的信息
  - name: '${SA_NAME}'
    user:
      token: '${SA_TOKEN}'
clusters:  #远程授权服务
  - cluster:
      certificate-authority-data: '${CERTIFICATE_AUTHORITY_DATA}'  #验证远程授权服务的 CA证书
      server: '${APISERVER}'  #  远程授权服务 URL, 必须使用 HTTPS
    name: '${SA_NAME}'-cluster
contexts:
  - context:
      cluster: '${SA_NAME}'-cluster
      namespace: '${NAMESPACE}'
      user: '${SA_NAME}'
    name: '${SA_NAME}'-cluster
current-context: '${SA_NAME}'-cluster' > ${SA_NAME}_kubeconfig
```

在正常情况下，为了确保 Kubernetes 集群的安全， API Server都会对客户端进行身份 认证，认证失败的客户端无法进行 API 调用。此外，在 Pod 中访问 Kubernetes API Server 服务时，是以 Service 方式访问名为 Kubernetes 的这个服务的，而 Kubernetes 服务又只在 HTTPS 安全端 口 443 上提供，那么如何进行身份认证呢?这的确是个谜，因为 Kubernetes 的官方文档并没有清楚说明这个问题。

通过查看官方源码，我们发现这是在用一种类似 HTTP Token 的新认证方式一 Service Account Auth, Pod 中的客户端调用 Kubernetes API 时，在 HTTP Header 中传递了 一个 Token 字符串，这类似于之前提到的 HTTP Token 认证方式，但有以下几个不同之处 。

- 这个 Token 的内容来自 Pod 里指定路径下的一个文件 (/run/secrets/kubernetes. io/serviceaccount/token)，该 Token 是动态生成的，确切地说，是由 Kubernetes Controller 进 程用 API Server 的私钥 ( --service-account-private-key-file 指定的 私钥) 签名生成的一个 JWTSecret。
- 在官方提供的客户端 REST 框架代码里，通过 HTTPS 方式与 API Server 建立连接 后，会用 Pod 里指定路径下的一个 CA 证书 (/run/secrets/kubernetes.io/serviceaccount/ ca.crt) 验证 API Server发来的证书，验证是否为 CA 证书签名的合法证书。
- API Server 在收到 这个 Token 以后， 会采 用自己的私 钥(实际 上 是使 用 service-account- key-file 参数指定的私钥，如果没有设置此参数，则默认采用 tis-private-key-file 指定的参数，即自己的私钥)对 Token 进行合法性验证 。

![image-20220623222556194](https://cdn.jsdelivr.net/gh/631068264/img/e6c9d24egy1h3ijnpzexjj21460fejsq.jpg)



接下来看看 default-token-77oyg都有什么内容

![image-20220623222631592](https://cdn.jsdelivr.net/gh/631068264/img/e6c9d24egy1h3ijoaslonj21400u0gqs.jpg)



从上面的输出信息中可以看到， default-token-77oyg包含三个数据项，分别是 token、 ca.crt、 namespace。 联想到 Mountable secrets 的标记，以及之前看到的 Pod 中的三个文件 的文件名 ，

我们恍然大悟 : **在每个命名空间中都有一个名为 default 的默认 Service A ccount 对象 ， 在这个 ServiceAccount 里面有一个名为 Tokens 的可以作为 Volume 被挂载到 Pod 里 的 Secret, Pod启动时，这个 Secret会自动被挂载到 Pod 的指定目录下，用来协助完成 Pod 中的进程访问 API Server 时的 身份鉴权。**

![image-20220623223800142](https://cdn.jsdelivr.net/gh/631068264/img/e6c9d24egy1h3ik08i4thj21pe0p8gpa.jpg)

Kubernetes 之所以要创建两套独立的账号系统

- User 账号是给人用的， Service Account 是给 Pod 里的 进程用的，面向的对象不同 

- User 账号是全局性的， Service Account 则属千某个具体的命名空间 。

- 通常来说， User 账号是与后端的用户数据库同步的，创建一个新用户时通常要走

  一套复杂的业务流程才能实现， Service Account 的创建则需要极轻量级的实现方 式，集群管理员可以很容易地为某些特定任务创建一个 ServiceAccount。

**接下来深入分析 ServiceAccount 与 Secret 相关的一些运行机制 。**

Service Account 的正常工作离不开以下控制器: Service Account Controller、 Token Controller 、 Admission Controller。

**Service Account Controller** 的工作相对简单，它会监听 Service Account 和 Namespace 这两种资源对象的事件，如果在一个 Namespace 中没有默认的 Service Account, 那么**它会 为该 Namespace 创建一个默认的 Service Account 对象，这就是在 每个 Namespace 下都有 一个名为 default 的 Service Account 的原因 。**

**Token Controller 也监听 Service Account 的事件，如果发现在 新建的 Service Account 里没有对应的 Service Account Secret**, 则会用 API Server私钥 (--service-account-private­ key-file 指定的文件)创建一个 Token, 并用该 Token、 API Server 的 CA 证书等三个信息 产生一个新的 Secret 对象，然后放入刚才的 Service Account 中 。 如果监听到的事件是 ServiceAccount删除事件，则自动删除与该 ServiceAccount相关的所有 Secret。此外，Token Controller 对象也会同时监听 Secret 的创建和删除事件，确保与对 应的 Service Account 的 关联关系正确。
