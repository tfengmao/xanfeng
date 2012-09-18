---
title: Linux kernel debug
layout: post
tags: linux kernel debug crash
category: linux
---

**为什么要调试内核**？  
- kernel crash了，去找原因。  
- 某个模块的行为不是预期的。  

**kernel的特点**：  
- Supports runtime loading and unloading of additional code(kernel modules)  
- Conﬁgured using Kconﬁg, a domain specifc conﬁguration language.  
- Built using kbuild, a collection of complex Makeﬁles.  
- Heavily dependant on gcc and gccisms. Does not use or link with user space libraries, although supplies many of them - sprintf, memcpy, strlen, printk (not printf!).  

最原始却最有效的调试方法：  
- Your brain  
- printf。kernel 提供printk，可以在**中断上下文**中被调用，与printf类似，不支持float。

**kernel debugger**：  
- kdb  
- kgdb  
- LKCD  
- ksymoops  
- kdump  
- User Mode Linux

**kgdb**  
- Requires two machines, a slave and a master.  
- gdb runs on the master, controlling a gdb stub in the slave kernel via the [serial port](http://xanpeng.github.com/2012/04/06/linux-serial-port/).  
- When an OOPS or a panic occurs, you drop into the debugger.  
- Very very useful for the situations where you dump core in an interrupt handler and no oops data makes it to disk - you drop into the debugger with the correct backtrace.  

**User Mode Linux([1](http://en.wikipedia.org/wiki/User-mode_Linux),[2](http://user-mode-linux.sourceforge.net/index.html))**  
- For some kinds of kernel development(architecture independent, **file systems**, memory management), using UML is a life saver.  
- Allows you to run the Linux kernel in user space, and debug it with gdb.  
- Work is underway at making valgrind work on UML, which is expected to ﬁnd many bugs.  

但是目前User Mode Linux[不能为我理解和接受](http://xanpeng.github.com/2012/04/07/user-mode-linux/)。

参考  
- [Your kernel just oopsed - What do you do, hotshot?](http://www.mulix.org/lectures/kernel_oopsing/kernel_oopsing.pdf)  
- [搭建内核开发调试环境](http://adam8157.info/blog/2012/04/setup-kernel-developing-environment/)  
- [Linux kernel live debugging, how it's done and what tools are used?](http://stackoverflow.com/questions/4943857/linux-kernel-live-debugging-how-its-done-and-what-tools-are-used)  
