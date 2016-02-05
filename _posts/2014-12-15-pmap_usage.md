---
layout: post
title: "pmap查看进程内存分布"
description: ""
category: 系统管理
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [调试,系统管理,进程]
---

如果发现一个进程占用的内存非常大,或者存在内存泄露,可以使用pmap来查看该进程的内存映射,进行初步判断.pmap使用是少见的简单.

# 1. 选项说明

- -x 显示拓展格式
- -d 显示设备格式


**拓展格式**

更喜欢拓展的格式,所以对拓展的格式进行说明.
  
    fishcried@kali:~$ pmap -x 4751
    4751:   -bash
    Address           Kbytes     RSS   Dirty Mode   Mapping
    0000000000400000     916     716       0 r-x--  bash
    ......
    00007fffaa366000     132      32      32 rw---    [ stack ]
    00007fffaa3c5000       8       4       0 r-x--    [ anon ]
    ffffffffff600000       4       0       0 r-x--    [ anon ]
    ----------------  ------  ------  ------
    total kB           23560    6548    4684

|Address | Kbytes | Rss | Dirty | Mode Mapping|
|---------+-------+----+---------+----+-------|
|起始地址|大小 | 实际占用物理内存大小 | 脏页数 | 权限 | 名字 |

`pmap`命令读取了`/proc/pid/maps`和`/porc/pid/smaps`文件.可以直接查看以上两个文件.smaps的格式更为详细可读.

# 变更记录

|Why | Who | When |
|----|-----|------|
|创建文件,提醒日后完善内容|fishcired|2014-12-15 |
