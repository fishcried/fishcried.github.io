---
layout: post
title: "SaltStack(一)试水"
description: ""
category: 配置管理
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [saltstack,devops,配置管理]
serials: 配置管理
---

# 1. What is SaltStack?

**from 官方手册**

> 1. configuration management system, capable of maintaining remote nodes in defined states (for example, ensur-
> ing that specific packages are installed and specific services are running)
> 2. a distributed remote execution system used to execute commands and query data on remote nodes, either indi-
> vidually or by arbitrary selection criteria

**from民间**

> Salt = 强化的Func +  弱化的Puppet

**架构**

  功能上，SaltStack提供了配置管理和远程执行两大块。配置管理包括配置下发，版本更新，指定依赖关系和状态管理等;远程执行是指下发命令到minion机器执行，SaltStack包含了种类众多的命令模块，如cmd，mysql，package等。

  部署上，SaltStack分为master,syndic,minion三部分.如下图:

![SaltStack架构图](/img/saltstack_architecture.png)

- salt-master： 黑老大
	 - 安装有salt-master的节点我们称作Master节点，负责存储配置信息、对可信的Minion节点进行授权、通过ZeroMQ与Minion节点进行交互。

- salt-syndic： 小头目
	-   在特别庞大的部署环境中才会使用syndic，比如在多数据中心的部署中。syndic相当于一个正向代理节点，它代理了所有Master节点与Minion节点的通信。这样做一方面可以将Master的负载分担给多个syndic承担。另一方面，它也可以降低Master通过广域网访问Minion的成本，提高了安全性，使salt适用于夸数据中心的部署。

- salt-minion： 马仔
	- 安装有salt-minion的节点称作Minion节点，salt-minion就是一个agent进程，通过ZeroMQ接收来自master的命令，执行并返回结果；


**特点**

- 快:  salt的命令执行确实非常快，用一下就能感受到
- 并行: salt的命令执行是并行的
- 加密: 通信是加密的
- 灵活:
	- salt的远程执行,有很全的内置模块，也可以执行原生命令，而且可以执行脚本(自动推送的minion，然后执行)
	- 配置管理使用起来也是非常灵活的
- 简单:
	- 安装简单
	- 查看教程后，使用起来也非常简单

# 2 Why  use it

 之前没有用过相关工具，无法感受其与Puppet等工具的不同。试用了一下salt,感觉salt绝对是运维利器,尤其是大规模的集群管理，在云时代绝对有用武之处,更是大势所趋.

|场景| 手动方式| saltstack|
|---+------------+----------|
| N台机器上执行M条命令 |  开放ssh root登陆,打通秘钥| 自动认证 |
| N台机器上顺序安装X种软件,启动Y个服务 |  写脚本,调试很久,不能多次运行 | 配置管理长项 |
| N台机器中选择X台执行 | 通过ip来过滤,来个`hostlist.conf`文件 | 目标分组解决 |
| 管理100台主机| 求死 | so easy |

每个人使用会有不同的感受，面对的需求，过往的经历，个人偏好都是决定因素。salt让我感觉很惊讶。就像是一直用tcpdump(ssh,bash,screen)，然后突然知道了wireshark(salt)。salt的便利，灵活性，执行速度让人爱不释手。突然想用salt写一个linux安装后的初始配置工具，能够自动的配置bash，vim，firefox，还有安装必要软件等.[正在维护中](https://github.com/fishcried/linux_profile).有点大才小用,但是确实很方便.

# 2. 安装(On ubuntu14.04)

使用salt，安装非常简单。服务器端安装`salt-master`,客户端安装`salt-minion`并且配置服务器端ip，然后接受连接就可以使用。

**1. 服务器端**

    #安装软件包
    sudo apt-get install salt-master
    #编辑/etc/salt/master配置文件,修改监听地址
    - #interface: 0.0.0.0
    + interface: 10.0.0.1


更详细的配置见[官方](http://docs.saltstack.com/en/latest/ref/configuration/master.html)

**2. 客户端**

	#安装软件包
	sudo apt-get install salt-minion
	
	#配置/etc/salt/minion文件,指定server地址 
	- #master: salt
	+ master: 10.0.0.1

更详细的配置见[官方](http://docs.saltstack.com/en/latest/ref/configuration/minion.html)

**3. syndic**

TODO: 暂时没有使用，以后补上

# 3 使用`salt-key`认证

minion安装配置后，需要在master通过认证。使用`salt-key -A`接受认证后，即可进行日常管理.

	#列出所有的认证
	salt-key -L
	#接受所有新认证
	salt-key -A

salt-key的使用`man salt-key` or `salt-key --help`

> Note:
> 1. 编辑`/etc/salt/master`使`auto_accept: True `可以自动接受新的认证.
> 2. 如果网络不同，请确实是否是防火墙问题，如果只是为了试用，最好暂时关闭防火墙。否则请开放TCP4505,4046端口

测试连接,查看两端网络是否畅通

	salt "*" test.ping

查看连接情况,可以具体查看那些minion在线，那些不在线

	salt-run manage.status

# 4 Salt命令介绍

|命令  | 说明
|------+----
|salt |Salt主命令,比如执行命令模块
|satl-cp|复制文件到指定的系统上去
|salt-key|和 Minion 之间进行身份验证
|salt-master|Master 主守护进程,用于控制 Minion
|salt-run|前端命令执行
|salt-syndic|Salt syndic 守护进程,用于多级 salt-master 使用

需要帮助的时候就请出你的男仆`man`,

# 参考

1.《运维社区-Saltstack》
2. [官方安装文档](http://docs.saltstack.com/en/latest/topics/installation/index.html)

# 变更记录

|Why | Who | When |
|----|-----|------|
|create|fishcired|2014-08-04|
|v1.0| fishcried| 2014-08-10 |
|修改了salt使用情景表格,改正错别字| fishcried| 2014-12-04  |

