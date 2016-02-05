---
layout: post
title: "Linux Swap分区管理以及使用原则"
description: ""
category: 性能调优
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [内存管理,性能调优]
---

>　对于swap的原则，个人比较极端。如果考虑性能，那么就不要用。如果有问题就早点发生，早点解决。

**系统什么时候进行swap转换?**

- 要看swappiness的值,该值默认值是60.
- swappiness=0的时候表示最大限度使用物理内存，然后才是swap空间，
- swappiness＝100的时候表示积极的使用swap分区，并且把内存上的数据及时的搬运到swap空间里面

以上也说明了不是系统内存不够了才会用swap,是根据swappiness这个值来判定的,所以很有可能内存剩余很多，但是却频繁的进行swap交换，导致系统性能很差.

**如何查看swap的交换状况?**

1. `vmstat` 查看对应的swap的si/so值
2. `sar -S/-W`

**swappiness的修改?**

- `echo 0 > /proc/sys/vm/swappiness` 重启后会失效
- `sysctl vm.swappiness=0`这个重启后也会失效
- 编辑文件`/etc/sysctl.conf`添加`vm.swappiness=value`

**那么如何关闭/开启swap功能那?**

    swapon/swapoff

**如何强之swap刷新?**

    `swapoff -a && swapon -a`

**如果已经设置了swap分区,如何删除?**

关闭后,从/etc/fstab中删掉

# 变更记录

|Why | Who | When |
|----|-----|------|
|create|fishcired|2014-10-30 |
|修改错别字|fishcired|2014-12-15 |
|修改标题,整理内容|fishcired|2015-09-23 |
