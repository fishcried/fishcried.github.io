---
layout: post
title: "Cpu bindings (二) 绑定方法总结"
description: ""
category: 性能调优
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [cpu,openstack]
---

之前的[理解cpu topology](/2015-01-09/cpu_topology)主要是一些原理学习，本文想总结下常见的绑定方式.不单单是`host Pcus`还有`guest Vcpus`.

# **`isolcpus`隔离系统**

openstack的compute节点，系统主要使用0-4核，其他的留给hypervisor.如何做？　`isolcpus`闪亮登场．系统隔离使用`isolcpus`,hypervisor的控制使用`taskset`等．

内核启动参数`isolcpus`指定系统不能使用哪些cpu.如`isolcpus=4,5,6,7,8`,表示系统启动后用户进程不能使用4-8cpu.

    isolcpus = [KNL,SMP]
        format:
            <cpu number>,...,<cpu number>
            or
            <cpu number>-<cpu number>
            or 
            <cpu number>,...,<cpu number>-<cpu number>

isolcpus的原理也很简单:通过设置进程的cpu亲和性来实现.启动时设置init进程的亲和性,后续的进程均会继承init进程的亲和性设置.这样就达到了整个系统的亲和性一致．如果后续的用户想修改亲和性可以通过`taskset`来达到目的.

**永久生效需要在grub中修改kernel的启动参数.**

# **`tasket`设置进程亲和性**

`tasket`使用非常简单,能够实时的进行cpu亲和性设置.

    # 设置亲和性 taskset -cp mask pid
    taskset -cp 0-3 1234
    # 获取亲和性 taskset -cp pid
    taskset -cp 1234

# **Cgroup进行Cpu QOS**

Cgroup能够管理cpu资源.cgroup使用了特殊的文件系统，可以像正常的文件操作一样完成cgroup的设置．也可是使用特定的命令来进行设置．具体操作可以参考[Linux的Cgroup](http://www.cnblogs.com/yjf512/p/3298582.html)

Cgroup能够控制隔离cpu资源,但是更多的是QOS功能.


# **Vcpus Bindings on KVM**

    virsh vcpupin guest1 4 0,1,2,3,8,9,10,11

主要是使用`virsh vcpupin`命令来实现，上面就是将虚机guest1的vcpu 4绑定到host的0,1,2,3,8,9,10,11上.

## **Vcpus Bindings On Openstack**

I版的时候cpu bingdings非常简单,只要设置nova的vcpu_pin_set即可,也是挺粗糙的.

    #nova.conf
    [DEFAULT]
    ...
    vcpu_pin_set=4-31

J版的时候社区完善了功能,可以针对numa特性来进行绑定了.在numa体系中玩binding不是那么简单了.[理解cpu topology](/2015-01-09/cpu_topology)应该能提供一点帮助.

- Vcpus与Node的策略 更多查看[virt-driver-numa-placement bp](http://specs.openstack.org/openstack/nova-specs/specs/juno/implemented/virt-driver-numa-placement.html)
    - hw:numa_nodes=NN - numa of NUMA nodes to expose to the guest.
    - hw:numa_mempolicy=preferred\|strict - memory allocation policy
    - hw:numa_cpus.0=<cpu-list> - mapping of vCPUS N-M to NUMA node 0
    - hw:numa_cpus.1=<cpu-list> - mapping of vCPUS N-M to NUMA node 1
    - hw:numa_mem.0=<ram-size> - mapping N GB of RAM to NUMA node 0
    - hw:numa_mem.1=<ram-size> - mapping N GB of RAM to NUMA node 1
- Vcpus与Pcpus的绑定策略,更多查看[vcpu与pcpu的绑定[Virt driver pinning guest vCPUs to host pCPUs](http://specs.openstack.org/openstack/nova-specs/specs/juno/approved/virt-driver-cpu-pinning.html)
    - hw:cpu_policy=shared\|dedicated
    - hw:cpu_threads_policy=avoid\|separate\|isolate\|prefer, 只有hw:cpu_policy为dedicated时本属性才生效
        - avoid: the scheduler will not place the guest on a host which has hyperthreads.
        - separate: if the host has threads, each vCPU will be placed on a different core. ie no two vCPUs will be placed on thread siblings
        - isolate: if the host has threads, each vCPU will be placed on a different core and no vCPUs from other guests will be able to be placed on the same core. ie one thread sibling is guaranteed to always be unused,
        - prefer: if the host has threads, vCPU will be placed on the same core, so they are thread siblings.


**举例(未验证)**

1.建立相应的flavor,并设置属性

    nova flavor-key m1.large set hw:numa_mempolicy=strict hw:numa_cpus.0=0,1,2,3 hw:numa_cpus.1=4,5,6,7 hw:numa_mem.0=1 hw:numa_mem.1=1 hw:cpu_policy=decicated hw:cpu_threads_policy=separate


2.建立相应的image,并设置属性

    glance image-update image_id –property hw_numa_mempolicy=strict –property hw_numa_cpus.0=0,1,2,3 –property hw_numa_cpus.1=4,5,6,7 –property hw_numa_mem.0=1 –property hw_numa_mem.1=1 --property hw_cpu_policy=decicated --property hw_cpu_threads_policy=separate


# 参考文档

1. [Pinning processors by common threads and cores](http://www-01.ibm.com/support/knowledgecenter/linuxonibm/liaat/liaattunpinproctop.htm?lang=en)
2. [https://wiki.openstack.org/wiki/VirtDriverGuestCPUMemoryPlacement](https://wiki.openstack.org/wiki/VirtDriverGuestCPUMemoryPlacement)

以上文档建议阅读

# 变更记录

|Why | Who | When |
|----|-----|------|
|create|fishcired|2014-11-27 |
|整理|fishcired|2015-01-05 |
|完善|fishcired|2015-01-09 |
