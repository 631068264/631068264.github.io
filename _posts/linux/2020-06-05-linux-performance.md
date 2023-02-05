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



```shell
watch -n 1 -d ifconfig eth1
```

- -n或-interval来指定间隔的时间

# top



## cpu

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

CPU压测

```sh
for i in `seq 1 50`; do dd if=/dev/zero of=/dev/null & done

echo "PID of this script: $$"
```

停止

```sh
ps -ef | grep if=/dev/zero | grep -v grep |awk '{print $2}' | xargs kill -9
```



## 内存

- VIRT：virtual memory usage，虚拟内存
- RES：resident memory usage，物理内存
- SHR：shared memory，共享内存



### 物理内存

物理内存被划分为许多相同大小的部分，也称作内存页

内核使用的部分，所有的进程都需要分配物理内存页给它们的代码、数据和堆栈。进程消耗的这些物理内存被称为“驻留内存”，RSS。

**除去在内核和进程使用的部分**，物理内存剩下的部分被称为**页缓存**(page cache)。因为磁盘io的速度远远低于内存的访问速度，所以为了加快访问磁盘数据的速度，页缓存尽可能的保存着从磁盘读入的数据。

- buffer是缓存将要**写**到硬盘里的数据

- cache是缓存从硬盘**读**出来的数据



**page cache**不足当期驻留在内存中的数据置换到事先配置在磁盘上的**swap**空间

swap交换往往会带来磁盘IO的大量消耗，严重影响到系统正常的磁盘io。出现大量的swap交换说明系统已经快要不行了



sync命令会强制将数据写入磁盘中，并释放该数据对应的buffer，所以常常会在写磁盘后输入sync命令来将数据真正的写入磁盘。

### 虚拟内存

进程运行阶段所需申请的内存可能远远超过物理内存，并且系统不可能只跑一个进程，会有多个进程一起申请使用内存，如果都直接向物理内存进行申请使用肯定无法满足。通过引入虚拟内存，每个进程都有自己独立的虚拟地址空间，这个空间理论上可以无限大，因为它并不要钱。一个进程同一时刻不可能所有变量数据都会访问到，只需要在访问某部分数据时，把这一块虚拟内存映射到物理内存，其他没有实际访问过的虚拟地址空间并不会占用到物理内存，这样对物理内存的消耗就大大减少了 。

系统内核为每个进程都维护了一份**从虚拟内存到物理内存的映射表**，称为页表。页表根据虚拟地址，查找出锁映射的物理页位置和数据在物理页中的偏移量，便得到了实际需要访问的物理地址。

进程持有的虚拟地址（Virtual Address）会经过内存管理单元（Memory Mangament Unit）的转换变成物理地址

- 虚拟内存可以利用内存起到缓存的作用，提高进程访问磁盘的速度；
- 虚拟内存可以为进程提供独立的内存空间，简化程序的链接、加载过程并通过动态库共享内存；
- 虚拟内存可以控制进程对物理内存的访问，隔离不同进程的访问权限，提高系统的安全性；



虚拟内存看作是在**磁盘上一片空间**当这片空间中的一部分访问比较频繁时，**该部分数据会以页为单位被缓存到主存中**以加速 CPU 访问数据的性能，虚拟内存利用空间较大的磁盘存储作为『内存』并使用主存储缓存进行加速。



当用户程序访问未被缓存的虚拟页时，硬件就会触发**缺页中断**（Page Fault，PF），在部分情况下，被访问的页面已经加载到了物理内存中，但是用户程序的**页表**（Page Table）并不存在该对应关系，这时我们只需要在页表中建立虚拟内存到物理内存的关系；在其他情况下，操作系统需要将磁盘上未被缓存的虚拟页加载到物理内存中。



因为主内存的空间是有限的，当主内存中不包含可以使用的空间时，操作系统会从选择合适的物理内存页驱逐回磁盘，为新的内存页让出位置，选择待驱逐页的过程在操作系统中叫做页面替换（Page Replacement）。缺页中断和页面替换技术都是操作系统调页算法（Paging）的一部分，该算法的目的就是充分利用内存资源作为磁盘的缓存以提高程序的运行效率。

