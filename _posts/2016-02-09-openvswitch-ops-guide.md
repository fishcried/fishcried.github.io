---
layout: post
title: "OpenvSwitch使用指南"
subtitle: "OVS常用操作总结"
description: "对openvswitch日常使用的一次系统总结."
category: Openstack
author: "fishcried"
header-img: "img/openvswitch-ops-guide-bg.jpg"
tags: [neutron,openvswitch,ovs,openstack,sdn]
---

有一段时间，不太敢敲ovs相关命令,太复杂了，莫名恐惧。时间久了，发现也没啥，看手册都免了。下面总结一下OVS相关知识点，自我巩固一下，内容主要来自官方手册和一些openstack峰会的ppt。

# What is OpenvSwitch?

![](/img/openvswitch-overview.png)

官方解释:

> "OpenvSwitch is a production quality, multilayer virtual switch licensed under the open source Apache 2.0 license.
It is designed to enable massive network automation through programmatic extension,
while still supporting standard management interfaces and protocols (e.g. NetFlow, sFlow, SPAN, RSPAN, CLI, LACP, 802.1ag).
In addition, it is designed to support distribution across multiple physical servers similar to
VMware's vNetwork distributed vswitch or Cisco's Nexus 1000V."

简单来说,openvswitch就是软件交换机。

![](/img/openvswitch-usage-case.jpg)

OVS在网络虚拟化中扮演着重要的角色,尤其在云计算时代,OVS正是大展身手之时。OpenStack的Neutron项目将OVS作为基石，ovs演化出了专门为openstack服务的ovn项目.

# OpenvSwitch的架构与基本概念

## OVS构成

ovs的构成非常简单,每个部件负责各自的职责.

![](/img/openvswitch-arch.png)

| ovs-vswtiched | openvswitch的守护进程.                                    |
| ovsdb-server  | openvswitch的数据库服务,保存相关配置信息，非常轻量级 |
| kernel Datapath | Datapath是流的一个缓存，会把流的match结果cache起来，避免下一次流继续到用户空间进行flow match |
| controller    | 这个不是ovs自身的部件,而是一个抽象的概念，指的是ovs的控制者 |

## 基本概念

![](/img/openvswitch-contents.png)

**Packet**

  网络转发的最小数据单元，每个包都来自某个端口，最终会被发往一个或多个目标端口，转发数据包的过程就是网络的唯一功能。

**Bridge**

  OpenvSwitch中的网桥对应物理交换机，其功能是根据一定流规则，把从端口收到的数据包转发到另一个或多个端口。

**Port**

  端口是收发数据包的单元。OpenvSwitch中，每个端口都属于一个特定的网桥。端口收到的数据包会经过流规则的处理，发往其他端口；也会把其他端口来的数据包发送出去.主要有

| 类型    | 说明 |
|---------+------|
| Normal  | 用户可以把操作系统中的网卡绑定到ovs上，ovs会生成一个普通端口处理这块网卡进出的数据包。 |
| Internal| 端口类型为internal时，ovs会创建一块虚拟网卡，端口收到的所有数据包都会交给该网卡，发出的包会通过该端口交给ovs。当ovs创建一个新网桥时，默认会创建一个与网桥同名的Internal Port.|
| Patch   | 当机器中有多个ovs网桥时，可以使用Patch Port把两个网桥连起来。Patch Port总是成对出现，分别连接在两个网桥上，在两个网桥之间交换数据。|
| Tunnel  | 隧道端口是一种虚拟端口，支持使用gre或vxlan等隧道技术与位于网络上其他位置的远程端口通讯。|

**Interface**

  接口是ovs与外部交换数据包的组件。一个接口就是操作系统的一块网卡，这块网卡可能是ovs生成的虚拟网卡，也可能是物理网卡挂载在ovs上，也可能是操作系统的虚拟网卡（TUN/TAP）挂载在ovs上。

**FlowTable**

  流定义了端口之间数据包的交换规则.下面会对FlowTable做了详细的说明.

## 流表(FlowTable)

**流表定义**

流表是交换机进行转发策略控制的核心数据结构。交换机芯片通过查找流表表项来决策进入交换机网络流量采取合适的行为. 流表中包括一些流表项，每个表项包含若干个域:包头域，活动计数器,0个或多个执行行动.

![](/img/openflow-v1.0.png)

1. 包头域: 执行规则的条件，主要是数据包各层协议的指定,当数据包满足该条件时进行匹配.
2. 计数器: 用来统计流量的一些信息.比如存活时间,错误，发包数等.
3. 行动: 定义了对匹配规则的包的处理方式.可以drop,转发,修改等.

