---
layout: post
title: "xargs命令使用技巧"
description: ""
category: 系统管理
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags:
    - 命令行技巧
    - xargs
    - find
---

# 1. xargs的用途

使用`find`命令常常会看到`xargs`，他的用途是构造参数列表并运行命令。

`man xargs`的description字段如下说明:

> xargs reads items
> from  the  standard  input, delimited by blanks (which can be protected
> with double or single quotes or a backslash) or newlines, and  executes
> the  command (default is /bin/echo) one or more times with any initial-
> arguments followed by items read from standard input.  Blank  lines  on
> the standard input are ignored.

# 2. xargs常用的options

- `-I`	     replace_str 指定替换的字符串,`-i`为废弃选项
- `-0`		 使用'\0'来代替空白分隔符
- `-a file`	 从文件读取参数
- `-n n`	 指定每行参数的数量n
- `-d delim` 指定定界符
- `-r`		 如果没有参数传入，就不执行。默认是执行，gnu选项
- `-p`		 交互处理

# 3. 应用实例

1 将多行输入转换成单行输出 

	$cat example.txt
	a b c 
	d e
	f
	$cat example.txt | xargs
	a b c d e f

2 将单行输入转换成多行输出 

	$cat example.txt | xargs -n 2
	a b
	c d
	e f

3 -r处理没有输入数据的情况

	mkdir empty_dir
	cd !$
	ls | xargs rm

执行以上命令会出错，因为当前目录下没有文件可删除，rm后面没有参数。这个时候应该使用`-r`选项。

	ls | xargs -r rm

4 -0处理带空格命名的文件

	touch a\ b
	ls a\ b | xargs rm

执行以上命令会出错，因为传递给xargs的是`a b`,该文件中间带有空格，执行删除命令失败。正确的应该是

	find -name 'a b' -type f -print0 | xargs -r0 rm
	ls a\ b | xargs -I{} rm {}

5 find与xargs的联用

1. -exec 参数过长问题
1. find多进程问题
1. find的文件带空格处理方式

	find . -type f -name "*.txt" -print | xargs  rm -f
	find . -type f -name "*.txt" -print0 | xargs -r0 rm -f
	find . -type f -name "*.swp" -print0 | xargs -r0 rm -f

这是非常危险的，因为xargs是以空格为delimiter,如果文件名包含空格，那么就会出错，甚至误删

在使用find命令的-exec选项处理匹配到的文件时， find命令将所有匹配到的文件一起传递给exec执行。但有些系统对能够传递给exec的命令长度有限制，这样在find命令运行几分钟之后，就会出现溢出错误。错误信息通常是“参数列太长”或“参数列溢出”。这就是xargs命令的用处所在，特别是与find命令一起使用

# 修改记录

|Why | Who | When |
|----|-----|------|
|create|fishcired|2014-06-22|
