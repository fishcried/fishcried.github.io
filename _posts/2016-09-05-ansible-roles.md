---
layout: post
title: "Ansible(五) 编写Ansible Role"
description: ""
category: 配置管理
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [ansible,devops,配置管理]
series: 配置管理
---

# Ansible(五) 编写Ansible Role

当一个任务非常复杂时，playbooks的编写一会非常的庞大，以至于没人愿意把一个playbook从头看到尾,更没有原理维护了，当然同伴也不会愿意复用你的，可能更愿意自己新写一个。
所以出现了role功能，ansible role就是把一个复杂的playbooks模板化，分成不同的目录，每个目录都存放各自的东西，这样一个大任务被拆解开，编写与维护就都简单了。更主要的是你可以直接把你的role开放出来，别人可以复用，从而依靠大家的力量就能打造强大的ansible 角色库。

# 如何书写ansible role

编写ansible role并不需要什么特别的技术，甚至不需要书写，只是对原来的playbook进行逻辑上的划分。但实际上playbook的布局也不需要我们自己再去设计，而是根据早就规定好的布局对我们的playbook进行逻辑拆分就好了，看下面的webservers role布局:

```
...
roles/
   webservers/    #下面的子目录都不是必须提供的，没有的目录会自动忽略，不会出现问题，所以你可以只有tasks/子目录也没问题
     files/
     templates/
     tasks/
     handlers/
     vars/
     meta/
```

当把一个大的playbook做了逻辑拆分，然后对应到上面的布局，当引用role的时候，ansible会做相应幕后的工作:

- /path/roles/x/tasks/main.yml中的tasks都将自动添加到该play中
- /path/roles/x/handlers/main.yml中的handlers都将添加到该play中
- /path/roles/x/vars/main.yml中的所有变量都将自动添加到该play中
- /path/roles/x/meta/main.yml中的所有role依赖关系都将自动添加到roles列表中
- /path/roles/x/defaults/main.yml中为一些默认变量值，具有最低优先权，在没有其他任何地方指定该变量的值时，才会用到默认变量值
- task中的copy模块和script模块会自动从/path/roles/x/files寻找文件，也就是根本不需要指定文件绝对路径或相对路径，如src=foo.txt则自动转换为/path/roles/x/files/foo.txt
- task中的template模块会自动从/path/roles/x/templates/中加载模板文件，无需指定绝对路径或相对路径
- 通过include包含文件会自动从/path/roles/x/tasks/中加载文件，无需指定绝对路径或相对路径


> 这回是不是就理解了ansible role的宗旨:把配置文件整合到一起并最大程度保证其干净整洁,保证可重用性

## role依赖

可以在一个role的meta/main.yml中定义该role依赖其他的role，然后调用该role的时候，会自动去拉取其他依赖的role

```
---
dependencies:
  - { role: common, some_parameter: 3 }    #向依赖的role传递变量
  - { role: apache, port: 80 }
  - { role: postgres, dbname: blarg, other_parameter: 12 }
```

#  项目布局

有了role,整体项目的布局也就很好看了，一般都是下面这个样子。

```
    production                # inventory file for production servers
    stage                     # inventory file for stage environment
    group_vars/
       group1                 # here we assign variables to particular groups
       group2                 # ""
    host_vars/
       hostname1              # if systems need specific variables, put them here
       hostname2              # ""
    library/                  # if any custom modules, put them here (optional)
    filter_plugins/           # if any custom filter plugins, put them here (optional)
    site.yml                  # master playbook
    webservers.yml            # playbook for webserver tier
    dbservers.yml             # playbook for dbserver tier
    roles/
        common/               # this hierarchy represents a "role"
            tasks/            #
                main.yml      #  <-- tasks file can include smaller files if warranted
            handlers/         #
                main.yml      #  <-- handlers file
            templates/        #  <-- files for use with the template resource
                ntp.conf.j2   #  <------- templates end in .j2
            files/            #
                bar.txt       #  <-- files for use with the copy resource
                foo.sh        #  <-- script files for use with the script resource
            vars/             #
                main.yml      #  <-- variables associated with this role
            defaults/         #
                main.yml      #  <-- default lower priority variables for this role
            meta/             #
                main.yml      #  <-- role dependencies
        webtier/              # same kind of structure as "common" was above, done for the webtier role
        monitoring/           # ""
        fooapp/               # ""
```
