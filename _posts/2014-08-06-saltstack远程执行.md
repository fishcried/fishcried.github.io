---
layout: post
title: "SaltStack(二)远程执行"
description: ""
category: 系统管理
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [saltstack,配置管理]
---

# 1. 远程执行的基本需求

远程执行太常用，也太重要了。以前通过`ssh user@host cmd`来应付(太傻了)。我并没有很多的运维经验，只是提出一个场景来看一下远程执行的基本要求 。

## 1.1 苛刻一点的场景

>50台的一个集群，出现问题，怀疑机器时间不同步。现在要做两件事情:
>
> 1. 确认时间是否同步
> 2. 需要重启其中的13台机器的某个服务，这13台机器命名规则是`FLAG-N`,如`FLAG-1,FLAG-2......`

**解决第一个问题:确认时间是否同步**

该问题要求salt要有一种机制。这看似是一个很简单的问题,只要批量执行date命令不就可以了么,但是别忘记了这个与时间有关系,salt本身消耗的时间是占比重的.该问题要求salt具有一下特性:

1. 自动化认证
2. 并行执行
3. 执行速度要快,高效

一旦需要手工输入，时延，那么得到的结果都无法说明这50台机器时间是否存在问题。

**解决第二个问题:重启服务**

该问题要求salt

1. root权限
2. 目标过滤，如果通过ip来过滤13个服务器,那就没什么意思了。

## 1.2 远程脚本执行

远程执行脚本也是刚性需求。所以远程执行必须要解决，文件传输，权限修改的步骤。

总体来讲，远程执行要满足:

1. 认证自动化，且要安全
2. 要提供并行，高效的机制，而且要知道执行结果
3. root权限
4. 灵活的目标指定
5. 能够执行脚本

salt正中意!!!

# 2. salt远程执行使用

**salt远程执行的语法**

	 salt '<目标>' <function> [参数]

## 2.1 salt目标主机选择

salt的目标选择太灵活了，支持minion id，pillar，grains，分组，CIDR,组合等.这使得大规模集群管理变得简单。

|类别|用途|命令行使用|top中使用
|-----+---+----------+--------
|ID|通配符| `salt '*' test.ping` |默认的
||pcre| `salt -E 'compute(12)|(23)' test.ping` | `match:pcre`
||list| `salt -L 'list1,list2' test.ping` | `match:list`
|Pillar|master主动定义的变量|`salt -I 'master:auth_mode:1' test.ping`| `match:pillar`
|Grains|minion的系统信息| `salt -G 'os:Debian' test.ping `| `match:grain`
|CIDR|通过网络地址来过滤| `salt -S 192.168.1.0/24  test.ping `| `match:ipcidr`
|分组|可以进行分组| `salt -N 'compute' test.ping` | `match:nodegroup`
|compound|组合| `salt -C 'G@os:RedHat and webser* or E@database.*'  test.ping `| `match:compound`

### 2.1.1 minion id

minion id就是minion注册时使用的，可以通过`/etc/salt/minion`来自定义。monion id的选择自身支持通配符，正则表达式，列表的形式。

**1 通配符**

指定minion id可以使用通配符号，top文件与命令行默认都支持`shell style globbing`.
	
	# 命令行
	salt '*' test.ping
	
	#在top文件中使用时
	
	base:
	  '*':
	    - webserver

**2 正则表表达式**
	
	# 通过`-E`可以指定使用pcre。如：
	salt -E 'web1-(prod|devl)' test.ping
	
	#在top文件中使用时，必须通过`match: pcre`来指定，如:
	
	base:
	  'web1-(prod|devel)':
	    - match:pcre
	    - webserver

**3 列表的形式**

	# 命令行
	salt -L 'web1,web2,web3' test.ping
	
	# top文件中
	base:
	  'web1,web2,web3':
	    - match:list
	    - webserver

### 2.1.2 Grains

Grains主要用于描述被管理系统的系统信息。通过系统信息来匹配目标，如系统架构，内核版本等.

	#命令行中通过`-G`来表明，如：
	salt -G 'os:CentOS' test.ping
	
	#在top文件中使用
	base:
	  'node_type:web':
	    - match: grain
	    - webserver

> Note: Grains可以嵌套，而且可以使用通配符号: 
> saltstack -G 'ec2_tags:environment:*production*'

