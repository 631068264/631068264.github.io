---
layout:     post
rewards: false
title:   rancher 用户自定义Add-Ons
categories:
   - k8s
tags:
  - rancher


---







# 用户自定义Add-Ons

## 前提
在Rancher UI创建rke1集群的时候，遇到一种情况，如rancher配置了域名绑定访问，而该域名在rke创建完成之后不可达；此时，需要修改coredns的configmap文件才能实现

需要配置~/.kube/config, 再通过kubectl修改coredns, 主要配置hosts：
```
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        hosts {
          ##defineHarborHost##
          192.168.18.6 hzrancher.xxxx.cn
          fallthrough
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . "/etc/resolv.conf"
        cache 30
        loop
        reload
        loadbalance
    } # STUBDOMAINS - Rancher specific change
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
```
因此，为了应对这种创建rke集群需要较复杂的二次手动配置的情况，这次利用rke的[addons](https://rke.docs.rancher.com/config-options/add-ons/user-defined-add-ons)功能解决

## 实验环境：
- rancher v2.6.5 , 绑定入口为：https://hzrancher.xxxx.cn:443
- rke版本 v1.23.6-rancher1-1
- coredns 1.9.1(rke自带)
- docker 19.03.11

基于Rancher UI创建rke集群时，编辑YAML(注意addons配置处于rancher_kubernetes_engine_config部分的下一层)：
```
answers: {}
docker_root_dir: /var/lib/docker
enable_cluster_alerting: false
enable_cluster_monitoring: false
enable_network_policy: false
fleet_workspace_name: fleet-default
local_cluster_auth_endpoint:
  enabled: false
name: hztest-0705
rancher_kubernetes_engine_config:
  addon_job_timeout: 45
  addons: |-
    ---
    apiVersion: v1
    data:
      Corefile: |
        .:53 {
            errors
            health {
              lameduck 5s
            }
            hosts {
              ##defineHarborHost##
              192.168.18.6 hzrancher.xxxx.cn
              fallthrough
            }
            ready
            kubernetes cluster.local in-addr.arpa ip6.arpa {
              pods insecure
              fallthrough in-addr.arpa ip6.arpa
            }
            prometheus :9153
            forward . "/etc/resolv.conf"
            cache 30
            loop
            reload
            loadbalance
        } # STUBDOMAINS - Rancher specific change
    kind: ConfigMap
    metadata:
      name: coredns
      namespace: kube-system
  authentication:
    strategy: x509
  authorization: {}
  bastion_host:
    ignore_proxy_env_vars: false
    ssh_agent_auth: false
  cloud_provider: {}
  enable_cri_dockerd: false
  ignore_docker_version: true
  ingress:
    default_backend: false
    default_ingress_class: true
    http_port: 0
    https_port: 0
    provider: nginx
  kubernetes_version: v1.23.6-rancher1-1
  monitoring:
    provider: metrics-server
    replicas: 11
  network:
    mtu: 1500
    plugin: canal
  restore:
    restore: false
  rotate_encryption_key: false
  services:
    etcd:
      backup_config:
        enabled: true
        interval_hours: 12
        retention: 6
        safe_timestamp: false
        timeout: 300
      creation: 12h
      extra_args:
        election-timeout: '5000'
        heartbeat-interval: '500'
      gid: 0
      retention: 72h
      snapshot: false
      uid: 0
    kube-api:
      always_pull_images: false
      pod_security_policy: false
      service_node_port_range: 30000-32767
    kube-controller: {}
    kubelet:
      fail_swap_on: false
      generate_serving_certificate: false
    kubeproxy: {}
    scheduler: {}
  ssh_agent_auth: false
  upgrade_strategy:
    drain: false
    max_unavailable_controlplane: '1'
    max_unavailable_worker: '10'
    node_drain_input:
      delete_local_data: false
      force: false
      grace_period: 30
      ignore_daemon_sets: true
      timeout: 30
```
其中，addon_job_timeout为部署addons插件执行的连接时间，默认45秒

## 验证
当上面Rancher UI配置好所有参数之后，生成命令：
```bash
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run  cc/cc-agent:v2.6.5 --server https://hzrancher.xxxx.cn:443 --token 2fdj7wq88hz87zd4wnvk2v8c5pk44cl92cck8rsd9crddlg4dvz6mm --ca-checksum e244da828e023bbf709b7f9e3ff503cae602f11f163257dd3a4ff14668345d20 --etcd --controlplane --worker
be275f38ce2d5f710d8bddfa765e31f43b8cdef7dc83d13160cda30cef8ff807
```
粘贴到rke集群的节点机器执行，之后留意Rancher集群管理-配置日志：
```
[INFO ] [addons] Saving ConfigMap for addon rke-ingress-controller to Kubernetes
[INFO ] [addons] Successfully saved ConfigMap for addon rke-ingress-controller to Kubernetes
[INFO ] [addons] Executing deploy job rke-ingress-controller
[INFO ] [ingress] removing default backend service and deployment if they exist
[INFO ] [ingress] ingress controller nginx deployed successfully
[INFO ] [addons] Setting up user addons
[INFO ] [addons] Saving ConfigMap for addon rke-user-addon to Kubernetes
[INFO ] [addons] Successfully saved ConfigMap for addon rke-user-addon to Kubernetes
[INFO ] [addons] Executing deploy job rke-user-addon
[INFO ] [addons] User addons deployed successfully
[INFO ] Finished building Kubernetes cluster successfully
```
看到日志记录显示：User addons deployed successfully, 说明addons配置安装成功。

也可以通过kubectl查看addons执行记录，以便查询addons配置错误：
```bash
$ kubectl -n kube-system get pod  | grep user-addon
rke-user-addon-deploy-job-2rx4r           0/1     Completed   0          3h44m
$ kubectl -n kube-system logs rke-user-addon-deploy-job-2rx4r
configmap/coredns configured
```

## 其他探索
在addons功能测试过程中，也可以自定义coredns，禁止rke默认安装的coredns，使用addons将coredns所有yaml部署文件添加上去。
基于现在rke集群导出coredns的所有yaml部署文件：
```bash
$ vi generate_coredns_yamls.sh

#!/bin/bash

function write() {
    echo "---" >> output.yaml
    cat $1 >> output.yaml
}

function create() {
    kubectl get sa coredns -n kube-system -o yaml > coredns-sa.yaml
    kubectl get sa coredns-autoscaler -n kube-system -o yaml > coredns-autoscaler-sa.yaml

    kubectl get ClusterRole system:coredns -n kube-system -o yaml > coredns-rabc.yaml
    kubectl get ClusterRole system:coredns-autoscaler -n kube-system -o yaml > coredns-autoscaler-rabc.yaml

    kubectl -n kube-system get cm coredns -o yaml > coredns-cm.yaml
    kubectl -n kube-system get cm coredns-autoscaler -o yaml > coredns-autoscaler-cm.yaml

    kubectl get ClusterRoleBinding system:coredns -n kube-system -o yaml > coredns-rolebinding.yaml
    kubectl get ClusterRoleBinding system:coredns-autoscaler -n kube-system -o yaml > coredns-autoscaler-rolebinding.yaml

    kubectl get deployment coredns -n kube-system -o yaml > coredns-deployment.yaml
    kubectl get deployment coredns-autoscaler -n kube-system -o yaml > coredns-autoscaler-deployment.yaml

    kubectl -n kube-system get svc kube-dns -o yaml > kube-dns.yaml

    for file in "$(dirname "$0")"/*.yaml; do
      echo "$file"
      cp "$file" "$file.orign"
      yq eval 'del(.secrets, .metadata.annotations, .metadata.generation, .metadata.resourceVersion, .metadata.uid, .metadata.creationTimestamp, .spec.progressDeadlineSeconds, .spec.revisionHistoryLimit, .spec.template.metadata.annotations, .spec.template.metadata.creationTimestamp, .spec.template.spec.terminationMessagePath, .spec.template.spec.terminationMessagePolicy, .spec.template.securityContext, .status.availableReplicas, .status.conditions, .status.observedGeneration, .status.readyReplicas, .status.replicas, .status.updatedReplicas, .spec.clusterIPs, .status.loadBalancer)' "$file" > "$file.tmp" && mv "$file.tmp" "$file"
    done
}

create

rm -rf output.yaml
touch output.yaml
write coredns-sa.yaml
write coredns-autoscaler-sa.yaml
write coredns-rabc.yaml
write coredns-autoscaler-rabc.yaml
write coredns-cm.yaml
write coredns-autoscaler-cm.yaml
write coredns-rolebinding.yaml
write coredns-autoscaler-rolebinding.yaml
write coredns-deployment.yaml
write coredns-autoscaler-deployment.yaml
write kube-dns.yaml
```
执行generate_coredns_yamls.sh获取汇总后的output.yaml文件(结合实际情况修改)，添加到addons中，同时需要禁用rke自带的coredns
```
rancher_kubernetes_engine_config:
  dns:
    provider: none
```

# 参考
[https://rke.docs.rancher.com/config-options/add-ons/user-defined-add-ons](https://rke.docs.rancher.com/config-options/add-ons/user-defined-add-ons)

[https://rke.docs.rancher.com/config-options/add-ons/dns#disabling-deployment-of-a-dns-provider](https://rke.docs.rancher.com/config-options/add-ons/dns#disabling-deployment-of-a-dns-provider)
