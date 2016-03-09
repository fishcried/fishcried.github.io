---
layout: post
title: "使用ELK调试OpenStack"
subtitle: "沈阳云计算大数据Meetup第一期Topic之一"
description: ""
category: 云计算
author: "fishcried"
header-img: "img/openstack-elk-bg.png"
tags: [openstack,elk]
---


> 之前简单的阐述过使用ELK调试OpenStack的思路,今天在"[沈阳云计算大数据meetup](http://mp.weixin.qq.com/s?__biz=MzA5ODU1ODIwOA==&mid=535930856&idx=1&sn=982f4fda81f5e677e8b84b60d5792b2b&scene=0#wechat_redirect)"上详细的分享了该主题，这里对之前的blog进行重写。
> 之所以使用ELK分析OpenStack日志，主要是OpenStack环境下的问题定位时查看日志是最有效的方式，而ELK成为目前日志分析的标配，自然而然的就行了尝试与实践. 这里是[PPT](http://pan.baidu.com/s/1jHkEtd8)下载地址.
>
> ps: 如果沈阳的朋友有好的议题欢迎来分享 ;-)

![](/img/elk-debug-openstack-topic.png)

**Agenda**

![](/img/elk-debug-openstack-agenda.png)


# What's ELK?

![](/img/elk-debug-openstack-001.png)


ELK提供了日志分析的一整套产品.正是这一整套解决方案让日志分析变得非常简单.

1. Logstash用于采集分析日志，设计的非常灵活，插件式的架构;
2. Elasticsearch用于存储分析，能够提供实时性的搜索性能.作者当初是为了给他妻子设计一个能够搜索菜谱的引擎，结果这个需求到现在也没完成，倒是鼓捣出了这么火的Elasticsearch,说好的不忘初心那!
3. Kibana用于可视化数据展示,数据分析，没有可视化展示是很难打动人的.


# Why use ELK to debug OpenStack?

在弄清为什么使用之前先来看看日常问题定位的模式.


**普通开发模式下的调试**

![](/img/elk-debug-openstack-002.png)

普通开发模式的调试环境往往是单点的，而且可以使用调试器，所以问题发生时可以replay/restart。一般来讲掌握了一些高级的调试技巧就能够成为bug killer.
下面举两个例子:

1. GDB自动定位死锁(上图左下).

   gdb能够执行脚本。获取lock&unlock函数的地址，对两个函数地址下断点，然后循环执行. 最有一次堆栈的输出就是死锁的发生的现场.里面的quit是为了ctrl+C退出.
2. GDB可视化数据包.

   之前做IPS开发时调试网络数据包非常困难，因为数据包被协议解析之后存放在全局的g_package变量中，具体的协议字段会使用很多宏，所以查看协议的变量非常的麻烦.
   后来就用GDB自定义了一个可视化数据包的命令，这样可以一键就打印出数据包各个字段的含义，类似上图中右下侧的样子，当然表格使用'+-----+-----+'这种方式表示的,
   这些命令给数据包的分析带来了很大的方便.在验证协议解析，以及特征检测时提高了很大的效率.


上面的两个例子对于C开发者，尤其是常常使用GDB的同学深有感触。开发者配合一些技巧解决一些问题还是很高效的.


**OpenStack调试模式**

![](/img/elk-debug-openstack-003.png)

OpenStack的调试非常的麻烦，节点分布式，很难用调试器，日志成为救命稻草，倒是日志真的很难看. 而且多节点查看日志常常会陷入下面的窘境:

1. 用xshell或者screen来查看不同节点的日志时需要不停的切换tag，经常会把自己迷失.
2. 用tmux多panes查看日志时发现屏幕不够大(Tmux还是好用的,Ctrl+B+Z全屏快捷键).
3. 更要命的是查看日志只能`tailf xxx.log`,没法停下来，感觉就像漫天雪花一样.

综上，普通开发模式的调试技巧已经不太对OpenStack试用，而日志成为调试OpenStack最有效的方式.所以就提了以下的需求，也是验证ELK是否能够胜任的关键.

![](/img/elk-debug-openstack-004.png)

Basic真的是基础，即使实现不了Advance，日志的中心化显示对于OpenStack调试来说都是福音。
而Advance需求正是ELK的强项，但是可视化展示的好不好还是要看日志的解析程度以及对展示图形对特征提取的敏感度。

关于专家感觉突然想到了《程序员的思维修炼：开发认知潜能的九堂课》,建议看看.


# How to debug OpenStack by ELK?

![](/img/elk-debug-openstack-005.png)

