创建自定义RKE时候报错

```
Failed to set up SSH tunneling for host [192.168.1.203]: Can't retrieve Docker Info: error during connect: Get http://%2Fvar%2Frun%2Fdocker.sock/v1.24/info: Unable to access the service on /var/run/docker.sock.

Removing host [192.168.1.203] from node lists
```

![image-20220803161511163](/Users/wyx/union_workspce/docs/问题集/images/image-20220803161511163.png)

- 根据https://github.com/rancher/rke/issues/1417排查

- 排除用户`/var/run/docker.sock`的权限问题，因为用的是root用户

- 确认是https://rancher.com/docs/rke/latest/en/os/#ssh-server-configuration和这个有关

  ```
  vim /etc/ssh/sshd_config
  
  AllowTcpForwarding yes
  ```

- 遇到其他问题也最好参考[rke的安装环境的配置](https://rancher.com/docs/rke/latest/en/os/)。

另外的错误怀疑rancher的bug，即使是配置了好了也会报同样的错，在部署多节点集群时候。

**测试时候使用三个全角色节点**，过程遇到问题不会继续安装，**程序并行下发安装命令**

```sh
[INFO ] [dialer] Setup tunnel for host [A]
[INFO ] [dialer] Setup tunnel for host [B]
[INFO ] [dialer] Setup tunnel for host [C]
[ERROR] Failed to set up SSH tunneling for host [A]: Can't retrieve Docker Info: error during connect: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/info": can not build dialer to [c-gkgpl:m-564c4031e739]
[ERROR] Removing host [A] from node lists

```

这里A被排除了，先安装BC节点，**当BC装好后，并不会重新安装A，直接active，**这错误还概率出现，报错后rke就不再往下跑，得再试多几遍。

而且看见agent报相关的错

```sh
time="2022-08-04T09:21:48Z" level=info msg="Connecting to proxy" url="wss://10.19.64.205:3443/v3/connect/register"
time="2022-08-04T09:21:48Z" level=error msg="Failed to connect to proxy. Response status: 400 - 400 Bad Request. Response body: Operation cannot be fulfilled on nodes.management.cattle.io \"m-564c4031e739\": the object has been modified; please apply your changes to the latest version and try again" error="websocket: bad handshake"
time="2022-08-04T09:21:48Z" level=error msg="Remotedialer proxy error" error="websocket: bad handshake"
```

https://github.com/rancher/rancher/issues/24450，这里也有相关的讨论，最后[这里给了我启示](https://github.com/rancher/rancher/issues/24450#issuecomment-705753168)

![image-20220805142656775](/Users/wyx/union_workspce/docs/问题集/images/image-20220805142656775.png)

尝试不并行，使用并行原因：**不需要关心命令成功与否，赶紧完成。**

**最后方案**

先挑选一个全节点命令先执行，sleep若干sec，剩下的并行执行

```sh
[INFO ] [dialer] Setup tunnel for host [A]
[INFO ] [dialer] Setup tunnel for host [B]
[INFO ] [dialer] Setup tunnel for host [C]
[ERROR] Failed to set up SSH tunneling for host [A]: Can't retrieve Docker Info: error during connect: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/info": can not build dialer to [c-gkgpl:m-564c4031e739]
[ERROR] Removing host [A] from node lists


.....安装完BC节点后，重新运行rke

[ERROR] Provisioning incomplete, host(s) [A] skipped because they could not be contacted
[INFO ] Initiating Kubernetes cluster
[INFO ] Successfully Deployed state file at [management-state/rke/rke-2548767367/cluster.rkestate]
[INFO ] Building Kubernetes cluster
[INFO ] [dialer] Setup tunnel for host [A]
[INFO ] [dialer] Setup tunnel for host [B]
[INFO ] [dialer] Setup tunnel for host [C]
```



