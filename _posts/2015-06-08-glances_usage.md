---
layout: post
title: "linux系统性能概览工具(三):glances"
description: ""
category: 系统管理
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [系统管理,性能调优,glances]
---

非常喜欢glances这个工具,名字起的非常好,真像是时时刻刻有一只眼睛在上方盯着你.

使用glance非常简单,直接在使用命令即可`glances`.下面是输出图:

![](/img/glances.jpg)

通过上图可以看到,glances最吸引人的地方是将系统各个自己系统的数据全部展现出来了.真对的其`glances`这个词.而且一些系统消耗比较高的指标会变成红色.

- 绿色：OK（一切正常）
- 蓝色：CAREFUL（需要注意）
- 紫色：WARNING（警告）
- 红色：CRITICAL（严重）


**glances的特点**

上面的图就是glances最大的特点,能够将系统每个子系统的性能展现在一个界面,真的非常非常方便.


**glances的常用选项以及快捷键**

- a – 对进程自动排序
- c – 按 CPU 百分比对进程排序
- m – 按内存百分比对进程排序
- p – 按进程名字母顺序对进程排序
- i – 按读写频率（I/O）对进程排序
- d – 显示/隐藏磁盘 I/O 统计信息
- f – 显示/隐藏文件系统统计信息
- n – 显示/隐藏网络接口统计信息
- s – 显示/隐藏传感器统计信息
- y – 显示/隐藏硬盘温度统计信息
- l – 显示/隐藏日志（log）
- b – 切换网络 I/O 单位（Bytes/bits）
- w – 删除警告日志
- x – 删除警告和严重日志
- 1 – 切换全局 CPU 使用情况和每个 CPU 的使用情况
- h – 显示/隐藏这个帮助画面
- t – 以组合形式浏览网络 I/O
- u – 以累计形式浏览网络 I/O
- q – 退出（‘ESC‘ 和 ‘Ctrl&C‘ 也可以）


# glances的高级用法

**远程使用glances**

服务端:

    glances -s

会提示使用密码,也可以不使用(直接回车即可)

客户端:

    glances -c ip 

要要求输入密码,如果没有直接回车.这种c/s的模式挺有意思.

**获取原始数据,便于数据分析**

    glances -o CSV -f file.csv

随后数据就会写入file.csv文件,后续就可以将该数据作为原始数据进行自己的分析了.


# 变更记录

|Why | Who | When |
|----|-----|------|
|创建文件,提醒日后完善内容|fishcired|2015-06-05 |
