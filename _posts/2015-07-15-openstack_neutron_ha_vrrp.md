---
layout: post
title: "Neutorn通过VRRP实现l3的HA"
description: ""
category: 云计算
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [openstack, neutron, vrrp]
---

L3 agent是Neutron网络中南北东西流量的必经之地,所以L3 agent可靠性在生产环境中至关重要.

- juno版之前,l3 agent没有提供内置的ha功能.虽然可以使用多个l3 agent,但是当一个down掉,其上的路由都会丢失.想要提供ha,只能通过pacemaker或者脚本的方式.但是切换是会造成服务中断,虚拟路由越多,中断的时间越长.
- juno版提供了内置的ha功能,使用vrrp协议在协议层解决高可用问题,但是没有解决性能问题.一下相关的spec和bp链接.
  - [Neutron spec:l3-high-availability](http://specs.openstack.org/openstack/neutron-specs/specs/juno/l3-high-availability.html)
  - [Blueprint:Add High Availability Features on l3 agent ](https://blueprints.launchpad.net/neutron/+spec/l3-high-availability)

> 关于Neutron L3 Agent HA讨论有一篇很好的文章:[原文](http://assafmuller.com/2014/08/16/layer-3-high-availability/), [译文](http://blog.csdn.net/matt_mao/article/details/38676873).

# What's VRRP?

虚拟路由冗余协议VRRP（Virtual Router Redundancy Protocol）通过把几台路由设备联合组成一台虚拟的路由设备，使用一定的机制保证当主机的下一跳路由器出现故障时，及时将业务切换到备份路由器，从而保持业务的连续性和可靠性。


使用VRRP的优势在于：既不需要改变组网情况，也不需要在主机上配置任何动态路由或者路由发现协议，就可以获得更高可靠性的缺省路由。

**VRRP的工作原理**

![](/img/vrrp01.png)

RouterA、RouterB和RouterC属于同一个VRRP组，组成一个虚拟路由器，这个虚拟路由器有自己的IP地址10.110.10.1。虚拟IP地址可以直接指定，也可以借用该VRRP组所包含的路由器上某接口地址。

- 物理路由器RouterA、RouterB和RouterC的实际IP地址分别是10.110.10.5、10.110.10.6和10.110.10.7。
- 局域网内的主机只需要将缺省路由设为10.110.10.1即可，无需知道具体路由器上的接口地址。

主机利用该虚拟网关与外部网络通信。路由器工作机制如下：

- 根据优先级的大小挑选Master设备。Master的选举有两种方法：
  - 比较优先级的大小，优先级高者当选为Master。
  - 当两台优先级相同的路由器同时竞争Master时，比较接口IP地址大小。接口地址大者当选为Master。
- 其它路由器作为备份路由器，随时监听Master的状态。
  - 当主路由器正常工作时，它会每隔一段时间（Advertisement_Interval）发送一个VRRP组播报文，以通知组内的备份路由器，主路由器处于正常工作状态。
  - 当组内的备份路由器一段时间（Master_Down_Interval）内没有接收到来自主路由器的报文，则将自己转为主路由器。一个VRRP组里有多台备份路由器时，短时间内可能产生多个Master，此时，路由器将会将收到的VRRP报文中的优先级与本地优先级做比较。从而选取优先级高的设备做Master。


更多的vrrp介绍可参看[VRRP技术白皮书](http://www.h3c.com.cn/Products___Technology/Technology/Dependability/Other_technology/Technology_book/200802/335873_30003_0.htm),或者[RFC3768](http://tools.ietf.org/html/rfc3768).

**What's Keepalived?**

Keepalived是一个基于VRRP协议来实现的WEB服务高可用方案，可以利用其来避免单点故障。一个WEB服务至少会有2台服务器运行Keepalived，一台为主服务器（MASTER），一台为备份服务器（BACKUP），但是对外表现为一个虚拟IP，主服务器会发送特定的消息给备份服务器，当备份服务器收不到这个消息的时候，即主服务器宕机的时候，备份服务器就会接管虚拟IP，继续提供服务，从而保证了高可用性。


# L3 Agent HA

**实现**

如果创建的路由启用了HA功能(只有管理员可以创建),那么在每个网络节点都会创建该路由的命名空间并启动keepalived进程,不同的router之间通过HA专用网络通信.路由命名空间会有ha,qg,qr网口.ha就是专用的vrrp的控制网口.只有master的qg,gr有ip,backup的qg,gr网口都不会有ip.

![](/img/l3_agent_vrrp.png)

**安装 & 测试**

kilo上使用l3 agent ha非常简单.配置多个网络节点后,`/etc/neutron/neutron.conf`内的`l3_ha`为`true`即可,其他不用配置.juno版相关配置好像在l3_agent.ini中.

    # =========== items for l3 extension ==============
    # Enable high availability for virtual routers.
        l3_ha = True
    #
    # Maximum number of l3 agents which a HA router will be scheduled on. If it
    # is set to 0 the router will be scheduled on every agent.
    #max_l3_agents_per_router = 3
    #
    # Minimum number of l3 agents which a HA router will be scheduled on. The
    # default value is 2.
    #min_l3_agents_per_router = 2
    #
    # CIDR of the administrative network if HA mode is enabled
    # l3_ha_net_cidr = 169.254.192.0/18
    # =========== end of items for l3 extension =======

l3_ha_net_cidr定义的是ha专用的管理网段.

下面是验证步骤:

1 kilo版portal创建router时可以选择是否支持ha.将horizon配置文件/etc/openstack-dashboard/local_settings.py中的enable_ha_router改为true,然后重启apache服务.在horizon上创建HA路由.并记录该路由的id.

![](/img/vrrp02.png)

2 到网络节点`/var/lib/neutron/ha_confs/$router_id/`查看state文件,该文件为节点的状态,或者执行`ip net exec qrouter-xxx ifconfig`查看网口信息.找到master节点,然后手动关闭命名空间下的ha网口,然后测试网络是否依旧通.

下面为master节点的网口配置:

    root@kilo-network-2:~# ip netns exec qrouter-fc2bb98b-42af-46cd-9566-17658cf424d6 ifconfig
    ha-f4a66796-58 Link encap:Ethernet  HWaddr fa:16:3e:09:61:de
              inet addr:169.254.192.18  Bcast:169.254.255.255  Mask:255.255.192.0
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              RX packets:1356 errors:0 dropped:0 overruns:0 frame:0
              TX packets:172647 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:0
              RX bytes:75936 (75.9 KB)  TX bytes:9322338 (9.3 MB)

    lo        Link encap:Local Loopback
              inet addr:127.0.0.1  Mask:255.0.0.0
              UP LOOPBACK RUNNING  MTU:65536  Metric:1
              RX packets:0 errors:0 dropped:0 overruns:0 frame:0
              TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:0
              RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

    qg-b49711fc-73 Link encap:Ethernet  HWaddr fa:16:3e:33:89:f2
              inet addr:192.168.252.220  Bcast:0.0.0.0  Mask:255.255.255.0
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              RX packets:403826 errors:0 dropped:0 overruns:0 frame:0
              TX packets:317151 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:0
              RX bytes:62033724 (62.0 MB)  TX bytes:50201098 (50.2 MB)

    qr-628cc539-6b Link encap:Ethernet  HWaddr fa:16:3e:f2:1d:1b
              inet addr:192.168.34.1  Bcast:0.0.0.0  Mask:255.255.255.0
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              RX packets:302597 errors:0 dropped:0 overruns:0 frame:0
              TX packets:301586 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:0
              RX bytes:29300566 (29.3 MB)  TX bytes:29084716 (29.0 MB)


下面是backup节点的.可以看出来backup节点的qr,qg都是没有ip配置的.

    ha-bcfdf132-fd Link encap:Ethernet  HWaddr fa:16:3e:43:11:0b
              inet addr:169.254.192.17  Bcast:169.254.255.255  Mask:255.255.192.0
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              RX packets:172690 errors:0 dropped:0 overruns:0 frame:0
              TX packets:1358 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:0
              RX bytes:9670640 (9.6 MB)  TX bytes:72852 (72.8 KB)

    lo        Link encap:Local Loopback
              inet addr:127.0.0.1  Mask:255.0.0.0
              UP LOOPBACK RUNNING  MTU:65536  Metric:1
              RX packets:0 errors:0 dropped:0 overruns:0 frame:0
              TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:0
              RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

    qg-b49711fc-73 Link encap:Ethernet  HWaddr fa:16:3e:33:89:f2
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              RX packets:87589 errors:0 dropped:0 overruns:0 frame:0
              TX packets:572 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:0
              RX bytes:11609481 (11.6 MB)  TX bytes:53872 (53.8 KB)

    qr-628cc539-6b Link encap:Ethernet  HWaddr fa:16:3e:f2:1d:1b
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              RX packets:2737 errors:0 dropped:0 overruns:0 frame:0
              TX packets:2186 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:0
              RX bytes:262846 (262.8 KB)  TX bytes:207504 (207.5 KB)

# 参考文档

- [Wiki:L3_High_Availability_VRRP](https://wiki.openstack.org/wiki/Neutron/L3_High_Availability_VRRP)
- [VRRP技术白皮书](http://www.h3c.com.cn/Products___Technology/Technology/Dependability/Other_technology/Technology_book/200802/335873_30003_0.htm)
- [Neutron spec:l3-high-availability](http://specs.openstack.org/openstack/neutron-specs/specs/juno/l3-high-availability.html)
- [Blueprint:Add High Availability Features on l3 agent ](https://blueprints.launchpad.net/neutron/+spec/l3-high-availability)

# 变更记录

|Why | Who | When |
|----|-----|------|
|创建文件,提醒日后完善内容|fishcired|2015-07-02 |
|提纲 | fishcired|2015-07-13|
|完善内容 | fishcired|2015-07-15 |
|修改错别字 | fishcired|2015-07-16  |
|添加相应段落的截图 | fishcired|2015-07-17 |
