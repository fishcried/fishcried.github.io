---
layout: post
title: "调优: 使用sar收集系统性能数据"
description: ""
category: 性能调优
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [性能调优,sar]
---

**sar是一个非常优秀的工具,为什么那?**

> 1. 一站式服务.sar基本可以获取各个子系统的信息.主要是读取proc文件系统,然后解析.有种一sa在手,天下我有的感觉.
> 2. sar每个10分钟就将统计信息写入统计文件,任何时候你都可以查看.信息收集这一点非常重要,系统出问题的时候常常向前回溯的.

当然,sar由于提供"一站式"服务,所以选项非常多,掌握起来有点麻烦,不过有个印象即可,随时换出你的男仆就好了.下面一张sar的总结图非常精彩,出自大神[Brendan Gregg](http://www.brendangregg.com/linuxperf.html).

![Berendan Gregg大牛总结的图](/img/sar.jpeg)

通过这张图可以看到,sar获取的信息覆盖了各个子系统.


# 1. sar选项总结

**常用格式**

    sar [options] [-A] [-o file]  t      [n]
    #             全部           间隔    次数

- 控制选项
    - -s [hh:mm:ss]: 设置报告显示的开始时间
    - -e：设k显示报告的结束时间
    - -f：从指定文件提取报告
    - -i：设状态信息刷新的间隔时间
- 内存管理
    - -B: 显示换页状态
    - -H: 大页统计
    - -r: 报告内存信息
    - -R：显示内存状态
    - -S: swap使用信息
    - -W: 交换统计
- cpu与调度相关
    - -P {cpu [,...] \| ALL}：报告CPU的状态
    - -q: 报告运行队列信息
    - -u [ALL]：显示CPU利用率
    - -w：显示进程创建与交换信息
    - -I {int [,...] \| SUM \| ALL \| XALL}: 显示中断情况
- 文件系统
    - -b: 显示I/O速率
    - -d：显示每个块设备的状态
    - -v：显示索引节点，文件和其他内核表的状态
    - -n NFS
    - -n NFSD
- 网络系统
    - -n { keyword [,...] \| ALL } :显示网络信息. keyword(DEV,EDEV,NFS,NFSD,SOCK,IP,EIP,ICMP,EICMP,TCP,ETCP,UDP...)
- 其他
    - -A：显示所有的报告信息
    - -m { keyword [,...] \| ALL} 显示电源信息.keyword(CPU,FAN,FREQ,IN,TEMP,USB)
    - -y: tty设备

# 2. SAR使用举例

## 2.1 cpu调度评估

**1 `sar -P ALL 1 1`或者`sar -u 1 1`查看cpu整体状态**

    root@kali:~# sar -P ALL 1 1
    Linux 3.14-kali1-amd64 (kali)   12/15/2014      _x86_64_        (4 CPU)

    01:33:47 PM     CPU     %user     %nice   %system   %iowait    %steal     %idle
    01:33:48 PM     all      1.25      0.00      0.25      0.00      0.00     98.50
    01:33:48 PM       0      0.00      0.00      1.00      0.00      0.00     99.00
    01:33:48 PM       1      4.00      0.00      0.00      0.00      0.00     96.00
    01:33:48 PM       2      1.00      0.00      0.00      0.00      0.00     99.00
    01:33:48 PM       3      0.00      0.00      1.98      0.00      0.00     98.02

    Average:        CPU     %user     %nice   %system   %iowait    %steal     %idle
    Average:        all      1.25      0.00      0.25      0.00      0.00     98.50
    Average:          0      0.00      0.00      1.00      0.00      0.00     99.00
    Average:          1      4.00      0.00      0.00      0.00      0.00     96.00
    Average:          2      1.00      0.00      0.00      0.00      0.00     99.00
    Average:          3      0.00      0.00      1.98      0.00      0.00     98.02

`-u`选项显示的整体的情况, `-u ALL`会显示更多列的信息哦,`-P`可以分core来显示.关于cpu利用率启示需要关注user,system,iowait字段.

- user高 用户空间进程造成
- system高 系统调用频繁
    可以用strace等工具进一步查看那个系统造成,程序是否需要优化
- iowait io可能存在问题
    可以利用`iostat -x`,`sar -d`,`iotop`进一步查看

> 1. steal只有在虚拟机中才有可能有值,表示被hyperv偷走的时间

**2 `sar -q 1 1`查看调度情况**

    root@kali:~# sar -q 1 1
    Linux 3.14-kali1-amd64 (kali)   12/15/2014      _x86_64_        (4 CPU)

    01:35:16 PM   runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15   blocked
    01:35:17 PM         0       486      0.15      0.24      0.27         0
    Average:            0       486      0.15      0.24      0.27         0

| 项目 | 说明
|------+-----
| runq-sz | 运行队列长度
| plist-sz | 任务列表中的任务数,包含了线程
| ldavg-1/5/15 | 均衡负载
| blocked | 当前被io阻塞的任务数


**3 `sar -w 1 3`查看进程的创建与上下文交换情况**

    root@kali:~# sar -w 1 3
    Linux 3.14-kali1-amd64 (kali)   12/15/2014      _x86_64_        (4 CPU)

    02:00:00 PM    proc/s   cswch/s
    02:00:01 PM      5.00   1784.00
    02:00:02 PM      0.00    684.00
    02:00:03 PM      0.00    616.00
    Average:         1.67   1028.00

**4 `sar -I ALL 3 1`查看中断情况**

    root@kali:~# sar -I ALL 1 1
    Linux 3.14-kali1-amd64 (kali)   12/15/2014      _x86_64_        (4 CPU)

    01:32:15 PM      INTR    intr/s
    01:32:16 PM         0      0.00
    01:32:16 PM         1      0.00
    01:32:16 PM         2      0.00
    01:32:16 PM         3      0.00
    01:32:16 PM         4      0.00
    01:32:16 PM         5      0.00
    01:32:16 PM         6      0.00
    01:32:16 PM         7      0.00
    01:32:16 PM         8      0.00
    01:32:16 PM         9      0.00
    01:32:16 PM        10      0.00
    01:32:16 PM        11      0.00
    01:32:16 PM        12      0.00
    01:32:16 PM        13      0.00
    01:32:16 PM        14      0.00
    01:32:16 PM        15      0.00

## 2.2 内存相关

**1. `sar -r 1 1 `查看内存使用状况**

    root@kali:~# sar -r 1 1
    Linux 3.14-kali1-amd64 (kali)   12/15/2014      _x86_64_        (4 CPU)

    02:02:26 PM kbmemfree kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact
    02:02:27 PM   1506888   2448124     61.90    134384    815160   3770280     31.07   1528428    712184
    Average:      1506888   2448124     61.90    134384    815160   3770280     31.07   1528428    712184

| 项目 | 说明
|------+-----
|kbmemfree  | 可使用的内存(单位k)
|kbmemused  | 已经使用的内存 (单位k)
|%memused   | 使用内存的百分比
|kbbuffers  | buffers占用的内存
|kbcached   | cached占用的内存
|kbcommit   | 当前需要的RAM+swap量
|%commit    | 百分比
|kbactive   | 活跃内存量
|kbinact    | 非活跃内存量

**2. `sar -R 1 1`**

    root@kali:~# sar -R 1 1
    Linux 3.14-kali1-amd64 (kali)   12/15/2014      _x86_64_        (4 CPU)

    02:02:34 PM   frmpg/s   bufpg/s   campg/s
    02:02:35 PM  -1738.00      0.00   1940.00
    Average:     -1738.00      0.00   1940.00

| 项目 | 说明
|------+-----
|frmpg/s   | 每秒空闲页数,负值表示系统申请了
|bufpg/s   | 每秒buffer占用的页数
|campg/s | 每秒cache占用的页数

**3. 查看swap使用状况**

    root@kali:~# sar -S 1 1
    Linux 3.14-kali1-amd64 (kali)   12/15/2014      _x86_64_        (4 CPU)

    02:03:35 PM kbswpfree kbswpused  %swpused  kbswpcad   %swpcad
    02:03:36 PM   8178684         0      0.00         0      0.00
    Average:      8178684         0      0.00         0      0.00


    root@kali:~# sar -W 1 1
    Linux 3.14-kali1-amd64 (kali)   12/15/2014      _x86_64_        (4 CPU)

    02:03:42 PM  pswpin/s pswpout/s
    02:03:43 PM      0.00      0.00
    Average:         0.00      0.00


**4. `sar -B 1 3`查看换页统计**

    root@kali:~# sar -B 1 3
    Linux 3.14-kali1-amd64 (kali)   12/15/2014      _x86_64_        (4 CPU)

    11:19:42 AM  pgpgin/s pgpgout/s   fault/s  majflt/s  pgfree/s pgscank/s pgscand/s pgsteal/s    %vmeff
    11:19:43 AM      0.00      0.00    168.00      0.00    101.00      0.00      0.00      0.00      0.00
    11:19:44 AM      0.00     12.00    115.00      0.00    107.00      0.00      0.00      0.00      0.00
    11:19:45 AM      0.00      0.00    158.00      0.00    119.00      0.00      0.00      0.00      0.00
    Average:         0.00      4.00    147.00      0.00    109.00      0.00      0.00      0.00      0.00

| 项目 | 说明
|------+-----
|pgpgin/s | 每秒从磁盘paged到内存的kilobytes
|pgpgout/s| 每秒paged到磁盘的kilobytes
|fault/s  | 每秒page faults数量(major +minor)
|majflt/s  | major faults number per second
|pgfree/s | free pages number per second
|pgscank/s | Number of pages scanned by the kswapd daemon per second
|pgscand/s | Number of pages scaned directly per second
|pgsteal/s    | 每秒从pagecache和swapcache回收的pages
|%vmeff | pgsteal / pgscan

## 2.3 io系统评估

**1. `sar -b 1 3`查看io请求**

    root@kali:~# sar -b 1 3

    Linux 3.14-kali1-amd64 (kali)   12/15/2014      _x86_64_        (4 CPU)

    11:11:31 AM       tps      rtps      wtps   bread/s   bwrtn/s
    11:11:32 AM      0.00      0.00      0.00      0.00      0.00
    11:11:33 AM     18.00      0.00     18.00      0.00    152.00
    11:11:34 AM      0.00      0.00      0.00      0.00      0.00

    Average:         6.00      0.00      6.00      0.00     50.67

| 项目 | 说明
|------+-----
|tps    |  每秒向物理设备发起的I/O请求数,iops
|rtps   | 每秒向物理设备发起的读I/O请求数
|wtps   | 每秒向物理设备发起的写I/O请求数
|bread/s   |  每秒从物理设备去读的块个数
|bwrtn/s   |  每秒从物理设备去写的块个数

**2. `sar -d 1 3`查看块设备性能**

    root@kali:/etc/cron.d# sar -d 1 3
    Linux 3.14-kali1-amd64 (kali)   12/15/2014      _x86_64_        (4 CPU)

    11:05:06 AM       DEV       tps  rd_sec/s  wr_sec/s  avgrq-sz  avgqu-sz     await     svctm     %util
    11:05:07 AM    dev8-0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00

    11:05:07 AM       DEV       tps  rd_sec/s  wr_sec/s  avgrq-sz  avgqu-sz     await     svctm     %util
    11:05:08 AM    dev8-0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00

    11:05:08 AM       DEV       tps  rd_sec/s  wr_sec/s  avgrq-sz  avgqu-sz     await     svctm     %util
    11:05:09 AM    dev8-0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00

    Average:          DEV       tps  rd_sec/s  wr_sec/s  avgrq-sz  avgqu-sz     await     svctm     %util
    Average:       dev8-0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00

| 项目 | 说明
|------+-----
|tps   |  每秒向物理设备发起的I/O请求数
|rd_sec/s   | 每秒读取的扇区数
|wr_sec/s   | 每秒写入的扇区数
|avgrq-sz|   设备扇区请求的平均大小
|avgqu-sz   | 设备队列请求的平均大小
|await   | 平均每次io设备操作的等待时间,毫秒
|svctm|  平均每次io设备操作的服务时间,毫秒
|%util   | 设备io请求期间的cpu时间的百分比,一秒钟百分之几的时间用于io操作

一般比较关注svctm,await.一般svctm小于await.如果两者接近,表示io基本没有等待,性能很好.如果await远远大于svctm,可能是io队列较长.如果util比较接近100%,表示系统满载,可能磁盘性能存在性能瓶颈.

`iostat -x`也能提供详细的io信息.


**3. `sar -v 1 1`查看索引信息**

    root@kali:/etc/cron.d# sar -v 1 1
    Linux 3.14-kali1-amd64 (kali)   12/15/2014      _x86_64_        (4 CPU)

    02:20:54 PM dentunusd   file-nr  inode-nr    pty-nr
    02:20:55 PM    104936      7904     46795        16
    Average:       104936      7904     46795        16

| 项目 | 说明
|------+-----
|dentunusd   |  Number of unused cache entries in the direcotry cache
|file-nr  | Num ber of file handlers used by the system
|inode-nr    | Number of inode handlers used by the system
|pty-nr | Number of pseudo-terminals used by the system.

## 2.4 网络相关

**1. `sar -n DEV`查看网口数据统计**

    root@kali:~# sar -n DEV 1 1
    Linux 3.14-kali1-amd64 (kali)   12/15/2014      _x86_64_        (4 CPU)

    02:30:10 PM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s
    02:30:11 PM      eth0     28.00     20.00      7.87      1.43      0.00      0.00      0.00
    02:30:11 PM        lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00
    02:30:11 PM    virbr0      0.00      0.00      0.00      0.00      0.00      0.00      0.00
    02:30:11 PM     wlan0      0.00      0.00      0.00      0.00      0.00      0.00      0.00

    Average:        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s
    Average:         eth0     28.00     20.00      7.87      1.43      0.00      0.00      0.00
    Average:           lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00
    Average:       virbr0      0.00      0.00      0.00      0.00      0.00      0.00      0.00
    Average:        wlan0      0.00      0.00      0.00      0.00      0.00      0.00      0.00

`sar -n EDEV`可以查看错误统计.

**2. `sar -n SOCK`查看sock统计**

    root@kali:~# sar -n SOCK 1 1
    Linux 3.14-kali1-amd64 (kali)   12/15/2014      _x86_64_        (4 CPU)

    02:32:01 PM    totsck    tcpsck    udpsck    rawsck   ip-frag    tcp-tw
    02:32:02 PM       618        31         3         0         0         0
    Average:          618        31         3         0         0         0
    root@kali:~#

可以通过关键字`IP,ICMP,TCP,UPD`查看各个协议的情况,前面加上`E`表示错误统计.


## 3. 数据可视化

将sar数据可视化很诱人啊,sadf能够将`/var/log/sysstat/saNN`数据转换成各种格式,python将格式化的数据进行可视化并不难.既然我有这个想法,那么有没有人已经实现了那?到github search一下,发现确实有几个[项目](https://github.com/search?utf8=✓&q=sar+graph&type=Repositories&ref=searchresults).

大概看了一下几个项目,都是几年前的.没由来啊,可视化sar数据需求应该很强烈的,又翻了翻,终于发现了[ksar](http://sourceforge.net/projects/ksar/).这个东西不进行详细介绍了,可以参考[使用 ksar 工具分析系统性能 ](http://www.ibm.com/developerworks/cn/linux/1303_caojh_ksar/).贴一张图,表示我确实用过.

![ksar](/img/ksar.png)

## 4. 问题

**1.手动建立数据文件**

执行sar命令需要使用`/var/log/saNN`文件,nn为日期,可以使用`sar -o`建立

**2.版本问题**

如果遇到参数不同,可以升级到最新版本,当前最新版本为11.0.2.不过本篇中使用的是10.0.5.最新版本提供了irqstat,irqtop试用命令.

## 5. 推荐文档

1. [sar访谈](http://roclinux.cn/?p=1647),入门很好
2. [10 Useful Sar (Sysstat) Examples for UNIX / Linux Performance Monitoring](http://www.thegeekstuff.com/2011/03/sar-examples/),geek stuff网站的
3. [使用 ksar 工具分析系统性能 ](http://www.ibm.com/developerworks/cn/linux/1303_caojh_ksar/)

# 变更记录

|Why | Who | When |
|----|-----|------|
|记录sar的选项|fishcired|2014-11-05 |
|添加举例,与ksaZ|fishcired|2014-12-15  |
