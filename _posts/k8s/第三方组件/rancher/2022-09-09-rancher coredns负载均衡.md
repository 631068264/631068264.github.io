---
layout:     post
rewards: false
title:   rancher coredns 负载均衡
categories:
    - k8s
tags:
 - rancher


---

参考：https://coredns.io/plugins/etcd/

# 获取etcd证书



## rke k8s

```sh
docker ps --no-trunc |grep etcd
```

获得 `trusted-ca-file`、`cert-file` 和 `key-file`。

```sh
/usr/local/bin/etcd --name=etcd-d-ecs-38357230 --initial-advertise-peer-urls=https://10.xx.131:2380 --peer-key-file=/etc/kubernetes/ssl/kube-etcd-10-xx-131-key.pem --advertise-client-urls=https://xxx:2379 --peer-client-cert-auth=true --heartbeat-interval=500 --initial-cluster-token=etcd-cluster-1 --listen-peer-urls=https://0.0.0.0:2380 --peer-trusted-ca-file=/etc/kubernetes/ssl/kube-ca.pem --cert-file=/etc/kubernetes/ssl/kube-etcd-10-xx-131.pem --key-file=/etc/kubernetes/ssl/kube-etcd-10-xx-131-key.pem --client-cert-auth=true --data-dir=/var/lib/rancher/etcd/ --listen-client-urls=https://0.0.0.0:2379 --peer-cert-file=/etc/kubernetes/ssl/kube-etcd-10-xx-131.pem --cipher-suites=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 --initial-cluster=etcd-d-ecs-38357230=https://10.xx.131:2380 --initial-cluster-state=new --trusted-ca-file=/etc/kubernetes/ssl/kube-ca.pem --election-timeout=5000
```



```
trusted-ca-file=/etc/kubernetes/ssl/kube-ca.pem
cert-file=/etc/kubernetes/ssl/kube-etcd-10-xx-131.pem
key-file=/etc/kubernetes/ssl/kube-etcd-10-xx-131-key.pem
```



## k3s

```sh
export ETCDCTL_ENDPOINTS='https://127.0.0.1:2379'
export ETCDCTL_CACERT='/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt'
export ETCDCTL_CERT='/var/lib/rancher/k3s/server/tls/etcd/server-client.crt'
export ETCDCTL_KEY='/var/lib/rancher/k3s/server/tls/etcd/server-client.key'
export ETCDCTL_API=3

```





# 获取etcdctl

https://github.com/etcd-io/etcd/releases 解压对应包获取到etcdctl

# 添加证书到coredns容器



cmetcd.sh，根据实际情况填入

```sh
ETCD_CERT=
ETCD_kEY=
ETCD_CA=

cp ${ETCD_CERT} etcd.pem
cp ${ETCD_kEY} etcd-key.pem
cp ${ETCD_CA} etcd-ca.pem

# etcd证书
kubectl create cm coredns-etcd-pem --from-file etcd.pem -n kube-system
# etcd证书key
kubectl create cm coredns-etcd-key --from-file etcd-key.pem -n kube-system
# etcd ca证书
kubectl create cm coredns-etcd-ca  --from-file etcd-ca.pem -n kube-system
```

添加volumes

```sh
kubectl edit deploy  coredns -n kube-system
```

Example

```yaml
        volumeMounts:
        ...
        - name: coredns-etcd-key
          mountPath: /etc/etcd-key
          readOnly: true
        - name: coredns-etcd-pem
          mountPath: /etc/etcd-pem
          readOnly: true
        - name: coredns-etcd-ca
          mountPath: /etc/etcd-ca
          readOnly: true

      volumes:
.....
      - name: coredns-etcd-key
        configMap:
          name: coredns-etcd-key
      - name: coredns-etcd-pem
        configMap:
          name: coredns-etcd-pem
      - name: coredns-etcd-ca
        configMap:
          name: coredns-etcd-ca
```

修改cm

```sh
kubectl edit cm  coredns -n kube-system
```



example

```yaml
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        etcd {
          path /coredns
          # etcd集群端点地址
          endpoint xxx:2379
          upstream /etc/resolv.conf
          # 配置访问etcd证书，注意顺序一定要正确（无证书删除此配置即可）
          tls /etc/etcd-pem/etcd.pem /etc/etcd-key/etcd-key.pem  /etc/etcd-ca/etcd-ca.pem
          fallthrough
        }
        prometheus :9153
        forward . "/etc/resolv.conf"
        cache 30
        loop
```



# 添加DNS A记录

etcda.sh

```sh
ETCD_CERT=
ETCD_kEY=
ETCD_CA=

cp ${ETCD_CERT} etcd.pem
cp ${ETCD_kEY} etcd-key.pem
cp ${ETCD_CA} etcd-ca.pem

export ETCDCTL_API=3

etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=etcd-ca.pem --cert=etcd.pem --key=etcd-key.pem \
  put /coredns/com/test/www/ep1 '{"host":"192.16.58.114","ttl":10}'

etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=etcd-ca.pem --cert=etcd.pem --key=etcd-key.pem \
  put /coredns/com/test/www/ep2 '{"host":"192.16.58.115","ttl":10}'

etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=etcd-ca.pem --cert=etcd.pem --key=etcd-key.pem \
  put /coredns/com/test/www/ep3 '{"host":"192.16.58.116","ttl":10}'
```

A记录添加是反向的即www.test.com要配成 /com/test/www/，后面的ep1为自定义内容，代表www.test.com对应的3个IP记录192.16.58.114 ~ 116三个IP地址

**验证**

```sh
kubectl get svc -n kube-system | grep dns
kube-dns                     ClusterIP   10.43.0.10      <none>        53/UDP,53/TCP,9153/TCP         43d
```

使用dig验证配置的A记录

```
dig @10.43.0.10 www.test.com +short
192.16.58.116
192.16.58.115
192.16.58.114
```

此时域名 www.test.com 已配置好DNS负载均衡，K8s中的Pod访问 www.test.com域名，将负载均衡到3个IP上。

删除记录

```sh
export ETCDCTL_API=3

etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=etcd-ca.pem --cert=etcd.pem --key=etcd-key.pem \
  delete /coredns/com/test/www/ep1
```

