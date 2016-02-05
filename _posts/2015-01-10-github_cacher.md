---
layout: post
title: "一个git sync脚本实现github cacher功能"
description: ""
category: 程序设计
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [cacher,github]
---


国内clone github上的项目速度还是有点慢，尤其像openstack这样由多个子项目组成的项目，网络问题让人恼火.所以想搞个cacher，来本地存储github上关注的项目.尝试坐火箭而不是老爷车．

一般来讲，逻辑上的设计可能是这样:

![simple-git-sync](/img/git-sync-simple.png)

通过一台本地的git server做为cacher来mirror github上的项目.用户直接获取本地的cacher即可．但是觉得自己玩不转git server,而且需要自己去维护管理git server，那可能会陷入另一个坑.更好的合适的解决方式可能是借用别人或者团队的维护的本地git server.然后自己pull from github,再push到本地的git server来保证项目的同步.这样既解决了问题.也免去了维护的负担.幸好，团队内部有本地的gitlab．我的方案就成了下面的样子:

![git-sync](/img/git-sync.png)

这样只要实现中间的`git sync`功能就可以了，并不复杂，`crontab`加上一些`git`命令就能搞出个脚本来解决.脚本的调用可以在随意的一台机器上．前提是做好执行`git sync`脚本这台机器与github,gitlab的认证.如果没有本地的gitlab．也可以用国内免费的代码托管服务,比如csdn code等,只不过不能同步那么多项目.

脚本名字叫`git-sync.sh`.使用起来非常简单: `git-sync.sh projects.conf`.只需要通过一个文件来指定cache哪些项目.文件格式如下:

    项目名称 项目源地址　项目cacher地址

比如我将github上的openstack的很多子项目mirror到了本地网络上的192.168.250.3(提供gitlab服务)这台机器,cacher后的项目名称是手动到gitlab上建立的.

    #　cat projects.conf
    keystone git@github.com:openstack/keystone.git git@192.168.250.3:mirror/keystone.git
    requirements git@github.com:openstack/requirements.git git@192.168.250.3:mirror/requirements.git
    devstack git@github.com:openstack-dev/devstack.git git@192.168.250.3:mirror/devstack.git
    ...

**`git-sync.sh脚本代码`**

脚本同步了每个项目的所有分支以及标签.最新的代码也可查看[github](https://github.com/fishcried/git-sync)

    #!/bin/bash
    #
    # The MIT License (MIT)
    # Copyright (c) 2014 fishcried(tianqing.w@gmail.com)
    #

    trap 'echo "ERRTRAP $LINENO" && exit 2' ERR

    LOG() { 
      [ "$DEBUG" == "true" ] && echo 1>&2 "$@" 
    }

    WORKSPACE=/var/cache/git-mirrors
    DEBUG=true

    if [ $# -ne 1 ];then
      echo "$(basename $0) project-file"
      exit 1
    fi

    PROJECTS=$1

    [ -e $PROJECTS ] || { 
      echo "Can't find the projects file - $PROJECTS" 
      exit 1
    }


    if [ ! -e $WORKSPACE ];then
      mkdir $WORKSPACE
    fi


    cd $WORKSPACE

    cat $PROJECTS | while read line
    do
      # parse the config -> name origin mirror
      eval $(echo $line | awk '{printf("project_dir=%s; origin_url=%s; mirror_url=%s;",$1,$2,$3)}')
      

      LOG "Starting $project_dir:  [$origin_url] --->> [$mirror_url]"
      
      if [ ! -d "$project_dir" ];then
        # clone the new project
        LOG "Clone $project_dir from $origin_url"
        git clone $origin_url
        cd $project_dir

        # set the back mirror
        LOG "Add mirror repository $mirror_url"
        git remote add mirror $mirror_url

        LOG "Starting track remote branchs"
        for br in $(git branch -r | sed '1,2d;s/\*//;s/ //g')
        do
          LOG "Track branch $br"
          git checkout --track $br
        done
        cd ..
      fi

      # pull from the origin and push to the mirror

      cd $project_dir
      LOG "Starting sync mirror"
      for br in $(git branch | sed 's/\*//;s/ //g')
      do
        LOG "Pulling $br from $origin_url"
        git pull origin 
        LOG "Pushing $br to $mirror_url"
        git push mirror $br 
        git push mirror --tags
      done
      cd ..

      LOG "Finish mirror $project_dir"

    done

脚本能够优化的地方还是有的,比如发现从github pull时发现代码是最新的,就没有必要向本地的cache push了.没有进行这种优化为了保证简单健壮,因为速度不是首要因素.

**建立crontab任务**

将脚本与配置文件放到想放的位置.每小时更新一次.

    # crontab -e
    ...
    0 */1 * * * bash PATH/git-sync.sh PATH/projects.conf

# 变更记录

|Why | Who | When |
|----|-----|------|
|创建文件,提醒日后完善内容|fishcired|2015-01-07 |
|完善|fishcired|2015-01-10  |
|修改错别字|fishcired|2015-01-14 |
