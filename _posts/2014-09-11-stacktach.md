---
layout: post
title: "使用stacktach调试OpenStack"
subtitle: "通过web查看openstack的rpc消息"
description: ""
category: openstack
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [openstack,stacktach]
---

# 1. StackTach 是什么

[网易分享的部署运维实战](http://www.ibm.com/developerworks/cn/cloud/library/1408_zhangxl_openstack/)里提到stacktach进行调试，
部署试用了一下，感觉对调试非常有帮助．该项目出自rackspace，开发者是nova的核心开发人员,不过该项目最近６个月已经没有人提交代码了．

**What's stacktach?**

> StackTach was initially created as a browser-based debugging tool for OpenStack Nova.
> Since that time, StackTach has evolved into a tool that can do debugging, performance
> monitoring and perform audit, validation and reconcilation of Nova and Glance usage 
> in a manner suitable for billing.

看一张部署之后的图:

![stacktach](/img/stacktach.png)

# 2. 工作原理

事件发生时，Openstack的各个组件能够发送通知消息到消息队列．通过监控这些通知信息就能洞悉openstack流程，并进行审计调试等．

![stacktach](/img/stacktach.gif)

# 3. 安装StackTach( on ubuntu14.04)

>  安装过程主要参考[官方文档](http://stacktach.readthedocs.org/en/latest/setup.html),该文档**"Hurry Up"**.

**1. 安装依赖的软件包**

    # 安装一些依赖的包
    apt-get install -y python-pip libmysqlclient-dev  python-dev

    # 配置pip
    apt-get install -y python-pip

    mkdir ~/.pip
    cd !$
    touch pip.conf

    cat pip.conf
    [global]
    index-url = http://pypi.v2ex.com/simple

**2. 下载stacktach源码**

    # 对该项目fork下,去除了对google资源的引用,要不然得翻墙
    git clone https://github.com/fishcried/stacktach.git
    cd stacktach
    pip install -r etc/pip-requires.txt

**3. 修改配置文件**

应为要使用数据库，消息队列．所以需要配置相关信息．

- stacktach_config.sh
- stacktach_worker_config.json

以上文件的详细说明见[官方文档](http://stacktach.readthedocs.org/en/latest/setup.html)

**4. 准备数据库**

创建数据库

	cat create_db.mysql
	create database stacktach
	
	mysql -u root -h mysqlhost -ppassword | create_db.mysql

同步表

	python manage.py syncdb --noinput
	python manage.py migrate --noinput

**4. 配置nova**

nova修改非常简单,只需要在nova.conf中加上

	notification_driver=nova.openstack.common.notifier.rpc_notifier
	notification_topics=monitor

然后重启服务即可．

但是考虑到需要修改的节点非常多,就写成脚本的方式,然后通过`saltstack`来执行．使用的配置函数来自`devstack`．
	
	# Determinate is the given option present in the INI file
	# ini_has_option config-file section option
	function ini_has_option {
	    local xtrace=$(set +o | grep xtrace)
	    set +o xtrace
	    local file=$1
	    local section=$2
	    local option=$3
	    local line
	    line=$(sed -ne "/^\[$section\]/,/^\[.*\]/ { /^$option[ \t]*=/ p; }" "$file")
	    $xtrace
	    [ -n "$line" ]
	}
	
	# Set an option in an INI file
	# iniset config-file section option value
	function iniset {
	    local xtrace=$(set +o | grep xtrace)
	    set +o xtrace
	    local file=$1
	    local section=$2
	    local option=$3
	    local value=$4
	
	    [[ -z $section || -z $option ]] && return
	
	    if ! grep -q "^\[$section\]" "$file" 2>/dev/null; then
	        # Add section at the end
	        echo -e "\n[$section]" >>"$file"
	    fi
	    if ! ini_has_option "$file" "$section" "$option"; then
	        # Add it
	        sed -i -e "/^\[$section\]/ a\\
	$option = $value
	" "$file"
	    else
	        local sep=$(echo -ne "\x01")
	        # Replace it
	        sed -i -e '/^\['${section}'\]/,/^\[.*\]/ s'${sep}'^\('${option}'[ \t]*=[ \t]*\).*$'${sep}'\1'"${value}"${sep} "$file"
	    fi
	    $xtrace
	}
	
	# Set a multiple line option in an INI file
	# iniset_multiline config-file section option value1 value2 valu3 ...
	function iniset_multiline {
	    local xtrace=$(set +o | grep xtrace)
	    set +o xtrace
	    local file=$1
	    local section=$2
	    local option=$3
	    shift 3
	    local values
	    for v in $@; do
	        # The later sed command inserts each new value in the line next to
	        # the section identifier, which causes the values to be inserted in
	        # the reverse order. Do a reverse here to keep the original order.
	        values="$v ${values}"
	    done
	    if ! grep -q "^\[$section\]" "$file"; then
	        # Add section at the end
	        echo -e "\n[$section]" >>"$file"
	    else
	        # Remove old values
	        sed -i -e "/^\[$section\]/,/^\[.*\]/ { /^$option[ \t]*=/ d; }" "$file"
	    fi
	    # Add new ones
	    for v in $values; do
	        sed -i -e "/^\[$section\]/ a\\
	$option = $v
	" "$file"
	    done
	    $xtrace
	}
	
	# Uncomment an option in an INI file
	# iniuncomment config-file section option
	function iniuncomment {
	    local xtrace=$(set +o | grep xtrace)
	    set +o xtrace
	    local file=$1
	    local section=$2
	    local option=$3
	    sed -i -e "/^\[$section\]/,/^\[.*\]/ s|[^ \t]*#[ \t]*\($option[ \t]*=.*$\)|\1|" "$file"
	    $xtrace
	}
	CONF_FILE=/tmp/nova.conf
	
	iniset $CONF_FILE DEFAULT notification_driver nova.openstack.common.notifier.rpc_notifier
	iniset $CONF_FILE DEFAULT notification_topics monitor

**5. 运行**
	
	# 要创建该目录，否则woker无法启动
	mkdir -p /var/log/stacktach/
	touch /var/log/stacktach/worker.log
	python worker/start_worker.py &
	
	python manage.py runserver --insecure 0.0.0.0:8000 &

# 4. 通过docker来部署stacktach

安装这个东西还是挺麻烦的,用docker来吧．(docker可能存在问题,没有详细测试过.有时间会修改的)

[项目地址](https://github.com/fishcried/docker-stacktach)

# 变更记录

|Why | Who | When |
|----|-----|------|
|create|fishcired|2014-09-11|
|编写ubuntu14.04安装过程|fishcired|2014-09-11|
|完善细节|fishcired|2014-09-12|
|fork了该项目,去除google资源引用,避免翻墙|fishcired|2014-12-24 |
