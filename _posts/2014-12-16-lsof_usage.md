---
layout: post
title: "文件描述符管理工具 lsof & fuser"
description: ""
category: 系统管理
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [系统管理,调试,lsof,fuser]
---

# 1. lsof(list open files)


lsof可以看做是一个搜索器,输入搜索条件,返回对应打开的文件.所以使用lsof主要就是掌握怎么输入条件.更高级一点就是按照lsof提供的高级语法来让加速lsof.这个工具真的是只做一件事,做就做到极致!

看一个命令, `lsof | grep xxx`,有人会这么用么,当然有了,不过很慢,也有点不负责.


> 1. lsof的过滤词一般支持`^`(取反),`-`(区间),`,`(列表)操作
> 2. 两个过滤词可以通过-a选项来表示必须同时满足
> 3. 同时指定关键词,表示or的关系

下面只列出比较常用的选项:

**1. 通过文件名查看**

- `lsof filename` 
- `lsof -c string`
- `lsof -d FD` FD可以是文件的fd,也可以是类型

**2. 通过进程归属来查看**

- lsof -p pid
- lsof -u user

**3. 过滤socket**

- `lsof -i [46][protocol][@hostname|hostaddr][:service|port]`

**4. 输出格式**

|COMMAND | PID | USER | FD        | TYPE    | DEVICE | SIZE/OFF | NODE | NAME |
|进程名  | pid | user | 文件描述符|文件类型 | 设备名 | 文件大小 | inode | 文件名称 |

# 2. fuser(identify processes using files or sockets)

fuser的功能与lsof基本一样.但是有个方便的`-k`参数,能够kill掉打开某文件的所有进程.这个在umount的时候非常方便.至于其他的选项就不介绍了,lsof就好.

# 变更记录

|Why | Who | When |
|----|-----|------|
|创建文件,提醒日后完善内容|fishcired|2014-12-16 |
