---
layout: post
title: "OpenStack disable ipv6"
description: ""
category: 云计算
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [ipv6,openstack]
---

openstack ipv6功能的历史,可以查看这篇文章,[Journey of IPv6 in OpenStack](http://techs.enovance.com/7199/journey-of-ipv6-in-openstack). 

I版时openstack对ipv6只是有限支持.而且默认是关闭的,所以不用任何配置.虽然openstack没有开启ipv6,但是创建的vm由于加载了ipv6模块,所有能够看到ipv6地址.


宿主机不需要ipv6,所以禁用掉.配置很简单,且有多种方式.


**在`sysctl.conf`中禁用**

    #tail -f /etc/sysctl.conf
    ...
    net.ipv6.conf.all.disable_ipv6 = 1
    net.ipv6.conf.default.disable_ipv6 = 1
    net.ipv6.conf.lo.disable_ipv6 = 1

    #sysctl -p

这种方式的好处是不用重启.

**修改kernel启动参数,禁用ipv6**

以下使用的是grub2.

1. 在`/etc/default/grub`中修改修改内核启动参数,在里面添加`ipv6.disable=1`即可.如`GRUB_CMDLINE_LINUX_DEFAULT="xxxx ipv6.disable=1"`
1. 然后更新grub配置. `update-grub2`
1. reboot
1. 验证. `ifconfig`/`ip addr`/`netstt -tnl`都可以.

# More

1. [IPv6 archlinux wiki](https://wiki.archlinux.org/index.php/IPv6_(简体中文)#.E7.A6.81.E7.94.A8_IPv6)
1. [Journey of IPv6 in OpenStack](http://techs.enovance.com/7199/journey-of-ipv6-in-openstack)
1. [Neutron/IPv6](https://wiki.openstack.org/wiki/Neutron/IPv6)
1. [Exploring OpenStack IPv6](https://wiki.opnfv.org/ipv6_opnfv_project/openstack_ipv6)

# 变更记录

|Why | Who | When |
|----|-----|------|
|创建文件,提醒日后完善内容|fishcired|2015-05-13 |
|记录禁用方式,并测试|fishcired|2015-07-23 |
