---
title: 中断 interrupt
layout: post
tags: interrupt hardware
category: linux
---

中文wiki页面表达得很好了: [中断](http://zh.wikipedia.org/wiki/%E4%B8%AD%E6%96%B7)

从中可知, 中断分为硬件中断和软件中断, 硬件中断有些可以通过硬件屏蔽, 有些不能被屏蔽, 如时钟中断. 软件中断是用特殊的 CPU 指令模拟的中断, 通常要运行一个切换到 kernel space 的子例程, 一般被用来实现 syscall.

中断的目的很简单. 比如, 厨师在做菜, 客人让你问菜做好了没有, 你要么定期去问问(轮询), 要么就是问一次, 然后让厨师好了时通知你一声. 在计算机里面, 程序问 OS 设备是不是 ready 了, OS 如果去轮询的话, 会浪费很多 CPU time, 所以不如让设备在 ready 后通过中断通知 OS.
