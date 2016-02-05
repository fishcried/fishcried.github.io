---
layout: post
title: "vim实用技巧之多buffer操作"
description: ""
category: vim
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [vim]
---

> `使用多buffer操作多文件,是vim的一条必经之路.早点到达这个阶段,会节省很多时间. `


# 1. 很久的一个疑问

> **多文件编辑,使用buffer还是tab?**这是我曾经遇到的问题. 一般来讲,tab是首选,因为tab编辑标题可见,而buffer只能看到当前文件名称,其他文件名称则被 隐藏,有种失控的感觉.所以平常我都尽力使用tab功能进行多文件编辑.可是tab也有缺陷,但是tab不能一下子开太多.而且tab的切换并不迅捷.所以又回到原来的问题.看一下别人是否遇到这种困境:

- 1.[stackoverflow的讨论](http://stackoverflow.com/questions/102384/using-vims-tabs-like-buffers/103590#103590)
- 2.[ruby-china.org的讨论](https://ruby-china.org/topics/1271)
- 3.[Vim Tab Madness. Buffers vs Tabs](Vim Tab Madness. Buffers vs Tabs)

以上讨论建议尽量使用buffer来进行多文件编辑.原因这里不概括,非常希望能仔细阅读上面链接.不得不感叹,与外面的世界保持畅通多么重要,否则只能坐井观天.

一再强调的三个概念:

> A buffer is the in-memory text of a file.
> A window is a viewport on a buffer.
> A tab page is a collection of windows.

# 2. 高效操作buffers技巧

牛人多推荐使用buffer,那么就试试吧.但是如果只是用系统的配置来进行操作,只能吐了.vim默认操作buffer的方式使用起来不太方便.`:bnext`,`:bpre`简单的buffer切换需要按几次键盘(而且需要左手按shift),那么太痛苦了.所以需要简单的配置下.

**配置**
    
    set hidden " 避免必须保存修改才可以跳转buffer

    " buffer快速导航
    nnoremap <Leader>b :bp<CR>
    nnoremap <Leader>f :bn<CR>

    " 查看buffers
    nnoremap <Leader>l :ls<CR>

    " 通过索引快速跳转
    nnoremap <Leader>1 :1b<CR>
    nnoremap <Leader>2 :2b<CR>
    nnoremap <Leader>3 :3b<CR>
    nnoremap <Leader>4 :4b<CR>
    nnoremap <Leader>5 :5b<CR>
    nnoremap <Leader>6 :6b<CR>
    nnoremap <Leader>7 :7b<CR>
    nnoremap <Leader>8 :8b<CR>
    nnoremap <Leader>9 :9b<CR>
    nnoremap <Leader>0 :10b<CR>

**快速切换技巧**

1. 切换buffer自动补全非常重要,方便. `:b pattern<tab>`会触发自动补全,而且pattern不必非得是名称的开始.
1. `c-^`可以在两个buffer中转轮切换. 
1. `'"`,上一次离开的位置, `'.`上一次编辑的位置.大写的字母是全局标记,可用用于文件跳转.
1. `Ctrl-I` `Ctrl-O`也可以用于文件的标记的快速跳转.

**推荐的buffer插件**

- MiniBufExploer 查看和操作buf的插件
	主要使用该插件的查看功能.`:ls`太麻烦了,没人愿意切换前都来这么一下.该插件能同时显示buffer的名称和索引.正好为上面的跳转快捷键准备的.每次删除buffer后,新建buffer索引会递增.如果索引大于10了,请尽量使用`b pattern<tab>`来跳转,而不要使用MiniBufExploer来操作,用几次就知道哪个高效.
- [CtrlP](http://blog.codepiano.com/pages/ctrlp-cn.dark.html) 神级插件
	推荐一个[视频](http://happycasts.net/episodes/64)
- vim-one 保持只有一个vim窗口
	如果使用多个vim打开多个文件,那么vim进程之间是不能直接复制粘贴等操作.所以文件都在一个vim中打开成为一种需求.这个插件正好解决了该问题.该插件主要是通过vim选项(`--servername`,`--remote-slient`)来做到的.上面的视频链接也提到了.

# 3. buffer的基础知识

**获取帮助**

`:help buffers`

**buffer标志**

| 标志 | 含义
|--------+---------
| u |列表外缓冲区 |unlisted-buffer|。
| % | 当前缓冲区。
| # | 轮换缓冲区。
| a | 激活缓冲区,缓冲区被加载且显示。
| h | 隐藏缓冲区,缓冲区被加载但不显示。
| = | 只读缓冲区。
| - | 不可改缓冲区, ’modifiable’ 选项不置位。
| + |已修改缓冲区。



**创建**

编辑新文件，就会自动创建.

- `:edit file`
- `new`
- `:vnew`
- `:badd 1.txt`

**查看**

- `ls`
- `buffers`
- `files`

**删除**

- `bdelete f1.txt`
- `bdelete 1`
- `bwipeout` 完全删除buffer
- `3,5bdelete`
- `bdelete 1.txt 2.txt 3.txt`
- `bunload`

**多buffer查找替换**

`:bufdo %s/pattern/replace/ge | update`

# 变更记录

|Why | Who | When |
|----|-----|------|
|编写|fishcired|2014-10-22 |
|完善|fishcired|2014-10-24  |
|添加插件说明|fishcired|2014-10-25  |
|整理格式|fishcired|2014-11-28   |
