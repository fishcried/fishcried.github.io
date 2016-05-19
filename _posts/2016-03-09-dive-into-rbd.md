---
layout: post
title: "本地修改OpenStack虚机RBD块"
description: ""
category: OpenStack
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [openstack,ceph,rbd]
---

> 有台名为nginx的vm出现了些怪异的行为, 想把根磁盘导出来挂载在本地看看到底是怎么回事. 如果nova使用的是本地存储,vm disk会
存在相应compute节点的`/var/lib/nova/instance/xxxx/disk`中,直接使用mount命令井进行本地挂载即可，或者使用`libguestfs-tools`工具套件.
但是如果使用的是rbd存储,vm的disk会存在ceph pool里，需要先解决如何把相应的rbd image搞到本地，再做相应的挂载修改处理.

下面记录一下整体的处理流程，重点有两个:

1. 如何把rbd搞到本地.
2. 挂载image时需要跳过起始扇区,否则挂载不成功.

>  **如果磁盘不加密,其实就是在裸奔.**


### **1. 查看vm所在compute节点**

通过horizon确认nginx虚拟id，然后通过管理员权限执行`nova show`命令,可以看到vm所在物理机器,以及xml中的instance_name.

    fishcried@:~# nova show be3eab24-81f0-48f0-bf6d-1387da64184b
    +--------------------------------------+-----------------------------------------------------------------+
    | Property                             | Value                                                           |
    +--------------------------------------+-----------------------------------------------------------------+
    | OS-EXT-SRV-ATTR:host                 | compute-1                                                     |
    | OS-EXT-SRV-ATTR:instance_name        | instance-00000135                                               |
    | .............................        | .................                                               |
    +--------------------------------------+-----------------------------------------------------------------+

compute节点为`compute-1`, instance-name为`instance-00000135`.这个instance-name便于`virsh dumpxml`.

### **2. 通过xml文件查看相应的rbd image id**

登陆上面的compute节点,然后执行`virsh dumpxml instance-00000135`查看xml定义,主要找到disk device中rbd的image id.

    fishcried@:~# virsh dumpxml instance-00000135
    <domain type='kvm' id='74'>
      <name>instance-00000135</name>
      ...
      <devices>
        <disk type='network' device='disk'>
          <driver name='qemu' type='raw' cache='writeback' discard='unmap'/>
          <auth username='cinder'>
            <secret type='ceph' uuid='df995543-c5b7-48f7-b812-aed8c37a6d4a'/>
          </auth>
          <source protocol='rbd' name='liberty-pool/be3eab24-81f0-48f0-bf6d-1387da64184b_disk'>
            <host name='192.168.xxx.96' port='6789'/>
            <host name='192.168.xxx.90' port='6789'/>
            <host name='192.168.xxx.91' port='6789'/>
          </source>
          <backingStore/>
          <target dev='vda' bus='virtio'/>
          <alias name='virtio-disk0'/>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
        </disk>
      </devices>
    </domain>

看到了吧,rbd信息为`liberty-pool/be3eab24-81f0-48f0-bf6d-1387da64184b_disk`,格式是`pool/image-id`的形式.

### **3. 将rbd image导入到本地**

> 想导出rbd image肯定需要ceph的权限的.

导出rbd image的方式有两种,一种是`rbd map`,另一种是`rbd export`. 如果是map的方式,执行后会在`/dev`目录下看到相应的
rbdx设备(其实是软连接到相应的文件), 如果是export的方式,执行时需要指定导出后的文件名.

**map**

    # map的方式
    fishcried@:~# rbd map liberty-pool/be3eab24-81f0-48f0-bf6d-1387da64184b_disk
    /dev/rbd0

**export**

    # export的方式
    rbd export liberty-pool/be3eab24-81f0-48f0-bf6d-1387da64184b_disk     nginx-rbd

### **4. 挂载image**

以上面`map`的方式继续,map后rbd块映射到了本地的/dev/rbd0.

通过`fdisk -lu /dev/rbd0`查看下image的分区信息, 尤其要注意`Units`和`Start`这里,因为要跳过Start字节数.

    fishcried@:~/nginx# fdisk -lu /dev/rbd0
    GNU Fdisk 1.2.4
    Copyright (C) 1998 - 2006 Free Software Foundation, Inc.
    This program is free software, covered by the GNU General Public License.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU General Public License for more details.


    Disk /dev/rbd0: 85 GB, 85896599040 bytes
    255 heads, 63 sectors/track, 10443 cylinders, total 167766795 sectors
    Units = sectors of 1 * 512 = 512 bytes

        Device Boot      Start         End      Blocks   Id  System 
    /dev/rbd0p1   *        2048   167765961    83883366   83  Linux
    Warning: Partition 1 does not end on cylinder boundary.  

上面看到一个Units为512字节, Start为2048,所以起始为512*2048,使用`mount -o offset=xxx`命令即可.

    fishcried@:~/# echo "512*2048" | bc
    1048576

上面用的是`bc`计算器算一下,其实心算我也能算出来,不过不能紧张.

    fishcried@:~/# mkdir nginx
    fishcried@:~/# mount -o offset=1048576 /dev/rbd0 nginx
    fishcried@:~/nginx# ls
    bin  boot  dev  etc  home  initrd.img  lib  lib64  lost+found  media  mnt  opt  proc  fishcried  run  sbin  srv  sys  tmp  usr  var  vmlinuz
    fishcried@:~/nginx# cd ..
    fishcried@:~/# umount  nginx

### **4. 收场**

修改后,需要将rbd image再导入到ceph中,如果是map的方式则不用这么麻烦了,只要`rbd unmap /dev/rbd0`即可.如果是`export`的方式,则需要导入一下.


    rbd import liberty-pool/be3eab24-81f0-48f0-bf6d-1387da64184b_disk   nginx-rbd
