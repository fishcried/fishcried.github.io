---
layout: post
title: "Python调试器pdb/ipdb"
description: ""
category: 程序设计
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [调试,python,pdb]
---

> The module pdb defines an interactive source code debugger for Python programs. It supports setting (conditional) breakpoints and single stepping at the source line level, inspection of stack frames, source code listing, and evaluation of arbitrary Python code in the context of any stack frame. It also supports post-mortem debugging and can be called under program control.

# 1. 调用pdb

**源码调用**
`file: myscript.py`

	import pdb
	pdb.set_trace()
	...

**交互方式调用**

	>>> import pdb
	>>> import mymodule
	>>> pdb.run('mymodule.test()')
	> <string>(0)?()
	(Pdb) continue
	> <string>(1)?()
	(Pdb) continue
	NameError: 'spam'
	> <string>(1)?()
	(Pdb)

**命令行式**
`python -m pdb myscript.py`

# 2. 常用命令

![pdb](/img/pdb.png)

# 3. pdb的替代

pdb如果真的使用了一会,会感觉到诸多不变,什么玩意,和gdb差远了! 其实主角搞错了,真正的一号应该是ipdb,ipdb提供了自动补全,更多的命令.试用ipdb一会,感觉ipdb和gdb更像.这个才算是调试器.


# 变更记录

|Why | Who | When |
|----|-----|------|
|create|fishcired|2014-09-02 |
|添加ipdb|fishcired|2014-11-06 |

