---
layout: post
title: "OpenStack消息队列学习(二): RabbitMq操作指南"
description: ""
category: OpenStack
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [openstack,rabbitmq]
---

管理rabbitmq的时候能有web ui就用ui,毕竟方便,次之是rabbitmqctl,这个是基础.

# 0. Web UI

![web ui](/img/rabbitmq_ui.png)

开启方法很简单`rabbitmq-plugins enable rabbitmq_management`,然后访问http://server-name:15672/即可.

> 还有个[rabbitmqadmin](http://www.rabbitmq.com/management-cli.html),比rabbitmqctl要好用点,可以去搞一下.但是好用的有ui,基本有的rabbitmqctl,再去掌握admin有点负担太重,看个人精力吧.

# 1. rabbitmqctl子命令的规律

1. 子命令都是动作在前,而且后面一般是复数.如`list_queues`, `add_user`, `list_queues`
2. 大部分命令是list动作.如: `list_users,list_vhosts,list_permissions`

# 2. 服务管理

    stop [<pid_file>]
    stop_app
    start_app
    wait <pid_file>
    reset
    force_reset
    rotate_logs <suffix>

# 3.  常用操作

## 3.1 操作用户(list,add,delete,change)

| 新建 |add_user <username> <password>
| 删除 |delete_user <username>
| 修改密码 |change_password <username> <newpassword>
| 清除 |clear_password <username>
| 查看 |list_users

    # 查看都有哪些用户
    root@kali:~# rabbitmqctl list_users
    Listing users ...
    guest   [administrator]
    ...done.

    # 新建用户新建用户user1,密码passwd1
    root@kali:~# rabbitmqctl add_user user1 passwd1
    Creating user "user1" ...
    ...done.

    # 更改user1的密码为123456
    root@kali:~# rabbitmqctl change_password user1 123456
    Changing password for user "user1" ...
    ...done.

    # 删除user1
    root@kali:~# rabbitmqctl delete_user user1
    Deleting user "user1" ...
    ...done.
    root@kali:~# rabbitmqctl list_users
    Listing users ...
    guest   [administrator]
    ...done.


## 3.2 操作虚拟主机(add,delete,list)


|新建 |add_vhost <vhostpath>
|删除 |delete_vhost <vhostpath>
|查看 |list_vhosts [<vhostinfoitem> ...]

    # 查看当前虚拟主机
    root@kali:~# rabbitmqctl  list_vhosts
    Listing vhosts ...
    /
    ...done.

    # create a new vhost host1
    root@kali:~# rabbitmqctl  add_vhost host1
    Creating vhost "host1" ...
    ...done.


## 3.3 设置权限

权限分为conf,write,read三类,对应如下如:

![rabbit permissions](/img/rabbit_permissions.png)



| 设置    | set_permissions [-p <vhostpath>] <user> <conf> <write> <read>
| 清除    |  clear_permissions [-p <vhostpath>] <username>
| 查看虚拟主机权限  | list_permissions [-p <vhostpath>]
| 查看用户权限 | list_user_permissions [-p <vhostpath>] <username>


## 3.4 查看exchanges,bindings,queues,connetions,channels,consumers

查看起来非常简单,list_xxx即可,-p可以指定虚拟主机,如果想查看详细项目的话,后面可以添加infoitems.一般不需要定制输出,默认已经足够.

**`rabbitmqctl list_queues name messages consumers arguments`**

> <queueinfoitem> must be a member of the list [name, durable, auto_delete,
arguments, pid, owner_pid, exclusive_consumer_pid, exclusive_consumer_tag,
messages_ready, messages_unacknowledged, messages, consumers, memory,
slave_pids, synchronised_slave_pids].

**`rabbitmqctl list_exchanges`** 默认输出数据就够用了

**`rabbitmqctl list_bindings`** 


# 4. 参考

1.[Management Plugin](http://www.rabbitmq.com/management.html)

# 变更记录

|Why | Who | When |
|----|-----|------|
|create|fishcired|2014-09-22|
|总结内容|fishcired|2014-10-13 |
|整理 | fishcried | 2014-12-16 |
|修改错别字 | fishcried | 2014-12-17  |
|将ui调整到最前| fishcried | 2014-12-18 |
