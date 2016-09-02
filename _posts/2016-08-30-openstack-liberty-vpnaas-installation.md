---
layout: post
title: OpenStack(Liberty) Neutron VPNaas安装配置
subtitle:
description:
category: OpenStack
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags:
    - openstack
    - vpnaas
    - neutron
---

# OpenStack Neutron VPNaas安装配置

用ansible部署vpnaas，发现没有详细的官方文档，这里简单梳理下，供日后查阅.

## 安装

| 软件　|  版本 |
|-------|-------|
| OpenStack | Liberty |

**网络节点安装vpn-agent**

```
apt-get install openswan neutron-plugin-vpn-agent
```


> 安装neutron-plugin-vpn-agent时会卸载neutron-l3-agent,后续在网络节点启动neutron-vpn-agent即可，其代码继承了l3-agent.

**配置内核网络参数**

```
cat /etc/sysctl.conf
net.ipv4.ip_forward=1
net.ipv4.conf.default.accept_redirects=0
net.ipv4.conf.default.send_redirects=0
```

**ipsec verify**

在网络节点执行`ipsec verify`进行检查看看是否满足要求.


**控制节点安装python-neutron-vpnaas**

```
apt-get install python-neutron-vpnaas
```

## 配置

**配置vpn device driver.**

```
cat /etc/neutron/vpn_agent.ini
...
[vpnagnet]
vpn_device_driver=neutron.services.vpn.device_drivers.ipsec.OpenSwanDriver
...
```

> 该文件中还可配置`[DEFAULT]`段的interface_driver,由于该可以继承l3的配置,所以可以不进行配置.

**在horizon上启用vpn plugin**

```
cat /etc/openstack-dashboard/local_settings.py
...
OPENSTACK_NEUTRON_NETWORK = {
    'enable_router': True,
    'enable_quotas': True,
    'enable_ipv6': False,
    'enable_distributed_router': False,
    'enable_ha_router': True,
    'enable_lb': True,
    'enable_firewall': True,
    'enable_vpn': True,
    'enable_fip_topology_check': True,
...
```

**在控制节点启用vpnaas plugin**

```
cat /etc/neutron/neutron.conf
...
service_plugins = ...,vpnaas
...
```

**重启服务**

- 在控制节点`service neutron-service restart`
- 在网络节点`service nturon-vpn-agent restart`


这样登陆horizon后即可使用vpn.