上面是openflow v1.0对flowtable的定义，openflow目前已经发布多个版本，之间会有所差别，但是整体上是一致的的。

**流表的匹配**

![](/img/openvswitch-openflow-match.png)

1. 按照从小到大的顺序匹配流表
2. 同表内的表项按照优先级顺序匹配

下面简单查看下流表规则(neutron gre网络br-tun流表)，体会一下就会明白。

    # ovs-ofctl dump-flows br-tun
    NXST_FLOW reply (xid=0x4):
      # 从port1进来的包转到表1处理
      cookie=0x0, duration=10970.064s, table=0, n_packets=189, n_bytes=16232, idle_age=16, priority=1,in_port=1 actions=resubmit(,1)
      # 从port2进来的包转到表2处理
      cookie=0x0, duration=10906.954s, table=0, n_packets=29, n_bytes=5736, idle_age=16, priority=1,in_port=2 actions=resubmit(,2)
      # 不匹配上面两条则drop
      cookie=0x0, duration=10969.922s, table=0, n_packets=3, n_bytes=230, idle_age=10962, priority=0 actions=drop
      # 表1，单播包转到表20处理
      cookie=0x0, duration=10969.777s, table=1, n_packets=26, n_bytes=5266, idle_age=16, priority=0,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,20)
      # 多播包转到表21处理
      cookie=0x0, duration=10969.631s, table=1, n_packets=163, n_bytes=10966, idle_age=21, priority=0,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,21)
      # 表2，port2进来的包在这里处理了.同样是转给表10处理
      cookie=0x0, duration=688.456s, table=2, n_packets=29, n_bytes=5736, idle_age=16, priority=1,tun_id=0x1 actions=mod_vlan_vid:1,resubmit(,10)
      # 表10，进行规则学习,具体就不解释了。学习到的规则后续会给表20来使用
      cookie=0x0, duration=10969.2s, table=10, n_packets=29, n_bytes=5736, idle_age=16, priority=1 actions=learn(table=20,hard_timeout=300,priority=1,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:NXM_OF_IN_PORT[]),output:1
      # 表20, 根据目的mac设置tun_id,通过指定的port发出去
      cookie=0x0, duration=682.603s, table=20, n_packets=26, n_bytes=5266, hard_timeout=300, idle_age=16, hard_age=16, priority=1,vlan_tci=0x0001/0x0fff,dl_dst=fa:16:3e:32:0d:db actions=load:0->NXM_OF_VLAN_TCI[],load:0x1->NXM_NX_TUN_ID[],output:2
      # 无规则的交给表21处理
      cookie=0x0, duration=10969.057s, table=20, n_packets=0, n_bytes=0, idle_age=10969, priority=0 actions=resubmit(,21)
      # 表21，根据vlan找到对应的出去的口
      cookie=0x0, duration=688.6s, table=21, n_packets=161, n_bytes=10818, idle_age=21, priority=1,dl_vlan=1 actions=strip_vlan,set_tunnel:0x1,output:2
      # drop
      cookie=0x0, duration=10968.912s, table=21, n_packets=2, n_bytes=148, idle_age=689, priority=0 actions=drop

简单查看以上规则，明白个大概就能对流表有个理解.

## `ovs-*`相关命令快速指南

ovs命令行挺复杂的，提供多个命令，每个命令又有多个不同的子命令，刚刚接触会觉得一团糟，但仔细理解ovs架构和基本概念后会发现命令使用起来非常简单.

1. ovs对每个组件都会单独提供一个命令行,所以命令行的功能都非常的专一,理解了组件的用途，也就记住了命令行的用途和大概的子命令.
2. 命令行的子命令其实就是对数据库的CURD，而数据库表其实就是OVS的核心概念,如Bridge,Ports,FlowTables等.

![](/img/openvswitch-details.png)

这是一张详细一点的架构图.颜色深一点的线是数据包的处理过程,而浅颜色的是Management Workflow.可以很清晰的看到每个组件都会有专门的client与其进行交互.

**client与组件对应关系**

| ovs-dpctl | datapath控制器,可以创建删除DP,控制DP中的FlowTables,最常使用`show`命令，其他很少手动操作 |
| ovs-ofctl | 流表控制器，控制bridge上的流表，查看端口统计信息等 |
| ovsdb-tool | 专门管理ovsdb的client |
| ovs-vsctl | 最常用的命令,通过操作ovsdb去管理相关的bridge,ports什么的 |
| ovs-appctl | 这个可以直接与openvswitch daemon进行交互,上图中没有列出来,这么命令较少使用 |

