---
layout: post
title: "linux下dns查询相关命令笔记"
description: ""
category: 个人笔记
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [linux,host,dns,nslookup, dig]
---

# DNS的类型

| 类型 | 用途 |
|-------+---------------------|
| A   | 主机记录，名称到ip的映射 |
| SOA | 指出当前区域内谁是主DNS服务器 |
| NS  | 指出当前区域内有几个DNS服务器在提供服务  |
| CNAME | 别名记录，用于定义A记录的别名 |
| MX  | 邮件交换记录。指出当前区域内 SMTP邮件服务器IP |
| PTR | 反向解析。将IP解析为域名FQND　  |

# 常用的DNS工具

**`dig`**

    dig [options] FQDN [@server]

| 常用选项 | 含义                    |
|----------+-------------------------|
| @server       |  手动指定dns服务器
| -t type       |  指定查询的数据类型,MX,NS,SOA,A等
| -x            |  反向查询
| +trace        |  调试模式

这里只是一个简单的记录,这个篇[《dig挖出DNS的秘密》-linux命令五分钟系列之三十四](http://roclinux.cn/?p=2449)不错可以查看下.

**`host`**

    Usage: host [-aCdlriTwv] [-c class] [-N ndots] [-t 类型 ] [-W time] [-R number] [-m flag] hostname [dnsserver]

| 常用选项 | 含义                    |
|----------+-------------------------|
| -a       | 列出所有信息            |
| -l       | 进行区域传送            |
| -t type  | 指定查询类型            |

**`nslookup`**

nslookup用着不爽,不做总结.

后续会对dig做深入的总结.相关命令使用选项很多,但只要吃透了dns协议,记起来就不难了.命令都会提供指定domain,type,使用udp还是tpc,延迟,debug,以及输出控制的选项.

# 变更记录

|Why | Who | When |
|----|-----|------|
|创建文件,提醒日后完善内容|fishcired|2015-04-15 |
|放了挺久,都快忘记了该文档,简单整理下publish,驱动后续的完善|fishcired|2015-06-17 |
