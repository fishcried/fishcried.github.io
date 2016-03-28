---
layout: post
title: "Cpu bindings (一) 理解cpu topology"
description: ""
category: 性能调优
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [cpu,性能调优]
---

> **cpu bindings**来提升性能很常用.但是实施绑定之前需要对当前cpu的topology有所了解，太随意可能会导致性能下降．本文对**Intel(R) Xeon(R) CPU E5-2650 v2 @ 2.60GHz**的topology进行了简单的分析.

# **Node,Sockets,Cores,Threads**

有些基本概念还是要较真一下的.比如sockets,cores,threads.很多时候觉得理解了，但闭嘴一想，心里又会咯噔一下...

曾在kvm启动虚机时指定参数时遇到过,由于不是必须指定,所以错过了研究机会,直到最近才仔细的查找资料.试图阐述总结,发现这张图表达的非常好:

![socket&core&thread](/img/mc_support.gif)

- Sockets
    可以理解成主板上cpu的插槽数.物理cpu的颗数.一般同一socket上的core共享三级缓存.
- Cores
    常说的核,核有独立的物理资源.比如单独的一级二级缓存什么的.
- Threads
    我理解成逻辑cpu了.如果不开超线程,threads应该与cores相等,如果开了超线程,threads应该是cores的倍数.相互之间共享物理资源.
- Nodes 上图中没有提及
    Node是NUMA体系中的概念．由于SMP体系中各个CPU访问内存只能通过单一的通道．导致内存访问成为瓶颈,cpu再多也无用．后来引入了NUMA．通过划分node,每个node有本地RAM,这样node内访问RAM速度会非常快．但跨Node的RAM访问代价会相对高一点.可以看下面的示意图.

![SMP vs NUMA](/img/smp_vs_numa.png)

由此可以总结这样的逻辑关系(包含):**`Node > Socket > Core > Thread`**.
区分这几个概念为了了解cache的分布,因为cpu绑定的目的就是提高cache的命中率,降低cpu颠簸.所以了解cache与cpu之间的mapping关系是非常重要的.通常来讲:

2. 同Socket内的cpu共享三级级缓存.
3. 每个Core有自己独立的二级缓存.
4. 一个Core上超线程出来的Threads,避免绑定，看似可能会提高L2 cache命中率,但也可能有严重的cpu争抢，导致性能非常差.

# 查看CPU信息

最简单直接的方式就是使用`lscpu`,`numactl`.

    root@ncloud-config:~# lscpu
    Architecture:          x86_64
    CPU op-mode(s):        32-bit, 64-bit
    Byte Order:            Little Endian
    CPU(s):                32
    On-line CPU(s) list:   0-31
    Thread(s) per core:    2
    Core(s) per socket:    8
    Socket(s):             2
    NUMA node(s):          2
    Vendor ID:             GenuineIntel
    CPU family:            6
    Model:                 62
    Stepping:              4
    CPU MHz:               1200.000
    BogoMIPS:              5189.35
    Virtualization:        VT-x
    L1d cache:             32K
    L1i cache:             32K
    L2 cache:              256K
    L3 cache:              20480K
    NUMA node0 CPU(s):     0-7,16-23
    NUMA node1 CPU(s):     8-15,24-31 

看到了么,2颗8核双线程,一共是32 processors.也可以看到是numa体系.可以使用一下命令详细查看numa信息.非NUMA体系时,所有cpu都划分为一个Node.

    root@ncloud-config:~# numactl --hardware
    available: 2 nodes (0-1)
    node 0 cpus: 0 1 2 3 4 5 6 7 16 17 18 19 20 21 22 23
    node 0 size: 64395 MB
    node 0 free: 62846 MB
    node 1 cpus: 8 9 10 11 12 13 14 15 24 25 26 27 28 29 30 31
    node 1 size: 64507 MB
    node 1 free: 63676 MB
    node distances:
    node   0   1
      0:  10  20
      1:  20  10


两个Node,每个Node内RAM为64G.注意,Node内的cpu id不连续哦！

## 装X的方式(通过proc & sys的手动获取)

    # 获取cpu名称与主频
    cat /proc/cpuinfo | grep 'model name'  | cut -f2 -d: | head -n1 | sed 's/^ //'
    
    # 获取逻辑核数
    cat /proc/cpuinfo | grep 'model name'  | wc -l
    
    # 获取物理核数
    cat /proc/cpuinfo | grep 'physical id' | sort | uniq | wc -l
    
    # 查看cpu的flags
    cat /proc/cpuinfo | grep flags | uniq | cut -f2 -d : | sed 's/^ //'

    # 是否打开超线程
    cat /proc/cpuinfo | grep -qi "core id" 
    if [$? -ne 0 ]; then
        echo "hyper-threading yes"
    else
        echo "hyper-threading no"
    fi
    
    # 查看cache大小,X自省替换
    sudo cat /sys/devices/system/cpu/cpuX/cache/indexX/size
    # 查看各个cpu之间与cache的mapping
    cat /sys/devices/system/cpu/cpuX/cache/indexX/shared_cpu_list


**装X**并不是贬义不赞成，而是这种方式真的很酷，稀里哗啦的一顿键盘就像变魔术一样．而且这种方式又是最原始的．`lscpu`,`numactl`都是读取`proc`,`sys`文件系统信息并进行格式化，输出人性化的内容．当没有网络,而lscpu,numactl都没有安装时，只能使用这种方式了

能用工具还是用工具，工具就是解放双手的，只要别把大脑也解放了就好．

# 3. Cpu Topology可视化

通过`lscpu`与`numactl`获取的信息，必要的时候查询了`/sys/devices/system/cpu/cpuX/*`的数据将正在使用的 **Intel(R) Xeon(R) CPU E5-2650 v2 @ 2.60GHz**的topology进行可视化．

![cpu topology](/img/cpu_to.png)

**Numa分组信息**

- 通过图可以看到cpu为numa架构,且有两个node.
- 将同一socket内的cpu(threads)都划分在一个node中.通过上图也解释了node中cpu序列不连续的问题.因为同一个Core上的两个Threads是超线程出来的.超线程Thread的cpu id在原有的core id基础上增长的
- 每个node中有64507MB的本地RAM可用.

**cache信息**

- 每个core都有独立的二级缓存,而不是socket中所有的core共享二级缓存.nice!
- 同node中的cpu共享三级缓存.

**cpu绑定注意的几点**

- Numa体系中,如果夸node绑定,性能会下降.因为L3 cache命中率低,跨node内存访问代价高.
- 绑定同Node,同一个Core中的两个超线程出来的cpu,性能会急剧下降.cpu密集型的线程硬件争用严重."玩转CPU Topology"中也提到了.
- Numa架构可能引起swap insanity.需要注意．[参看](http://sohulinux.blog.sohu.com/181968823.html)


有了以上信息,进行cpu绑定也应该有底了吧.

# 参考文章

1. [SMP、NUMA、MPP体系结构介绍](http://www.cnblogs.com/yubo/archive/2010/04/23/1718810.html)
2. [玩转CPU Topology](http://www.searchtb.com/2012/12/玩转cpu-topology.html)

# 变更记录

|Why | Who | When |
|----|-----|------|
|创建|fishcired|2014-07-28 |
|添加cpu topology图|fishcired|2015-01-04 |
|整理发布|fishcired|2015-01-09 |
|修正错别字|fishcired|2015-01-14 |
|修正错别字|fishcired|2016-03-28 |
