---
layout:     post
rewards: false
title:      docker 磁盘满了
categories:
    - docker

---



```sh
df -h
```



每次创建一个容器时，都会有一些文件和目录被创建，例如：

- `/var/lib/docker/containers/ID`目录，如果容器使用了默认的日志模式，他的所有日志都会以JSON形式保存到此目录下。
- `/var/lib/docker/overlay2` 目录下含有容器的读写层，如果容器使用自己的文件系统保存了数据，那么就会写到此目录下。

**当停止容器后，容器占用的空间就会变为可回收的**（使用`docker system df`查看），**删除容器时会删除其关联的读写层占用的空间。**

```sh
# 所有已经停止的容器
docker container prune
```



# 查看

## system

```sh
# 用于查询镜像（Images）、容器（Containers）和本地卷（Local Volumes）等空间使用大户的空间占用情况
➜  ~ docker system df
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          0         0         0B        0B
Containers      0         0         0B        0B
Local Volumes   0         0         0B        0B
Build Cache     0         0         0B        0B
```
**TYPE 列出了docker 使用磁盘的 4 种类型**：

- Images：所有镜像占用的空间，包括拉取下来的镜像，和本地构建的。
- Containers：运行的容器占用的空间，表示每个容器的读写层的空间。
- Local Volumes：容器挂载本地数据卷的空间。
- Build Cache：镜像构建过程中产生的缓存空间（只有在使用 BuildKit 时才有，Docker 18.09 以后可用）。
- 最后的 RECLAIMABLE 是可回收大小。

**查看空间占用细节**

可以进一步通过 `-v` 参数查看空间占用细节，以确定具体是哪个镜像、容器或本地卷占用了过高空间。示例输出如下：

```sh
docker system df -v


# 镜像的空间使用情况
Images space usage:

REPOSITORY                                                   TAG                 IMAGE ID            CREATED             SIZE                SHARED SIZE         UNIQUE SIZE         CONTAINERS
busybox                                                      latest              6ad733544a63        5 days ago          1.129 MB            0 B                 1.129 MB            1
nginx                                                        latest              b8efb18f159b        3 months ago        107.5 MB            107.5 MB            0 B                 4
ubuntu                                                       latest              14f60031763d        3 months ago        119.5 MB            0 B                 119.5 MB            0
alpine                                                       3.3                 606fed0878ec        4 months ago        4.809 MB            0 B                 4.809 MB            0
tutum/curl                                                   latest              01176385d84a        3 years ago         224.4 MB            0 B                 224.4 MB            1

# 容器的空间使用情况
Containers space usage:

CONTAINER ID        IMAGE                                                                    COMMAND                  LOCAL VOLUMES       SIZE                CREATED             STATUS                     NAMES
d1da451ceeab        busybox                                                                  "ping 127.0.0.1"         0                   10.7 GB             About an hour ago   Up About an hour           dstest
956ae1d241e8        nginx:latest                                                             "nginx -g 'daemon ..."   0                   26 B                3 months ago        Up 3 months                localTest_restserver_2
74973d237a06        nginx:latest                                                             "nginx -g 'daemon ..."   0                   2 B                 3 months ago        Up 3 months                

# 本地卷的空间使用情况
Local Volumes space usage:

VOLUME NAME                                                        LINKS               SIZE
83ba8747f4172a3c02a15f85b71e1565affca59f01352b4a94e0d28e65c26d1c   0                   830 B
a479c303b278f1442f66644f694a554aac630e72b7a27065a11ef85c4d87b648   0                   22.16 MB
79a25b6376e0d6587d8f4f24e08f9467981f04daad14bf3353a12d727d065503   1                   18.83 MB
 
```

## 目录文件大小


```sh
# 查看目录大小
du -sh /var/lib/docker/

# 排序
#参数说明：
#--max-depth：表示要查看几层目录
#sort -r：反向显示
#sort -h：compare human readable numbers (e.g., 2k 1G)
du -h --max-depth=1 . | sort -hr


ls    -Slh

```



## 查看占用空间的pid，以及对应的容器名称

```sh
docker ps -q | xargs docker inspect --format '{{.State.Pid}}, {{.Name}}, {{.GraphDriver.Data.WorkDir}}' | grep "overlay2 hash"
```





## 镜像/容器内空间

### 镜像

```sh
docker history image:tag
```

结合业务情况做进一步处理，重新 build 镜像。

### 容器

按容器显示磁盘使用情况

该`docker ps -s`命令为每个容器显示两个不同的磁盘大小：

```
$ docker ps -s

CONTAINER ID   IMAGE          COMMAND                  CREATED        STATUS       PORTS   NAMES        SIZE                                                                                      SIZE
e90b8831a4b8   nginx          "/bin/bash -c 'mkdir "   11 weeks ago   Up 4 hours           my_nginx     35.58 kB (virtual 109.2 MB)
00c6131c5e30   telegraf:1.5   "/entrypoint.sh"         11 weeks ago   Up 11 weeks          my_telegraf  0 B (virtual 209.5 MB)
```

