---
layout: post
title: OpenStack测试工具Rally简介
subtitle:
description:
category: OpenStack
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags:
    - openstack
    - rally
---

# OpenStack测试工具Rally简介

## Rally是什么?

> Rally is a benchmarking tool that automates and unifies multi-node OpenStack deployment, cloud verification, benchmarking & profiling. It can be used as a basic tool for an OpenStack CI/CD system that would continuously improve its SLA, performance and stability.

以上是官方说明，从中我们可以知道Rally是一个OpenStack基准测试工具，可以用于功能验证，基准测试和性能分析。

![](/img/Rally-Actions.png)

通过上面这个图能够更好的理解Rally能干什么：

1. Deploy(部署) 

   这里其实不是说用rally能够直接部署一套OpenStack，而是Deploy模块能够结合devstack等项目来部署OpenStack环境。或者直接引用已经部署好的环境，这样就能够基于这套环境进行相应的测试了。
   
1. Verify(验证) 

   可以对上面部署的环境(deployment)进行验证，其实是集成的tempest。如果单独使用tempest项目的话会发现非常的麻烦，这里rally对其做了集成，整个安装配置使用都非常简单，而且能够生成html结果，非常的好用。
   
1. Benchmark(基准测试) 这里Rally提供了非常多的测试用例，能够非常容易的对OpenStack环境进行评估。测试用例文件可以是json或者yaml格式，非常的方便。最主要的是能够生成可视化的结果。

1. Generate Report(报表生成) 这个报表生成才是终极功能，而且还能对两次结果进行对比

**OpenStack中Rally使用场景**

下面三种场景可以仔细的看一下,不做解释了.

![](/img/Rally-UseCases.png)


### Rally架构


![](/img/Rally_Architecture.png)

上图主要是说Rally部署时可以有两种模式:

1. 最为App部署，其实就是个CLI
2. 作为Service部署，这样整个团队可以使用。这种要是直接有个web UI就好了。


> service模式只是个愿景: https://answers.launchpad.net/rally/+question/253017


## Rally安装试用
 
 rally项目提供了安装脚本，可以很方便的安装；也提供了dockerfile可以自己构建docker image。直接上docker hub上pull相应的image也可以。由于docker太方便了，所以下面就用docker的方式吧。
 
 
### 使用docker安装rally

```
# mkdir ~/rally
# chown 65500 ~/rally
# docker pull rallyforge/rally
# docker run -t -i -v ~/rally:/home/rally rallyforge/rally
# echo 'alias dock_rally="docker run -t -i -v ~/rally:/home/rally rallyforge/rally"' >> ~/.bashrc

# 第一次使用时需要重建数据库
# docker_rally
root@5716a367e606:~# rally-manage db recreate   
 ```
 
> 上面的chown步骤非常重要，否则会出现权限问题. 

### 添加deployment进行benchmark测试

将已经存在的openstack添加到rally中，可以参照下面的json文件修改。我预先建立了rally的测试用户，这样能隔离开测试数据。

**定义环境文件**
```
$ cat existing.json
{
    "type": "ExistingCloud",
    "auth_url": "http://192.168.xxx.7:5000/v2.0/",
    "region_name": "RegionOne",
    "endpoint_type": "public",
    "admin": {
        "username": "rally_admin",
        "password": "111111",
        "tenant_name": "rally_test"
    },
    "users": [
        {
            "username": "rally_test1",
            "password": "123456",
            "tenant_name": "rally_test"
        },
        {
            "username": "rally_test2",
            "password": "123456",
            "tenant_name": "rally_test"
        }
    ]
}
```

```
# 创建deployment
#rally deployment create  --file existing.json --name test-env1

# 查看deployment
# rally deployment list
+--------------------------------------+----------------------------+-------------+------------------+--------+
| uuid                                 | created_at                 | name        | status           | active |
+--------------------------------------+----------------------------+-------------+------------------+--------+
| 5b225ece-537e-493e-8f19-f57709a739a5 | 2016-06-17 02:01:59.026426 | test-env1 | deploy->finished | *      |
+--------------------------------------+----------------------------+-------------+------------------+--------+

```

