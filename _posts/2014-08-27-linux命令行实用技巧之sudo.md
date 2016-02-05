---
layout: post
title: "linux commands tips: sudo"
description: ""
category: 系统管理
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [命令行使用]
---

linux系统中,如果把各个用户与官位对应起来,那么root是至高无上的皇帝.大部分用户为平民.sudo命令能够让平民具有高级权限.sudo的核心就是适当放权,避免root事事亲为.

但是可怕的是很多系统中,sudo赋予了普通用户任何权限,也就是人人都能成为root.哪天谁叛变了,国家随时灭亡.这一切都是随意造成,没有仔细的了解sudo,对sudo进行详细配置.

sudo特性:

- 指定哪些用户执行哪些命令,可配置
- 提供审计,日志功能
- 提供有效期的概念,就是第一次输入密码后,五分钟之内不用再次输入密码

# 1. 常用选项

	   sudo -h | -K | -k | -V
	
	   sudo -v [-AknS] [-g group name|#gid] [-p prompt] [-u user name|#uid]
	
	   sudo -l[l] [-AknS] [-g group name|#gid] [-p prompt] [-U user name]
	   [-u user name|#uid] [command]
	
	   sudo [-AbEHnPS] [-C fd] [-g group name|#gid] [-p prompt] [-r role] [-t type]
	   [-u user name|#uid] [VAR=value] [-i | -s] [command]
	
	   sudoedit [-AnS] [-C fd] [-g group name|#gid] [-p prompt] [-u user name|#uid]
	   file ...

- `-l` 列出当前用户可以执行那些命令
- `-u username [#uid]` 以指定用户的身份执行
- `-g group name[$gui]` 以指定组的身份执行
- `-k ` 清楚session的有效性
- `-b ` 后台执行命令
- `-e file` 相当与sudoedit

# 2. 配置文件`/etc/sudoers`

# 2.1 配置格式

	users      hosts      =      (run_as)      [:NOPASSWD]      commands
	
	谁         来自哪些主机      作为哪个用户  不用密码         做哪些操作

- users:
	- user1,user2
	- %manager 表示用户组
- hosts
	- host1,host2
- run_as
	- user1,user2
- commands
	- 必须是绝对路径
	- cmd1,cmd2
	- !cmd2 排除的意思

## 2.2 使用别名简化复杂配置

	User_Alias  USERS  user1,user2
	Host_Alias  HOSTS  host1
	Runas_Alias RUNAS  fishcried
	Cmnd_Alias  CMDS   /bin/su
	
	USERS HOSTS=(RUNAS) CMDS

> 1. 编辑配置文件时请使用`visudo`,因为`visudo`保护性更高
> 2. 允许执行的命令,应该采用白名单原则

#  3. 开启日志`syslog`日志

	echo 'Defaults logfile=/var/log/sudo.log' > /etc/sudoers
	
	echo 'local2.debug /var/log/sudo.log' > /etc/syslog.conf
	service syslog restart

# 变更记录

|Why | Who | When |
|----|-----|------|
|create|fishcired|2014-08-27 |
