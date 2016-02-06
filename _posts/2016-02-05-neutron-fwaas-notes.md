---
layout: post
title: "Neutron防火墙(FWaas)"
description: ""
category: 云计算
subtitle:
author: "fishcried"
header-img: "img/neutron-fwaas-bg.jpg"
tags: [openstack,neutron,fwaas,防火墙]
---

Neutron早就实现了Fwaas插件，提供防火墙的基本功能。通过与路由的绑定，能够保护用户的网络安全。

# 防火墙与安全组

![](/img/neutron-fwaas-secgrp.png)

通过上图可以看到:

1. 安全组作用于vm，在compute节点的qbr上实现。
1. 防火墙作用于router，进而控制网络。在网络节点的路由命名空间内通过iptables实现

# 防火墙的实现原理

![](/img/neutron-fwaas-iptables.png)

**安全组的实现非常简单，默认使用的是linux系统的iptables，在指定路由的命名空间上设置相应的策略.**

上图中可以看到，当创建一个防火墙后，会在相应的路由命名空间添加三个链：neutron-vpn-agent-Iv4xxx,neutron-vpn-agent-ov4xxx,neutron-agent-fwaas-default.

1. 前两个链，一个作用于入口流量，一个作用于出口流量，i代表ingress，o代表egress，后面的v4是协议的版本，xxx其实是防火墙id。
1. neutron-vpn-agent-Iv4xxx,neutron-vpn-agent-Ov4xxx这两个链默认会有两条规则,这两条规则非常有用。这里的状态不是TCP连接的状态而是Iptables的概念，可以google之.
1. default链内只有一条规则，就是丢弃。

这样整个流程就是ingress流量会先经过neutron-vpn-agent-iv4xxx,进行匹配如果没有匹配，最后走default链,drop。egress流量先匹配neutron-vpn-agent-ov4xxx,如果没有匹配则走default链，drop掉.

# API说明

当前的Firewall-as-a-Service(FWaas) API Version为2.0,可实现:

1. 对租户网络上下行流量应用防火墙规则。
1. 支持TCP，UDP，ICMP或者其他协议。
1. 可以创建或者共享防火墙策略(有序的防火墙规则集).
1. 支持防火墙规则和策略的审计功能.

API操作的主要的对象为firewalls,firewall_policies,firewall_rules.

| 资源           | 说明                                                                                 |
|----------------+--------------------------------------------------------------------------------------|
|firewall        | 租户可实例化并进行管理的逻辑防火墙。一个firewall只能有一个防火墙策略。               |
|firewall_policy | 有序的防火墙规则集合。                                                               |
|firewall_rule   |具体的防火墙规则，可以根据端口，ip地址等属性进行条件定义并制定具体动作(如allow,deny等)|

数据库中也是对应的三个表，ER图:

![](/img/neutron-fwaas-ER.png)

关于API的具体说明可以阅读官方文档http://developer.openstack.org/api-ref-networking-v2-ext.html.

**API 2.0增强**

M版社区有个specs "[Firewall as Service API 2.0][http://specs.openstack.org/openstack/neutron-specs/specs/mitaka/fwaas-api-2.0.html]"，打算对防火墙API进行增强（安全组也进行了增强).如:

1. 控制粒度调整到neutron ports,不单单是路由集.
1. 如果使用dvr可以控制东西流量.
1. 引入防火墙组，地址组等新对象.
1. ...

关键是M版没有相关代码实现，release已经推迟到了N版，日后具体如何还是未知。

# 防火墙的安装部署

**Enable the Fwaas plugin-in**

    # /etc/neutron/neutron.conf
    service_plugins = firewall

    [service_providers]
    service_provider = FIREWALL:Iptables:neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver:default

    # /etc/neutron/fwaas_driver.ini
    [fwaas]
    driver = neutron_fwaas.services.firewall.drivers.linux.iptables_fwaas.IptablesFwaasDriver
    enabled = True

**Create the required tables in database**

    neutron-db-manager --service fwaas upgrade head

**Enable the option in the local_settings.py of horizon**

    OPENSTACK_NEUTRON_NETWORK = {
        ...
        'enable_firewall' = True,
        ...
    }

**Retart the neutron-l3-agent and neutron-server services**

    #Network Node
    service neutron-l3-agent
    #Controller Node
    service neutron-service