**跑测试用例**

自己手动创建一个task文件,文件格式为yaml,json也行，但是我觉得yaml方便.


```
# cat task1.yaml 
---
# Authenticate
Authenticate.validate_nova:
  - args:
      repetitions: 2
    runner:
      type: "constant"
      times: 100
      concurrency: 2
      
      
```
```
# rally task start task1.yaml 
--------------------------------------------------------------------------------
 Preparing input task
--------------------------------------------------------------------------------

Input task is:
---
# Authenticate
Authenticate.validate_nova:
  - args:
      repetitions: 2
    runner:
      type: "constant"
      times: 100
      concurrency: 2

Task syntax is correct :)
2016-06-20 07:12:53.049 3549 INFO rally.task.engine [-] Task 3432752c-972e-4c2c-a483-d6ea6dbb202c | Starting:  Task validation.
2016-06-20 07:12:53.290 3549 INFO rally.task.engine [-] Task 3432752c-972e-4c2c-a483-d6ea6dbb202c | Starting:  Task validation of scenarios names.
2016-06-20 07:12:53.293 3549 INFO rally.task.engine [-] Task 3432752c-972e-4c2c-a483-d6ea6dbb202c | Completed: Task validation of scenarios names.
2016-06-20 07:12:53.294 3549 INFO rally.task.engine [-] Task 3432752c-972e-4c2c-a483-d6ea6dbb202c | Starting:  Task validation of syntax.
2016-06-20 07:12:53.297 3549 INFO rally.task.engine [-] Task 3432752c-972e-4c2c-a483-d6ea6dbb202c | Completed: Task validation of syntax.
2016-06-20 07:12:53.297 3549 INFO rally.task.engine [-] Task 3432752c-972e-4c2c-a483-d6ea6dbb202c | Starting:  Task validation of semantic.
2016-06-20 07:12:53.298 3549 INFO rally.task.engine [-] Task 3432752c-972e-4c2c-a483-d6ea6dbb202c | Starting:  Task validation check cloud.

```

**查看结果**
```
rally task report --out=report1.html --open

```

![](/img/rally-task-task1.png)

> 测试用例其实不用自己写,rally项目自己已经携带了很多现成的用例.在`samples/tasks/scenarios`目录下.


## 编写task input文件


任务格式如下:

```
{
    "args": { <参数> },
    "runner": { <运行次数和并发相关控制> },
    "context": { <上下文参数> },
    "sla": { <指明什么情况下该测试算ok> }
}

# 例子
NovaSecGroup.boot_and_delete_server_with_secgroups:
  - args:
      flavor:
        name: "m1.small"
      image:
        name: "ubuntu14.04"
      security_group_count: 10
      rules_per_security_group: 10
    runner:
      type: "constant"
      times: 100
      concurrency: 2
    context:
      network:
        start_cidr: "100.1.0.0/26"
```

> 多个用例可以写在同一个文件中，这样生成报表的时候比较方便.语法如下：

```
{
   "<ScenarioName1>": [<benchmark_config>, <benchmark_config2>, ...]
   "<ScenarioName2>": [<benchmark_config>, ...]
}
```

**模板与传参**

任务文件中可以使用jinjia2语法,自然就可以使用变量了.

```
\{\% set image_name = image_name or "^cirros.*uec$" \%\}
  NovaServers.boot_and_delete_server:
    -
      args:
        flavor:
            name: "m1.tiny"
        image:
            name: {{image_name}}
      runner:
        type: "constant"
        times: 2
        concurrency: 1
      context:
        users:
          tenants: 1
          users_per_tenant: 1
    ...
```

启动任务时传参

```
rally task start task.yaml --task-args 'image_name: "^cirros.*uec$"'
```

## Rally Cli使用

rally cli使用起来非常的简单,主要有`task`,`deploymnet`,`verify`几个子命令,每个命令提供了非常多的参数,所以多用help命令查看参数信息.

你所需要的，基本都被想到了，参数都是现成的.
