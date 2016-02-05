---
layout: post
title: "OpenStack Neutron Dhcp-agent的高可用"
description: ""
category: openstack
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [openstack,neutron,dhcp]
---

OpenStack中创建虚机时，由于dhcp获取ip失败的情况比较常见，尤其是批量操作。所以对dhcp做HA非常必要。
以下对部署多dhcp做了记录.

# 1 查看下当前系统的部署情况

首先，需要了解下自己的网络环境。

- 当前有那些网络
- 多少个dhcp agent
- 以及你关注的子网有几个dncp agent。

##  1.1 查看当前有哪些网络

	root@ncloud-keystone-1:~# neutron net-list
	+--------------------------------------+-------------+-------------------------------------------------------+
	| id                                   | name        | subnets                                               |
	+--------------------------------------+-------------+-------------------------------------------------------+
	| 92df65b1-8237-46e5-b8ba-f4d8044c15b9 | Ext-Net-1   | d81eddb8-4fe0-4b22-8c1f-2202db8e6a2c 192.168.252.0/24 |
	| ab6c75f1-396d-4c13-977e-ab2003d3a6eb | 172.1.1.0   | ade33a99-1b57-423c-81ca-d3112860a993 172.1.1.0/24     |
	+--------------------------------------+-------------+-------------------------------------------------------+

以上可以看到当前有一个`Ext-Net-1`，用于连接外网。 还有一个`172.1.1.0`，这个是租户的子网。

## 1.2 查看当前有多少个dhcp-agent

	root@ncloud-keystone-1:~#neutron agent-list | grep DHCP
	 | ec821072-37ae-46c9-9abf-9c1410e1cc06 | DHCP agent         | ncloud-network-1  | :-)   | True           |

当前只有一个dhcp-agent,没有做HA。

# 2 增加dhcp agent

决定在一台compute节点安装下dhcp agent，做为HA来使用。

## 2.1 安装neutron-dhcp-agent

	root@ncloud-compute-16:~# apt-get install neutron-dhcp-agent
	Reading package lists... Done
	Building dependency tree       
	...
	neutron-metadata-agent start/running, process 41235
	Processing triggers for ureadahead (0.100.0-16) ...
	Setting up neutron-dhcp-agent (1:2014.1.1-0ubuntu2) ...
	neutron-dhcp-agent start/running, process 41288

可以看到在安装neutron-dhcp-agent的同时，neutron-metadata-agent也被安装了。 这两个agent是需要同时装的，如果没有neutron-matadata-agent，cloud-init将 无法获取数据。 

## 2.2 修改`/etc/neutron/dhcp_agent.ini`配置文件

	# 通过sed脚本来修改
	root@ncloud-compute-16:~# cat dhcp-config.sed 
	s:^# (interface_driver = .*OVSInterfaceDriver).*$:\1:
	s:^# (dhcp_driver = .*$):\1:
	s:^# (use_namespaces) .*$:\1 = True:
	s:^# (enable_isolated_metadata) .*$:\1 = True:
	s:^# (enable_metadata_network) .*$:\1 = True:
	s:^# (dnsmasq_config_file) .*$:\1 = /etc/neutron/dnsmasq-neutron.conf:
	
	# 执行脚本
	root@ncloud-compute-16:~# sed -i.bak  -rf dhcp-config.sed /etc/neutron/dhcp_agent.ini 
	
	# 创建dnsmasq使用的配置文件
	root@ncloud-compute-16:~# echo "dhcp-option-force=26,1400" >  /etc/neutron/dnsmasq-neutron.conf

**通过dhcp将MTU调小为1400,否则数据包通过GRE隧道后长度会超过1500,出现问题**.

## 2.3 配置metadata-agent

	root@ncloud-compute-16:~# cat metadata-config.sed
	s|^(auth_url) .*$|\1 = http://ncloud-keystone-1:5000/v2.0|
	s|^(auth_region) .*$|\1 = regionOne|
	s|^(admin_tenant_name) .*$|\1 = service|
	s|^(admin_user) .*$|\1 = neutron|
	s|^(admin_password) .*$|\1 =neutron|
	s|^# (nova_metadata_ip) .*$|\1 = ncloud-nova-1|
	s|^# (metadata_proxy_shared_secret) .*$|\1 = neunn|
	
	root@ncloud-compute-16:~# sed -i.bak -rf metadata-config.sed /etc/neutron/metadata_agent.ini 