### 2.1.3 pillar

pillar主要用于变量定义，用于保护私有数据。在master端定义

	# 命令行
	salt -I 'master:auth_mode:1' test.ping
	
	#在top文件中使用
	base:
	  'master:auth_mode:1':
	    - match: pillar
	    - webserver

### 2.1.4 CIDR

	# 命令行
	salt -S '192.168.1.0/24' test.ping
	
	#在top文件中使用
	base:
	  '192.168.1.0/24': 
	    - match: pillar
	    - webserver

### 2.1.5 Range

没有用过,需要探索

### 2.1.6 组

将目标进行分组,进一步进行管理，这个也非常有用。

#### 2.1.6.1  定义组

.编辑`/etc/salt/master`配置文件

	nodegroups:
	  group1: 'L@foo.domain.com,bar.domain.com,baz.domain.com or bl*.domain.com'
	  group2: 'G@os:Debian and foo.domain.com'

上面有些语法糖，挺有意思的，想要理解的话，忍忍，继续看下去。

#### 2.1.6.2  使用组

	1 命令行通过`-N`来制定
	salt -N group1 test.ping
	
	#通过topfile,match来指定
	base:
	  group1:
	    - match: nodegroup
	    - webserver

### 2.1.7 翻滚吧，使用组合匹配目标

组合匹配就是将上面可用的方式通过逻辑运算来组合起来，让人兴奋啊。看看下面的表进行体会。逻辑运算包括`and`,`or`,`not`.

|Letter|	Match Type|			例如
|------|--------------|-------------
|G 	|Grains glob 	|G@os:Ubuntu
|E 	|CRE Minion ID 	|"E@web\d+\.(dev\|qa\|prod)\.loc"
|P 	|Grains PCRE 	|"P@os:(RedHat\|Fedora\|CentOS)"
|L 	|List of minions|	L@minion1.example.com,minion3.domain.com or bl*.domain.com
|I 	|Pillar glob 	|I@pdata:foobar
|S 	|Subnet/IP address|	S@192.168.1.0/24 or S@192.168.1.100
|R |	Range cluster|		R@%foo.bar

#### 2.1.7.1 使用

	# 命令行
	salt -C 'webserv* and G@os:Debin or E@web-dec-serv.*' test.ping
	
	# top文件
	base:
	  'webserv* and G@os:Debian or E@web-dc1-serv.*':
	  - match: compound
	  - webserver

> Note:
> 1. 单独的`not`不能使用，需要与`and`连用。 如：` salt -C '* and not G@kernel:Darwin' test.ping`
> 2. 不能通过分组去定义组合

我觉得大多数的情况是通过组合去定义一个组，然后使用这个组去匹配目标。否则书写起来过于费力。

## 2.2 Functions

salt支持的内容模块非常多，`salt 'ip-200-0-0-124' sys.doc`即可查看具体的functions。我的版本是`salt 2014.1.4`.

**1 查看有多少条内置的functions**

	salt 'ip-200-0-0-124' sys.doc | grep -E '^[a-z]+' | wc -l
	611

**2 查看function的分类**

	salt 'ip-200-0-0-124' sys.doc | grep -E '^[a-z]+' | awk -F'.' '{print $1}' | uniq  | wc -l
	67

**3 `cmd.run`**

`cmd.run`能够允许你执行远程主机上的命令，这是一个框架级别的function。

`salt '*' cmd.run 'ps -ef | grep kvm`

**4 `cmd.script`**

`cmd.scripte`允许将脚本上传到远端，然后执行，这也是一个框架级别的function。

`salt '*' cmd.script salt://test.sh`

**5 More**

如以上统计所见，salt内置了611个函数。想详细使用的时候只要`salt '*' sys.doc `查看下具体帮助即可。

## 2.3. 传递参数

function通过空格来界定参数:

	salt '*' cmd.exec_code python 'import sys; print sys.version'

可选的, 也支持keyword参数:

	salt '*' pip.install salt timeout=5 upgrade=True

## 2.4. 自定义function

TBD

# 变更记录

|Why | Who | When |
|----|-----|------|
|Create|fishcired|2014-08-05|
|添加详细的目标使用实例|fishcired|2014-08-10|
|修正错别字| fishcried | 2014-12-04 |