**常用子命令说明**

- ovs-dpctl `show -s`
- ovs-ofctl `show, dump-ports, dump-flows, add-flow, mod-flows, del-flows`
- ovsdb-tools `show-log -m`
- ovs-vsctl
  - show 显示数据库内容
  - 关于桥的操作 `add-br, list-br, del-br, br-exists`.
  - 关于port的操作 `list-ports, add-port, del-port, add-bond, port-to-br`.
  - 关于interface的操作 `list-ifaces, iface-to-br`
  - `ovs-vsctl list/set/get/add/remove/clear/destroy table record column [value]`,
    常见的表有bridge, controller,interface,mirror,netflow,open_vswitch,port,qos,queue,ssl,sflow.
- ovs-appctl `list-commands, fdb/show, qos/show`

# OpenvSwitch操作实例

## 日常操作

**查看bridge,ports,interfaces以及相互之间的对应关系**

`ovs-vsctl show`查看整体信息,这个命令会把相关信息都列出来，信息量大的时候就不太方便了.

    root@l-network-1:~# ovs-vsctl show
        Bridge br-tun
            fail_mode: secure
            Port "vxlan-ac1c050d"
                Interface "vxlan-ac1c050d"
                    type: vxlan
                    options: {df_default="true", in_key=flow, local_ip="172.28.5.14", out_key=flow, remote_ip="172.28.5.13"}
            Port "vxlan-ac1c053f"
                Interface "vxlan-ac1c053f"
                    type: vxlan
                    options: {df_default="true", in_key=flow, local_ip="172.28.5.14", out_key=flow, remote_ip="172.28.5.63"}

查看有哪些桥，桥中有哪些ports，哪些interfaces

    root@l-network-1:~# ovs-vsctl list-br
    br-ex
    br-int
    br-tun

    root@l-network-1:~# ovs-vsctl list-ports br-tun
    patch-int
    vxlan-ac1c0509
    vxlan-ac1c050d
    vxlan-ac1c051c
    vxlan-ac1c053f

    root@l-network-1:~# ovs-vsctl list-ifaces br-tun
    patch-int
    vxlan-ac1c0509
    vxlan-ac1c050d
    vxlan-ac1c051c
    vxlan-ac1c053f
    # iface与ports同名.

查看port,interface属于哪个bridge,xxx-to-br即可.

    root@l-network-1:~# ovs-vsctl port-to-br vxlan-ac1c0509
    br-tun
    root@l-network-1:~# ovs-vsctl iface-to-br vxlan-ac1c0509
    br-tun

**查看bridge中的流表**

查看某个bridge的flowtables

    root@l-network-1:# ovs-ofctl dump-flows br-ex
    NXST_FLOW reply (xid=0x4):
     cookie=0x0, duration=6067267.035s, table=0, n_packets=1313858912, n_bytes=881486996325, idle_age=0, hard_age=65534, priority=0 actions=NORMAL

流表规则中往往使用portid来指定相关的port，可以使用`show`命令来对应port name与port id.

    root@l-network-1:# ovs-ofctl show br-ex
    OFPT_FEATURES_REPLY (xid=0x2): dpid:00002c44fd8a32ce
    n_tables:254, n_buffers:256
    capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
    actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
     1(qg-e275cd74-7d): addr:fa:16:3e:6b:0b:f3
         config:     PORT_DOWN
         state:      LINK_DOWN
         speed: 0 Mbps now, 0 Mbps max
     2(qg-7b2a2e0d-c0): addr:fa:16:3e:45:c7:c5
         config:     PORT_DOWN
         state:      LINK_DOWN
         speed: 0 Mbps now, 0 Mbps max
     3(qg-73dbbd87-ed): addr:00:00:00:00:00:00
         config:     PORT_DOWN
         state:      LINK_DOWN
         speed: 0 Mbps now, 0 Mbps max

查看隐藏的流表规则，很少使用

    root@l-network-1:# ovs-appctl bridge/dump-flows br-ex
    duration=6067771s, n_packets=1313936898, n_bytes=881574100116, priority=0,actions=NORMAL
    table_id=254, duration=6067771s, n_packets=0, n_bytes=0, priority=2,recirc_id=0,actions=drop
    table_id=254, duration=6067771s, n_packets=0, n_bytes=0, priority=0,reg0=0x1,actions=controller(reason=no_match)
    table_id=254, duration=6067771s, n_packets=0, n_bytes=0, priority=0,reg0=0x2,actions=drop
    table_id=254, duration=6067771s, n_packets=0, n_bytes=0, priority=0,reg0=0x3,actions=drop

