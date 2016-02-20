---
layout: post
title: "OpenStack网络基础知识: Linux中常用网络设备总结"
subtitle: "bridge,tap/tun,veth,vlan"
category: OpenStack
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [openstack,网络,neutron]
---

OpenStack的Neutron网络使用了很多的虚拟设备，比如qbrXXX网桥,而qbr网桥与集成网桥br-int之间的联接使用的是veth设备(qvbXXX/qvoXXX),
而qrouter和qdhcp网络命名空间中又创建了很多tap设备来完成相应的功能. 本文对这些常见的网络设备及其操作进行简单的总结说明，
对理解OpenStack网络和诊断问题时可能会更加顺利些。

# 网桥(Bridge)

网桥工作在数据链路层，是二层网络设备，用于将两个LAN连起来。主要靠的MAC地址学习机制。当网桥的Port收到包时会将包的源mac和port ID关联起来记入mac学习表,通过这个学习过程来完善mac表。
也就是收包时自动学习源mac,学习的目的就是转发包的时候来使用. 转发时会检索表目的地址时候在mac学习表中，如果找到就将包通过对应的port转发出去，没有就对所有其他port进行广播。过程就是这个样子。mac表可以
通过`brctl showmacs brname`来查看。当然了，网络一旦形成环路就完蛋了，为了学习mac，port会广播数据包，但是由于环路的存在，广播包在整个网络中走不出去，整个网络会很快就瘫痪。而STP协议就是为了解决这个问题.

    root@l-compute-3:~# brctl showmacs qbrc903a495-b4
    port no mac addr                is local?       ageing timer
      1     42:3e:54:7d:05:5a       yes                0.00
      2     fe:16:3e:9b:7d:69       yes                0.00

> linux中使用网桥时有个地方需要注意: 网卡加入网桥后它的ip就失效了,网桥是根据mac来进行转发的。但是网桥可以配置ip，因为网桥是个虚拟设备，可以通过ip进行管理.

**`brctl`命令**

1. 最常用的命令`brctl help`
2. 新建/删除网桥`brctl addbr/delbr <bridge>`
3. 网口操作·`brctl addif/delif <bridge> <device>`
4. 查看mac表`brctl showmacs <bridge>`
5. 查看stp协议`brctl showstp <bridge>`

# VLAN配置

VLAN不是网络设备，中文叫虚拟局域网，是一种协议。提出VLAN技术是由于当网络中物理机器多起来后，交换机地址学习时广播包非常多，常常会发生广播风暴。
而VLAN技术可以通过tag把同一局域网络进行逻辑上的分离划分,广播包只在同一VLAN中广播，不同VLAN中需要通过三层路由才能联通。这样就一定程度上隔离了广播包，而且可以在不改变物理部署的情况下
通过逻辑划分来规划网络，非常便利,同时网络安全性也提升了，所以VLAN技术非常受欢迎。OpenStack租户网络使用VLAN技术时网络性能还是非常优秀的，可以VLAN
技术存在tag数的限制，最大数为4096,也就是同一二层网络最大只能划分4096个逻辑LAN.

Linux中配置VLAN也很简单，首先必须要确保加载`8021q`模块，然后使用`vconfig`命令进行配置,查看vlan信息可以通过查看`/proc/net/vlan/*`下的文件来查看.

**linux vlan配置**

加载8021q模块

    apt-get install vlan
    modprobe 8021q

之后就可以使用`vconfig`命令来进行vlan配置了.

    vconfig add eth0 100
    ifconfig eth0.100 192.168.100.2 netmask 255.255.255.0  up
    vconfig rem eth0.100
    cat /proc/net/vlan/eth0.100

# 3. TAP/TUN设备

TAP/TUN设备是一种让用户态程序向内核协议栈注入数据的设备，一个工作在三层，一个工作在二层，使用较多的是TAP设备。

    ip tuntap add mode tap
    ip tuntap add mode tun

设备创建来后就可以像正常的物理网卡一样配置了.

# 4. VETH设备

VETH设备出现较早，它的作用是反转通讯数据的方向，需要发送的数据会被转换成需要收到的数据重新送入内核网络层进行处理，从而间接的完成数据的注入。就是完成网线的作用。
VETH设备常常用于不同的网络命名空间，就像一根网线一段插入命名空间A，另一端插入明空空间B,这样两个网络命名空间就能够联通了.

    # 创建veth设备
    ip link add name veth-in type veth peer name veth-peer

veth设备一般是成对出现的，那么怎么通过一段查看另一端那?可以通过查看`peer_ifindex`属性来确定.

    ethtool -S veth1
    root@fishcried-horse:~# ethtool -S veth-in
    NIC statistics:
         peer_ifindex: 5

    ip link 
    root@fishcried-horse:~# ip link
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
        link/ether fa:16:3e:19:f8:c8 brd ff:ff:ff:ff:ff:ff
    3: br0: <BROADCAST,MULTICAST> mtu 1400 qdisc noop state DOWN mode DEFAULT group default
        link/ether 00:00:00:00:00:00 brd ff:ff:ff:ff:ff:ff
    5: veth-peer: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
        link/ether ce:1a:44:c1:22:cf brd ff:ff:ff:ff:ff:ff
    6: veth-in: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
        link/ether b2:85:1b:77:17:cd brd ff:ff:ff:ff:ff:ff

# 修改记录

| 日期            | 说明                                             |
|-----------------+--------------------------------------------------|
| 2014-08-13      | 根据网络内容形成草稿                             |
| 2016-02-20      | 修改标题，删减内容重新整理                       |
