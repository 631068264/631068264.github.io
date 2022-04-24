---
layout:     post
rewards: false
title:   keepalived
categories:
    - k8s
---

# 简介

Keepalived 是运行在 lvs 之上，是一个用于做双机热备（HA）的软件，它的主要功能是实现真实机的故障隔离及负载均衡器间的失败切换，提高系统的可用性。

## 运行原理

keepalived 通过选举（看服务器设置的权重）挑选出一台热备服务器做 MASTER 机器，MASTER 机器会被分配到一个指定的虚拟 ip，外部程序可通过该 ip 访问这台服务器，如果这台服务器出现故障（断网，重启，或者本机器上的 keepalived crash 等），keepalived 会从其他的备份机器上重选（还是看服务器设置的权重）一台机器做 MASTER 并分配同样的虚拟 IP，充当前一台 MASTER 的角色。

## 选举策略

选举策略是根据 **VRRP 协议**，完全按照**权重大小**，权重最大（0～255）的是 MASTER 机器，下面几种情况会触发选举

- keepalived 启动的时候

- master 服务器出现故障（断网，重启，或者本机器上的 keepalived crash 等，而本机器上其他应用程序 crash 不算）

- 有新的备份服务器加入且权重最大

# 配置

修改 `/etc/keepalived/keepalived.conf`

```sh
# 全局定义 (global definition) 
global_defs {
   # 机器标识，通常配置主机名
   router_id NFS-Master
}

# VRRPD 配置分成三个类vrrp_sync_group，vrrp_instance，vrrp_script
vrrp_instance VI_1 {
    # 指定 instance(Initial)[MASTER|BACKUP]的初始状态就是说在配置好后这台 服务器的初始状态就是这里指定的但这里指定的不算还是得要通过竞选通过优先级来确定里如果这里设置为 master 但如若他的优先级不及另外一台 那么这台在发送通告时会发送自己的优先级另外一台发现优先级不如自己的高那么他会就回抢占为 master
    state MASTER
    #查看当前主机网卡 如"enp0s3" 配置虚拟 VIP 的时候必须是在已有的网卡上添加的
    interface enp0s3
    # 不同的instance，要唯一
    virtual_router_id 51
    # 设置本节点的优先级,优先级高的为 master
    priority 150
    # 设置 MASTER 与 BACKUP 负载均衡之间同步即主备间通告时间检查的时间间隔, 单位为秒，默认 1s
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass nfsuser123
    }
    # 这里设置的就是 VIP 也就是虚拟 IP 地址他随着 state 的变化而增加删除
    # 当 state 为 master 的时候就添加
    # 当 state 为 backup 的时候删除这里主要是有优先级来决定的和 state 设置的值没有多大关系,这里可以设置多个 IP 地址
    virtual_ipaddress {
        10.21.17.94  
    }
}
```



启动

```
systemctl start  keepalived.service && systemctl enable keepalived.service
```





# 双活

> 只需要修改 state 和 priority

 slave state设置BACKUP , priority设置比master低就好了

- 当 master 恢复后会自动回切，影响业务流量



# 常见问题

- virtual_router_id  不是唯一
- vrrp_strict 没有关闭 ，VIP ping不通



# nfs维护脚本

### 

| 角色        | ip          |
| ----------- | ----------- |
| client      | 10.21.17.92 |
| master      | 10.21.17.91 |
| slave       | 10.21.17.93 |
| 虚拟（VIP） | 10.21.17.94 |

在 Master 上编写一个定时任务来检测 nfs 服务是否宕机

```bash
cd /usr/local/sbin
# 生成文件check_nfs.sh
#!/bin/sh
# 每秒执行一次
step=1 #间隔的秒数，不能大于60 
for (( i = 0; i < 60; i=(i+step) )); do 
  ###检查nfs可用性：进程和是否能够挂载
  /sbin/service nfs status &>/dev/null
  if [ $? -ne 0 ];then
    ###如果服务状态不正常，先尝试重启服务
    /sbin/service nfs restart
    /sbin/service nfs status &>/dev/null
    if [ $? -ne 0 ];then
       # 如服务仍不正常，停止 keepalived
       systemctl stop keepalived.service
    fi
  fi
  sleep $step 
done 


```

加入定时任务

```bash
chmod 777 /usr/local/sbin/check_nfs.sh
crontab -e
# 输入定时任务
* * * * *  /usr/local/sbin/check_nfs.sh &> /dev/null
```

在 Client 添加定时任务，当 Master 宕机时进行重新挂载

```bash
cd /usr/local/sbin
# 生成文件check_mount.sh

#!/bin/sh
# 每秒执行一次
step=1 #间隔的秒数，不能大于60 
for (( i = 0; i < 60; i=(i+step) )); do 
  mount=`df -Th|grep /data/nfsdata`
  if [ "$mount" = "" ];then
     umount /data/nfsdata
     mount -t nfs 10.21.17.94:/data/nfs /data/nfsdata
  fi
  sleep $step 
done 
```

加入定时任务

```bash
chmod 777 /usr/local/sbin/check_mount.sh
crontab -e
# 输入定时任务
* * * * *  /usr/local/sbin/check_mount.sh &> /dev/null
```

