---
layout: post
title:  Ceph与OpenStack集成配置
subtitle:
description:
category: OpenStack
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags:
    - openstack
    - mysql
    - 高可用
---

# Ceph与OpenStack集成配置


目前，使用OpenStack的用户存储基本都使用ceph。参考链接2中的一张图非常好，里面清晰的展示了Ceph与OpenStack的关联。


![]( http://7xt00t.com2.z0.glb.clouddn.com/ceph-in-openstack.png)

## 基础准备

由于实验环境中OpenStack没有启动对象存储，所以下面的配置只有nova,cinder,glance的。OpenStack版本为liberty.

### 创建相关ceph用户和存储尺

| 账号        | pool            |      openstack组件|
|--------------|----------------|-------------------------|
| m-cinder | mitaka-volumes  | cinder |
|  m-glance              | mitaka-images    | glance |
|  m-cinder              | mitaka-vms         | nova   |

创建相关的pools. pg-num,pgp-num根据自己的集群进行设置.

```
root@l-controller-1:~# ceph osd pool create mitaka-volumes 2048 2048
pool 'mitaka-volumes' created
root@l-controller-1:~# ceph osd pool create mitaka-images 2048 2048
pool 'mitaka-images' created
root@l-controller-1:~# ceph osd pool create mitaka-vms 2048 2048
pool 'mitaka-vms' created
root@l-controller-1:~#
```

创建相关的账号，并设置权限

```
root@l-controller-1:~# ceph auth get-or-create client.m-cinder mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=mitaka-volumes, allow rwx pool=mitaka-vms, allow rx pool=mitaka-images'
[client.m-cinder]
        key = AQC4shBXx57uNBAAZqD736Eslal1bVvKvntJjg==

root@l-controller-1:~# ceph auth get-or-create client.m-glance mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=mitaka-images'
[client.m-glance]
        key = AQDdshBX6yL7CRAApMNLYQf5SXqH5EozUHmuGg==
```

### 同步ceph配置到所有节点

同步`ceph.conf`到所有openstack节点


```
ssh {your-openstack-server} sudo tee /etc/ceph/ceph.conf </etc/ceph/ceph.conf
```

### 安装相关的软件包

**配置软件仓库**

```
wget -q -O- 'https://ceph.com/git/?p=ceph.git;a=blob_plain;f=keys/release.asc' | sudo apt-key add -
sudo apt-add-repository 'deb http://download.ceph.com/debian-{release-name}/ {codename} main'
```


**glance-api节点**


```
sudo apt-get install python-rbd
sudo yum install python-rbd
```

**nova-compute/cinder-volume/cinder-backup节点**

```
sudo apt-get install ceph-common
```

### 配置认证

配置glance-api和cinder-volume节点的keyring.

```
# 将m-glance user的keyring拷贝到glance-api节点，并修改相应权限.
ceph auth get-or-create client.m-glance | ssh 192.168.250.34 sudo tee /etc/ceph/ceph.client.m-glance.keyring
ssh 192.168.250.34 sudo chown glance:glance /etc/ceph/ceph.client.glance.keyring

# 将m-cinder user的keyring拷贝到cinder volume节点,并更改相应的权限.
ceph auth get-or-create client.cinder | ssh 192.168.250.34 sudo tee /etc/ceph/ceph.client.cinder.keyring
ssh 192.168.250.34 sudo chown cinder:cinder /etc/ceph/ceph.client.cinder.keyring
```

nova-compute需要cinder.keyring

```
ceph auth get-key client.m-cinder | ssh {your-compute-node} tee client.m-cinder.key

uuidgen
457eb676-33da-42ec-9a8c-9293d545c337

cat > secret.xml <<EOF
<secret ephemeral='no' private='no'>
  <uuid>457eb676-33da-42ec-9a8c-9293d545c337</uuid>
  <usage type='ceph'>
    <name>client.m-cinder secret</name>
  </usage>
</secret>
EOF
sudo virsh secret-define --file secret.xml
Secret 457eb676-33da-42ec-9a8c-9293d545c337 created
sudo virsh secret-set-value --secret 457eb676-33da-42ec-9a8c-9293d545c337 --base64 $(cat client.cinder.key) && rm client.m-cinder.key secret.xml
```


## Glance集成Ceph配置

```
cat /etc/glance/glance-api.conf
[DEFAULT]
...
# enable copy-on-write
show_image_direct_url = True

[glance_store]
...
stores = rbd
default_store = rbd
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_user = m-glance
rbd_store_pool = mitaka-images
rbd_store_chunk_size = 8
...
[paste_deploy]
flavor = keystone
```

## Cinder与Ceph集成配置.


```
cat cinder.conf
[DEFAULT]
...
enabled_backends = ceph
...
[rbd]
volume_driver = cinder.volume.drivers.rbd.RBDDriver

rados_connect_timeout = -1
rados_connection_interval = 5
rados_connection_retries = 3
rdb_ceph_conf = /etc/ceph/ceph.conf
rbd_cluster_name = ceph

rbd_flatten_volume_from_snapshot = True
rbd_max_clone_depth = 5
rbd_pool = mitaka-volumes
rbd_user = m-cinder
rbd_secret_uuid = 457eb676-33da-42ec-9a8c-9293d545c337
rbd_store_chunk_size = 4
```

## Nova与Ceph集成配置


```
cat /etc/nova-compute.conf
[libvirt]
virt_type=kvm
...

images_type = rbd
images_rbd_pool = mitaka-vms
images_rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = m-cinder
rbd_secret_uuid = 457eb676-33da-42ec-9a8c-9293d545c337
disk_cachemodes = "network=writeback"
inject_password = false
inject_key = false
inject_partition = -2
live_migration_flag="VIR_MIGRATE_UNDEFINE_SOURCE,VIR_MIGRATE_PEER2PEER,VIR_MIGRATE_LIVE,VIR_MIGRATE_PERSIST_DEST,VIR_MIGRATE_TUNNELLED"
hw_disk_discard = unmap
```

# 参考链接
- [官方文档](http://docs.ceph.com/docs/master/rbd/rbd-openstack/)
- [Ceph And OpenStack  Current integration, roadmap and more](http://www.sebastien-han.fr/down/OpenStack%20_%20Ceph%20-%20Liberty.pdf)

# 更改记录

| 日期     |  备注                        |
|----------|-----------------------------|
| 2016-06-28 | 整理草稿为v1.0             |

