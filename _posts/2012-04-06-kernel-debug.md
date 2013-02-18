---
title: 调试内核
layout: post
tags: linux kernel debug crash
category: linux
---

**为什么要调试内核**？  
- kernel crash了，去找原因。  
- 某个模块的行为不是预期的。  
－ 理解代码。  

**kernel的特点**：  
- 支持动态加载／卸载内核模块代码。  
- 内核是通过[Kconfig](http://www.kernel.org/doc/Documentation/kbuild/kconfig-language.txt)配置的。  
- 内核是通过kbuild(a collection of complex Makeﬁles)构建的。  
- 重度依赖于gcc和gccisms，不使用任何用户态函数库。  

最原始却最有效的调试方法：  
- Your brain。  
- printf。kernel 提供printk，可以在中断上下文中被调用，与printf类似，不支持float。

**有哪些工具？**：  
- kdb  
- kgdb  
- LKCD  
- ksymoops  
- kdump  
- User Mode Linux

或许你会问：这些是什么东西，我只知道gdb...  
不错，使用gdb可以瞟到running kernel，执行命令`gdb vmlinux /proc/kcore`即可。  
但使用gdb有根本的限制：不能修改数据，不能单步执行。  

**kgdb**  
- 是一个补丁，支持通过serial line，使用gdb远程全方位调试内核：读写变量、设置断点和watchpoints、单步执行等。  
- 需要两台机器，第一台运行打好kgdb补丁的内核，第二台通过serial line调试第一台。  

**User Mode Linux([1](http://en.wikipedia.org/wiki/User-mode_Linux),[2](http://user-mode-linux.sourceforge.net/index.html))**  
让kernel像一个用户态进程一样跑起来，可以使用gdb直接调试。  
一些资源：  
1、[Installing User Mode Linux](http://web2.clarkson.edu/class/cs644/kernel/setup/uml/uml.html)  
2、[UML/User-mode Linux](http://lenky.info/2012/04/06/uml-user-mode-linux/)，[利用UML调试内核](http://lenky.info/2012/04/21/%E5%88%A9%E7%94%A8uml%E8%B0%83%E8%AF%95%E5%86%85%E6%A0%B8/)，[uml使用详细](http://lenky.info/2012/08/26/uml%E4%BD%BF%E7%94%A8%E8%AF%A6%E7%BB%86/)  
xen和UML可能是冲突的，因为参照资料中的步骤，make时会出现各种错误。不得其解，暂行搁置。  

**kdb**  
自己调试自己。  
据[elinux.org/KDB](http://elinux.org/KDB)，2.6.35内核才开始支持KDB，之前的通过补丁支持。  
使用Putty的工作模式下，好像不适合使用kdb，网络可能被关闭，于是Putty断连。([更多...](http://debug-sai.blogbus.com/logs/47460470.html))。  

参考  
- [Your kernel just oopsed - What do you do, hotshot?](http://www.mulix.org/lectures/kernel_oopsing/kernel_oopsing.pdf)  
- [搭建内核开发调试环境](http://adam8157.info/blog/2012/04/setup-kernel-developing-environment/)  
- [Linux kernel live debugging, how it's done and what tools are used?](http://stackoverflow.com/questions/4943857/linux-kernel-live-debugging-how-its-done-and-what-tools-are-used)  
- [Kernel Hacking Environment](http://unix.stackexchange.com/questions/9330/kernel-hacking-environment)  
- [使用KGDB调试内核 on QEMU(一步一步跟我学)](http://www.kgdb.info/kgdb/use_kgdb/using_kgdb_base_qemu/)  
- Linux Kernel Development, Chapter "Debugging".  