## 操作

- E  切换内存单位
- 1 核心数详情
- -c 显示cmd 的完整路径



# 磁盘空间

显示文件或目录所占的磁盘空间

**du -sh [目录名]**：返回该目录的大小

**du -h [目录名]**：查看指定文件夹下的所有文件大小（包含子文件夹）

**du --max-depth=1 -h [目录名]**



**df -h [目录名]**: 检查磁盘空间占用情况(并不能查看某个目录占用的磁盘大小)





du df 区别

- du，disk usage

  通过搜索文件来计算每个文件的大小然后累加，du能看到的文件只是一些当前存在的，没有被删除的。他计算的大小就是当前他认为存在的所有文件大小的累加和

- df，disk free

  通过文件系统来快速获取空间大小的信息，当我们删除一个文件的时候，这个文件不是马上就在文件系统当中消失了，而是暂时消失了，当所有程序都不用时，才会根据OS的规则释放掉已经删除的文件， df记录的是通过文件系统获取到的文件的大小，他比du强的地方就是能够看到已经删除的文件，而且计算大小的时候，把这一部分的空间也加上了，更精确了

当文件系统也确定删除了该文件后，这时候du与df就一致了。



# tcp连接

获取tcp状态数量

```shell
netstat -n| awk '/^tcp/ {++S[$NF]} END {for(a in S) print a,S[a]}'
```

