---
layout: post
title: "linux commands tips: grep"
description: ""
category: 系统管理
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [命令行使用]
---

阅读完全文需要一点耐心,只希望能提供一点参考价值.

# 1. 使用技巧

## 1.1 多个关键字查找

通过使用频率来看,多关键查询并不常见.但很多时候确实需要,简单了解下吧.

### 1.1. 关键字间的或操作

**0x01 使用`-e`选项**

    #包含了string.h或者stdlib.h的头文件
    grep -l -e 'string\.h' -e 'stdlib\.h' /usr/include/*.h

> -e 选项还可以避免 关键字是-开头的导致选项解读失败。

**0x02 使用元字符 `\|`**

    grep 'strint\.h\|stdlib\.h' /usr/include/*.h

> 使用-E,可以避免很多的\,书写和阅读都便利

    grep -E 'string\.h|stdlig\.h' /usr/include/*.h

**0x03 使用-f file**

    cat >multi_pattern
    stdlib\.h
    string\.h
    grep -l -f multi_pattern /usr/include/*.h

### 1.1.2 关键字间之间的与操作

**0x01 通过管道**

    #同时包含'hello','world'的行
    echo hello world | grep '\<hello\>' | grep '\<world\>'

> 通过管道是最常见的方式

**0x02 通过正则 `|`**

    grep -E 'pattern1.*pattern2|pattern2.*pattern1

如果包含2个关键字还好，要是n个就有n全排序列种可能！   

## 1.2. 全词匹配

**0x01 使用选项-w(gnu 选项)**

    grep -w 'main' /usr/include/*.h

**0x02 使用`\<\>`**

grep '\<main\>' /usr/include/*.h

两者比较来看,`-w`无疑是更为便利的.全词匹配是常见的需求.如果可以,非常推荐随时指定全词匹配.

##  1.3.善用 -E

    -E选项启用 extended expression,正则写起来更加灵活

    #查看gcc帮助文件里两个the/that/and/or连在一起的行
    man gcc | grep -E '(\<the\>|\<that\>|\<and\>|\<or\>) \1'
    man gcc | grep -E -w '(the|that|and|or) \1'
    #查看gcc帮助文件里含两个连续单词的行
    man gcc | grep -E -w '([a-zA-Z] ) \1'

> 使用-E让书写更方便，省去很多的\,同时功能更强大。在多关键字查询里也提到了.

## 1.4. 忽略大小写 -i

    #查看INT_MAX的值
    grep -i 'int_max' /usr/include/limits.h

    -i与\n同时使用的乱象

    #匹配连续相同单词
    echo 'it IT' | grep -i -w -E '([a-z] ) \1'
    echo 'it IT' | grep -E -w '([a-zA-Z] ) \1'

这是两个相同的单词吗？是的，因为告诉grep不计大小写的！有的时候不要光图方便会不准确。

## 1.5. 递归查找 -r(posix 未说明)

    #查看日志的错误信息
    grep -i -w -r -E 'error|failed|failure' /var/log |less


## 1.6. 显示匹配行周围行 (posix 未说明)

    B/A/C(before/after/context
    -B n
    -A n
    -C n

## 1.7. 取反-v

    grep -v -w 'hello' filename

如果没有取反，世界将不再美丽

## 1.8. 匹配数 -c

    echo aaaa | grep -c 'a'

这个输出是1！因为grep是行匹配的

## 1.9. 输出文件名 -l

    grep -l -r -i -w 'filename_max' /usr/include/*.h


## 1.10. 只输出匹配部分-o (gnu 选项)

    echo abcddf |grep -o 'dd'

很多时候,就是用这个参数来验证正则书写是否正确

## 1.11. 如果是纯字符串搜索，-F 速度更快。

做个实验：

    #用gcc manual生成个纯字符串文件作为搜索关键字
    man gcc | tr -cs '[:alpha:]' '\n' >grep.date
    wc -l grep.date
    97288 #这么多！       


    #比较不带-F，与带-F
    time `man gcc | grep -F -f grep.date > /dev/null`
    real 0m0.499s
    user 0m0.741s
    sys 0m0.056s
    time `man gcc | grep -f grep.date > /dev/null`
    real 4m9.630s
    user 4m7.602s
    sys 0m0.713s

   
仔细看看时间的对比！ 当纯字符串匹配，尤其是要匹配的字符串非常多，-F不可不用。

## 1.12. 在查找进程的时候，利用[]实现同时grep -v grep的功能

举例：

    [Bob]@[Fck_without_U]:[~]-> ps -ef | grep "java -jar"
    tdlteman 22023 22006 0 Oct07 ? 00:09:58 java -jar slave.jar
    xiabao 31501 30737 0 11:08 pts/8 00:00:00 grep java -jar 
	#grep 自身也出来了

    [Bob]@[Fck_without_U]:[~]-> ps -ef | grep "java -jar" | grep -v grep
    tdlteman 22023 22006 0 Oct07 ? 00:09:58 java -jar slave.jar
    #grep -v grep 精确结果

    [Bob]@[Fck_without_U]:[~]-> ps -ef | grep "[j]ava -jar"
    tdlteman 22023 22006 0 Oct07 ? 00:09:58 java -jar slave.jar
    #看，神奇的事情发生了

巧妙！关于原理其实也很简单,自己想想应该可以理解.

# 2. 注意事项

**1. [a-d] 与 [abcd] 不一定等价**

正则表达式中出现range[]是里面的序列是受字符集和环境变量影响的。典型的，许多 locale 将字符以字典顺序排序，
在这些 locale 中， [a-d] 不与 [abcd] 等价；例如它可能与 [aBbCcDd] 等价。

    [iscs@linux-sp1]:/users/iscs>$ echo "ABC
    > abc"|grep '[a-c]'
    ABC
    abc
    [iscs@linux-sp1]:/users/iscs>$ echo "ABC
    abc"|grep -E '[a-c]'
    ABC
    abc

为了提高可移植性:

1. 使用POSIX定义的字符组
2. 定义环境变量LC_ALL

**2. 匹配 - 开头的关键字 如(--shit)grep出错**

如果关键字以-开头，会打乱grep对选项的解析.解决： 使用 -e选项 或 --

    [iscs@linux-sp1]:/users/iscs>$ echo "--shit"|grep --shit
    grep: unrecognized option '--shit'
    Usage: grep [OPTION]... PATTERN [FILE]...
    Try `grep --help' for more information.
    [iscs@linux-sp1]:/users/iscs>$ echo "--shit"|grep -- --shit
    --shit
    [test@test~]$ echo '--shit' | grep -e '--shit'
    --shit

**3. 关于效率问题**

多人服务器开发的时候,经常遇到`vim`不响应,cpu load飙到令人发指的情况.top发现`grep`是元凶.原因就是有人很随意的执行`grep "xxx" . -rn`.
该命令看似没有什么问题,但是当你的目录内文件非常多且大,就会导致cpu利用率上来.而且多人共同使用的机器,其他人很容易感觉到.有比这个更狠的.
`find / --name "xxx" `.

每次使用`grep`,`find`之类的命令,最好应该提供更多的信息. 

- `-F` 如果你要查询的是一个明确的字符串,不需要正则.那么使用`-F`加速
- `include/exclude` 如果你要匹配的内容肯定在`.c`文件内,那么可以使用`include`来指定文件.
- `- w` 如果你要匹配的是单词

指定很多参数确实是很麻烦的.一般可以通过别名来简化.例如定义一个`cgrep`,专门在查询c代码.`alias cgrep='grep -nr --include *.[ch]'`

> 负担总是存在,把负担留给你的大脑还是留给机器.看情况而定.但前提是不要影响他人.
> 大脑多一点负担也是一种知识巩固.时间久了手指会形成机械记忆,只是需要一个坚持的过程罢了.

# 变更记录

|Why | Who | When |
|----|-----|------|
|将早期的<grep技巧12则>移植过来,重新维护|fishcired|2014-09-16 |
|添加关于效率问题的注意事项|fishcired|2014-09-16 |

