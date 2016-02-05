---
layout: post
title: "linux commands tips: crontab"
description: ""
category: 系统管理
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [系统管理]
---

linux有个定期执行的服务cron,是一个非常听话的秘书．cron由crond服务与crontab命令组成．cron服务是可以根据
时间、日期、月份、星期的组合来调度对重复任务执行的守护进程。linux的cron服务是每隔一分钟去读取一次
/var/spool/cron，/etc/crontab,/etc/cron.d下面所有的内容。

# 1.关键文件

- 系统任务
    - `/etc/cron.d/*` 系统定期执行的任务，但不是按照小时，天，月的方式
    - `/etc/crontab` 方便日周月的一个例行工作.
        - `/etc/cron.hourly/*` 每小时执行的任务
        - `/etc/cron.daily/*`  每天
        - `/etc/cron.montyly/*` 每月
- 用户自己的任务
    - `/var/spool/cron/crontabs/$USER` 用户自己的定期任务

**服务管理**

    service crond start
    service crond stop
    service crond status

# 2.crontab的使用

     crontab -u //设定某个用户的cron服务，一般root用户在执行这个命令的时候需要此参数
     crontab -l //列出某个用户cron服务的详细内容
     crontab -r //删除某个用户的cron服务
     crontab -e //编辑某个用户的cron服务

具体到用户可以通过`-u`来指定．

**crontab格式**

    分钟　小时　日 　月　 星期　命令
    0-59  0-23  1-31 1-12 0-6   默认使用sh

**几种特殊的表示:**

- ＊ 所有
- ／ 每　/5 每５个单位
- － 表示区间　１-5 第一个到第五个单位
- ， 离散的单位


    # 每晚11:30
    30 23 * * * xxx
    # 每月的1,10,22日的３点
    0 3 1,10,22 * * xxx
    # 每两个小时
    0 */2 * * * xxx
    # 晚上11点到早上８点之间的每两个小时,早上８点
    ０　23-8/2,8 * * * xxx

> cron执行后信息输出到邮件系统,不会在终端显示．避免邮件系统收到信息过大．很多时候使用重定向技术．
`* * * * * ls -l > /dev/null 2 > &1`

# 变更记录

|Why | Who | When |
|----|-----|------|
|create|fishcired|2014-10-09 |

