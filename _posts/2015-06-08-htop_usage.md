---
layout: post
title: "linux系统性能概览工具(二): htop"
description: ""
category: 系统管理
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [系统管理,性能调优,htop]
---

# htop是什么?

    # whatis htop
    htop (1)             - interactive process viewer

htop是交互式的进程查看器,新一代的top工具,可以完全替换掉top.

**htop vs top**

- htop比top展示效果更加美观.颜色鲜明,而且可以上下左右移动,便于查看所有信息
- htop交互式的特性
   强大的交互式操作.可以不用退查htop进行strace,lsof,change affinity等操作
- htop更加人性化
   以上两点已经体现了人性化,而且支持鼠标操作.查看时排序也比top方便.
- top是自动自带,htop需要单独安装. 这个可能是top唯一的优势了


# htop的使用

**htop输出展示**

![htop](/img/htop.jpg)

htop与top两者的信息展示上还是很像的,可能第一眼看到会觉得htop的颜色鲜明,要好于top.

但是关于头部信息的展示,htop与top各有特点:

- top头部能看到cpu的详细信息,比如io,usr,sys等.但是如果需要看每个核的需要单独按下`1`.htop默认把各个核全部展示,但是不能具体到sys,usr,io等.好处是带有颜色展示,但是不利于对比查看.
- top关于任务的显示能看到总任务,运行中,睡眠中,僵死的.而htop看不到详细分类.
- 同样的内存信息,top显示也更为精确.而htop则主要以图来展示趋势.

**以上也能看出,top与htop在展示总体信息时,top讲究精细化(数据化),而htop则是可视化.两个各有优点.如果是性能调优,个人觉得通过top来获取全局数据更好,而htop则能快速掌握当前压力状态.**


**htop的帮助**

开启htop后按`h`就可查看htop的帮助页面.很简洁.

![htop](/img/htop_usage.jpg)


**常用**

- 显示
  - 方向键  通过方向键可以水平竖直移动显示区,方便查看进程命令信息
  - t/+/- 以树的方式显示进行,方便查看进程之间的关系,+/-用于展示关闭节点
- 过滤
  - / 通过字符快速搜索进程
  - \ 只显示与pattern匹配的进程,安静世界的
  - H/K 是否显示用户/内核线程,头部的tasks数目会变
  - u  通过用户过滤进程
  - pid 直接按数字当成pid,过滤进程
- 排序
  - `<>` 自行排序
  - M 根据内存排序
  - P 根据cpu排序
  - T 根据使用cpu时间排序
  - I 反向排序
- 交互
  - space,U 按space会高亮进程,再按取消; U会取消全部
  - k kill process
  - s 调用strace去attach进程,需要是root启动
  - l 获取该进程打开的文件描述符(lsof)
  - a 设置进程的cpu afinity
  - i 设置进程的io优先级
  - `[/]` 升降nice值, 降时需要是root


htop确实比top更加的人性化.但是无法完全替换top.top与htop使用场景往往是:

- 快速获取系统全局性能信息
- 通过对子系统排序,获取topN,定位到谁在消耗系统性能
- 通过搜索快速得到想要跟踪的进程

两者都能做到,而且top是系统自带的,满足需求即可.但是htop的交互式也很方便,比如kill,改变亲和性,查看打开的文件等,但是trace有点扯,不实用.

# 变更记录

|Why | Who | When |
|----|-----|------|
|创建文件,提醒日后完善内容|fishcired|2015-05-06 |
