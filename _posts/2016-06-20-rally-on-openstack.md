---
layout: post
title: Rally与OpenStack集成
subtitle:
description:
category: OpenStack
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags:
    - openstack
    - rally
---

# Rally与OpenStack集成

最近在探索怎样将rally与OpenStack结合起来,成熟后会进行整理。这里先记录几点rally使用注意点吧。

**使用独立用户运行rally**
 
> 使用已经建立的用户能够隔离测试环境.主要是在环境配置时指定users,然后在task的context中去除user信息.

 
 ```
$ cat existing.json
{
    "type": "ExistingCloud",
    "auth_url": "http://192.168.250.7:5000/v2.0/",
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
# rally deployment create --file=existing.json --name=existing
```

上面deployment定义时指定了以后task运行时使用的用户，这样测试数据就能个离开。

**rally对OpenStack环境进行功能验证**

1. rally整个tempest来做功能验证非常方便了，需要注意的是在安装时可以通过--source指定维护的tempest用例分支.如果都是跟着官方的肯定会有很多用例不过的.
```
$ rally verify install
$ rally verify install --deployment <UUID or name of a deployment>
$ rally verify install --source /home/ubuntu/tempest/
$ rally verify install --source https://github.com/openstack/tempest.git
$ rally verify install --source /home/ubuntu/tempest/ --version 198e5b4b871c3d09c20afb56dca9637a8cf86ac8
$ rally verify install --source /home/ubuntu/tempest/ --version 10.0.0
$ rally verify install --source /home/ubuntu/tempest/ --version 10.0.0 --system-wide
```
2. 整理所有集成的task场景到一个文件中，将task的timers设置为1，这样也可以做到功能验证的目的.

> 上面两种方式，前者在于测试的精度非常细，是api级别(单元测试),而且控制粒度也比较细，冒烟测试,特定项目(通过--set来指定)测试等.
> 
> 后者的精度是场景，是功能级别。如果是为了验证一套刚刚部署好的环境可用行，比较推荐这种，简明扼要。但是最CICD，前者则是必须的.
> 
> 使用tempest还有一种优势就是第二次运行时，可以只运行上一次失败的用例.
