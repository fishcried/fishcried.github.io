---
layout: post
title: "OpenStack Juno版本ironic服务原理篇"
description: ""
category: OpenStack
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [openstack,ironic]
---

> 最近看了一下ironic,挺诱人的一个项目.I版以前叫baremetal与nova代码很合再一起,后期独立成ironic项目,J版发布了一个driver.翻看了一下,做个笔记,主要是ironic的原理.

# 1.What is Ironic

> Ironic is an OpenStack project which provisions bare metal (as opposed to virtual) machines by leveraging common technologies such as PXE boot and IPMI to cover a wide range of hardware, while supporting pluggable drivers to allow vendor-specific functionality to be added.
>
> If one thinks of traditional hypervisor functionality (eg, creating a VM, enumerating virtual devices, managing the power state, loading an OS onto the VM, and so on), then Ironic may be thought of as a hypervisor API gluing together multiple drivers, each of which implement some portion of that functionality with respect to physical hardware

**Why Provision Bare Metal**

- High-performance computing clusters
- Computing tasks that require access to hardware devices which can't be virtualized
- Database hosting(some databases run poorly in a hypervisor)
- Single tenant, dedicated hardware for performance, security, dependability and other regulatory requirements.
- Or, rapidly deploying a cloud infrastructure.

# 2.关键技术

像openstack管理虚拟机一样管理物理机,不得不赞叹的这个思路很赞.那么实现起来到底难不难那,或者说问题点在哪里,凭感觉:

1. 物理机的管理与虚拟管理体系融合,最起码操作上不能是两套东西,这样就没有意义了.
2. 初始期,物理机如何自动加电? (IPMI能解决)
3. 物理机初始系统如何安装? (装什么系统必须是可以选择的,理想是使用glance管理系统镜像.安装方式可以用PXE方式)

技术解决上,很多成熟的东西都有.ironic关键是粘合他们,并且与openstack融合.

# 3.ironic构成

- API service.
  只允许管理员访问的REST　API.
- Conductor service
  实现核心功能,是API后端实现.与API通过进行RPC通信.
- 消息队列
- Database & DB api.
  存储
- 用于部署的Ramdisk([diskimage-builder](https://github.com/openstack/diskimage-builder))

# 4.架构

**1 官方的ironic架构图**

![conceptual architecture](/img/ironic_conceptual_architecture.png)

架构图还是很清晰的,通过baremetal驱动,物理机网络使用neutron,系统镜像使用glance,volume挂载使用了cinder.真的与openstack融合了.

**2 官方的ironic逻辑架构**

![logical architecutre](/img/ironic_logical_architecture.png)

用户通过nova api像启动一个实例,scheduler选举出合适的compute节点,然后该compute请求ironic api,真正的处理有ironic conductor来做.像neutron申请网络,到glance获取image,通过cinder获取volume,然后操作物理机.针对不同的屋里机,需要厂商提供ironic的driver.

看到这个图的时候,我有几个疑问:

1. Nova api启动实例,如何分辨是虚机还是物理机
2. Scheduler如何调度nova compute节点,nova compute还能正常使用么?


# 5. 部署一台物理机的流程

**ironic管理硬件逻辑**

![部署](/img/ironic_deployment_architecture.png)

部署ironic需要一些必要的前置准备工作:

- ironc-conductor的节点上需要安装tftp-server,ipmi,syslinux等工具
- 使用硬件的flavor需要被创建.nova必须知道flavor从哪里启动.
- glance上需要有必要的镜像.
  - bm-deploy-kernel
  - bm-deploy-ramdisk
  - user-image
  - user-image-vmlinuz
  - user-image-initrd
- 硬件已经注册

![baremetal_deployment_steps](/img/ironic_deployment_steps.png)

过程:

1. 通过Nova API创建实例,请求通过消息队列发送到Nova scheduler.
2. Nova scheduler调用filter,找到合适的计算节点.Scheduler的过程使用了flavor的附带信息作为选择条件如"cpu_arch,baremetal:deploy_kernel_id,baremetal:deploy_ramdisk_id'.解决了我的疑问,boot instance如何区分是虚机还是物理机,通过特定flavor.
3. Compute Driver生成一个task(含有必要的启动信息)
4. 检索baremtal node信息
5. ironic Conductor向glance获取镜像
6. 获取VIF
7. Nova's ironic驱动向ironic api发送部署请求
8. PXE driver prepares tftp bootloader.
9. IPMI驱动开启网络启动并加电
10. 通过pxe安装系统
11. 重启机器
12. 更新bare node状态

# 7. 疑问解答

1. Nova api启动实例,如何分辨是虚机还是物理机
  通过flavor来区分,flavor需要附带特殊信息.
2. Scheduler如何调度nova compute节点,nova compute还能正常使用么?
  通过flavor提供的信息来进行调度.nova compute还能正常使用,可以查看源码.

目前通过devstack安装ironic,初步使用了一下.后续会进行真正的环境部署来试验.当前的[ironic安装文档](http://docs.openstack.org/developer/ironic/deploy/install-guide.html)还不完善,坑挺多的.

# 8. 参考文档

1. [ironic开发文档](http://docs.openstack.org/developer/ironic/)
2. [ironic简介](http://docs.openstack.org/developer/ironic/deploy/user-guide.html)

# 变更记录

|Why | Who | When |
|----|-----|------|
|create|fishcired|2014-11-12 |
|整理笔记,完善文档|fishcired|2014-11-28  |
