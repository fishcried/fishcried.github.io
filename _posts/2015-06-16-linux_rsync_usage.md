---
layout: post
title: "linux下rsync命令使用"
description: ""
category: 个人笔记
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [linux,rsync]
---


![](/img/rsync_logo.png)

> rsync is an open source utility that provides fast incremental file transfer. rsync is freely available under the GNU General Public License and is currently being maintained by Wayne Davison. 



# 使用

    Local:  rsync [OPTION...] SRC... [DEST]

           Access via remote shell:
             Pull: rsync [OPTION...] [USER@]HOST:SRC... [DEST]
             Push: rsync [OPTION...] SRC... [USER@]HOST:DEST

           Access via rsync daemon:
             Pull: rsync [OPTION...] [USER@]HOST::SRC... [DEST]
                   rsync [OPTION...] rsync://[USER@]HOST[:PORT]/SRC... [DEST]
             Push: rsync [OPTION...] SRC... [USER@]HOST::DEST
                   rsync [OPTION...] SRC... rsync://[USER@]HOST[:PORT]/DEST


**选项说明**

| 选项 | 含义 |
| -t | 同步modify time |
| -I | 放弃 quick check策略,一个一个的同步 |
| -v | 输出更多信息 | 
| -z | 如果通过网络的话,进行数据压缩 |
| -r | recursive | 
| -l | 同步同步软连接 | 
| -p | 同步权限 |
| -g/-o | 同步group和owner属性 |
|-a | -riptgoD |
| --delete | 目的与源保持一致,多余的会被删除 |
| --exclude | 指定哪些不同步 |

# 变更记录

|Why | Who | When |
|----|-----|------|
|创建文件,提醒日后完善内容|fishcired|2015-06-16 |
