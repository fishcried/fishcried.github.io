---
layout: post
title: "Linux下Vlan配置"
description: ""
category: Linux系统管理
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [linux,vlan]
---


# linux下配置VLAN

**一些知识点**

| 交换机端口模式| 用途 |
| -- | -- |
| access | 用于连接计算机端.接收到不带vlan tag的包则打native id，然后转发；发送时则剥离掉vlan tag |
| trunk | 用于交换机之间的互联。接收包时，若包有vlan tag则判断是否允许该vlan id进入，进而转发或丢弃；没有vlan tag的包则打native vlan id然后交换转发 |
| hybird | 混合了上面两种模式。接受时同trunk，发送时判断端口属性，untag则剥离tag，如果是tag则直接发送|




# linux下vlan配置

**1. 查看内核是否支持vlan功能(内核模块为8021q)**

```
root@l-all-3:~# modinfo 8021q
filename:       /lib/modules/3.13.0-24-generic/kernel/net/8021q/8021q.ko
version:        1.8
license:        GPL
alias:          rtnl-link-vlan
srcversion:     EF43AFFE12EA6E25FAC0538
depends:        mrp,garp
intree:         Y
vermagic:       3.13.0-24-generic SMP mod_unload modversions 
signer:         Magrathea: Glacier signing key
sig_key:        00:A5:A6:57:59:DE:47:4B:C5:C4:31:20:88:0C:1B:94:A5:39:F4:31
sig_hashalgo:   sha512
```

让系统开机默认加载8201q模块

```
# echo "modprobe 8021q" >  /etc/rc.local
# echo "8021q" >> /etc/modules
```

**查看vlan配置**

```
root@l-all-3:~# cat  /proc/net/vlan/config
VLAN Dev name    | VLAN ID
Name-Type: VLAN_NAME_TYPE_PLUS_VID_NO_PAD
vlan1000       | 1000  | eth4
```

## 方法一:通过vlan命令配置

```
# apt-get install vlan
# modprobe 8021q
# vconfig add eth0 100
# vconfig add eth0 200
```

然后对vlan100和vlan200配置ip就可以了.


删除

```
# vconfig rem vlan100
# vconfig rem vlan200
```

## 方法二:修改配置文件方式

```
root@l-all-1:~# cat /etc/network/interfaces
...
auto vlan1000
iface vlan1000 inet static
    address 192.168.1.20
    netmask 255.255.255.0
    vlan-raw_device eth4
...
root@l-all-1:~# ifup vlan1000
```
