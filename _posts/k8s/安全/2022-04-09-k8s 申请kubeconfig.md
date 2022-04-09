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
users:
  - name: '${SA_NAME}'
    user:
      token: '${SA_TOKEN}'
clusters:
  - cluster:
      certificate-authority-data: '${CERTIFICATE_AUTHORITY_DATA}'
      server: '${APISERVER}'
    name: '${SA_NAME}'-cluster
contexts:
  - context:
      cluster: '${SA_NAME}'-cluster
      namespace: '${NAMESPACE}'
      user: '${SA_NAME}'
    name: '${SA_NAME}'-cluster
current-context: '${SA_NAME}'-cluster' > ${SA_NAME}_kubeconfig
```

