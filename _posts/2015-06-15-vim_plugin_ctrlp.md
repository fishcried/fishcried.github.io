---
layout: post
title: "vim插件ctrlp"
description: ""
category: vim
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [vim,ctrlp]
---

**What's ctrlp?**

![](http://zuyunfei.com/images/ctrl-p-buttons.png)

> Full path fuzzy file, buffer, mru, tag, ... finder for Vim.

ctrlp主要用来查找文件,vim必备插件.平常vim多文件编辑时文件切换都是用这个插件,只要敲入文件的一两个字母就能快速打开文件,不需要多余的思考.具体如何使用可以看一遍[中文手册](http://blog.codepiano.com/pages/ctrlp-cn.dark.html).一下只是记录自己经常用的快捷键.


- `c-d` 
  在全路径搜索和文件名搜索间切换。
  注意: 在文件名搜索模式，提示符面板的提示符是'\>d\>'，而不是'\>\>\>'
- `c-r`
  在字符串搜索模式和正则表达式模式之间切换。
  注意: 在全正则表达式模式，提示符面板的提示符是'r\>\>'，而不是'\>\>\>'
- 支持readline快捷键
- `c-t`
  使用标签打开文件
- `c-v`
  竖立分割窗口打开
- `c-s`
  水平分割窗口打开
- `c-y`
  创建一个文件和他的父目录
- `c-z`
  标记或取消一个文件
- `c-o`
  打开标记文件
- `c-\\`
  打开一个终端对话框来粘贴 `cword`， `cfile`，搜索寄存器的文本，上一次可视 化模式的选择，剪贴板或者任何寄存器到提示符面板中。


# 变更记录

|Why | Who | When |
|----|-----|------|
|create|fishcired|2014-10-21 |
|添加快捷键记录|fishcired|2014-11-27  |
|简单整理|fishcired|2015-06-15   |
