---
layout: post
title: "Kali安装sogou输入法"
description: ""
category: 个人笔记
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [issue]
---

linux下，输入法使用是个心病．搜狗出了个linux的版本,最近有时间正好折腾下．因为使用的是kali，官方不支持，只能手动来．简单记录．

# 1. 安装记录

**1 卸载fcitx相关软件包**

如果系统安装了fcitx相关东西,需要卸载，因为源的fcitx版本太低．请谨慎，后果自负．

	apt-get purge fcitx-*

**2 手动下载最新的fcitx软件包**

手动麻烦，且安装顺序有依赖，上个脚本．

	#!/bin/bash
	#
	# The MIT License (MIT)
	# Copyright (c) 2014 fishcried(tianqing.w@gmail.com)
	#
	
	pkgs="fcitx-libs_4.2.8.4-3~bpo70+1_amd64.deb \
		fcitx-libs-qt_4.2.8.4-3~bpo70+1_amd64.deb \
		fcitx-bin_4.2.8.4-3~bpo70+1_amd64.deb \
		fcitx-data_4.2.8.4-3~bpo70+1_all.deb \
		fcitx-modules_4.2.8.4-3~bpo70+1_amd64.deb \
		fcitx-module-dbus_4.2.8.4-3~bpo70+1_amd64.deb \
		fcitx-module-kimpanel_4.2.8.4-3~bpo70+1_amd64.deb \
		fcitx-module-lua_4.2.8.4-3~bpo70+1_amd64.deb \
		fcitx-module-x11_4.2.8.4-3~bpo70+1_amd64.deb \
		fcitx_4.2.8.4-3~bpo70+1_all.deb \
		fcitx-tools_4.2.8.4-3~bpo70+1_amd64.deb \
		fcitx-ui-classic_4.2.8.4-3~bpo70+1_amd64.deb "
	
	ftpaddr="http://ftp.debian.org/debian/pool/main/f/fcitx"
	
	for pkg in $pkgs
	do
		echo "wget  $ftpaddr/$pkg && dpkg -i $pkg "
	done
	
	#dpkg -i $1

**3. 下载[官方包](http://pinyin.sogou.com/linux/),通过`dpkg` -i 来安装.**

**4. 安装缺少的库**

启动的时候发现缺少的一个库,安装一下.

	apt-get install blua5.2-0

# 2. 遇到的问题

输入框是黑的，非常难看．需要安装`xcompmgr`美化解决．

	apt-get install xcompmgr
	
	cat .config/autostart/xcompmgr.desktop
	
	[Desktop Entry]
	Type=Application
	Encoding=UTF-8
	Name="xcompmgr"
	Comment=""
	Exec="xcompmgr"
	Hidden=false
	NoDisplay=false
	Terminal=false

# 变更记录

|Why | Who | When |
|----|-----|------|
|create|fishcired|2014-08-29 |
|将之前的草稿整理下|fishcired|2014-09-11 |