**查看一些有用的统计信息**

查看datapath统计信息.主要关注lost数值.hit表示datapath命中数,missed未命中，lost表示没有传递到用户空间就丢弃了.
主要关注lost值是否上升，如果上升说明存在问题了.该命令可以使用`-s`选项，会将每个port的统计信息也显示出来.

    root@l-network-1:# ovs-dpctl show
    system@ovs-system:
            lookups: hit:2596765540 missed:6438005 lost:0
            flows: 39
            masks: hit:19416324706 total:10 hit/pkt:7.46
            port 0: ovs-system (internal)
            port 1: tapd9a71635-53 (internal)
            port 2: tap82059645-8e (internal)
            port 3: tap3f0507b3-70 (internal)

    root@l-network-1:# ovs-dpctl show -s
    system@ovs-system:
            lookups: hit:2596836956 missed:6438236 lost:0
            flows: 42
            masks: hit:19416902233 total:10 hit/pkt:7.46
            port 0: ovs-system (internal)
                    RX packets:0 errors:0 dropped:0 overruns:0 frame:0
                    TX packets:0 errors:0 dropped:0 aborted:0 carrier:0
                    collisions:0
                    RX bytes:0  TX bytes:0
            port 1: tapd9a71635-53 (internal)
                    RX packets:4170 errors:0 dropped:0 overruns:0 frame:0
                    TX packets:1091973 errors:0 dropped:0 aborted:0 carrier:0
                    collisions:0
                    RX bytes:906944 (885.7 KiB)  TX bytes:48392325 (46.2 MiB)
            port 2: tap82059645-8e (internal)
                    RX packets:7130 errors:0 dropped:0 overruns:0 frame:0
                    TX packets:7379 errors:0 dropped:0 aborted:0 carrier:0
                    collisions:0
                    RX bytes:641578 (626.5 KiB)  TX bytes:650492 (635.2 KiB)
            port 3: tap3f0507b3-70 (internal)
                    RX packets:5505 errors:0 dropped:0 overruns:0 frame:0
                    TX packets:7549 errors:0 dropped:0 aborted:0 carrier:0

`ovs-ofctl dump-ports br [port]`也可以查看port的统计信息.这个命令优势是可以指定port.

    root@l-network-1:# ovs-ofctl dump-ports br-ex
    OFPST_PORT reply (xid=0x2): 19 ports
      port LOCAL: rx pkts=78668524, bytes=34522064349, drop=0, errs=0, frame=0, over=0, crc=0
               tx pkts=78827893, bytes=5746347600, drop=0, errs=0, coll=0
      port 10: rx pkts=14386485, bytes=10843773932, drop=0, errs=0, frame=0, over=0, crc=0
               tx pkts=15666927, bytes=10659584526, drop=0, errs=0, coll=0
      port 26: rx pkts=3830, bytes=161388, drop=0, errs=0, frame=0, over=0, crc=0
               tx pkts=3025213, bytes=244511981, drop=0, errs=0, coll=0
      port 36: rx pkts=16, bytes=1200, drop=0, errs=0, frame=0, over=0, crc=0
               tx pkts=898463, bytes=68248745, drop=0, errs=0, coll=0
      port 25: rx pkts=4693, bytes=206958, drop=0, errs=0, frame=0, over=0, crc=0
               tx pkts=3256605, bytes=264430395, drop=0, errs=0, coll=0
      port 35: rx pkts=229447, bytes=21077911, drop=0, errs=0, frame=0, over=0, crc=0
               tx pkts=1905393, bytes=758729599, drop=0, errs=0, coll=0
      port 28: rx pkts=13, bytes=1074, drop=0, errs=0, frame=0, over=0, crc=0
               tx pkts=2266685, bytes=179166266, drop=0, errs=0, coll=0
      port 23: rx pkts=10345, bytes=636977, drop=0, errs=0, frame=0, over=0, crc=0
               tx pkts=3345437, bytes=272099392, drop=0, errs=0, coll=0

