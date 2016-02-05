---
layout: post
title: "linux系统中手动绑定IRQ"
description: ""
category: 性能调优
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [linux,性能调优,中断]
---

**查看中断**

![](/img/cat_interrupts.jpg)

**绑定中断**

如果发现大部分中断都落在单一的cpu上,没有充分利用多核特性,可以通过设置亲和性文件`/proc/irq/中断号/smp_affinity`开进行调节,进而改善系统性能.


**irqbalance**

irqbalance服务实现了自动均衡irq负载的功能,是在实践中大家都不推荐启用该服务,发现该服务启用,可考虑关闭.相关阅读可以参看下面文章.

- [深度剖析告诉你irqbalance有用吗？](http://blog.yufeng.info/archives/2422)
- [cpuspeed和irqbalance服务器的两大性能杀手](http://wubx.net/stop-irqbalance-and-cpuspeed/)

> 现在最新的irqbalance版本为1.0.9,可能已经进行了改进,具体可浏览下代码.

# 更多阅读

- [Linux 内核中断内幕](http://www.ibm.com/developerworks/cn/linux/l-cn-linuxkernelint/)

# 变更记录

|Why | Who | When |
|----|-----|------|
|创建文件,提醒日后完善内容|fishcired|2015-04-15 |
|简单整理,供日后复习 | fishcried | 2015-06-08 |
