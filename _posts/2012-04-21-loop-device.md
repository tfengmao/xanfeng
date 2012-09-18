---
title: loop device
layout: post
tags: loop device block pseudo
category: linux
---

[loop device](http://en.wikipedia.org/wiki/Loop_device) 的 loop 让我纠结了很多年了(...夸张了, 我接触 linux 也才两年), 我一直为 "loop" 的含义感到疑惑. 根据 [wiki](http://en.wikipedia.org/wiki/Loop_device#Availability):

> Sometimes, the loop device is erroneously referred to as 'loopback' device, but this term is reserved for a networking device in the Linux kernel (cf. loopback). The concept of the 'loop' device is distinct from that of 'loopback', although similar in name.

看来很多人都有类似的疑惑了, 都搞混了 "loop" 和 "[loopback](http://en.wikipedia.org/wiki/Loopback)" 的含义了, 后者是 network 的一个专有词汇.

那么 loop device 到底是什么呢?  
它只是一个 pseudo ("fake") device (autually just a file) that acts as a block-based device ([ref](http://unix.stackexchange.com/a/4536/12224)).  
那么为什么名为 "loop" 呢? 根据[这个](http://www.groad.net/bbs/read.php?tid-2352.html):  

> 在使用之前，一个 loop 设备必须要和一个文件进行连接。这种结合方式给用户提供了一个替代块特殊文件的接口。因此，如果这个文件包含有一个完整的文件系统，那么这个文件就可以像一个磁盘设备一样被 mount 起来。

> 理解一下 loop 之含义：对于第一层文件系统，它直接安装在我们计算机的物理设备之上；而对于这种被 mount 起来的镜像文件(它也包含有文件系统)，它是建立在第一层文件系统之上，这样看来，它就像是在第一层文件系统之上再绕了一圈的文件系统，所以称为 loop。

查看 loop device 文件: `ls /dev/loop*`.  
loop device 也可能不够用, 可以修改其数目配置, 然后重新 `modprobe loop` 即可.
