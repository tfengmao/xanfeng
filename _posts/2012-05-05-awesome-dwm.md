---
title: Awesome -> dwm
layout: post
tags: window manager awesome gnome dwm
category: desktop
---

[Awesome](http://awesome.naquadah.org/) is awesome!  
[My first awesome](http://awesome.naquadah.org/wiki/My_first_awesome)  
[Awesome 3 configuration](http://awesome.naquadah.org/wiki/Awesome_3_configuration)  
[List of user configuration files](http://awesome.naquadah.org/wiki/User_Configuration_Files)

**Awesome** 已成历史, 它是我首次接触 Tiling Window Manager 用到的 wm, 就像初恋一样, 肯定会让我印象深刻, 如果没有出错的话, 也将肯定是我一生使用的 wm. 

然而, 遗憾的是, 它出错了, 而且不能恢复. 我不知道错在哪里, 我尝试了一些方法, 却仍然回不到从前. 我只是想换一个更好的 theme, 加一些更好的配件而已.

后来我凑巧碰到了 **[dwm](http://dwm.suckless.org/)**, 有着和初恋 Awesome 类似的外表, 却从命名上就更加地实在: dynamic window manager, 来自 suckless. ---- 就是这么简单直白!

它的内部也是一样的简单直白, 就是那么几个文件, dwm.c + config.h + Makefile, 通过 config.h 配置, 重编译安装后生效. 通过 patch 增加新的功能, 照样是重编译后生效.

当然, dwm 也会出错, 我也遇到了和 Awesome 一样无法进入的问题, 但却不再无计可施, 进入 tty0 重新编译安装原来的 dwm.c + config.h, 重新进入 X 就可以了.

喜欢一个人, 能不断地发现她的优点, 是一件很幸福的事情. 对于 dwm 来说, 我发现了它在 windows 下的版本: **[bug.n](http://www.autohotkey.net/~joten/bug.n.html)**, 对于当下必须使用 windows 的我来说, 也是一件幸福的事情:)