![](https://cdn.jsdelivr.net/gh/631068264/img/008eGmZEgy1goqkn3plpmj31hc0u041x.jpg)

![](https://cdn.jsdelivr.net/gh/631068264/img/008eGmZEgy1goqklzidoqj31860megmm.jpg)

**我们看一下系统默认是如何控制连接建立超时时间的？**

TCP三次握手的第一个SYN报文没有收到ACK，系统会自动对SYN报文进行重试，

最大重试次数由系统参数`net.ipv4.tcp_syn_retries`控制，$$\mathrm{最大超时时间}=2^{x+1}-1\;(x是net.ipv4.tcp\_syn\_etries)$$，默认值为6。初始RTO为1s，如果一直收不到SYN ACK，依次等待1s、2s、4s、8s、16s、32s发起重传，最后一次重传等待64s后放弃，最终在127s后才会返回ETIMEOUT超时错误。

建议根据整个公司的业务场景，调整系统参数进行兜底。该参数设为3，即最大15s左右可返回超时错误。   



提到网络异常检测，大家可能首先想到的是**TCP Keepalive**。系统TCP Keepalive相关的三个参数为

- net.ipv4.tcp_keepalive_time 7200s 如果7200s没有收到对端的数据，就开始发送TCP Keepalive报文

- net.ipv4.tcp_keepalive_intvl 75s  如果75s内，没有收到响应，会继续重试
- net.ipv4.tcp_keepalive_probes  9 直到重试9次都失败后，返回应用层错误信息

为什么需要实现应用层的心跳检查呢？系统的TCP Keepalive满足不了需求吗？是的，系统的TCP Keepalive只能作为一个最基本的防御方案，而满足不了高稳定性、高可靠性场景的需求。原因有如下几点：

- TCP Keepalive是扩展选项，不一定所有的设备都支持；
- TCP Keepalive报文可能被设备特意过滤或屏蔽，如运营商设备；
- TCP Keepalive无法检测应用层状态，如进程阻塞、死锁、TCP缓冲区满等情况；
- TCP Keepalive容易与TCP重传控制冲突，从而导致失效。

**实现应用层心跳需要考虑如下点**

- 心跳间隔不能太小也不能太大。间隔太小心可能会对轻微抖动过于敏感，造成过度反应，反而会影响稳定性，同时也有一定的性能开销；间隔太大会导致异常检测延迟比较高。可以严格地定期发送心跳，也可以一段时间内没有收到对端数据才发起心跳。建议心跳间隔为5s~20s。
- 设置连续失败阈值，避免瞬间抖动造成误判等。建议连续失败阈值为2~5。
- 不要使用独立的TCP连接进行心跳检查，因为不同连接的网络路径、TCP缓冲区等都不同，无法真实反映业务通信连接的真实状态。

**大量客户端可能会同时发起TCP重连及进行应用层请求**

- 当网络异常恢复后，大量客户端可能会同时发起TCP重连及进行应用层请求，可能会造成服务端过载、网络带宽耗尽等问题，从而导致客户端连接与请求处理失败，进而客户端触发新的重试。如果没有退让与窗口抖动机制，该状况可能会一直持续下去，很难快速收敛。

- 建议增加指数退让，如1s、2s、4s、8s...，同时必须限制最大退让时间（如64s），否则重试等待时间可能越来越大，同样导致无法快速收敛。同时，为了降低大量客户端同时建连并请求，也需要增加窗口抖动，窗口大小可以与退让等待时间保持一致，如: $$nextRetryWaitTime\;=\;backOffWaitTime\;+\;rand(0.0,\;1.0)\;\ast\;backOffWaitTime$$

- 在进行网络异常测试或演练时，需要把网络异常时间变量考虑进来，因为不同的时长，给应用带来的影响可能会完全不同。



![](https://cdn.jsdelivr.net/gh/631068264/img/008eGmZEgy1goqkm7dhh3j314s0mwt9y.jpg)

- LISTENING 服务启动处于侦听状态

- ESTABLISHED 建立连接。表示两台机器正在通信

- CLOSE_WAIT **对方**主动关闭连接或者网络异常导致连接中断

- TIME_WAIT  **我方**主动调用close()断开连接，收到对方确认后状态变为TIME_WAIT

  TIME_WAIT 是主动关闭链接时形成的，等待2MSL时间，约4分钟。主要是防止最后一个ACK丢失。  

- TCP连接主动关闭方存在持续2MSL的TIME_WAIT状态；
- TCP连接由是由四元组<本地地址，本地端口，远程地址，远程端口>来确定的

**why 2MSL**

- 确保最后一个确认报文能够到达。如果没能到达，服务端就会重发FIN请求释放连接。等待一段时间没有收到重发就说明服务的已经CLOSE了。如果有重发，则客户端再发送一次LAST ack信号
- 确保当前连接所产生的所有报文都从网络中消失，使得下一个新的连接不会出现旧的连接请求报文

**TIME_WAIT存在的意义主要有两点**：

1. 维护连接状态，使TCP连接能够可靠地关闭。如果连接主动关闭端发送的最后一条ACK丢失，连接被动关闭端会重传FIN报文。因此，主动关闭方必须维持连接状态，以支持收到重传的FIN后再次发送ACK。如果没有TIME_WAIT，并且最后一个ACK丢失，那么此时被动关闭端还会处于LAST_ACK一段时间，并等待重传；如果此时主动关闭方又立即创建新TCP连接且恰好使用了相同的四元组，连接会创建失败，会被对端重置。
2. 等待网络中所有此连接老的重复的、走失的报文消亡，避免此类报文对新的相同四元组的TCP连接造成干扰，因为这些报文的序号可能恰好落在新连接的接收窗口内。

因为每个TCP报文最大存活时间为MSL，一个往返最大是2*MSL，所以TIME_WAIT需要等待2MSL。

当进程关闭时，进程会发起连接的主动关闭，连接最后会进入TIME_WAIT状态。当新进程bind监听端口时，就会报错，因为有对应本地端口的连接还处于TIME_WAIT状态。

实际上，**只有当新的TCP连接和老的TCP连接四元组完全一致，且老的迷走的报文序号落在新连接的接收窗口内时，才会造成干扰**。为了使用TIME_WAIT状态的端口，现在大部分系统的实现都做了相关改进与扩展：

- 新连接SYN告知的初始序列号，要求一定要比TIME_WAIT状态老连接的序列号大，可以一定程度保证不会与老连接的报文序列号重叠。
- 开启TCP timestamps扩展选项后，新连接的时间戳要求一定要比TIME_WAIT状态老连接的时间戳大，可以保证老连接的报文不会影响新连接。

因此，在开启了TCP timestamps扩展选项的情况下（net.ipv4.tcp_timestamps = 1），可以放心的设置SO_REUSEADDR选项，**支持程序快速重启**。

> 注意不要与**net.ipv4.tcp_tw_reuse**系统参数混淆，该参数仅在客户端调用**connect创建连接时才生效**，可以使用TIME_WAIT状态超过1秒的端口（防止最后一个ACK丢失）
>
> 而**SO_REUSEADDR是在bind端口时生效**，一般用于服务端监听时，可以使用本地非LISTEN状态的端口（另一个端口也必须设置SO_REUSEADDR），不仅仅是TIME_WAIT状态端口。

**过多危害**

- socket的TIME_WAIT状态结束之前，该socket所占用的本地端口号将一直无法释放。
- 在高并发（每秒几万qps）并且采用短连接方式进行交互的系统中运行一段时间后，系统中就会存在大量的time_wait状态，如果time_wait状态把系统所有可用端口都占完了且尚未被系统回收时，就会出现无法向服务端创建新的socket连接的情况。此时系统几乎停转，任何链接都不能建立。
- 大量的time_wait状态也会系统一定的fd，内存和cpu资源，当然这个量一般比较小，并不是主要危害

**优化TIME_WAIT过多**


- 调整系统内核参数

  ```
  net.ipv4.tcp_tw_reuse = 1 表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；
  net.ipv4.tcp_tw_recycle = 1 表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。
  ```

  `sysctl -p`命令，来激活上面的设置永久生效

- 短链接为长链接

  - client到nginx的连接是长连接

    	**默认情况下，nginx已经自动开启了对client连接的keep alive支持（同时client发送的HTTP请求要求keep alive）**。一般场景可以直接使用，但是对于一些比较特殊的场景，还是有必要调整个别参数（keepalive_timeout和keepalive_requests）。

    ```
    http {
        keepalive_timeout  120s 120s;
        keepalive_requests 10000;
    }
    ```

    - keepalive_timeout: 第一个参数：设置keep-alive客户端连接在服务器端保持开启的超时值（默认75s）；值为0会禁用keep-alive客户端连接；
    - 第二个参数：可选、在响应的header域中设置一个值“Keep-Alive: timeout=time”；通常可以不用设置；
	  
  - 保持和server的长连接
  
    为了让nginx和后端server（nginx称为upstream）之间保持长连接，典型设置如下：（默认nginx访问后端都是用的短连接(HTTP1.0)一个请求来了，Nginx 新开一个端口和后端建立连接，后端执行完毕后主动关闭该链接）
  
    upstream已经支持keep-alive的，所以我们可以开启Nginx proxy的keep-alive来减少tcp连接
  
    ```
    upstream http_backend {
     server 127.0.0.1:8080;
    
     keepalive 1000;//设置nginx到upstream服务器的空闲keepalive连接的最大数量
    }
    
    server {
     ...
    
    location /http/ {
     proxy_pass http://http_backend;
     proxy_http_version 1.1;//开启长链接
     proxy_set_header Connection "";
     ...
     }
    }
    
    ```



**why CLOSE_WAIT很多**

表示说要么是你的应用程序写的有问题，没有合适的关闭socket；要么是说，你的服务器CPU处理不过来（CPU太忙）或者你的应用程序一直睡眠到其它地方(锁，或者文件I/O等等)，你的应用程序获得不到合适的调度时间，造成你的程序没法真正的执行close操作。

- **响应太慢或者超时设置过小：如果连接双方不和谐，一方不耐烦直接 timeout，另一方却还在忙于耗时逻辑，就会导致 close 被延后**。响应太慢是首要问题，不过换个角度看，也可能是 timeout 设置过小。







每个应用、每个通信协议要有固定统一的监听端口，便于在公司内部形成共识，降低协作成本，提升运维效率。如对于一些网络ACL控制，规范统一的端口会给运维带来极大的便利。

应用监听端口不能在net.ipv4.ip_local_port_range区间内，这个区间是操作系统用于本地端口号自动分配的（bind或connect时没有指定端口号）


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



提高多核负载

```sh
# 运行




# 删除dd
ps -ef | grep if=/dev/zero | grep -v grep |awk '{print $2}' | xargs kill -9

# 个数
ps -ef | grep if=/dev/zero | grep -v grep |wc -l
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

du -h --max-depth=1 | sort -hr
参数说明：
--max-depth：表示要查看几层目录
sort -r：反向显示
sort -h：compare human readable numbers (e.g., 2k 1G)

du -d1 -h /var/lib/docker/containers | sort -h
du -d1 -h /var/lib/docker/overlay2 | sort -h
docker ps -q | xargs docker inspect --format '{{.State.Pid}}, {{.Name}}, {{.GraphDriver.Data.WorkDir}}' | grep "11f9eb2a7369936346812b6d8d95e8441ff4f2d2bd37723ea15f27c84e17a85a"


1. 使用ls也可以实现文件按大小排序，使用-S参数。
ls -lSh
 
注意： 它不能识别视频文件和封装的其他系统文件的大小
 
2. 把文件按时间排序，如下命令可实现
ls -ltu --time-style=long-iso
 
参数说明：
-u: 和lt一起用，也就是-ltu，会按访问时间进行排序
-t：一般和l一起用，也就是-lt，按修改时间排序，最新的在第一位；有时候需要逆排序，那个参数r就行了，也就是ls -ltr
也可以试试ls -ltu --full-time，显示的时间很全。



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

## 系统版本

```sh
# redheat 系列看系统版本
cat /etc/redhat-release

# 其他
cat /etc/issue
lsb_release -a

# 内核版本
cat /proc/version
uname -a
```







## RAID

将多个较小的磁盘整合成为一个较大的磁盘设备； 而这个较大的磁盘功能可不止是**储存**而已，他还具有**数据保护**的功能呢



### raid 0

使用相同型号与容量的磁盘来组成时，效果较佳。等量模式，**性能最佳**

当你有 100MB 的数据要写入时，每个磁盘会各被分配到 50MB 的储存量



越多颗磁盘组成的 RAID-0 性能会越好，因为每颗负责的数据量就更低了！ 这表示我的数据可以分散让多颗磁盘来储存，当然性能会变的更好啊！此外，磁盘总容量也变大了！



RAID-0 只要有任何一颗磁盘损毁，在 RAID 上面的所有数据都会遗失而无法读取。

不同容量的磁盘来组成 RAID-0 时，当小容量磁盘的区块被用完了， 那么所有的数据都将被写入到最大的那颗磁盘去，性能就变差。



### raid1

映射模式 **完整备份**

如果我有一个 100MB 的文件，且我仅有两颗磁盘组成 RAID-1 时， 那么这两颗磁盘将会同步写入 100MB 到他们的储存空间去。 因此，整体 RAID 的容量几乎少了 50%。



大量写入 RAID-1 的情况下，写入的性能可能会变的非常差(hardware raid 主动的复制一份而不使用系统的 I/O 总线)



### RAID 1+0，RAID 0+1

![RAID-1+0 的磁盘写入示意图](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gfhm8wrvbmj30c506aaa8.jpg)

为何会推荐 RAID 1+0 呢？想像你有 20 颗磁盘组成的系统，每两颗组成一个 RAID1，因此你就有总共 10组可以自己复原的系统了！ 然后这 10组再组成一个新的 RAID0，速度立刻拉升 10倍了！同时要注意，因为每组 RAID1 是个别独立存在的，因此任何一颗磁盘损毁， 数据都是从另一颗磁盘直接复制过来重建，并不像 RAID5/RAID6 必须要整组 RAID 的磁盘共同重建一颗独立的磁盘系统！性能上差非常多！ 而且 RAID 1 与 RAID 0 是不需要经过计算的 （striping） ！读写性能也比其他的 RAID 等级好太多了！



### raid 5

RAID-5 至少需要三颗以上的磁盘才能够组成这种类型的磁盘阵列。这种磁盘阵列的数据写入有点类似 RAID-0 ， 不过每个循环的写入过程中 （striping），在每颗磁盘还加入一个同位检查数据 （Parity）



![RAID-5 的磁盘写入示意图](https://cdn.jsdelivr.net/gh/631068264/img/007S8ZIlgy1gfhmqvuyg6g30b205974b.gif)

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