---
layout: post
title: "Ansible(二) 主机管理"
description: ""
category: Linux系统管理
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [ansible,devops,配置管理]
---

# Ansible(二) 主机管理

## 主机资产

其实所谓的资产的管理无非就是对主机的管理，主机管理的关键清晰的识别主机。那么如何清晰的对主机进行识别那，当然是赋予主机清晰的属性（元数据），然后对主机进行分组了。

上面主机属性其实就是变量，不但主机具有变量，主机组也具有变量,scope更大一些。Inventory主要就是怎么对进行主机分组，怎样定义变量。主机资产通过一个ini格式文件来定义，例如下面的例子,感受一下:


```ini
# file: hosts

[atlanta-webservers]
www-atl-1.example.com 
www-atl-2.example.com 

[boston-webservers]
www-bos-1.example.com
www-bos-2.example.com

[atlanta-dbservers]
db-atl-1.example.com
db-atl-2.example.com

[boston-dbservers]
db-bos-1.example.com

# webservers in all geos
[webservers:children]
atlanta-webservers
boston-webservers

# dbservers in all geos
[dbservers:children]
atlanta-dbservers
boston-dbservers

# everything in the atlanta geo
[atlanta:children]
atlanta-webservers
atlanta-dbservers

# everything in the boston geo
[boston:children]
boston-webservers
boston-dbservers
```

## 怎样分组

inventory其实就是一个ini文件，遵循ini语法，由于ini本身就有配置分组的语法，所以分组很简单，指定section就是一个组。

### 普通分组

```
[atlanta-webservers]
www-atl-1.example.com 
www-atl-2.example.com 

[boston-webservers]
www-bos-1.example.com
www-bos-2.example.com

[atlanta-dbservers]
db-atl-1.example.com
db-atl-2.example.com

[boston-dbservers]
db-bos-1.example.com
```

上面有三个组，每组里面定义自己的主机。

### 组中组
```ini
# webservers in all geos

[atlanta-webservers]
www-atl-1.example.com 
www-atl-2.example.com 

[boston-webservers]
www-bos-1.example.com
www-bos-2.example.com

[webservers:children]
atlanta-webservers
boston-webservers
```


组webservers的成员是前两个组的全部成员，当然不用一个个的重写一遍，如果重写，当前面任意一组变换时,webservers成员也需要调整，这也太傻了。所以出现了新的语法`[xxx:childer]`来解决问题.

### 指定主机时的一些花架子

```ini
[webservers]
www[01:50].example.com
[databases]
db-[a:f].example.com
```

`:`可以表示`to`的«义,`[01:50]`表示从01到50，这样就不用一个一个的写了,同样`[a:f]`表示a-f的意思。

## 变量

### 主机变量

在主机定义时，直接定义相关变量.
```
[atlanta]
host1 http_port=80 maxRequestsPerChild=808
host2 http_port=303 maxRequestsPerChild=909
```

### 组属性

同一组的主机具有相同的变量时使用组变量比较方便，同样适用vars来进行识别。

```
[atlanta]
host1
host2
[atlanta:vars]
ntp_server=ntp.atlanta.example.com
proxy=proxy.atlanta.example.com
```

**常用变量**

有些变量是系统预设含义的,下面列举几个常用的

- ansible_ssh_host 将要连接的远程主机名.与你想要设定的主机的别名不同的话,可通过此变量设置.
- ansible_ssh_port ssh端口号.如果不是默认的端口号,通过此变量设置.
- ansible_ssh_user 默认的 ssh 用户名
- ansible_ssh_private_key_file ssh 使用的私钥文件.适用于有多个密钥,而你不想使用 SSH 代理的情况.




## 主机匹配

一般使用组名进行匹配就可以了，下面总结了主机匹配时的与或非。

| 逻辑 | 语法 |  含义  |
|------|------|--------|
| all  | `*`,all | 匹配所有 |
| and  | `g1:&g2` | 在组g1且也在组g2中 |
| or  | `g1:g2` | 在组g1或组g2中的主º|
| not  | `g1:!g2` | 组g1但不能是在g2中的主机 |
| 正则  | `~`开头的表示使用的是正则  | 通过正则表达式去匹配 |
