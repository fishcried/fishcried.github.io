---
layout: post
title: "OpenStack消息队列学习(一): AMQP基本概念"
description: ""
category: OpenStack
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [openstack,rabbitmq]
---

**什么是消息队列?**

- 一句话, 接收并转发消息的一个组件
- 复杂一点的类比就是邮件系统,你写了一封邮件,指定了一些收件人,然后邮件就发送到这些人的收件箱了.

**AMQP是什么?**

AMQP就是一套协议规范,类似于STMP,POP3等,规定了消息队列的实现规范.不然大家都有各自的实现,都不相互兼容,那还怎么愉快的玩耍那!

**AMQP的核心组件**

![amqp](/img/rabbitmq.png)

- 消息(Message):由Header与Body组成,由producer发出,经过exchange路由到相应的queue,然后consumer从queue中取走消费.
  - Header是由生产者添加的各种属性的集合(优先级等)
  - Body存储真正的数据,为json格式
- 队列(queue):装消息的容器．消息存储队列里，直到有消费者连接队列并取走为止
- 邦定(binding):将queue与exchange之间基于指定好的规则建立映射,就像建立网络路由表一样,通过binding动作规定了exchange如何路由消息到队列.
- 交换机(exchange)
  - exchage就是路由．每个消息都有一个路由键的属性(一个简单的字符串).交换机中有一些队列的邦定(路由规则).由此路由．
	- 交换机有多种类型
		- topic  routing_key与bind_key进行模式匹配
			- `*` 匹配一个词.`a.*`能匹配`a.b`
			- `#` 匹配一个或多个词. `a.#`能匹配`a.b.c`
		- direct routing_key与bind_key完全匹配
		- fanout 就是广播的作用．将消息发送到所有与该交互机邦定的队列．
    - headers 匹配各个键-值对的逻辑

![rabbit exchange](/img/rabbit_exchange.png)

**AMQP其他基础组件**

- 服务器(server) 接受客户端连接，实现amqp消息队列和路由功能的进程
- 虚拟主机gvirtual host)
	- 一个虚拟主机有一组交换机，队列和邦定
	- 用户只能在虚拟主机的粒度进行权限控制
	- 每一个服务器都有一个默认的虚拟主机`/`. 
- 连接(Connection) 客户端与broker之间的tcp连接
- 信道(Channel) 比连接更小的单位．创建连接后需要在其内创建信道发送消息．
    - 连接内可以有多个信道．这样设计是为了减少tcp连接．总不至于发送个消息就创建个tcp连接，然后就销毁
    - 客户端线程尽量共用连接，不共用channel 比连接更小的单位．创建连接后需要在其内创建信道发送消息．
	- 客户端线程尽量公用连接，不共用channel


# 推荐阅读

- [rabbitmq官网](http://www.rabbitmq.com/features.html)
- [兔子和兔子窝](http://blog.ftofficer.com/2010/03/translation-rabbitmq-python-rabbits-and-warrens/)
- [消息队列 RabbitMQ 入门介绍](http://www.ityen.com/archives/578)
- [AMQP 0-9-1 Model Explained](https://www.rabbitmq.com/tutorials/amqp-concepts.html)

# 变更记录

|Why | Who | When |
|----|-----|------|
|create|fishcired|2014-09-18|
|整理|fishcired|2014-12-16 |
|添加AMQP 0-9-1 Model Explained的链接| fishcried | 2015-03-19 |
