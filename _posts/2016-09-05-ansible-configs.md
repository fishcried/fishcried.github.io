---
layout: post
title: "Ansible(三) 命令行与配置文件"
description: ""
category: 配置管理
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [ansible,devops,配置管理]
series: 配置管理
---

# Ansible(三)命令行与配置文件

## ansible命令行常用参数

简单的试用了ansible后觉得挺爽的，像个元帅指挥者百万大军一样。有的时候不能总是`ansible xxx -a cmd`就了事的，所以根据具体情况适当的改变，所以先了解下ansible命令行参数:

```
Usage: ansible <host-pattern> [options]
    Options:
      -m MODULE_NAME, --module-name=MODULE_NAME         要执行的模块，默认为command
      -a MODULE_ARGS, --args=MODULE_ARGS                模块的参数
      -u REMOTE_USER, --user=REMOTE_USER                ssh连接的用户名，默认用root，ansible.cfg中可以配置
      -k, --ask-pass                                    提示输入ssh登录密码，当使用密码验证登录的时候用
      -s, --sudo                                        sudo模式运行
      -U SUDO_USER, --sudo-user=SUDO_USER               sudo到哪个用户，默认为root
      -K, --ask-sudo-pass                               提示输入sudo密码，当不是NOPASSWD模式时使用
      -B SECONDS, --background=SECONDS                  run asynchronously, failing after X seconds(default=N/A)
      -P POLL_INTERVAL, --poll=POLL_INTERVAL            set the poll interval if using -B (default=15)
      -C, --check                                       只是测试一下会改变什么内容，不会真正去执行
      -c CONNECTION                                     连接类型(default=smart)
      -f FORKS, --forks=FORKS                           fork多少个进程并发处理，默认5
      -i INVENTORY, --inventory-file=INVENTORY          指定hosts文件路径，默认default=/etc/ansible/hosts
      -l SUBSET, --limit=SUBSET                         指定一个pattern，对<host_pattern>已经匹配的主机中再过滤一次
      --list-hosts                                      只打印有哪些主机会执行这个playbook文件，不是实际执行该playboo
      -M MODULE_PATH, --module-path=MODULE_PATH         要执行的模块的路径，默认为/usr/share/ansible/
      -o, --one-line                                    压缩输出，摘要输出
      --private-key=PRIVATE_KEY_FILE                    私钥路径
      -T TIMEOUT, --timeout=TIMEOUT                     ssh连接超时时间，默认10秒
      -t TREE, --tree=TREE                              日志输出到该目录，日志文件名会以主机名命名
      -v, --verbose                                     verbose mode (-vvv for more, -vvvv to enable connection debugging)
  ```


   
**多使用`--list-hosts`查看组内的主机**

当想查看某个组内成员时不用非得查看hosts文件，使用`ansible 

**参数`-f`指定并发数，加速执行**

这个参数可以指定任务执行的并发数，和主机cpu数相同即可.

**通过参数--sudo [-K]来使用sudo模式**

需要特殊操作的时候sud很有用.

**参数`-a`与`-m`的区别**

   -a指定命令参数,-m指定执行模块，当没有指定模块的时候使用的是command模块，这个模块不能用于管道什么的。所以命令复杂时要`-m shell -a find ... | xargs ...`,命令不复杂时直接-a就可以了,如`-a date`.

    

##配置文件

每次配置总不能通过命令行传递吧，执行一次两次也好，但是多了就麻烦了，所有好工具肯定都提供命令行和配置文件双重控制的，下面就看下配置文件。


### 配置文件的优先级

linux下配置文件通常都在`/etc/xxx/`下面,这个是全局的配置文件,然后个人的配置文件一般都是home目录下以.开头的形式存在,ansible也遵循这样的规则。ansible主要有以下的几种配置文件，优先级递减。

1. ansible.cfg (当前目录下)
1. ANSIBLE_CONFIG (环境变量)
1. .ansible.cfg (用户家目录下)
1. /etc/ansible/ansible.cfg

### 常用配置项说明

上面说明了配置文件的优先级，那么接下来总结下配置文件的内容，这里只摘录出常用的配置项，详细的还得去看官方文档。


| 配置项  | 含义  
|---------|-----------
| hostfile | 指定hostfile文件，这样就不用-i指定le |
| private_key_file | 指定秘钥 |
| gathtering | smart/implicit/explicit. 该配置用于控制facts变量，implict就是每次都要收集,explict表示只有task

### 配置文件样例

```
[defaults]
hostfile=/Users/the5fire/hosts
private_key_file=/xx
host_key_checking = False
```

