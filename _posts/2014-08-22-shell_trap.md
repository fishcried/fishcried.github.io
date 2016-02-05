---
layout: post
title: "Shell脚本编写细节之信号处理"
description: ""
category:  程序设计
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [shell,信号处理]
---

信号处理是比不可少的，很多时候用来保证环境的清洁，这也是程序的基本素养。

对信号有有三种响应方式:忽略，执行默认动作，捕获，并自定义动作。

**1. 忽略**

	trap "" signal-list

**2. 默认动作**

	trap signal-list

**3. 自定义动作**

	trap "commands" signal-list

支持的信号`kill -l`,`SIGKILL`，`SIGSTOP`不能捕获。

# 变更记录

|Why | Who | When |
|----|-----|------|
|整理资料|fishcired|2014-08-22|

