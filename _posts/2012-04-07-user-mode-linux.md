---
title: 试用UML、kdb和kgdb
layout: post
tags: user process linux kernel
category: linux
---

都是内核调试工具。UML：用户模式Linux，KDB：自己调试自己，KGDB：开发机调试目标机。  
关于内核调试，stackexhange上有总结：[Kernel Hacking Environment](http://unix.stackexchange.com/questions/9330/kernel-hacking-environment)。

###UML

UML，让kernel像一个用户态进程一样跑起来，可以使用gdb直接调试。

一些资源：  
1、[Installing User Mode Linux](http://web2.clarkson.edu/class/cs644/kernel/setup/uml/uml.html)  
2、[UML/User-mode Linux](http://lenky.info/2012/04/06/uml-user-mode-linux/)，[利用UML调试内核](http://lenky.info/2012/04/21/%E5%88%A9%E7%94%A8uml%E8%B0%83%E8%AF%95%E5%86%85%E6%A0%B8/)，[uml使用详细](http://lenky.info/2012/08/26/uml%E4%BD%BF%E7%94%A8%E8%AF%A6%E7%BB%86/)

xen和UML可能是冲突的，因为参照资料中的步骤，make时会出现各种错误。不得其解，暂行搁置。

###kdb

据[elinux.org/KDB](http://elinux.org/KDB)，2.6.35内核才开始支持KDB，之前的通过补丁支持。  
使用Putty的工作模式下，好像不适合使用kdb，网络可能被关闭，于是Putty断连。([更多...](http://debug-sai.blogbus.com/logs/47460470.html))。

###kgdb

[使用KGDB调试内核 on QEMU](http://www.kgdb.info/kgdb/use_kgdb/using_kgdb_base_qemu/)