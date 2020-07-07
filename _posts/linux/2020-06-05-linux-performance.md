---
layout: post
rewards: false
title:  Linux 性能
categories:
    - Linux

---



# ifconfig 流量监控

```shell
eth2: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 9000
        inet 192.168.99.49  netmask 255.255.255.0  broadcast 192.168.99.255
        inet6 fe80::329c:23ff:febb:e25e  prefixlen 64  scopeid 0x20<link>
        ether 30:9c:23:bb:e2:5e  txqueuelen 1000  (Ethernet)
        RX packets 2116672816  bytes 1619602963167 (1.6 TB)
        RX errors 0  dropped 1022340  overruns 0  frame 0
        TX packets 3664698944  bytes 441913769791 (441.9 GB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 
```

- RX==receive，**接收**，从开启到现在接收封包的情况，是下行流量。

- TX==Transmit，**发送**，从开启到现在发送封包的情况，是上行流量。



# top load average

```shell
load average: 0.00, 0.01, 0.05
1min 5min 15min 系统的平均负荷。
```

- 当CPU完全空闲的时候，平均负荷为0
- 当CPU工作量饱和的时候，平均负荷为1  (n个CPU的服务器，可接受的系统负荷最大为n.0 )
- 大于1 进程在等待

>  值越低，说明电脑的工作量越小，系统负荷比较轻。



```shell
%Cpu(s): 0.0 us, 0.1 sy, 0.0 ni, 99.9 ``id``, 0.0 wa, 0.0 hi, 0.0 si, 0.0 st
```

- us（user cpu time）：用户态使用的cpu时间比。该值较高时，说明用户进程消耗的 CPU 时间比较多，比如，如果该值长期超过 50%，则需要对程序算法或代码等进行优化。
- id（idle cpu time）：空闲的cpu时间比。如果该值持续为0，同时sy是us的两倍，则通常说明系统则面临着 CPU 资源的短缺。 **这个值过低，表明系统CPU存在瓶颈**
- wa（io wait cpu  time）：cpu等待磁盘写入完成时间。**该值较高时，说明IO等待比较严重**，这可能磁盘大量作随机访问造成的，也可能是磁盘性能出现了瓶颈。



## 操作

- E  切换内存单位
- 1 核心数详情
- -c 显示cmd 的完整路径



# 磁盘空间

**du -sh [目录名]**：返回该目录的大小

**du -h [目录名]**：查看指定文件夹下的所有文件大小（包含子文件夹）

**df -hl**  





# 机器参数

## CPU核数

- 总核数 = 物理CPU个数 X 每颗物理CPU的核数
- 总逻辑CPU数 = 总核数  X 超线程数



```shell
物理CPU个数
cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l

每个物理CPU中core的个数(即核数)
cat /proc/cpuinfo| grep "cpu cores"| uniq

逻辑CPU的个数
cat /proc/cpuinfo| grep "processor"| wc -l
```

## 内存 

```shell
free -h
```



##  磁盘

```shell
查看逻辑磁盘大小

fdisk -l |grep Disk

查看分区情况

df -h

目录大小
du -sh /var/lib/*
```



测速

- 顺序读写指文件指针只能从头移动到尾。

- 随机读写指文件指针可以根据需要，随意移动





/dev/null 伪设备，回收站，不会产生写IO
/dev/zero 伪设备，会产生空字符流，它不会产生读IO



```shell
随机4k写入 300 kB/s

dd if=/dev/zero of=test bs=4k count=1k oflag=sync

随机4k读取 500 MB/s

dd if=test bs=4k count=1k of=/dev/null

顺序写入  200 MB/s 即可
dd if=/dev/zero of=test bs=256M count=50 oflag=sync

dd if=/root/test of=test oflag=sync


```



## RAID

将多个较小的磁盘整合成为一个较大的磁盘设备； 而这个较大的磁盘功能可不止是**储存**而已，他还具有**数据保护**的功能呢



## raid 0

使用相同型号与容量的磁盘来组成时，效果较佳。等量模式，**性能最佳**