- “大小”信息显示用于每个容器的*可写*层的数据量（在磁盘上）
- “虚拟大小”是用于容器和可写层使用的只读*图像数据的磁盘空间总量。*







# 清理

**慎重执行**

```sh
# Remove all unused images not just dangling ones
docker system prune -a
WARNING! This will remove:
  - all stopped containers
  - all networks not used by at least one container
  - all images without at least one container associated to them
  - all build cache
  
docker volume prune
```



清理container log

```sh
du -d1 -h /var/lib/docker/containers | sort -h

> xxx-json.log
```



修改文件 `/etc/docker/daemon.json`，并增加以下配置，控制日志的文件个数和单个文件的大小

```json
{
    "log-driver":"json-file",
    "log-opts":{
        "max-size" :"50m","max-file":"1"
    }
}
```



```sh
sudo systemctl daemon-reload
sudo systemctl restart docker
```





# 检查磁盘分区挂载情况

[Linux 硬盘分区、分区、删除分区、格式化、挂载、卸载](https://cloud.tencent.com/developer/article/1504165)

[Linux下mount挂载新硬盘和开机自动挂载](https://www.cnblogs.com/sirdong/p/11969148.html)

```sh
# 检查磁盘分区挂载
lsblk
```

如图有个200G的盘没有挂载，mount point 是空的

![企业微信截图_b34aed63-4667-4eb7-a1a1-dea18aad3b21](https://cdn.jsdelivr.net/gh/631068264/img/e6c9d24egy1h4ebipfkcgj20r20audhh.jpg)



**格式化分区**， [Linux操作系统分区格式Ext2,Ext3,Ext4的区别](https://blog.csdn.net/Gwyp82ndlf/article/details/76032942)

```sh 
# 格式化成ext4
mkfs.ext4 /dev/vdb1 
```

格式化之后，就可以**挂载分区**了。

```sh
mkdir /data

mount /dev/vdb1 /data

# 检查
lsblk


NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0    7:0    0 113.9M  1 loop /snap/core/13308
loop1    7:1    0   114M  1 loop /snap/core/13425
sr0     11:0    1   872K  0 rom
vda    252:0    0    50G  0 disk
├─vda1 252:1    0     1M  0 part
├─vda2 252:2    0   500M  0 part /boot
├─vda3 252:3    0     4G  0 part
└─vda4 252:4    0  45.5G  0 part /
vdb    252:16   0   200G  0 disk
└─vdb1 252:17   0   200G  0 part /data
```

这样设置，机器重启后挂载点消失，所以必须设置永久挂载

```sh
# 查看盘的UUID 格式
blkid

...
/dev/vdb1: UUID="bf731254-57ae-4653-b3bf-a1383360c2e1" TYPE="ext3" PARTUUID="b85efa80-01"
..

# 修改配置
vim /etc/fstab

# 分区设备号 挂载点 文件系统类型	挂载选项	是否备份	
UUID=bf731254-57ae-4653-b3bf-a1383360c2e1 /data ext4 defaults 0 0



# 将 /etc/fstab 中定义的所有档案系统挂上。 检验编辑的内容是否有错
mount -a

```

 

| 要挂载的分区设备号                        | 挂载点 | 文件系统类型 | 挂载选项 | 是否使用dump备份 | 是否开机的时候使用fsck检验所挂载的磁盘 |
| ----------------------------------------- | ------ | ------------ | -------- | ---------------- | -------------------------------------- |
| UUID=bf731254-57ae-4653-b3bf-a1383360c2e1 | /data  | ext4         | defaults | 0                | 0                                      |

 

# docker数据迁移

默认docker数据目录 `/var/lib/docker `

[最方便的docker数据目录迁移](https://blog.51cto.com/u_6364219/5077941)

```sh
# 查看具体位置
docker info | grep "Docker Root Dir"


# stop
systemctl stop docker

# 重启断开，防止自启，断开docker挂载，防止迁移过程出现Device or resource busy
systemctl disable docker


# 迁移
cp -r /var/lib/docker /data/docker
nohup cp -ravf /var/lib/docker /data/docker &
# 加入软连接
ln -s /var/lib/docker /data/docker 

# 或者修改 /etc/docker/daemon.json
vim /etc/docker/daemon.json
{
    ...
    "data-root": "/data/docker",
    ...
}


# 启动dokcer
systemctl start docker
systemctl enable docker
```

## 遇到Error response from daemon: layer does not exist

pod起不来，ImageInspectError之类的报错。[参考](https://www.gushiciku.cn/pl/ptJi)

- 重新pull成功，但是images看不到
- rmi会报错,`Error response from daemon: unrecognized image ID sha256：xxx`

后面删除`/data/docker/image`目录，重启docker，才重新拉取成功，pod也最好delete一下
