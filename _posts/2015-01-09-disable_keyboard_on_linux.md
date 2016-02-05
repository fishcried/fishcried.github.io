---
layout: post
title: "linux中禁用笔记本键盘"
description: ""
category: 个人笔记
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [问题解决,个人笔记]
---

笔记本内置键盘不好用，桌子地方又小,只能这样了.

![hhkb](/img/hhkb_on_keyboard.png)


**查看输入系统信息**

    # xinput --list
    ⎡ Virtual core pointer                          id=2    [master pointer  (3)]
    ⎜   ↳ Virtual core XTEST pointer                id=4    [slave  pointer  (2)]
    ⎜   ↳ 2.4G Receiver                             id=13   [slave  pointer  (2)]
    ⎜   ↳ 2.4G Receiver                             id=14   [slave  pointer  (2)]
    ⎜   ↳ SynPS/2 Synaptics TouchPad                id=16   [slave  pointer  (2)]
    ⎣ Virtual core keyboard                         id=3    [master keyboard (2)]
        ↳ Virtual core XTEST keyboard               id=5    [slave  keyboard (3)]
        ↳ Power Button                              id=6    [slave  keyboard (3)]
        ↳ Video Bus                                 id=7    [slave  keyboard (3)]
        ↳ Video Bus                                 id=8    [slave  keyboard (3)]
        ↳ Power Button                              id=9    [slave  keyboard (3)]
        ↳ HP Truevision HD                          id=10   [slave  keyboard (3)]
        ↳ Topre Corporation HHKB Professional       id=11   [slave  keyboard (3)]
        ↳ 2.4G Receiver                             id=12   [slave  keyboard (3)]
        ↳ AT Translated Set 2 keyboard              id=15   [slave  keyboard (3)]
        ↳ HP WMI hotkeys                            id=17   [slave  keyboard (3)]
        ↳ HP Wireless hotkeys                       id=18   [slave  keyboard (3)]

**禁用内置键盘**
我的内置键盘是`AT Translated Set 2 keyboard`,禁用即可．

    # xinput set-prop 15 "Device Enabled" 0


# 变更记录

|Why | Who | When |
|----|-----|------|
|创建文件,提醒日后完善内容|fishcired|2015-01-09 |
