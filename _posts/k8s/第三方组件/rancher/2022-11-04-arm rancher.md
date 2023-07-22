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

## buildx

使用 [buildx](https://docs.docker.com/build/building/multi-platform/)

示例

```sh
docker buildx build --platform linux/arm64,linux/arm64,linux/arm --no-cache -f arm.Dockerfile -t image:tag .
```

macOS 或 Windows 系统的 Docker Desktop，以及 Linux 发行版通过 `deb` 或者 `rpm` 包所安装的 `docker` 内置了 `buildx`，不需要另行安装。 [官方安装教程](https://docs.docker.com/build/install-buildx/)

如果你的 `docker` 没有 `buildx` 命令，可以下载二进制包进行安装：

- 首先从 [Docker buildx](https://github.com/docker/buildx/releases/latest) 项目的 release 页面找到适合自己平台的二进制文件。
- 下载二进制文件到本地并重命名为 `docker-buildx`，移动到 docker 的插件目录 `~/.docker/cli-plugins`。
- 向二进制文件授予可执行权限。



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
  
- 创建不了worker节点

  kubelet  看到大量类似报错 `dial tcp 127.0.0.1:6443: connect: connection refused`
  
  ```
  E0106 08:40:08.885824  275376 controller.go:144] failed to ensure lease exists, will retry in 200ms, error: Get "https://127.0.0.1:6443/apis/coordination.k8s.io/v1/namespaces/kube-node-lease/leases/xc-10-81-133-2?timeout=10s": dial tcp 127.0.0.1:6443: connect: connection refused
  
  ```
  
  nginx-proxy，[某个二进制编译格式不对,](https://github.com/rancher/confd/pull/8)导致nginx-proxy没有正常起来，监听6443。
  
  ```
  /usr/bin/nginx-proxy: line 4: /usr/bin/confd: cannot execute binary file: Exec format error
  2023/01/09 02:47:22 [notice] 9#9: using the "epoll" event method
  2023/01/09 02:47:22 [notice] 9#9: nginx/1.21.0
  2023/01/09 02:47:22 [notice] 9#9: built by gcc 10.2.1 20201203 (Alpine 10.2.1_pre1) 
  2023/01/09 02:47:22 [notice] 9#9: OS: Linux 4.19.90-24.4.v2101.ky10.aarch64
  2023/01/09 02:47:22 [notice] 9#9: getrlimit(RLIMIT_NOFILE): 1048576:1048576
  2023/01/09 02:47:22 [notice] 9#9: start worker processes
  2023/01/09 02:47:22 [notice] 9#9: start worker process 10
  ```
  
  [根据这个issue](https://github.com/rancher/rancher/issues/37762), 拿到编译好的**confd**，重新commit 出新的镜像
  
  ```sh
  docker pull jerrychina2020/rke-tools:v0.1.78-linux-arm64
  
  # 下面步骤在arm64机器上进行
  docker -it --rm --name xxx jerrychina2020/rke-tools:v0.1.78-linux-arm64
  # 拿到编译好的confd
  docker cp xxx:/usr/bin/confd .
  
  
  
  docker -it --rm --name xxx rancher/rke-tools:v0.1.80
  
  docker cp confd xxx:/usr/bin/confd
  
  docker commit xxx rancher/rke-tools:v0.1.80
  ```
  
  

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

**第一次**跑**make compile_redis** ,需要注释掉

```sh
function create_sandbox() {
#  docker ps -f "name=$CONTAINER" && docker rm -f $CONTAINER
#  docker inspect --format='{{.Created}}' photon_build_spec:$VERSION.0
#  local status=$?
#  local cdate
#  cdate=$(date --date="$(docker inspect --format='{{.Created}}' photon_build_spec:$VERSION.0)" '+%s')
#  # image exists?
#  if [ $status -eq 0 ]; then
#    local vdate
#    vdate=$(($(date '+%s') - 1209600))
#    # image is less then 2 weeks
#    if [ "$cdate" -gt "$vdate" ]; then
#      # use this image
#      run "Use local build template image" docker run -d --name $CONTAINER --network="host" photon_build_spec:$VERSION.0 tail -f /dev/null
#      return 0
#    else
#      # remove old image
#      docker image rm photon_build_spec:$VERSION.0
#    fi
#  fi

  run "Pull photon image" docker run -d --name $CONTAINER --network="host" photon:$VERSION.0 tail -f /dev/null

  # replace toybox with coreutils and install default build tools
  run "Replace toybox with coreutils" in_sandbox tdnf remove -y toybox
```

详细log 通过**LOGFILE**=/tmp/redis.Ek4eRN/stage/LOGS/redis.log 路径是临时路径





**如果要有multi-arch支持，harbor版本必须大于等于2.0.0。同时push时候要用到[docker mainfest](https://docs.docker.com/engine/reference/commandline/manifest/)**

例子

https 自签名的话使用`--insecure`，`-p, --purge-镜像 push 成功后删除本地的多架构镜像mainfest`

```sh
docker manifest create --insecure myprivateregistry.mycompany.com/repo/image:1.0 \
    myprivateregistry.mycompany.com/repo/image-linux-ppc64le:1.0 \
    myprivateregistry.mycompany.com/repo/image-linux-s390x:1.0 \
    myprivateregistry.mycompany.com/repo/image-linux-arm:1.0 \
    myprivateregistry.mycompany.com/repo/image-linux-armhf:1.0 \
    myprivateregistry.mycompany.com/repo/image-windows-amd64:1.0 \
    myprivateregistry.mycompany.com/repo/image-linux-amd64:1.0
    

docker manifest push -p --insecure myprivateregistry.mycompany.com/repo/image:tag
```

- 为不同arch的镜像**本地**创建清单（**指向同一个image:tag**  例如myprivateregistry.mycompany.com/repo/image:1.0），而且其他arch镜像也得预先push，不能只是本地镜像，不然创建失败。
- push 实际上只是把清单内容push上去





