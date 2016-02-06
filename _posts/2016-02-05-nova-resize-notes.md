---
layout: post
title: "OpenStack Nova Resize功能"
subtitle: "改变Flavor方式的冷迁移"
description: ""
category: openstack
author: "fishcried"
header-img: "img/nova-resize-bg.jpg"
tags: [openstack,nova,resize]
---

> 最近基于Icehouse对nova resize功能进行调试.I版不支持RBD的resize，其中合并upstream的很多patch，好费力啊!下面对resize功能做一下简单梳理，巩固一下.

# Nova Resize功能使用场景

当用户使用的虚机配置不满足当前业务，想提高性能时，可以在不改变该客户虚机ip，mac，已有数据的情况下变更虚机的配置。也就是使用更多的cpu，mem，disk等.
其实现在libvirt已经支持cpu,mem的热插拔技术，openstack也有相关的bp，不过没有被认可，一些大牛希望热插拔技术能够使大部分虚拟化技术通用，如果只有kvm支持
那他们并不愿意接受。可想而知，如果总有一天热插拔技术会替代掉resize的，或者resize改名为live resize.


# Nova Resize的实质

如副标题所示，resize其实就是冷迁移。nova resize的实质是冷迁移.将一台vm从host，迁移到host2，只不过迁移后使用了一个较大的flavor.如何查看代码也可以发现，如果不传入flavor_ref，则认为是cold migrate.

限制:

* resize时， 新flavor -> 原flavor
* resize过程中，vm会关闭，resize结束后vm会start。

# Nova Resize流程

![](/img/nova-resize-state.png)

nova resize不是单单的一个resize动作，其实还有confirm(确认),恢复(revert).

1. resize: 请求进行resize操作。
1. confirm: resize执行后,src与dest host都有vm数据，confirm后会把src host上vm清理掉。
1. revert: revert是把dest host上vm信息丢弃掉，恢复使用src的

下面一个说明一下resize过程：

![](/img/nova-resize-phase.png)

上面可以看到，resize功能主要可以分为4个阶段:

**1 调度阶段（`ComputeTaskManager.migrate_server调用self.scheduler_rpcapi.select_destinations()`）**

当horizon/cli请求resize后，nova-schedule需要选择出最佳的目的主机.如果nova配置了允许本地迁移(allow_resize_to_same_host =True)，则源主机也作为候选者。整个schedule过程分为filter和weight过程。下面是非常经典的图:

![](/img/nova-scheduler.png)

filtert阶段会根据条件去除掉不合理的主机，而weight阶段则是将剩余的主机进行权重计算，然后选择最合适的主机来作为目标(主要是剩余RAM).

**2 pre阶段(`ComputeManager.prep_resize()`)**

目的主机确定后，nova-conductor会通过rpc通知dest host做resize准备，也就是进入pre阶段。主要是暂存instance新旧状态，方便日后恢复，同时做一些参数校验工作.主要代码在self._pre_resize()中.

**3 start阶段(`ComputeManager.resize_instance()`)**

dest host准备工作做好后,会通过rpc的方式告诉src host开始进行resize。最主要的工作是src compute host调用migrate_disk_and_power_off（），将虚机的disk同步到目的compute node.主要代码在self._resize_instance().

**4 post阶段(`ComputeManager.finish_resize()`)**

这个阶段Dest Compute Node会建立intance相关的网络，Extend disk，然后使用新的flavor重现create虚机。之后会start instance.主要代码在self._finish_resize()

# Nova resize部署时配置

Nova Resize功能主要配置有两点.

1. 本地迁移.
2. compute node间nova用户免密码登陆.

**本地迁移**

如果要开启本地迁移，也就是resize的虚拟机还在原来的compute node上，则需要将nova.conf中启用allow_resize_to_same_host=True.nova-scheduler节点与compute节点都需要配置.

**Compute Node间nova用户免密码登陆**

    # 准备秘钥
    mkdir ssh
    $ ssh-keygen -t rsa
    Generating public/private rsa key pair.
    Enter file in which to save the key (/home/ubuntu/.ssh/id_rsa): ssh/id_rsa
    Enter passphrase (empty for no passphrase):
    Enter same passphrase again:
    Your identification has been saved in ssh/id_rsa.
    Your public key has been saved in ssh/id_rsa.pub.
    The key fingerprint is:
    15:a8:43:8b:23:3a:73:77:db:36:65:18:cf:d0:22:55 ubuntu@fishcried-horse
    The key's randomart image is:
    +--[ RSA 2048]----+
    |         oE      |
    |      . o  .     |
    |     o + ..      |
    |  . o = +..      |
    | . . . oSB       |
    |+ . . . . =      |
    | + . . o o       |
    |      . +        |
    |       . .       |
    +-----------------+


    ubuntu at fishcried-horse in ~/temp


    ubuntu at fishcried-horse in ~/temp
    $ tree
    .
    └── ssh
        ├── authorized_keys
        ├── config
        ├── id_rsa
        └── id_rsa.pub


    1 directory, 4 files


    cat << EOF > ssh/config
     Host * StrictHostKeyChecking no
    UserKnownHostsFile=/dev/null
    EOF


    chmod 600
    chmod 600 authorized_keys id_rsa

    # 将秘钥与相关配置同步到每一个计算节点
    scp -r ssh root@compute-node:/var/lib/nova/.ssh

    # 登陆compute节点，调整nova用户
    ssh root@compute-node
    usermod -s /bin/bash nova
    chown -R nova:nova /var/lib/nova/.ssh

    # 测试联通性

    ssh compute-node-1
