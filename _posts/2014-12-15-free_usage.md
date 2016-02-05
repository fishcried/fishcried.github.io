---
layout: post
title: "linux commands tips: free"
description: ""
category: 系统管理
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [性能调优,内存管理,系统管理]
---

free命令简单又很常用.使用free命令的关键在于理解buffers/cached是什么,不然很多人都会上来就吓一跳,"我的内存怎么就剩这么少".

一般使用free命令的时候加上单位,这样好读一点,如`free -m`显示如下:

                  total      used       free     shared        buffers     cached
    Mem:          3862       3063        799          0           182       1242
    -/+ buffers/cache:       1638       2223
    Swap:         7986          0       7986
    fishcried@kali:~/Workspaces/fishcried.github.io$

1. 第2行free为799,剩余的内存都不到1G,不要惊讶.同样可以看到buffers使用了182M,cached使用了1242M.大部分都被他们用了.这是系统的缓存机制,当你需要内存的时候,他们会释放的.buffer缓存dir信息,cached缓存文件内容信息,读入目录buffer会变大,读取文件cache会大.系统将他们缓存到内存中,下次读取的时候就会非常快了.linux会尽量使用内存的,不会让他发霉.
2. 注意第三行,前面有个`-/+`,什么意思那?因为buffers/cached是可用的内存,当系统需要的时候可以用的.所以这一样在计算内存使用和剩余是要把他们折兑的.
    1. 这时候used = 3063 - 182 - 1242 = 1639, 呀,怎么差了1M! 你忘记了,我们单位使用了M,随意这里恰巧出现了误差.
    2. 这时候free = 799 + 182 + 1242 = 2223.
3. 第四行表示swap的使用量,当used不为零的时候,可能是个坏消息.很可能系统的内存已经不够用了,这时候需要将暂时不用的信息粗放到disk,一边腾出更多的内存.对与高性能需求的场景,这是个灾难.
    1. used有值不代表一定在发生交换,需要使用`vmstate`或者`sar -W`来看下si,so的值.
    2. 高性能场景就直接停用swap吧,任性一点而已.可以参见[swap分区管理](/2014-12-15/swap_usage)

如果想手动释放掉cached的内存.可以如下: `echo 3 > /proc/sys/vm/drop_caches`

可以使用`watch -d free -m`来连续观察free变化

# 变更记录

|Why | Who | When |
|----|-----|------|
|create|fishcired|2014-10-30 |
|修改错别字|fishcired|2014-12-15 |
