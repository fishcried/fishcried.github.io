---
layout: post
title: "Debug OpenStack by ELK"
description: ""
category: 云计算
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [openstack]
---

> 本文简单阐述ELK使用思路，不涉及ELK搭建. 但是会提供基于Liberty版本的shipper配置以及kibana的dashboards. 

![](https://github.com/fishcried/hawkeye-logstash/raw/master/screenshort.png)

# OpenStack问题定位是否有更好的方式?

openstack很庞大，出现问题的时候更是让人畏惧，尤其是生产环境出问题。openstack系统问题定位是否有更好更高效的方式，这个问题大家应该都曾想过.在探讨这个问题之前，先看看openstack问题诊断现状。

**openstack问题定位的尴尬**

openstack系统出现问题时定位起来很麻烦，多节点，多服务，多组件。当出现问题时一般流程: 开很多tab, ssh到各个节点，`tail -f xxx.log`.

- 各个tab之间需要快速切换，这时候考验的眼力和手速
- 如果开启了debug,那日志的刷新真是美，若是绿字黑底的主题，完全可以再现黑客帝国的屏保了.

总之，定位问题时涉及的组件太多，而且生产环境无法replay,更没法下断点。查看日志是唯一的方式。但这种方式真的很虐。这时候ELK呼之欲出，在使用ELK之前，我通过tmux([相关配置可以参考](https://github.com/fishcried/linux-profile/blob/master/tmux/workspaces/liberty.yaml))来解决自动登录等问题，以及tmux等分屏功能来同时盯着日志.奈何没有大屏。

憋屈到一定时候，解决方式总是会有的。openstack社区自己也有个通过elk展示日志的[网站](http://logstash.openstack.org/#/dashboard/file/logstash.json), searchlight,monasca项目都使用到了相关技术.故此尝试了一下ELK, 以下就是使用ELK debug openstack的思路。

# 调试需求

需求很简单，下面第一点为基本，第二点为锦上添花。

**1. 能够集中的查看日志，不在需要单点登录，能够对日志进行控制，格式清晰**

- 如果不再需要分别登录各个服务节点，通过统一portal就可以查看想要的日志多好？
- 能够对日志进行快速的过滤，排序，分组,日志格式清晰，不要乱糟糟的一片，然后眯好几次眼睛才能看清


![](/img/log-entry-with-pretty-format.png)

上面是通过kibana的discover查看的日志,由于对日志的显示字段可以随意配置,所以可以在前端只查看你需要的日志.而且支持搜索,排序.这样就可以非常快速的定位日志. 对日志进行重放或者定位最重要的就是时间流，通过kibana可以非常轻松的只查看一段时间内的ERROR日志，这个是非常高效的. bingo!


**2. 通过日志信息能够了解整个系统的状态，系统的大局观**

有什么比图像还能提供更多的信息那？

![](/img/vsz-ips-bomb-by-paths.png)

这里我只是通过上面的一个visualize来展示可视化定位思路.下面是该图含义:

- 横轴是实践
- 纵轴从上到下分别
  - critical级别日志
  - error级别日志
  - warning界别日志
  - http response code是4xx
  - http response code是5xx
- 后侧的legend指示颜色日志的path来源

通过这个图就可以快速的查看哪个模块存在问题.真是一图胜千言。发现问题后,只要在图上点击就可以过滤出相关的日志,进行详细的定位。其实可以制作出更多的图，比如各个组件的api响应时间,horizon用户使用情况(注册,登录等行为),nova组件错误类型等.


# 相关项目

上面是一些思路的介绍，但是如果自己想尝试的话不是那么容易，因为ELK是一套技术框架，日志数据怎样解析，怎样展示都是需要自己做的。解析和展示做的不好. 使用elk也是没有什么意义。所以我创建了hawkeye项目, 基于elk对openstack(liberty)的日志进行处理.  hawkeye分为三个子项目,项目的README有详细的说明:

- [hawkeye-logstash](https://github.com/fishcried/hawkeye-logstash)
- [hawkeye-kibana](https://github.com/fishcried/hawkeye-kibana)
- hawkeye-andible TBD


# 变更记录

|Why | Who | When |
|----|-----|------|
|创建文件,提醒日后完善内容|fishcired|2015-12-02 |
