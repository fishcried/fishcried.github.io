---
layout: post
title: "Ansible(六) 常用模块"
description: ""
category: 配置管理
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [ansible,devops,配置管理]
series: 配置管理
---

# Ansible(六) 常用模块

# 远程执行

*Ad-Hoc/ script*

```
    `ansible all -m command -a 'ls'`
    `ansible all -m script -a 'xxxx args'`
```

# 连接测试

**Ping**

    `ansible all -m ping`

# 软件包/服务相关

**Services**

    ansible all -m service -a "name=httpd state=started"

**crond**

    ansible all -m cron -a 'name="you autoupdate" weekday="2" minute=0 hour=12 user="root" job="/usr/sbin/yum-autoupdate" cron_file=ansible_yum-autoupdate'

apt

```
- name: Update apt sources
  apt:
    update_cache: yes
    cache_valid_time: 600

- name: Install galera pre packages
  apt:
    pkg: "{{ item }}"
    state: latest
  register: install_packages
  until: install_packages|success
  retries: 5
  delay: 2
  with_items: galera_pre_apt_packages
```

apt_key

```
- name: Add galera apt-keys
  apt_key:
    id: "{{ item.hash_id }}"
    keyserver: "{{ item.keyserver | default(omit) }}"
    data: "{{ item.data | default(omit) }}"
    url: "{{ item.url | default(omit) }}"
    state: "present"
```

apt_repository

```
- name: Add galera repo(s)
  apt_repository:
    repo: "{{ item.repo }}"
    state: "{{ item.state }}"
  register: add_repos
  until: add_repos|success
  retries: 5
  delay: 2
  with_items:
    - "{{ galera_apt_repo }}"
    - "{{ galera_apt_percona_xtrabackup_repo }}"
```

debconf

```
- name: Preseed galera password(s)
  debconf:
    name: "{{ item.name }}"
    question: "{{ item.question }}"
    value: "{{ item.value }}"
    vtype: "{{ item.vtype }}"
  with_items: galera_debconf_items
```

# 用户管理

```
ansible all -m user -a "name=foo password=xxxx"
ansible all -m user -a "name=foo state=absent"
```

# 文件配置管理

```
- name: Update hosts file remove stale IP entries
  lineinfile:
    dest: /etc/hosts
    regexp: "^{{ hostvars[item]['ansible_ssh_host'] }} (?!{{ item }}$)"
    state: absent
  with_items:
    - "{{ groups['all_containers'] }}"
    - "{{ groups['hosts'] }}"
  tags:
    - openstack-host-hostfile

- name: Update hosts file remove stale Host entries
  lineinfile:
    dest: /etc/hosts
    regexp: "(?<!^{{ hostvars[item]['ansible_ssh_host'] }}) {{ item }}$"
    state: absent
  with_items:
    - "{{ groups['all_containers'] }}"
    - "{{ groups['hosts'] }}"
  tags:
    - openstack-host-hostfile

- name: Update hosts file from ansible inventory
  lineinfile:
    dest: /etc/hosts
    line: "{{ hostvars[item]['ansible_ssh_host'] }} {{ item }}"
    state: present
  with_items:
    - "{{ groups['all_containers'] }}"
    - "{{ groups['hosts'] }}"
  tags:
    - openstack-host-hostfile
```

**文件传输与修改**

    ansible all  -m copy -a 'src=/etc/my.cnf dest=/etc/my.cnf owner=mysql group=mysql' 
    # 修改文件属性
    ansible all  -m file -a 'dest=/etc/rsync.d/secrets  mode=600 owner=root group=root'
    # 创建目录
    ansible all -m file -a 'dest=/data/install mode=755 owner=root group=root state=directory' 
    # 删除文件
    ansible all -m file -a 'dest=/data/tmp state=absent'
