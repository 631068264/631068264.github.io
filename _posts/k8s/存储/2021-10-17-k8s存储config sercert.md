---
layout:     post
rewards: false
title:      k8s存储 configMap secert
categories:
    - k8s
---

![image-20211213223333055](https://cdn.jsdelivr.net/gh/631068264/img/008i3skNgy1gxckwfwxqyj31vd0u079y.jpg)

- 第一，比如说一些可变的配置。因为我们不可能把一些可变的配置写到镜像里面，当这个配置需要变化的时候，可能需要我们重新编译一次镜像，这个肯定是不能接受的；
- 第二就是一些敏感信息的存储和使用。比如说应用需要使用一些密码，或者用一些 token；
- 第三就是我们容器要访问集群自身。比如我要访问 kube-apiserver，那么本身就有一个身份认证的问题；
- 第四就是容器在节点上运行之后，它的资源需求；
- 第五个就是容器在节点上，它们是共享内核的，那么它的一个安全管控怎么办？
- 最后一点我们说一下容器启动之前的一个前置条件检验。比如说，一个容器启动之前，我可能要确认一下 DNS 服务是不是好用？又或者确认一下网络是不是联通的？那么这些其实就是一些前置的校验。

![image-20211213224659115](https://cdn.jsdelivr.net/gh/631068264/img/008i3skNgy1gxclaf5jsnj321k0u0gpt.jpg)





# ConfigMap

`ConfigMap` 功能在 `Kubernetes1.2` 版本中引入，许多应用程序会从配置文件、命令行参数或环境变量中读取配置信息。`ConfigMap API` 给我们提供了向容器中注入配置信息的机制，`ConfigMap` 可以被用来保存单个属性，也可以用来保存整个配置文件或者 `JSON` 二进制大对象

## 介绍

![image-20211213225923394](https://cdn.jsdelivr.net/gh/631068264/img/008i3skNgy1gxclnc8tqej31bl0u0dk7.jpg)

## 注意点

![image-20211213231930396](https://cdn.jsdelivr.net/gh/631068264/img/008i3skNgy1gxcm89kiycj31vb0u045z.jpg)



## 创建

### 使用目录创建

`--from-file` 指定在目录下的所有文件都会被用在 `ConfigMap` 里面创建一个键值对，键的名字就是文件名，值就是文件的内容。

```bash
$ ls docs/user-guide/config-map/kubectl/
game.properties
ui.properties

# game.properties
$ cat docs/user-guide/config-map/kubectl/game.properties
enemies=aliens
lives= 3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAS
secret.code.allowed=true
secret.code.lives= 30

# ui.properties
$ cat docs/user-guide/config-map/kubectl/ui.properties
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice

# 创建名称为game-config的ConfigMap配置
$ kubectl create configmap game-config \
    --from-file=docs/user-guide/config-map/kubectl

# 查看存储的ConfigMap列表
$  kubectl get configmap
NAME               DATA   AGE
game-config        2      22s

# 查看对应内容
$ kubectl describe configmap game-config
$ kubectl get configmap game-config -o yaml
```

### 使用文件创建

只要指定为一个文件就可以从单个文件中创建 `ConfigMap`。`--from-file` 这个参数可以使用多次，你可以使用两次分别指定上个实例中的那两个配置文件，效果就跟指定整个目录是一样的。

```bash
# 创建名称为game-config-2的ConfigMap配置
$ kubectl create configmap game-config-2 \
    --from-file=docs/user-guide/config-map/kubectl/game.properties

# 查看存储的ConfigMap列表
$  kubectl get configmap
NAME               DATA   AGE
game-config        2      34s
game-config-2      1      2s

# 查看对应内容
$ kubectl describe configmap game-config-2
$ kubectl get configmap game-config-2 -o yaml
```

### 使用字面值创建

使用文字值创建，利用 `--from-literal` 参数传递配置信息，该参数可以使用多次，格式如下。

```bash
# 创建名称为special-config的ConfigMap配置
$ kubectl create configmap special-config \
    --from-literal=special.how=very \
    --from-literal=special.type=charm

# 查看对应内容
$ kubectl get configmaps special-config -o yaml
```

## 使用

设置

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.how: very
  special.type: charm
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
  namespace: default
data:
  log_level: INFO
```

- **使用 ConfigMap 来替代环境变量**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-test-pod
spec:
  restartPolicy: Never
  containers:
    - name: test-container
      image: hub.escape.com/library/myapp:v1
      command: [ "/bin/sh", "-c", "env" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.how

        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.type
      envFrom:
        - configMapRef:
          name: env-config
```

- **用 ConfigMap 设置命令行参数**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-test-pod
spec:
  restartPolicy: Never
  containers:
    - name: test-container
      image: hub.escape.com/library/myapp:v1
      command: [ "/bin/sh", "-c", "echo $(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.how
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.type
```

- **通过数据卷插件使用 ConfigMap**

在数据卷里面使用这个 `ConfigMap`，有不同的选项。最基本的就是将文件填入数据卷，在这个文件中，键就是文件名，键值就是文件内容。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-test-pod
spec:
  restartPolicy: Never
  containers:
    - name: test-container
      image: hub.escape.com/library/myapp:v1
      command: [ "/bin/sh", "-c", "cat /etc/config/special.how" ]
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: special-config
```

## 更新

正常情况下，我们可以通过如下配置，在启动的 `Pod` 容器里面获取到 `ConfigMap` 中配置的信息。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: log-config
  namespace: default
data:
  log_level: INFO

---
apiVersion: extensions/v1beta
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
        - name: my-nginx
          image: hub.escape.com/library/myapp:v1
          ports:
            - containerPort: 80
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
      volumes:
        - name: config-volume
          configMap:
            name: log-config
```

```bash
# 查找对应信息
$ kubectl exec \
    `kubectl get pods -l run=my-nginx -o=name | cut -d "/" -f2` \
    cat /etc/config/log_level
INFO
```

修改 `ConfigMap` 配置，修改 `log_level` 的值为 `DEBUG` 等待大概 `10` 秒钟时间，再次查看环境变量的值。

```bash
# 修改ConfigMap配置
$ kubectl edit configmap log-config

# 查找对应信息
$ kubectl exec \
    `kubectl get pods -l run=my-nginx -o=name|cut -d "/" -f2` \
    cat /etc/config/log_level
DEBUG
```

`ConfigMap` 更新后滚动更新 `Pod`，更新 `ConfigMap` 目前并不会触发相关 `Pod` 的滚动更新，可以通过修改 `pod annotations` 的方式强制触发滚动更新。这个例子里我们在 `.spec.template.metadata.annotations` 中添加 `version/config`，每次通过修改 `version/config` 来触发滚动更新。

```bash
$ kubectl patch deployment my-nginx \
    --patch '{"spec": {"template": {"metadata": {"annotations": \
    {"version/config": "20190411" }}}}}'
```

更新 `ConfigMap` 后

- 使用该 `ConfigMap` 挂载的 `Env` 不会同步更新
- 使用该 `ConfigMap` 挂载的 `Volume` 中的数据需要一段时间（实测大概 `10` 秒）才能同步更新

# Secret

Secret 解决了密码、token、密钥等敏感数据的配置问题，而不需要把这些敏感数据暴露到镜像或者 Pod Spec 中。Secret 可以以 Volume 或者环境变量的方式使用。Secret 有三种类型，分别是：

- Service Account
      - 用来访问 Kubernetes API，由 Kubernetes 自动创建，并且会自动挂载到 Pod 的特点目录中。
- Opaque
  - base64 编码格式的 Secret，用来存储密码、密钥等，相当来说不安全。普通的 Secret 文件
- kubernetes.io/dockerconfigjson
  - 用来存储私有 docker registry 的认证信息。
- bootstrap.token，是用于节点接入集群校验用的 Secret。

![image-20211218120216635](https://cdn.jsdelivr.net/gh/631068264/img/008i3skNgy1gxhur5dghfj31lk0u0dmn.jpg)

![image-20211218124618905](https://cdn.jsdelivr.net/gh/631068264/img/008i3skNgy1gxhw0yzg3aj320c0r4qar.jpg)

## Service Account

Service Account 是用来访问 Kubernetes API 接口的，由 Kubernetes 自动创建和管理的，并且会自动挂载到 Pod 的 /run/secrets/kubernetes.io/serviceaccount 目录中。

```bash
$ kubectl get pods -n kube-system
NAME                    READY  STATUS   RESTARTS  AGE
kube-proxy-md1u2        1/1    Running  0         13d

$ kubectl exec kube-proxy-md1u2 -- \
    ls /run/secrets/kubernetes.io/serviceaccount
ca.crt
namespace
token
```

![image-20211218165837349](https://cdn.jsdelivr.net/gh/631068264/img/008i3skNgy1gxi3bhsibdj31kw0u010e.jpg)



## Opaque

- 创建说明

Opaque 类型的数据是一个 map 类型，要求 value 是 base64 编码格式。

```
$ echo -n "admin" | base
YWRtaW4=

$ echo -n "1f2d1e2e67df" | base
MWYyZDFlMmU2N2Rm
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  password: MWYyZDFlMmU2N2Rm
  username: YWRtaW4=
```

- 使用方式 —— 将 Secret 挂载到 Volume 中

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: seret-test
spec:
  containers:
    - name: db
      image: hub.escape.com/library/myapp:v1
      volumeMounts:
        - name: secrets
          mountPath: "readOnly: true"
  volumes:
    - name: secrets
      secret:
        secretName: mysecret
```

- 使用方式 —— 将 Secret 导出到环境变量中

```yaml
apiVersion: extensions/v1beta
kind: Deployment
metadata:
  name: pod-deployment
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: pod-deployment
    spec:
      containers:
        - name: pod-1
          image: hub.escape.com/library/myapp:v1
          ports:
            - containerPort: 80
          env:
            - name: TEST_USER
              valueFrom:
                secretKeyRef:
                  name: mysecret
                  key: username
            - name: TEST_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysecret
                  key: password
      volumes:
        - name: config-volume
          configMap:
            name: mysecret
```

## dockerconfigjson

使用 Kuberctl 创建 docker registry 认证的 secret。

```bash
# 创建格式
$ kubectl create secret docker-registry myregistrykey \
    --docker-server=DOCKER_REGISTRY_SERVER \
    --docker-username=DOCKER_USER \
    --docker-password=DOCKER_PASSWORD \
    --docker-email=DOCKER_EMAIL

# 示例演示
$ kubectl create secret docker-registry myregistrykey \
    --docker-server=hub.escape.com \
    --docker-username=admin \
    --docker-password=harbor123456 \
    --docker-email=harbor@escape.com
```

在创建 Pod 的时候，通过 imagePullSecrets 来引用刚创建的 myregistrykey。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: registry-test
spec:
  containers:
    - name: registry-test
      image: hub.escape.com/library/myapp:v1
      imagePullSecrets:
        name: myregistrykey
```