ELK并不是开箱即用的，日志怎么解析你要配置，数据怎么可视化还是需要配置，这也是为什么ELK虽然强大，但是依然很难看到具体ELK日志分析案例, 
一配置起来很复杂，打磨的事件非常长，二十和业务过于绑定. 这也是为什么ELK公司后续主推Beat产品通过分析网络流量来可视化数据，毕竟网络协议是标准的，这样产品可以做到"on the fly".

**HawkEye项目**

HawkEye项目就是基于ELK对OpenStack(Liberty)进行日志分析的配置.ELK是骨架，加上详细的配置就成了活生生的恐龙.而恐龙最终的样子就像是上面下部分图，通过多个维度去分析OpenStack
不同组件的日志.主机，日志级别，日志路径，相应时间，响应码等.

上面看起来还是很炫的，但是真正使用起来才更酷，尤其是对很长时间的日志进行分析(比如一个月),会发现很多惊喜.下面通过几张图来讲解下.


**日志中心化展示**

![](/img/elk-debug-openstack-006.png)

登陆kibana，在web上就可以查看排序搜索过滤所有的日志，再也不用多节点的登陆了。而且日志看起来非常的清晰.

**快速定位出错模块和主机**

![](/img/elk-debug-openstack-007.png)

上图纵坐标根据日志级别和HTTP响应码进行了划分，左边的统计的Host维度，右边的是日志PATH维度。所以当看到这两个图中有数据就要警惕了，肯定发生了不想发生的事情.
而通过图形可以很快的发现是哪个日志文件哪台机器的日志出现了问题，当用鼠标在相应的位置进行选择后，就会显示具体的日志内容。

有没有"一图在手,全局我有"的感觉!

**http请求分析**

![](/img/elk-debug-openstack-008.png)

上面是对OpenStack各个模块的http请求的一个统计，图中的左右两个最高点很有意思，左边的是horizon http请求字节数非常大，右边是glance相应响应有点久，结合起来可以发现应该是用户进行了一次镜像上传.

**API请求Top10**

![](/img/elk-debug-openstack-009.png)

**ssh行为分析**

![](/img/elk-debug-openstack-010.png)

上面的图是对将近一个月的auth.log中ssh日志的一个分析.左边是rhost成功和失败的分析,右边是登陆用户的分析.

1. 上面rhost中244网段是个异常数据，本来是不应该有的，后来调查发现是内部wifi网络正常方式.
2. 右侧发现有root用户登陆,正常系统是禁止root的，调查后发现是个别的合法访问.
3. 还有有意思的是两个图中黄绿曲线走势非常一致。其实是nova resize功能的一个调试.resize时compute节点会用nova用户通过ssh隧道同步disk数据,而相应的zone里只有两台compute节点.还都是一个往另一个resize.

可视化真的是很有意思很有用的事情，能够在海量数据中找打特征，也能在海量数据中找到异常，哪怕是一丁点都逃不过.

**horizon用户行为分析**

![](/img/elk-debug-openstack-011.png)

上面由图中可以看到哪些用于登陆成功了哪些失败了,这里面有一天出现了高峰.

这里还可以做得更详细，比如用户注册，这样就能掌握平台用户的增长情况.


上面展示了几张图，简单的解释了用途和含义。当OpenStack发生问题时，结合这些可视化数据，问题的原因往往就出来了，最起码出错点能够很快的定位到. 再通过中心化的web界面查看具体日志进行进一步定位,
整体的效率应该是有所提高的.

# Dive into HawkEye

![](/img/elk-debug-openstack-012.png)

前面解释了HawkEye项目，就是针对OpenStack L版本的日志解析和可视化配置.分成了三个子项目:

- [hawkeye-logstash](https://github.com/fishcried/hawkeye-logstash), 主要是logstash解析OpenStack的具体配置.
- [hawkeye-kibana](https://github.com/fishcried/hawkeye-kibana), 上面的Dashboard的具体配置
- hawkeye-ansible. 本来想把自动化的ELK搭建和配置完成，结果一直搁置了.

上面前两个项目已经放在了github上,readme也进行了详细的说明.考虑到项目对业务的针对性太强和L版的生命周期，后续应该不会进行什么更新了。
当初提交这两个项目也是对ELK可视化OpenStack日志的一次尝试，之前也看过一些使用ELK的案例，包括OpenStack社区对ELK的使用方式，可是在可视化
上都不是很满意，所以造了下轮子.希望项目项目中的配置可以提供一些参考，因为对任何项目的日志分析思路都是通用的.

云时代，任何可视化都是值得重视的.

![](/img/elk-debug-openstack-013.png)


# 变更记录

|Why | Who | When |
|----|-----|------|
|创建文件,提醒日后完善内容|fishcired|2015-12-02 |
|根据PPT分享重新整理|fishcired|2016-03-06 |
