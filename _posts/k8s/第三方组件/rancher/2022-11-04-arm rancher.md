---
layout:     post
rewards: false
title:   rancher rke 适配arm64 Kylin Linux Advanced Server
categories:
    - k8s
tags:
   - rancher
---



# 查看系统版本

```sh
# 查看版本
uname -a

Linux localhost.localdomain 4.19.90-24.4.v2101.ky10.aarch64 #1 SMP Mon May 24 14:45:37 CST 2021 aarch64 aarch64 aarch64 GNU/Linux


cat /etc/kylin-release
Kylin Linux Advanced Server release V10 (Sword)
```

# sshd_config

```sh
vim /etc/ssh/sshd_config
# 修改
AllowTcpForwarding yes

# 方便后面安装rke
systemctl restart sshd

```







# docker

参考

- https://www.jianshu.com/p/f62df4b85761
- https://blog.csdn.net/LG_15011399296/article/details/126119349



https://download.docker.com/linux/static/stable/aarch64/

下载docker二进制压缩包

```sh
wget https://download.docker.com/linux/static/stable/aarch64/docker-20.10.7.tgz


# 解压
tar -zxvf docker-20.10.7.tgz
mv docker/* /usr/bin/
```



containerd.service

```sh
# Copyright The containerd Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=1048576
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```

docker.socket

```sh
[Unit]
Description=Docker Socket for the API
PartOf=docker.service

[Socket]
ListenStream=/var/run/docker.sock
SocketMode=0660
SocketUser=root
SocketGroup=root

[Install]
WantedBy=sockets.target
```

docker.service

```sh
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
BindsTo=containerd.service
After=network-online.target firewalld.service containerd.service
Wants=network-online.target
Requires=docker.socket

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd -H unix:// --containerd=/run/containerd/containerd.sock
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always

# Note that StartLimit* options were moved from "Service" to "Unit" in systemd 229.
# Both the old, and new location are accepted by systemd 229 and up, so using the old location
# to make them work for either version of systemd.
StartLimitBurst=3

# Note that StartLimitInterval was renamed to StartLimitIntervalSec in systemd 230.
# Both the old, and new name are accepted by systemd 230 and up, so using the old name to make
# this option work for either version of systemd.
StartLimitInterval=60s

# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity

# Comment TasksMax if your systemd version does not support it.
# Only systemd 226 and above support this option.
TasksMax=infinity

# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes

# kill only the docker process, not all processes in the cgroup
KillMode=process

[Install]
WantedBy=multi-user.target
```





启动docker

```sh
cp containerd.service /usr/lib/systemd/system/
cp docker.socket /usr/lib/systemd/system/
cp docker.service /usr/lib/systemd/system/

systemctl daemon-reload
systemctl enable containerd.service && systemctl start containerd.service
systemctl enable docker.socket
systemctl enable docker.service && systemctl start docker.service
```



# kubectl



```sh
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/arm64/kubectl"
```



# docker-compose

https://github.com/zhangguanzhang/docker-compose-aarch64/releases







# rke

rke从https://github.com/rancher/rke/releases/tag/v1.3.11下载

- 下载rke 镜像时候

  要使用

  ```sh
  docker pull --platform=arm64 镜像名
  ```

  查看需要下载的镜像，注意版本

  ```sh
  rke config --system-images --all
  ```

- 检查cni插件

  ```sh
  # 看看这个目录是否是空的
  ll /opt/cni/bin
  ```

  根据这个[issue](https://github.com/flannel-io/flannel/issues/890#issuecomment-349448629)，下载对应的[arm64二进制](https://github.com/containernetworking/plugins/releases)，解压全部放到上面的目录

  

- [rancher/calico-node不支持arm版本](rancher/calico-node不支持arm版本),遇到`standard_init_linux.go:220: exec user process caused ``"exec format error"`

  修改cluster.yml网络插件为flannel，默认是canal

  ```yaml
  # 设置flannel网络插件
  network:
    plugin: flannel
  ```

  UI创建集群时候网络驱动要选flannel
  
  ![image-20221123144548272](https://cdn.jsdelivr.net/gh/631068264/img/008vxvgGgy1h8f265kyyuj31ty0owac9.jpg)
  
  
  
- coredns error

  ```
  [INFO] plugin/ready: Still waiting on: "kubernetes"
  [INFO] plugin/ready: Still waiting on: "kubernetes"
  [INFO] plugin/ready: Still waiting on: "kubernetes"
  Still waiting on: "kubernetes" HINFO: read udp 114.114.114.114:53: i/o timeout
  Still waiting on: "kubernetes" HINFO: read udp 114.114.114.114:53: i/o timeout
  Still waiting on: "kubernetes" HINFO: read udp 114.114.114.114:53: i/o timeout
  
  ```
  
  可能是iptables设置问题，使用节点清除脚本，再重新安装
  
  - https://github.com/coredns/coredns/discussions/4990
  - https://github.com/coredns/coredns/issues/2693
  
  

# nfs



```
exportfs -r
exportfs: /etc/exports [1]: Neither 'subtree_check' or 'no_subtree_check' specified for export "10.81.133.1:/data/nfs".
  Assuming default behaviour ('no_subtree_check').
  NOTE: this default has changed since nfs-utils version 1.0.x
```



```sh
# 原来
echo '/data/nfs 10.81.133.1(rw,no_root_squash,no_all_squash,sync)' >> /etc/exports
# 添加subtree_check
echo '/data/nfs 10.81.133.1(rw,no_root_squash,no_all_squash,sync,subtree_check)' >> /etc/exports
```



# harbor

https://github.com/goharbor/harbor-arm  编译2.3.0，最好使用联网的arm64编译,同时解决一些错漏的地方



根据https://github.com/goharbor/harbor-arm/issues/37

```
chmod 777 -R  registry registryctl nginx

docker restart registry registryctl nginx
```

才能正常跑
