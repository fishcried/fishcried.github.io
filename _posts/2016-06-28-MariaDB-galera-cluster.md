---
layout: post
title:  搭建Mysql高可用集群(MariaDB Galera Cluster)
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

# 搭建Mysql高可用集群(MariaDB Galera Cluster)

对Mysql并不熟，但是OpenStack中使用mysql肯定要保证高可用，参照着OpenStack官方HA文档和一些资料整理MariaDB Galera Cluster搭建过程，省着忘了。以下就是实验的物理机器:

| hostname | ip | 用途 |
|---------------|-----|---|
| liberty-1  | 192.168.252.151 | first node |
| liberty-2  | 192.168.252.153 | other node |
| liberty-3  | 192.168.252.190 | other node |


## MariaDB Galera Cluster

> Galera Cluster for MySQL is  a true Multimaster Cluster based on synchronous replication. Galera Cluster is an easy-to-use, high-availability solution, which provides high system uptime, no data loss and scalability for future growth.

![](http://galeracluster.com/wp-content/uploads/2013/10/galera_replication1.png)

**World’s most advanced features**

- Synchronous replication
- Active-active multi-master topology
- Read and write to any cluster node
- Automatic membership control, failed nodes drop from the cluster
- Automatic node joining
- True parallel replication, on row level
- irect client connections, native MySQL look & feel


## 配置主节点

###  软件包安装

**添加软件仓库**

```
ubuntu@liberty-1:~$ sudo apt-get update
ubuntu@liberty-1:~$ sudo apt-get install python-software-properties 
ubuntu@liberty-1:~$ sudo apt-key adv --recv-keys --keyserver keyserver.ubuntu.com 0xcbcb082a1bb943db
ubuntu@liberty-1:~$ sudo add-apt-repository 'deb http://mirror.jmu.edu/pub/mariadb/repo/10.0/ubuntu trusty main'
ubuntu@liberty-1:~$ sudo apt-get update
```

**安装MariaDB Galera Cluster**

```
# 设置root密码为root,该密码为测试用途
ubuntu@liberty-1:~$ sudo apt-get install mariadb-galera-server galera -y
ubuntu@liberty-1:~$ sudo apt-get install rsync

```

### 防火墙配置


- 3306: Galera Cluster uses TCP for database client connections and State Snapshot Transfers methods that require the client, (that is, mysqldump).
- 4567: Galera Cluster uses TCP for replication traffic. Multicast replication uses both TCP and UDP on this port.
- 4568: Galera Cluster uses TCP for Incremental State Transfers.
- 4444: Galera Cluster uses TCP for all other State Snapshot Transfer methods.

下面配置中NODE-IP-ADDRESS自行替换

```
# iptables --append INPUT --in-interface eth0 \
   --protocol tcp --match tcp --dport 3306 \
   --jump ACCEPT
# iptables --append INPUT --in-interface eth0 \
   --protocol tcp --match tcp --dport 4567 \
   --jump ACCEPT
# iptables --append INPUT --in-interface eth0 \
   --protocol tcp --match tcp --dport 4568 \
   --jump ACCEPT
# iptables --append INPUT --in-interface eth0 \
   --protocol tcp --match tcp --dport 4444 \
   --jump ACCEPT
# service save iptables
```

### apparmor配置

```
# ln -s /etc/apparmor.d/usr /etc/apparmor.d/disable/.sbin.mysqld
# service apparmor restart
```

### 配置数据库

**/etc/mysql/conf.d/cluster.cnf配置**

```
cat /etc/mysql/conf.d/cluster.cnf
[mysqld]
query_cache_size=0
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
query_cache_type=0
bind-address=0.0.0.0

# Galera Provider Configuration
wsrep_provider=/usr/lib/galera/libgalera_smm.so

# Galera Cluster Configuration
wsrep_cluster_name="openstack-cluster"
wsrep_cluster_address="gcomm://liberty-1,liberty-2,liberty-3"
wsrep_node_address="192.168.252.153"
wsrep_node_name="liberty-1"
wsrep_sst_method=rsync
```

## 配置其他节点

> 在liberty-2,liberty-3上做相同的操作，设置repo，然后安装相关软件.


## 初始化集群

```
# 启动第一个节点
root@liberty-1:/home/ubuntu#  service mysql start --wsrep-new-cluster
root@liberty-1:/home/ubuntu#  mysql -u root -e "SHOW STATUS LIKE 'wsrep_cluster_size';
" -p
Enter password: 
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 1     |
+--------------------+-------+
root@liberty-1:/home/ubuntu# 
```

> --wsrep-new-cluster 这个参数只能在初始化集群使用，且只能在一个节点使用。


```
# 启动其他节点
root@liberty-2:~# service mysql  start
root@liberty-2:~# mysql -u root -e "SHOW STATUS LIKE 'wsrep_cluster_size';
> " -p
Enter password: 
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 2     |
+--------------------+-------+
root@liberty-3:~# service mysql  start
root@liberty-3:~# mysql -u root -e "SHOW STATUS LIKE 'wsrep_cluster_size';
> " -p
Enter password: 
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 3     |
+--------------------+-------+

```

## 测试集群

```
# from one node
root@liberty-1:/home/ubuntu# mysql  -uroot  -proot  -e  "create database galera_test"

# from other node
root@liberty-2:~# mysql  -uroot  -proot  -e  "show databases"
+--------------------+
| Database           |
+--------------------+
| galera_test        |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+

root@liberty-3:~# mysql  -uroot  -proot  -e  "show databases"
+--------------------+
| Database           |
+--------------------+
| galera_test        |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+

```

## 查看集群状态

```

# mysql -e "show status like 'wsrep%';"
+------------------------------+---------------------------------------------------+
| Variable_name                | Value                                             |
+------------------------------+---------------------------------------------------+
| wsrep_local_state_uuid       | 152542db-32cb-11e6-9a93-131ed66f4fdb              |
| wsrep_protocol_version       | 7                                                 |
| wsrep_last_committed         | 3149541                                           |
| wsrep_replicated             | 256466                                            |
| wsrep_replicated_bytes       | 138274219                                         |
| wsrep_repl_keys              | 1020696                                           |
| wsrep_repl_keys_bytes        | 14064350                                          |
| wsrep_repl_data_bytes        | 107796045                                         |
| wsrep_repl_other_bytes       | 0                                                 |
| wsrep_received               | 4954                                              |
| wsrep_received_bytes         | 40426                                             |
| wsrep_local_commits          | 256466                                            |
| wsrep_local_cert_failures    | 0                                                 |
| wsrep_local_replays          | 0                                                 |
| wsrep_local_send_queue       | 0                                                 |
| wsrep_local_send_queue_max   | 6                                                 |
| wsrep_local_send_queue_min   | 0                                                 |
| wsrep_local_send_queue_avg   | 0.025974                                          |
| wsrep_local_recv_queue       | 0                                                 |
| wsrep_local_recv_queue_max   | 2                                                 |
| wsrep_local_recv_queue_min   | 0                                                 |
| wsrep_local_recv_queue_avg   | 0.000404                                          |
| wsrep_local_cached_downto    | 2918055                                           |
| wsrep_flow_control_paused_ns | 0                                                 |
| wsrep_flow_control_paused    | 0.000000                                          |
| wsrep_flow_control_sent      | 0                                                 |
| wsrep_flow_control_recv      | 0                                                 |
| wsrep_cert_deps_distance     | 51.414320                                         |
| wsrep_apply_oooe             | 0.346506                                          |
| wsrep_apply_oool             | 0.000000                                          |
| wsrep_apply_window           | 6.688602                                          |
| wsrep_commit_oooe            | 0.000000                                          |
| wsrep_commit_oool            | 0.000000                                          |
| wsrep_commit_window          | 6.341971                                          |
| wsrep_local_state            | 4                                                 |
| wsrep_local_state_comment    | Synced                                            |
| wsrep_cert_index_size        | 122                                               |
| wsrep_causal_reads           | 0                                                 |
| wsrep_cert_interval          | 5.696556                                          |
| wsrep_incoming_addresses     | 10.2.9.1:3307,10.2.9.3:3307,10.2.9.2:3307         |
| wsrep_evs_delayed            |                                                   |
| wsrep_evs_evict_list         |                                                   |
| wsrep_evs_repl_latency       | 0.000258134/0.000565341/0.0016854/0.000160172/139 |
| wsrep_evs_state              | OPERATIONAL                                       |
| wsrep_gcomm_uuid             | 6b50600c-3c4b-11e6-b8cd-dbd1ab5f1f8f              |
| wsrep_cluster_conf_id        | 3                                                 |
| wsrep_cluster_size           | 3                                                 |
| wsrep_cluster_state_uuid     | 152542db-32cb-11e6-9a93-131ed66f4fdb              |
| wsrep_cluster_status         | Primary                                           |
| wsrep_connected              | ON                                                |
| wsrep_local_bf_aborts        | 0                                                 |
| wsrep_local_index            | 0                                                 |
| wsrep_provider_name          | Galera                                            |
| wsrep_provider_vendor        | Codership Oy <info@codership.com>                 |
| wsrep_provider_version       | 25.3.15(r3578)                                    |
| wsrep_ready                  | ON                                                |
| wsrep_thread_count           | 33                                                |
+------------------------------+---------------------------------------------------+
```

## 参考文档

- [OpenStack高可用官方文档数据库章节](http://docs.openstack.org/ha-guide/shared-database.html)
- http://galeracluster.com/documentation-webpages/


## 更改记录

| 日期 | 备注 |
|------------|--------------------|
| 2016-06-28 | 初步整理草稿到v1.0 |
