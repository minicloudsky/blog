---
title: Linux中swap与memory介绍
date: 2020-02-09 11:32:09
tags:
---

# 1. 背景介绍

 这篇文章介绍一下 Linux 中 swap 与 memory。memory 就是机器的物理内存ram，读写速度低于 cpu 一个量级，但是高于磁盘不止一个量级。所以，程序和数据如果在内存的话，会有非常快的读写速度。但是，内存的造价是要高于磁盘的，虽然相对来说价格一直在降低。除此之外，内存的断电丢失数据也是一个原因，因此不能把所有数据和程序都保存在内存中。既然不能全部使用内存，那数据还有程序肯定不可能一直霸占在内存中。当内存没有可用的，就必须要把内存中不经常运行的程序给踢出去。但是踢到哪里去，这时候 swap 就出现了。**swap 全称为 swap place，即交换区**，当内存不够的时候，被踢出的进程数据被暂时存储到交换区。当需要这条被踢出的进程的时候，就从交换区重新加载到内存，否则它不会主动交换到真实内存中。

<!-- more -->

# 2.swap 介绍

 在详细介绍 swap 之前，我们需要知道的是计算机对内存分为物理内存与虚拟内存（注意虚拟内存和虚拟地址空间的区别）。物理内存就是计算机的实际内存大小，由 RAM 芯片组成的。虚拟内存则是虚拟出来的、使用磁盘代替内存。虚拟内存的出现，让机器内存不够的情况得到部分解决。当程序运行起来由操作系统做具体虚拟内存到物理内存的替换和加载 (相应的页与段的虚拟内存管理)。**这里的虚拟内存即所谓的 swap**。

 当用户提交程序，然后产生进程，在机器上运行。机器会判断当前物理内存是否还有空闲允许进程调入内存运行，如果有那么则直接调入内存进行运行；如果没有，那么会根据优先级选择一个进程挂起，把该进程交换到 swap 中等待，然后把新的进程调入到内存中运行。根据这种换入和换出，实现了内存的循环利用，让用户感觉不到内存的限制。从这也可以看出 swap 扮演了一个非常重要的角色，就是暂存被换出的进程。 **内存与 swap 之间是按照内存页为单位来交换数据的，一般 Linux 中页的大小设置为 4kb。而内存与磁盘则是按照块来交换数据的。**

# 3.swap 的设置

 从上可以看出，当物理内存使用完或者达到一定比例之后，我们可以使用 swap 做临时的内存使用。当物理内存和 swap 都被使用完那么就会出错，out of memory。对于使用多大比例内存之后开始使用 swap，在系统的配置文件中可以通过调整参数进行修改。cat  /proc/sys/vm/swappiness 60

该参数可以从 0-100 进行设置。0 就是最大限度使用内存，尽量不使用 swap；100 就是积极使用 swap。这个具体的通过系统的算法进行确定。

 物理内存我们是无法更改的，所以 swap 的大小设置将会影响应用能否正常运行。那么 swap 大小如何确定。根据 centos 官网介绍可以得出如下公式：M = Amount of RAM in GB, and S = Amount of swap in GB, then If M <2, S = M *2 Else S = M + 2。而且其最小不应该小于 32M (never less than 32 MB.)。

 swap 分区的数量对性能也有很大的影响。因为 swap 毕竟还是以磁盘来伪装成内存，交换的操作是磁盘 IO 的操作而不是内存的 load 与 store 操作。如果有多个 swap 交换区，每个 swap 会有一定的优先级，该优先级也可以调整。swap 空间的分配会以轮流的方式操作于所有的 swap，这样会大大均衡 IO 的负载，加快 swap 交换的速度。

四：swap 相关命令

 swapon/swapoff swap-disk_name：启动和关闭相应的 swap_disk_name

 swapon -s ：可以查看当期 swap 的使用情况，也可以通过 cat /proc/swaps 命令查看

### centos开启虚拟内存

1 mkdir /swaps

2 cd /swaps

3   dd if=/dev/zero of=swaps bs=512k count=4096  

 swap大小为bs\*count=4096\*512/1024/1024=(2G)

4  swapon /swaps/swaps

5 开机挂载

cat /etc/fstab

/swaps/swaps swap swap defaults 0 0

### 手动创建swap分区

#### 使用cat /proc/swaps查看当前虚拟内存的情况

![](1.png)

#### 可见没有开启swap，手动创建一个文件夹，叫swaps

 在 /swaps 这个路径下执行**`dd if=/dev/zero of=swaps bs=512 count=4194308`**，在这里创建 **swap 大小为 bs\*count=21490217e9 (2G)**，这个过程需要一点时间，稍等片刻： 

![](2.png)

 通过**`mkswap swaps`**命令将上面新建出的 swaps 文件做成 swap 

![](3.png)

 此时使用 `cat /proc/sys/vm/swappiness` 查看数值应该是 0，需要**`sysctl -w vm.swappiness=60`**把它改成 60，这里 60 的含义是：100%-60%=40%，即物理内存剩下 40% 的时候时启用虚拟内存。若想永久修改，则编辑 `/etc/sysctl.conf` 文件，改文件中有 `vm.swappiness` 变量配置。 

![](4.png)

![](5.png)

 再**`swapon /swaps/swaps`**： 

![](6.png)

最后就是**添加开机自动挂载**，即在**`/etc/fstab`**文件添加如下一句：**/swaps/swaps swap swap defaults 0 0**

再用**`cat /proc/swaps`**命令检查一下 swap 分区是否启

![](7.png)

free -h查看下

![](8.png)

