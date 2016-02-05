---
layout: post
title: "python的数据类型"
description: ""
category: 程序设计
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [python]
---

# 1.序列

序列中的每个元素都有自己的编号。Python中有6种内建的序列。其中列表和元组是最常见的类型。其他包括字符串、Unicode字符串、buffer对象和xrange对象。

## 1.1 序列的通用操作

- 索引
  索引如果从左面则从0开始,从右面则从-1开始.
- 分片
  分片操作用来访问一定范围内的元素。分片通过冒号相隔的两个索引来实现.`obj[start:end[:step]]`
- 加法
- 乘法
- in
- max,min,len

## 1.2 列表

|方法 | 含义
|----+----------------------------------------
| list.append() | 追加成员，成员数据
| list.pop() | 删除成员,删除第i个成员
| list.count(x) | 计算列表中参数x出现的次数
| list.remove() | 删除列表中的成员，直接删除成员i
| list.extend(L) | 向列表中追加另一个列表L
| list.reverse() | 将列表中成员的顺序颠倒
| list.index(x) | 获得参数x在列表中的位置
| list.sort() | 将列表中的成员排序
| list.insert() | 向列表中插入数据insert(a,b)向列表中插入数据

**管用法**

    for i,v in enumerate(lst):
        print i,v

## 1.3 元组

TBD

## 1.4 字符串

格式化`str1='Hello,%s' % 'world.'`

## 1.4.1 常用方法

- 常用操作(查找,替换,分割等)
    - find() / rfind()
    - index() / rindex() / lindex()
    - replace()
    - join()
    - split()
    - strip() / lstrip() / rstrip()
    - upper()
    - lower()
    - count()
- 类别判断
    - isalpha()
    - isdigit()
    - islower()
    - isupper()
    - isspace()
- 转换
    - format()
    - center() / ljust() / rjust() / zfill()
    - decode()
    - encode()

# 2. 映射

## 2.1 字典

字典定义了键值的一一对应的关系,是非常有用的数据结构.python中内置了该数据类型,可惜的是c中默认没有该类
型.字典类型的特性:

- key不能重复,且大小写敏感
- values可以是任意数据类型.
- 随时可以加入新的键值对,语法同修改是一样的.这种多重语义不知道时好时坏.反正没遇到灾难的时候是美妙的.
- 字典是无序的.也就是两次遍历字典获取的元素顺序不一定是相同的.

###  2.1.1 字典常用操作

**1. 定义创建**

    >>> dict1 = {}
    >>> dict1
    {}

    >>> dict2 = {'name': 'earth', 'port':80}
    >>> dict2
    {'name': 'earth', 'port': 80}

    >>> fdict = dict((['x',1],['y',2]))
    >>> fdict
    {'y': 2, 'x': 1}

    >>> ddict = {}.fromkeys(('x','y'), -1)
    >>> ddict
    {'y': -1, 'x': -1}

**2. 添加与修改**

    # 添加
    >>> dict1 = {}
    >>> dict1['a'] = 'a'

    # 修改
    >>> dict1 = {'a': 'a'}
    >>> dict1['a'] = 'b'


**2. 访问**

    >>>> for key in dict2:
    ... print 'key=%s, value=%s' % (key, dict2[key])

**4. 删除**

    del dict2['name'] # 删除键为“name”的条目
    dict2.clear() # 删除dict2 中所有的条目
    del dict2 # 删除整个dict2 字典
    dict2.pop('name') # 删除并返回键为“name”的条目

###  2.1.2 字典内建方法

| 名称         |                         操作
|--------------+-----------------------------
| dict.clear()                       |   删除字典中所有元素
| dict.copy()                        |   返回字典(浅复制)的一个副本
| dict.fromkey()                     |   通过key快速创建字典 
| dict.get(key,default=None)         |   对字典dict中的键key,返回它对应的值value，如果字典中不存在此键，则返回default 的值(注意，参数default的默认值为None
| dict.has_key                       |   如果字典中有该key,返回True
| dict.keys()                        |   返回一个包含字典中键的列表
| dict.items()                       |   返回一个包含字典中(键, 值)对元组的列表
| dict.iteritems                     |   迭代items
| dict.values()                      |   返回一个包含字典中所有值的列表
| dict.itervalues                    |   迭代values
| dict.iter()                        |   方法iteritems(), iterkeys(), itervalues()与它们对应的非迭代方法一样，不同的是它们返回一个迭代，而不是一个列表。
| dict.pop(key[, default])           |   和方法get()相似，如果字典中key 键存在，删除并返回dict[key]，如果key键不存在，且没有给出default 的值，引发KeyError 异常。
| dict.popitem                       |   ...
| dict.setdefault(key,default=None)  |   和方法set()相似，如果字典中不存在key 键，由dict[key]=default为它赋值。
| dict.update                        |   ...
| dict.viewitems                     |   ...
| dict.viewkeys                      |   ...
| dict.viewvalues                    |   ...

### 2.1.3 惯用法

**键值存在**

    key in dct

**get with default value**
    
    dct[key] = dct.get(key, 0) + 1


**set with default value**

    dct = {}
    for (key,value) in data:
        group = dct.setdefault(key,[])
        group.append(value)
 
# 4 集合

- 元素唯一,重复会被忽略
- 没有顺序

## 4.1 常用方法

- union
- add
- remove

# 5 更多阅读

- http://blog.jobbole.com/65218/

# 变更记录

|Why | Who | When |
|----|-----|------|
|create|fishcired|2014-10-14 |
|审阅修正错误|fishcired|2014-10-19 |
|添加惯用法|fishcired|2014-11-11 |
