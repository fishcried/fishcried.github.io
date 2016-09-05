---
layout: post
title: "Ansible(四) 使用Ansible Playbooks"
description: ""
category: 配置管理
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [ansible,devops,配置管理]
series: 配置管理
---

# Ansible(四) 使用Ansible Playbooks

前面已经提到ansible可以非常方便的控制多台机器,比如查看各台机器上的apache服务状态,`ansible all -a "service apache status" --sudo`,如果任务更复杂一点可以写成一个脚本，然后ansible会在每台机器上执行脚本。但是如果任务中子任务有有依赖那，任务判断任务的执行状态那，总之还是有一系列的问题，总结起来就是能不能做任务编排?当然能，如果没有任务编排就不能称之为配置管理工具，这个功能就叫playbook.

playbooks正如名字一样"剧本集"，通过剧本来讲述一个复杂的故事。在这里playbooks是对tasks的编排，从而完成非常复杂的任务。所谓的配置管理，其实就是说任务编排。
既然playbooks能够实现对任务的编排，那么playbook必然会有自己的语法，幸运的是playbook使用了yaml，非常简单的一种文件格式，下面来简单学习一下如何编写playbook。

# Playbook的构成

- hosts,用于指定目标
- vars,使得任务更加灵活
- tasks,具体的行为动作
- Handler,用于通知


下面是一个playbook例子.

```yaml
- name: setup openstack compent
  hosts: all
  remote_user: root

  tasks:
  - name: say hello
    shell: echo Hello World.
```

## 如何在playbook中使用变量

### 变量命名

变量名可以为字母,数字以及下划线.

### 定义变量

**在playbook中定义变量**

```
- hosts: webservers
  vars:
    http_port: 80
```

**在文件和role中定义变量**

1 hosts文件中定义变量时使用

```
[groupname:vars]
key = value
```
2 group vars file

```
cat group_vars/groupname
key: value
```

3 如果是role的话，则在相应的`defaults/main.yml`或者`vars/main.yml`

**注册变量**

```
register: xxx
```
- xxx.stdout.find(sdf)
- xxx.stdout_lines

### 使用变量

Ansible允许你使用Jinja2模板系统在playbook中引用变量.

```
{{ var1 }}
```

**jinjia2过滤器**



```
# default(*value*, *default_value=u''*, *boolean=False*)
{{ my_variable|default('my_variable is not defined') }}

```
- http://jinja.pocoo.org/docs/dev/templates/#builtin-filters
- http://ju.outofmemory.cn/entry/97893

**使用Facts获取的信息**

```
ansible hostname -m setup
```

如果没有使用fact变量时可以强制地关闭,这样执行速度会快一点.

```
- hosts: whatever
  gather_facts: no
```

**魔法变量**

- hostvars允许你访问另一个主机的变量，当然前提是ansible已经收集到这个主机的变量了：
- group_names：是当前主机所在的group列表
- groups：是所有inventory的group列表
- inventory_hostname：是在inventory里定义的主机名（ip或主机名称）
- play_hosts是当前的playbook范围内的主机列表
- inventory_dir和inventory_file是定义inventory的目录和文件

```
# 查看test.example.com主机的ansible版本
{{ hostvars['test.example.com']['ansible_distribution'] }}
# 判断当前主机是否属于webserver组内
{% if 'webserver' in group_names %}
   # some part of a configuration file that only applies to webservers
{% endif %}
# 遍历app_servers组主机
% for host in groups['app_servers'] %}
   # something that applies to all app servers.
{% endfor %}
#  查看该组主机ip
{% for host in groups['app_servers'] %}
   {{ hostvars[host]['ansible_eth0']['ipv4']['address'] }}
{% endfor %}
```

### 优先级

1. role defaults
1. inventory vars
1. inventory group_vars
1. inventory host_vars
1. playbook group_vars
1. playbook host_vars
1. host facts
1. registered vars
1. set_facts
1. play vars
1. play vars_prompt
1. play vars_files
1. role and include vars
1. block vars (only for tasks in block)
1. task vars (only for the task)
1. extra vars (always win precedence)

> 从低到高.ansible中可以定义变量的地方太多了,没啥不一样的地方，就是优先级不同而已.

### 一些常用的变量

- `inventory_hostname`, 机器的hostname
- `{{ hostvars[inventory_hostname]['ansible_' + which_interface]['ipv4']['address'] }}` ip地址




## 条件选择

**when**
可以各种条件组合, ==, >=, not, and, is defined, is not defined.

