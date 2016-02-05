---
layout: post
title: "SaltStack(三)配置管理"
description: ""
category: 系统管理
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [saltstack,配置管理]
---

# 1. 如何做到配置管理

其实配置管理可以看作是对对象状态管理。主要解决对象的状态跟踪，依赖分析等问题,这就是state存在的意义。更难的是如果提供给用户一套简单灵活的接口来描述配置管理需求。state使用的是sls(salt state file),sls通过yaml格式来定义。简单灵活，并且可读性非常好。

让人惊讶的是定义sls并不需要一个新的dsl。sls内的没什么语法糖，只是一些数据结构。这些数据结构描述了你想使用什么模块，调用哪些方法，使用什么参数。书写起来感觉你在coding，coding什么时候这么简单了？！[yaml](http://yaml.org/spec/1.1/ "yaml手册")只是一种文件格式而已。

> Many of the most powerful and useful engineering solutions are founded on simple principles. Salt States strive to do just that: K.I.S.S. (Keep It Stupidly Simple

其实每一个作者都对**K.I.S.S**梦寐以求，但这真的不那么简单。太简单就会不灵活，灵活了就会复杂。salt state让人惊讶！

# 2. Salt State Tree

- State Tree
	- `file_roots`目录下的SLS文件的集合，用来组织SLS模块。
- top file
	- Salt State系统的入口文件，定义了minion处于哪个环境，加载哪些SLS模块。

### 2.1 编写top文件

top文件定义使用哪些minion使用哪个环境，加载那些sls模块. 在使用state系统前，需要设置Salt文件服务。编辑`/etc/salt/master`

    file_roots:
      base:
	    - /srv/salt

想要生效，需要重启master服务`service salt-master restart`

top文件实例:

    base:
      ’*’:
      - ldap-client
      - networking
      - salt.minion
    ’salt-master*’:
      -  salt.master
    ’^(memcache|web).(qa|prod).loc$’:
      - match: pcre
      - nagios.mon.web
      - apache.server
    ’os:Ubuntu’:
      - match: grain
      - repos.ubuntu
    ’os:(RedHat|CentOS)’:
      - match: grain_pcre
      - repos.epel
    ’foo,bar,baz’:
      - match: list
      - database
    ’somekey:abc’:
      - match: pillar
      - xyz
    ’nag1* or G@role:monitoring’:
      - match: compound
      - nagios.server

## 2.2 sls文件编写

sls文件默认使用`[yaml](http://docs.saltstack.cn/topics/yaml/index.html)`。其实yaml非常简单,只要记住:

1. 缩进用空格(一般是2个),不要用tab
2. `:`表示是字典
3. `-`表示是列表

**0. 一个完整的模板**

    <Include Declaration>:  # 一个list，其元素是要引用到本SLS文件的其他SLS模块。 只能用在highstate结构的顶层
      - <Module Reference>  # SLS模块的名字，以在Salt master上的文件结构命名。名为edit.vim的模块指向salt://edit/vim.sls
      - <Module Reference>
    
    <Extend Declaration>:  # 扩展被引用的SLS模块中的name声明。extend声明也是一个dict，其key必须是在被引用的SLS模块中定义的ID。 只能用在highstate结构的顶层。
    <ID Declaration>:
      [<overrides>]
    
    # standard declaration
    <ID Declaration>:      # 定义一个独立的highstate数据段。ID在highstate dict中作为key，其对应的value是包含state声明和requisit声明的
                           # 另一个dict 用在highstate结构的顶层或extend声明的下一层
      <State Declaration>: # 一个list，至少包含一个定义function声明的string，0个或多个function arg声明的dict。
        - <Function>       # state中要要执行的function。1个state声明中只能有1个function声明
        - <Function Arg>   # 只有1个key的dict，作为参数传递给function声明
        - <Function Arg>
        - <Function Arg>
        - <Name>: <name>   # 覆盖state声明中的name参数。name参数的默认值是ID声明
        - <Requisite Declaration>: # 一个list，其成员是requisite引用
          - <Requisite Reference>  # 有一个key的dict。key是被引用的state声明的名字，value是被引用的ID声明的名字
          - <Requisite Reference>
	
    # inline function and names
    <ID Declaration>:
      <State Declaration>.<Function>:
        - <Function Arg>
        - <Function Arg>
        - <Function Arg>
        - <Names>:         # 将1个state声明扩展为多个不同名的state声明。
          - <name>
          - <name>
          - <name>
        - <Requisite Declaration>:
          - <Requisite Reference>
          - <Requisite Reference>
    
    # multiple states for single id
    <ID Declaration>:
      <State Declaration>:
        - <Function>
        - <Function Arg>
        - <Name>: <name>
        - <Requisite Declaration>:
          - <Requisite Reference>
      <State Declaration>:
        - <Function>
        - <Function Arg>
        - <Names>:
          - <name>
          - <name>
        - <Requisite Declaration>:
          - <Requisite Reference>

**1. Include声明**

一个list，其元素是要引用到本SLS文件的其他SLS模块。 只能用在highstate结构的顶层

    include:
      - edit.vim      
      - http.server

类似与c语言的`include`,python的`import`.

引用其他环境变量的sls文件:

    include:
      - dev: http

相对引用:

    include:
      - .virt

**2. Moudle引用**

SLS模块的名字，以在Salt master上的文件结构命名。名为edit.vim的模块指向salt://edit/vim.sls

**3. ID声明**

定义一个独立的highstate数据段。ID在highstate dict中作为key，其对应的value是包含state声明和requisit声明的另一个dict 用在highstate结构的顶层或extend声明的下一层。 ID在整个State tree中必须是唯一的。如果同一个ID用了两次，只有最先匹配到的生效，其他所有的同名ID声明被忽略

> Note: ID要唯一，建议ID都要加前缀，然后通过`name`声明来修正。这样能更好的保证唯一性。

**4. Extend声明**

扩展被引用的SLS模块中的name声明。extend声明也是一个dict，其key必须是在被引用的SLS模块中定义的ID。 只能用在highstate结构的顶层。

mywebsite.sls:

    include:
      - apache
    
    extend:
      apache:
        service:
          - watch:
            - file: mywebsite
    
    mywebsite:
      file:
        - managed

以上的extend使得应用myswebsite.sls时安装apache服务自动重启，但是没有重写apache.sls，而是实现了拓展。

**5. State声明**

一个list，至少包含一个定义function声明的string，0个或多个function arg声明的dict。还有一些可选的成员，比如名字覆盖部分（name和names声明），requistie声明。只能用在ID声明的下一级。

**6. Requist声明**

一个list，其成员是requisite引用。 用来生成动作依赖树。Salt states被设计成按确定的顺序执行，require或watch其他Salt state可以调整执行的顺序。 做为list组件用在state声明下一级，或是作为key用在ID声明下一级。

**7. Requiste引用**

只有一个key的dict。key是被引用的state声明的名字，value是被引用的ID声明的名字。 只能用作requisite声明的成员。

require可以引用的类型包括,pkg,file,sls等。

    include:
      - network
    
    httpd:
      pkg:
        - installed
      service:
        - running
        - require:
          - pkg: httpd
          - sls: network

**8. Function声明**

state中要要执行的function。1个state声明中只能有1个function声明

下面的例子中，state声明调用了state模块pkg模块中的installed功能。

    httpd:
      pkg.installed

可以用行内缩写方式声明function（上面的例子中就是），使用完整写法使得数据结构更清晰：

    httpd:
      pkg:
        - installed

需要注意的是连续的两个简写形式是无效的，为了避免疑惑，建议全部采用完整写法。
INVALID:

    httpd:
      pkg.installed
      service.running

VALID:

    httpd:
      pkg:
        - installed
      service:
        - running

只能用作state声明的成员。

**9. Function arg声明**

只有1个key的dict，作为参数传递给function声明，其值为有效的Python类型。其类型必须满足function的需要。 用在function声明下一级。

    /etc/http/conf/http.conf:
      file.managed:
        - user: root
        - group: root
        - mode: 644

**10. Name声明**

覆盖state声明中的name参数。name参数的默认值是ID声明。 name总是1个单key字典，其值类型是string。

    motd_perms:
      file.managed:
        - name: /etc/motd
        - mode: 644
    
    motd_quote:
      file.append:
        - name: /etc/motd
        - text: "Of all smells, bread; of all tastes, salt."

name声明，常常用于避免id冲突和偷懒(一个长的id，如果经常被引用，那么就可以使用一个短的id来少敲键盘)

**11. Names声明**

将1个state声明扩展为多个不同名的state声明。

    python-pkgs:
      pkg.installed:
        - names:
          - python-django
          - python-crypto
          - python-yaml

==

    python-django:
      pkg.installed
    
    python-crypto:
      pkg.installed
    
    python-yaml:
      pkg.installed

## 2.3 解析sls文件的潜规则

**1 解析sls文件潜规则**

- `.sls`后缀是不出现在SLS定义中的. `test`等效`test.sls`
- 使用子目录进行清晰灵活的组织
	- `.`代表子目录
	- `webserver.dev`表示`/webserver/dev.sls`
- `init.sls`在一个子目录中表示引导文件,也就是表示子目录本身.`webserver/init.sls`就是`webserver`
- 如果同时存在`webserver.sls`和`webserver/init.sls`则前者有效.

**2 多环境变量时,关系映射的潜规则**

- `base`环境优先。 如果同时存在`base`，`dev`, `test`.所有重复的定义，选取`base`的.
- `base`环境内没有定义映射关系，而在其他环境中定义，则安装环境命令的字母顺序选择地一个。
- 针对其他环境变量，同上一条。

**3 states的执行顺序**

- 默认顺序安装编写sls的顺序
- 通过`require`,`watch`来进行顺序依赖，会自动处理
- 通过`order`来调整
	- order为N的时候，均为N的会一前一后执行，没有设置order的会在他们之后
	- last 最后

# 3. 常用模块

## 3.1 软件包管理

**1 仓库**

    base:
      pkgrepo.managed:
      - humanname: CentOS-$releasever - Base
      - mirrorlist: http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os
      - comments:
        - '#http://mirror.centos.org/centos/$releasever/os/$basearch/'
      - gpgcheck: 1
      - gpgkey: file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-6

**2 软件包**

    vim:
      pkg.installed

## 3.2 服务管理

    redis:
      service:
      - running
      - enable: True
      - reload: True
      - watch:
        - pkg: redis

## 3.3 文件管理模块

    /etc/http/conf/http.conf: 
      file.managed:
      - source: salt://apache/http.conf
      - user: root
      - group: root
      - mode: 644           
      - template: jinja     
      - defaults:           
        custom_var: "default value"
        other_var: 123    
    {% if grains['os'] == 'Ubuntu' %}
      - context:
        custom_var: "override"
    {% endif %}

## 3.4 管理用户和组

**1 用户管理**

    fred:
      user.present:
      - fullname: Fred Jones
      - shell: /bin/zsh
      - home: /home/fred
      - uid: 4000
      - gid: 4000
      - groups:
        - wheel
        - storage
        - games
    
    testuser:
      user.absent

**2 组管理**

    cheese:
      group.present:
      - gid: 7648
      - system: True

## 3.5 命令管理

    Run myscript:
      cmd.run:
      - name: /path/to/myscript
      - cwd: /
      - stateful: True
    
    Run only if myscript changed something:
      cmd.wait:
      - name: echo hello
      - cwd: /
      - watch:
        - cmd: Run myscript

    salt://scripts/foo.sh:
      cmd.script:     
      - env:
        - BATCH: 'yes'

> 想详细了解每个模块如何使用，最好的方式就是去看源码。`/usr/lib/python<version>/dist-packages/salt/sates/*.py`,只要扫一眼就能明白。

# 4. 执行

当一切都准备好后，只需要`salt '*' state.highstate`即可将sls应用到minion中。
使用`salt '*' state.highstate -v`可以看到更详细的输出。

1. 测试执行
如果只是试试看的话，并不真的执行只需要`salt '*' state.highstate test=True`
2. 主动推送
`salt '*' state.highstate`这个就是主动推送。
3. 被动拉取

# 5. 任务管理

    salt '*' saltutil.running
    salt '*' saltutil.find_job
    salt '*' saltutil.signal_job
    salt '*' saltutil.term_job
    salt '*' saltutil.kill_job
    salt-run jobs.active
    salt-run jobs.lookup_jid <job id number>

# 7. 调试

执行`state.highstate`后，如果只返回minion的主机名加上`:`，那么应该是出错了，很可能是SLS文件存在问题。提示方法:

- master  添加`-v`参数，查看具体输出， `salt '*' state.highstate -v`
- minion 
	- 如果salt-minion是以服务的形式启动，那么可以`salt-call state.highstate -l debug`进行查看过程
	- 也可以直接在前台启动`salt-minion -l debug`

# 6. 参考

1. [How Do I Use Salt States?](http://docs.saltstack.com/en/latest/topics/tutorials/starting_states.html)[译文](http://www.ituring.com.cn/article/42238)
2. [The Top file](http://docs.saltstack.com/en/latest/ref/states/top.html)
3. [Salt Formulas](http://docs.saltstack.com/en/latest/topics/development/conventions/formulas.html#conventions-formula)
4. [Hightstate date structure definitions](http://docs.saltstack.com/en/latest/ref/states/highstate.html#function-declaration)
5. [Highstate数据结构定义](http://www.ituring.com.cn/article/42245)

# 变更记录

|Why | Who | When |
|----|-----|------|
|create|fishcired|2014-08-05|
|完善sls文件编写|fishcried| 2014-08-12	|
|添加state执行顺序控制|fishcried|2014-08-13	|
|调整代码缩进错误| fishcried | 2014-11-12 |
