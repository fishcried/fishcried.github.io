---
layout: post
title: "vim多文件操作"
description: ""
category: vim
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [vim]
---

**多文件查找**

使用ack插件

    nmap <leader>ack :Ack
    nmap <leader>ackw :call Search_Word()<cr>
    function Search_Word()
        let w = expand("<cword>")
        execute "Ack " . w
    endfunction

**多文件替换**

    :args *.txt *.cpp
    :argdo %s/hate/love/gc | update

    :args **/*.txt 递归下级目录

**多文件编辑**

[vim实用技巧之多buffer操作](/2014-10-25/vim实用技巧之高效的buffer操作/)


# 变更记录

|Why | Who | When |
|----|-----|------|
|创建文件,提醒日后完善内容|fishcired|2015-03-31 |