```
- hosts: webservers
  roles:
     - { role: debian_stock_config, when: ansible_os_family == 'Debian' }

tasks:
  - command: /bin/false
    register: result
    ignore_errors: True
  - command: /bin/something
    when: result|failed
  - command: /bin/something_else
    when: result|success
  - command: /bin/still/something_else
    when: result|skipped
```

when后面可以接Jinja2的表达式或者过滤器 

**基于变量选择文件和模版**

```
- name: template a file
   template: src={{ item }} dest=/etc/myapp/foo.conf
   with_first_found:
     - files:
        - {{ ansible_distribution }}.conf
        - default.conf
       paths:
        - search_location_one/somedir/
        - /opt/other_location/somedir/
```

**变量为空或者未定义的判断**

```
when: varname is undefined
      or
      varname is none
      or
      varname | trim == ''
 ```


## 循环

**基本的循环结构 with_items**

```
# 迭代列表
- name: add several users
  user: name={{ item }} state=present groups=wheel
  with_items:
     - testuser1
     - testuser2
# 迭代哈希
- name: add several users
  user: name={{ item.name }} state=present groups={{ item.groups }}
  with_items:
    - { name: 'testuser1', groups: 'wheel' }
    - { name: 'testuser2', groups: 'root' }

```

**字典循环 with_dict**

```
---
users:
  alice:
    name: Alice Appleworth
    telephone: 123-456-7890
  bob:
    name: Bob Bananarama
    telephone: 987-654-3210
```
```
tasks:
  - name: Print phone records
    debug: msg="User {{ item.key }} is {{ item.value.name }} ({{ item.value.telephone }})"
    with_dict: "{{users}}"
```

**文件循环 with_fileglob**

```
---
- hosts: all

  tasks:

    # first ensure our target directory exists
    - file: dest=/etc/fooapp state=directory

    # copy each file over that matches the given pattern
    - copy: src={{ item }} dest=/etc/fooapp/ owner=root mode=600
      with_fileglob:
        - /playbooks/files/fooapp/*
```

## 错误处理

通常情况下, 当出现失败时 Ansible 会停止在宿主机上执行.有时候,你会想要继续执行下去.为此 你需要像这样编写任务:

```
- name: this will not be counted as a failure
  command: /bin/false
  ignore_errors: yes
```

**控制对失败的定义**

假设一条命令的错误码毫无意义只有它的输出结果能告诉你什么出了问题,比如说字符串 “FAILED” 出 现在输出结果中.

```
- name: this command prints FAILED when it fails
  command: /usr/bin/example-command -x -y -z
  register: command_result
  failed_when: "'FAILED' in command_result.stderr"
```

当一个 shell或命令或其他模块运行时,它们往往都会在它们认为其影响机器状态时报告 “changed” 状态.有时你可以通过返回码或是输出结果来知道它们其实并没有做出任何更改.你希望覆写结果的, “changed” 状态使它不会出现在输出的报告或不会触发其他处理程序:

```
tasks:

  - shell: /usr/bin/billybass --mode="take me to the river"
    register: bass_result
    changed_when: "bass_result.rc != 2"

  # this will never report 'changed' status
  - shell: wall 'beep'
    changed_when: False
```

## 如何调试

1. 当运行verbose模式时,会打印出所有模块运行后的变量.这对于你要使用register功能时候很重要.只需要在执行playbook命令时加上参数–verbose便可以.ansible-playbook –verbose playbook.yml
2. debug模块

```
# Example that prints the loopback address and gateway for each host
- debug: msg="System {{ inventory_hostname }} has uuid {{ ansible_product_uuid }}"

- debug: msg="System {{ inventory_hostname }} has gateway {{ ansible_default_ipv4.gateway }}"
  when: ansible_default_ipv4.gateway is defined

- shell: /usr/bin/uptime
  register: result

- debug: var=result verbosity=2

- name: Display all variables/facts known for a host
  debug: var=hostvars[inventory_hostname] verbosity=4
```

## 重复执行

常常用于软件安装或者重启的场景.

```
- name: Confirm service connectivity
  command: "user/bin/mysqladmin --defaults-file=/etc/mysql/debian.cnf ping"
  changed_when: _mysql_running.rc != 0
  register: _mysql_running
  until: _mysql_running == 0
  retries: "{{ num_retries }}"
  delay: "{{ wait_delay }}"
  failed_when: false
  changed_when: _mysql_running != 0
  tags:
    - galera-cluster-state-check
```