查看bridge的转发表

    root@l-network-1:# ovs-appctl fdb/show br-ex
     port  VLAN  MAC                Age
       10     0  fa:16:3e:f8:28:9f    9
       11     0  fa:16:3e:a7:d2:f5    8
       34     0  fa:16:3e:2f:b2:71    8
        4     0  2c:44:fd:89:cf:3a    6
        6     0  fa:16:3e:45:c7:c5    5
        4     0  fa:16:3e:44:15:eb    5
        4     0  2c:44:fd:8a:78:06    4
        1     0  fa:16:3e:be:08:35    2
        3     0  fa:16:3e:2f:dd:71    1
        2     0  fa:16:3e:d7:f0:c1    0
    LOCAL     0  2c:44:fd:8a:32:ce    0
        4     0  20:0b:c7:37:d0:05    0

## 设置qos

    ovs-vsctl set Interface tap0 ingress_policing_rate=100000
    ovs-vsctl set Intervace tap ingress_policing_burst=10000
    ovs-appctl qos/show <iface>

## 调试技巧

**查看流规则的匹配**

    watch -d -n 1 "ovs-ofctl dump-flows <bridge>"

**查看统计信息**

    ovs-dpctl show -s/ ovs-dpctl show
    ovs-ofctl dump-ports <br> [port]

**使用tcpdump抓包,需要设置端口镜像**

    ip link add name snooper0 type dummy
    ip link set dev snooper0 up
    ovs-vsctl add-port br-int snooper0

    # ovs-vsctl -- set Bridge br-int mirrors=@m  -- --id=@snooper0 \
      get Port snooper0  -- --id=@patch-tun get Port patch-tun \
      -- --id=@m create Mirror name=mymirror select-dst-port=@patch-tun \
      select-src-port=@patch-tun output-port=@snooper0 select_all=1

    tcpdump -i snooper0

    ovs-vsctl clear Bridge br-int mirrors
    ovs-vsctl del-port br-int snooper0
    ip link delete dev snooper0


**查看日志**

    # cat /var/log/openvswitch/*.log

    # ovsdb-tool show-log -m
    record 0: "Open_vSwitch" schema, version="7.12.1", cksum="2211824403 22535"

    record 1: 2015-10-20 03:41:17.728 "ovs-vsctl: ovs-vsctl --no-wait -- init -- set Open_vSwitch . db-version=7.12.1"
            table Open_vSwitch insert row 909bb8ed:

    record 2: 2015-10-20 03:41:17.740 "ovs-vsctl: ovs-vsctl --no-wait set Open_vSwitch . ovs-version=2.4.0 "external-ids:system-id=\"26c0e58d-8754-41f0-92c5-6863a2525456\"" "system-type=\"Ubuntu\"" "system-version=\"14.04-trusty\"""
            table Open_vSwitch row 909bb8ed (909bb8ed):

    record 3: 2015-10-20 03:41:17.753
            table Open_vSwitch row 909bb8ed (909bb8ed):

    record 4: 2015-10-20 03:42:51.586 "ovs-vsctl: ovs-vsctl add-br br-ex"
            table Interface insert row "br-ex" (a8df640b):
            table Port insert row "br-ex" (a042a3da):
            table Bridge insert row "br-ex" (b465bc4d):
            table Open_vSwitch row 909bb8ed (909bb8ed):

    record 5: 2015-10-20 03:42:51.683
            table Interface row "br-ex" (a8df640b):
            table Open_vSwitch row 909bb8ed (909bb8ed):

**使用`ovs-appctl ofproto/trace <bridge> k1=v1,k2=v2`测试流匹配**

    root@l-network-1:~# ovs-appctl ofproto/trace br-ex in_port=10,dl_src=66:4e:cc:ae:4d:20,dl_dst=46:54:8a:95:dd:f8 -generate
    Bridge: br-ex
    Flow: in_port=10,vlan_tci=0x0000,dl_src=66:4e:cc:ae:4d:20,dl_dst=46:54:8a:95:dd:f8,dl_type=0x0000

    Rule: table=0 cookie=0 priority=0
    OpenFlow actions=NORMAL
    no learned MAC for destination, flooding

    Final flow: in_port=10,vlan_tci=0x0000,dl_src=66:4e:cc:ae:4d:20,dl_dst=46:54:8a:95:dd:f8,dl_type=0x0000
    Megaflow: in_port=10,vlan_tci=0x0000/0x1fff,dl_src=66:4e:cc:ae:4d:20,dl_dst=46:54:8a:95:dd:f8,dl_type=0x0000
    Datapath actions: 27,30,12,14,15,17,20,37,9,28,47,48,54,57,29,25,61,21

# 变更记录

|变更说明 | 变更人 | 时间 |
|----|-----|------|
|整理个人笔记,形成v0.1草稿| fishcired|2016-02-09 |
|v1.0 | fishcired|2016-02-17 |
