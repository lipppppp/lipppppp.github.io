---
layout: post
title: Hadoop Tuning Summary (1)
date: '2020-03-17 09:48'
author: Eric Yin
catalog: 'true'
tags:
  - Hadoop
---

# Linux调优
本调优测试在CentOS7.4，可以通过/etc/sysctl.conf控制和配置Linux内核及网络设置，`sysctl -p`使配置生效。
## Linux Kernel Parameter
### vm.swappiness
`vm.swappiness`是从0到100的值，它控制应用程序数据（作为匿名页）从物理内存到磁盘上虚拟内存的交换。该值越高，不活跃的进程从物理内存中换出越活跃。该值越低，交换的次数就越少，从而迫使文件系统缓冲区被清空。

在大多数系统上`vm.swappiness`默认设置为60。这不适用于Hadoop集群，因为即使有足够的可用内存，有时也会交换进程。这可能会导致重要系统守护程序的垃圾回收暂停时间过长，从而影响稳定性和性能。

Cloudera建议设置`vm.swappiness`为在RHEL内核为2.6.32-642.el6或更高版本的系统上进行最小交换，请设置为1到10之间的值，最好为1。

查看`vm.swappiness`的当前设置：
```shell
cat /proc/sys/vm/swappiness
```
设置`vm.swappiness`为1：
```shell
sudo sysctl -w vm.swappiness=1
```

### vm.vfs_cache_pressure
该项表示内核回收用于directory和inode cache内存的倾向：

默认值100表示内核将根据pagecache和swapcache，把directory和inode cache保持在一个合理的百分比。降低该值低于100，将导致内核倾向于保留directory和inode cache。增加该值超过100，将导致内核倾向于回收directory和inode cache。

为了提升读性能，可将该值设置为0。
### vm.nr_hugepages
HugePages是一项集成到2.6版Linux内核中的功能。基本上，此功能提供了更大页面的4K页面大小的替代方法。HugePages是一种拥有较大页面的方法，在处理非常大的内存时非常有用。

:question:设置合理的`vm.nr_hugepages`值相对复杂，将在后期进行具体研讨。

### vm.overcommit_memory
`vm.overcommit_memory`指定了内核针对内存分配的策略，其值可以是0、1、2。
0：表示内核将检查是否有足够的可用内存供应用进程使用；如果有足够的可用内存，内存申请允许；否则，内存申请失败，并把错误返回给应用进程。
1：表示内核允许分配所有的物理内存，而不管当前的内存状态如何。
2： 表示内核允许分配超过所有物理内存和交换空间总和的内存。

## 网络配置
### net core配置
| 配置 | 描述 | 推荐值 |
|--|--|--|
|net.core.wmem_max|为TCP socket预留用于发送缓冲的内存最大值（单位：字节）|1048576|**|
|net.core.wmem_default|为TCP socket预留用于发送缓冲的内存默认值（单位：字节）|262144|
|net.core.rmem_max|为TCP socket预留用于接收缓冲的内存最大值（单位：字节）|4194304|
|net.core.rmem_default|为TCP socket预留用于接收缓冲的内存默认值（单位：字节）|262144|
|net.core.netdev_max_backlog|表示当每个网络接口接收数据包的速率比内核处理这些包的速率快时，允许发送到队列的数据包的最大数目，一般默认值为128。|2000|
|net.core.somaxconn|该参数用于调节系统同时发起的TCP连接数，一般默认值为128.在客户端存在高并发请求的情况下，该默认值较小，connect导致连接超时或重传问题，我们可以根据实际需要结合并发请求数来调节此值。|2048|

### net ipv4配置
| 配置 | 描述 | 推荐值 |
|--|--|--|
|net.ipv4.tcp_fin_timeout|表示如果套接字由本端要求关闭，这个参数决定了它保持在FIN-WAIT-2状态的时间。| 30|
|net.ipv4.tcp_keepalive_time|表示当keepalive起用的时候，TCP发送keepalive消息的频度，默认值为2小时。|1200|
|net.ipv4.tcp_keepalive_intvl|该参数以秒为单位，规定内核向远程主机发送探测指针的时间间隔|30|
|net.ipv4.tcp_keepalive_probes|该参数规定内核为了检测远程主机的存活而发送的探测指针的数量，如果探测指针的数量已经使用完毕仍旧没有得到客户端的响应，即断定客户端不可达，关闭与该客户端的连接，释放相关资源。|3|
|net.ipv4.tcp_tw_reuse|该参数设置TIME_WAIT重用，可以让处于TIME_WAIT的连接用于新的tcp连接。|1|
|net.ipv4.tcp_tw_recycle|该参数设置tcp连接中TIME_WAIT的快速回收。|1|
|net.ipv4.ip_local_port_range|规定了tcp/udp可用的本地端口的范围。|20000 65000|
|net.ipv4.tcp_max_syn_backlog|每一个连接请求(SYN报文)都需要排队，直至本地服务器接收，该变量就是控制每个端口的 TCP SYN队列长度的。如果连接请求多余该值，则请求会被丢弃。|8192|
|net.ipv4.tcp_rmem|TCP socket的读缓冲区的设置，每一项里面都有三个值，第一个值是缓冲区最小值，中间值是缓冲区的默认值，最后一个是缓冲区的最大值，虽然缓冲区的值不受core缓冲区的值的限制，但是缓冲区的最大值仍旧受限于core的最大值。|4096 87380 16777216|
|net.ipv4.tcp_wmem|TCP socket的读缓冲区的设置，每一项里面都有三个值，第一个值是缓冲区最小值，中间值是缓冲区的默认值，最后一个是缓冲区的最大值。|4096 65536 16777216|


## 其他配置
### 关闭tuned服务
CentOS7系列自带tuned服务。关闭tuned服务：
```shell
tuned-adm off
```
查看tuned服务：
```shell
tuned-adm list
```
停止并禁用tuned服务：
```shell
systemctl stop tuned
systemctl disable tuned
```

### 禁用Transparent Hugepages(THP)
大多数Linux平台都包含称为透明大页面的功能，该功能与Hadoop工作负载的交互很差，并且可能严重降低性能。
```shell
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

### 修改IO调度算法
IO调度算法默认有三种NOOP/Deadline/CFQ，其中CFQ更适用于Hadoop。
```shell
echo cfq > /sys/block/{磁盘名，如sda}/queue/scheduler
```

### noatime
默认的方式下linux会把文件访问的时间atime做记录，文件系统在文件被访问、创建、修改等的时候记录下了文件的一些时间戳，比如：文件创建时间、最近一次修改时间和最近一次访问时间；这在绝大部分的场合都是没有必要的。

因为系统运行的时候要访问大量文件，如果能减少一些动作（比如减少时间戳的记录次数等）将会显著提高磁盘 IO 的效率、提升文件系统的性能。

如果遇到机器IO负载高或是CPU WAIT高的情况，可以尝试使用noatime和nodiratime禁止记录最近一次访问时间戳。
```shell
mount -o noatime -o nodiratime -o remount {挂载目录名，如/home}
```