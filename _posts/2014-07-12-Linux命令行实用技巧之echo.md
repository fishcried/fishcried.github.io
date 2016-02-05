---
layout: post
title: "linux commands tips: echo"
category: 系统管理
tags: [命令行使用]
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
---

# 参数说明 

* -e print escaped characters
* -n suppress the newline

#  使用技巧

> *Be aware* that echo `command` deletes any linefeeds that the output of command generates.
>
> The $IFS(interal field separator) variable normally contains \n(linefeed) as one of its set of whitespace characters.Bash therefore splits output of command at linefeeds into arguments to echo. Then echo outputs these arguments, separated by spaces.

-  标志信息要出现在字符串之前，否则bash会将其是为另外一个字符串.`echo hello -e`.
-  -e 可是支持转义字符 `echo -e "1\t2\t3"`
-  打印彩色输出 `echo -e "\e[1;31m This is  red text\e[0m"` [More](http://www.freeos.com/guides/lsst/misc.html)
    
# 变更记录

|Why | Who | When |
|----|-----|------|
|整理下之前的记录|fishcired|2014-07-12 |
|添加颜色链接|fishcried|2014-08-22 |