## 2.4 重启服务

	rm -rf /var/lib/neutron/*
	killall dnsmasq
	neutron-ovs-cleanup
	neutron-netns-cleanup
	service neutron-plugin-openvswitch-agent restart
	service neutron-dhcp-agent restart
	service neutron-metadata-agent restart

## 2.5. 查看当前dhcp-agent

	root@ncloud-keystone-1:~# neutron agent-list | grep DHCP
	| 386c2d99-acbc-48b8-85eb-3427ee21f515 | DHCP agent         | ncloud-compute-16 | :-)   | True           |
	| ec821072-37ae-46c9-9abf-9c1410e1cc06 | DHCP agent         | ncloud-network-1  | :-)   | True           |

## 2.6 创建网络test_dhcp_on_c16

	root@ncloud-keystone-1:~# neutron net-create test_dhcp_on_c16
	Created a new network:
	+---------------------------+--------------------------------------+
	| Field                     | Value                                |
	+---------------------------+--------------------------------------+
	| admin_state_up            | True                                 |
	| id                        | e68a6446-ad4a-4e96-82a3-b28268f5fc37 |
	| name                      | test_dhcp_on_c16                     |
	| provider:network_type     | vxlan                                |
	| provider:physical_network |                                      |
	| provider:segmentation_id  | 14                                   |
	| shared                    | False                                |
	| status                    | ACTIVE                               |
	| subnets                   |                                      |
	| tenant_id                 | cbd4514eebb649f2b1bd26d4823b99ec     |
	+---------------------------+--------------------------------------+
	root@ncloud-keystone-1:~# neutron subnet-create test_dhcp_on_c16 23.0.0.0/24 --name subnet_test_dhcp
	Created a new subnet:
	+------------------+--------------------------------------------+
	| Field            | Value                                      |
	+------------------+--------------------------------------------+
	| allocation_pools | {"start": "23.0.0.2", "end": "23.0.0.254"} |
	| cidr             | 23.0.0.0/24                                |
	| dns_nameservers  |                                            |
	| enable_dhcp      | True                                       |
	| gateway_ip       | 23.0.0.1                                   |
	| host_routes      |                                            |
	| id               | 0a8a4c82-040d-4631-a810-65b62b3aca46       |
	| ip_version       | 4                                          |
	| name             | subnet_test_dhcp                           |
	| network_id       | e68a6446-ad4a-4e96-82a3-b28268f5fc37       |
	| tenant_id        | cbd4514eebb649f2b1bd26d4823b99ec           |
	+------------------+--------------------------------------------+
	neutron dhcp-agent-list-hosting-net  e68a6446-ad4a-4e96-82a3-b28268f5fc37
	+--------------------------------------+-------------------+----------------+-------+
	| id                                   | host              | admin_state_up | alive |
	+--------------------------------------+-------------------+----------------+-------+
	| 386c2d99-acbc-48b8-85eb-3427ee21f515 | ncloud-compute-16 | True           | :-)   |
	+--------------------------------------+-------------------+----------------+-------+

# 3 管理dhcp

上面可以看到新创建的网络只是用了一个dhcp，就是新创建的，还是没有做成HA。很简单，我们可以手动指定。

将network1的dhcp-agent也分配给新建的网络。

	root@ncloud-keystone-1:~# neutron dhcp-agent-network-add ec821072-37ae-46c9-9abf-9c1410e1cc06  test_dhcp_on_c16
	Added network test_dhcp_on_c16 to DHCP agent
	root@ncloud-keystone-1:~# neutron dhcp-agent-list-hosting-net test_dhcp_on_c16
	+--------------------------------------+-------------------+----------------+-------+
	| id                                   | host              | admin_state_up | alive |
	+--------------------------------------+-------------------+----------------+-------+
	| 386c2d99-acbc-48b8-85eb-3427ee21f515 | ncloud-compute-16 | True           | :-)   |
	| ec821072-37ae-46c9-9abf-9c1410e1cc06 | ncloud-network-1  | True           | :-)   |
	+--------------------------------------+-------------------+----------------+-------+

有添加当然就有移除了,如果想移除刚刚的dhcp agent

	root@ncloud-keystone-1:~# neutron dhcp-agent-network-remove  ec821072-37ae-46c9-9abf-9c1410e1cc06  test_dhcp_on_c16
	Removed network test_dhcp_on_c16 to DHCP agent
	root@ncloud-keystone-1:~# neutron dhcp-agent-list-hosting-net test_dhcp_on_c16
	+--------------------------------------+-------------------+----------------+-------+
	| id                                   | host              | admin_state_up | alive |
	+--------------------------------------+-------------------+----------------+-------+
	| 386c2d99-acbc-48b8-85eb-3427ee21f515 | ncloud-compute-16 | True           | :-)   |
	+--------------------------------------+-------------------+----------------+-------+

如果每次都需要手动指定，那是不是很麻烦，其实还有一个简单的办法。 设置neutron.conf的`DEFAULT`段`dhcp_agents_per_network`选项.
如果你希望新建的网络需要两个dhcp agent，那么`dhcp_agents_per_network=2`， 然后重启neutron服务.即可

# 4 总结

## 4.1 常用命令

	root@ncloud-keystone-1:~# neutron help | grep dhcp
	dhcp-agent-list-hosting-net    List DHCP agents hosting a network.
	dhcp-agent-network-add         Add a network to a DHCP agent.
	dhcp-agent-network-remove      Remove a network from a DHCP agent.
	net-list-on-dhcp-agent         List the networks on a DHCP agent.

## 4.2 坑

1. 配置文件内配置项前不能有空格,否则会读取失败
2. 如果创建虚机常常出现dhcp分配失败`dnsmasq-dhcp[13578]: DHCPDISCOVER(ns-ea864f58-69) 192.168.101.14 fa:16:3e:07:e2:8d no address available`
	- 可以尝试修改neutron.conf` report_interval=15 agent_down_time = 30` [具体参见](http://www.choudan.net/2014/03/10/OpenStack-Neutron-DHCP-问题.html)

# 5 参考

1. [管理员手册](http://docs.OpenStack.org/admin-guide-cloud/content/app_demo_multi_dhcp_agents.html)

# 变更记录

|Why | Who | When |
|----|-----|------|
|create |fishcired|2014-07-09 |
|详细步骤|fishcired|2014-07-21|
|修正了错字,添加细节|fishcired|2014-07-22 |