当你有 100MB 的数据要写入时，每个磁盘会各被分配到 50MB 的储存量



越多颗磁盘组成的 RAID-0 性能会越好，因为每颗负责的数据量就更低了！ 这表示我的数据可以分散让多颗磁盘来储存，当然性能会变的更好啊！此外，磁盘总容量也变大了！



RAID-0 只要有任何一颗磁盘损毁，在 RAID 上面的所有数据都会遗失而无法读取。

不同容量的磁盘来组成 RAID-0 时，当小容量磁盘的区块被用完了， 那么所有的数据都将被写入到最大的那颗磁盘去，性能就变差。



## raid1

映射模式 **完整备份**

如果我有一个 100MB 的文件，且我仅有两颗磁盘组成 RAID-1 时， 那么这两颗磁盘将会同步写入 100MB 到他们的储存空间去。 因此，整体 RAID 的容量几乎少了 50%。



大量写入 RAID-1 的情况下，写入的性能可能会变的非常差(hardware raid 主动的复制一份而不使用系统的 I/O 总线)



## RAID 1+0，RAID 0+1

![RAID-1+0 的磁盘写入示意图](https://tva1.sinaimg.cn/large/007S8ZIlgy1gfhm8wrvbmj30c506aaa8.jpg)

为何会推荐 RAID 1+0 呢？想像你有 20 颗磁盘组成的系统，每两颗组成一个 RAID1，因此你就有总共 10组可以自己复原的系统了！ 然后这 10组再组成一个新的 RAID0，速度立刻拉升 10倍了！同时要注意，因为每组 RAID1 是个别独立存在的，因此任何一颗磁盘损毁， 数据都是从另一颗磁盘直接复制过来重建，并不像 RAID5/RAID6 必须要整组 RAID 的磁盘共同重建一颗独立的磁盘系统！性能上差非常多！ 而且 RAID 1 与 RAID 0 是不需要经过计算的 （striping） ！读写性能也比其他的 RAID 等级好太多了！



## raid 5

RAID-5 至少需要三颗以上的磁盘才能够组成这种类型的磁盘阵列。这种磁盘阵列的数据写入有点类似 RAID-0 ， 不过每个循环的写入过程中 （striping），在每颗磁盘还加入一个同位检查数据 （Parity）



![RAID-5 的磁盘写入示意图](https://tva1.sinaimg.cn/large/007S8ZIlgy1gfhmqvuyg6g30b205974b.gif)

每个循环写入时，都会有部分的同位检查码 （parity） 被记录起来，并且记录的同位检查码每次都记录在不同的磁盘， 因此，任何一个磁盘损毁时都能够借由其他磁盘的检查码来重建原本磁盘内的数据喔！不过需要注意的是， 由于有同位检查码，因此 RAID 5 的总容量会是整体磁盘数量减一颗。以上图为例， 原本的 3 颗磁盘只会剩下 （3-1）=2 颗磁盘的容量。而且当损毁的磁盘数量大于等于两颗时，这整组 RAID 5 的数据就损毁了。 因为 RAID 5 默认仅能支持一颗磁盘的损毁情况。



| 项目                | RAID0                      | RAID1      | RAID10             | RAID5      | RAID6       |
| ------------------- | -------------------------- | ---------- | ------------------ | ---------- | ----------- |
| 最少磁盘数          | 2                          | 2          | 4                  | 3          | 4           |
| 最大容错磁盘数（1） | 无                         | n-1        | n/2                | 1          | 2           |
| 数据安全性（1）     | 完全没有                   | 最佳       | 最佳               | 好         | 比 RAID5 好 |
| 理论写入性能（2）   | n                          | 1          | n/2                | <n-1       | <n-2        |
| 理论读出性能（2）   | n                          | n          | n                  | <n-1       | <n-2        |
| 可用容量（3）       | n                          | 1          | n/2                | n-1        | n-2         |
| 一般应用            | 强调性能但数据不重要的环境 | 数据与备份 | 服务器、云系统常用 | 数据与备份 | 数据与备份  |