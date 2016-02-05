---
layout: post
title: "OpenStack Murano概览"
description: ""
category: 云计算
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [openstack, murano]
---

# 1. Murano是什么(Mission)?

Murano提供Application catalog功能,使得第三方开发者与管理员可以发布各种云应用.而用户能够通傻瓜式部署可靠的应用环境.Murano提供了UI和API完成综合环境构建部署,整个生命周期的管理.但是真正部署工作由OpenStack编排组件Heat完成,Murano只是完成各种应用与服务的集成.


> **What's Application Catalog?**
>
> 1. A single point to publish all the different apps
> 2. Language, OS and platform agnostic
> 3. Browsable and categorised
> 4. Built-in deployment engine
> 5. Maintenance and management tools
> 6. Details statistical reports
> 7. Access control
> 8. Billing rules
>
> From [MIRANTIS的胶片](http://www.slideshare.net/tivelkov/murano-application-catalog-for-openstack?qid=7cbf303b-6ac6-4f3e-b189-f5dfe31ac021&v=default&b=&from_search=1)

# 2. 原理(Rationale)

在任何环境中安装第三方应用或服务都不是容易的,OpenStack的动态特点加剧了这种复杂性.Murano通过添加一个中间层来解决这个问题.这个中间层整合了第三方组件与OpenStack,这样OpenStack同时能够提供IaaS和Paas层服务.用户通过portal即可完成应用部署.

## 2.1 Murano架构图

![架构图](/img/murano_design_diagram.png)

从这个图可以看到murano做的是一个集成工作,通过heat来做编排工作,ceilometer进行信息收集从而完成应用计费.

**主要组件**

- Murano Metadata Service
    存储应用的元数据,应用相关的heat模版,软件配置,脚本,UI定义等.
- Murano API
    API Service,供UI调用.
- Murano Engine
    核心引擎,后端实现均在次.
- Murano Agent
    运行在虚机内,执行部署计划

## 2.2 Murano部署工作流

![murano workflow](/img/murano_workflow.png)

1. API通过消息队列给Engine下发环境部署命令.
2. Engine向API发送一系列的请求下载需要的packages.
3. Engine通过openstack内部组件(heat)部署需要的资源,如网络,路由,主机等.
4. 部署的主机内运行murano-agent服务,engine通过管理murano-agent来部署相应的应用,完成应用的安装配置等.

上面只是简化的流程,关于每一步的详细说明可以阅读[官方文档](https://murano.readthedocs.org/en/latest/image_builders/index.html).后续会通过代码来整理更为详细的工作流描述.

# 3. 更多阅读

以上只是原理上的了解,想真正玩起来murano,以下文档必须阅读掌握.

1. [安装文档](https://murano.readthedocs.org/en/latest/install/index.html)
  该文档主要是通过devstack进行安装.关于通过源码手动安装的并不完善.尤其是关于murano-dashboard与horizon的集成.需要动手实验.
1. [murano编程语言-YAQL](https://murano.readthedocs.org/en/latest/murano_pl/murano_pl_index.html)
  murano项目单独设计了一种语言YAQL给engine使用.同时定义了系统类与核心库.第三方开发者必须掌握才能制作应用.
1. [定义动态UI](https://murano.readthedocs.org/en/latest/articles/dynamic_ui.html)
  说明如何制作动态UI.其实就是如何描述了web表单,输入参数的获取.达到'on-the-fly'的效果.开发者也是必须掌握的.
1. [手动制作应用](https://murano.readthedocs.org/en/latest/articles/app_pkg.html)
  说明了应用制作流程.
1. [制作murano镜像](https://murano.readthedocs.org/en/latest/image_builders/index.html)
  因为虚拟里要运行murano-agent服务,所以需要定制镜像.该文档对镜像制作进行了说明,以及如何上传镜像以便于murano能够是被该镜像.

# 4. 参考文档

1. [Application Catalog Project Overview](https://wiki.openstack.org/wiki/Murano/ApplicationCatalog)
2. [Murano workflow](https://murano.readthedocs.org/en/latest/articles/workflow.html)
3. [MIRANTIS's PPT](http://www.slideshare.net/tivelkov/murano-application-catalog-for-openstack?qid=7cbf303b-6ac6-4f3e-b189-f5dfe31ac021&v=default&b=&from_search=1)

# 变更记录

|Why | Who | When |
|----|-----|------|
|创建文件,提醒日后完善内容|fishcired|2015-02-26 |
|完善内容|fishcired|2015-03-06 |
