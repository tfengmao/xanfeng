---
title: 调试内核(debug)
layout: post
tags: linux kernel debug crash
category: linux
---

**为什么要调试内核**？  
- kernel crash了，去找原因。  
- 某个模块的行为不是预期的。  
- 理解代码。  

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
- 需要两台机器，测试机跑受测的内核(包含kgdb支持)，开发机跑gdb实例，gdb去连接kgdb。  
- 全方位调试：读写变量、设置断点和watchpoints、受限的单步执行等。  

upstream有对于kgdb和kdb的介绍："[Using kgdb, kdb and the kernel debugger internals](http://www.kernel.org/doc/htmldocs/kgdb.html)"。  
测试机启动受测内核时，要指定一些选项。  

选项之一是kgdboc，“kgdb over console”，主要用来配置gdb如何与kgdb通信。  
对于kgdb/gdb来说，kgdboc被设计为在串口下工作。有两种工作形式，第一种是用户使用serial console作为primary console并用来做内核调试，但是用串口作为console，用户需要将对应的支持编入内核（默认是未编入的，参考"[Linux Serial Console](http://www.kernel.org/doc/Documentation/serial-console.txt)"）；第二种是，即使串口不作console用途，kgdb仍然可以利用它。  
kgdboc可以被编译成内核的一部分，或者编译成内核模块。但是要使用kgdbwait做early debugging的话，kgdboc就必须被编译成built-in。  
kgdboc的具体配置参考[upstream文章](http://www.kernel.org/doc/htmldocs/kgdb.html#kgdboc)。  
串口波特率的查看：`setserial /dev/ttyS0 -a | grep baud_base`。  

选项之二是kgdbwait，它在boot内核时，让kgdb等着gdb的连接。上面说到，如果kgdboc是一个内核模块的话，kgdbwait是不会生效的。  

选项之三是kgdbcon，允许在gdb中看到被测内核的printk消息。怪异的是这个选项不能在作为system console的tty上使用。  

但是配置kgdb＋gdb的过程十分麻烦，我的希望两台虚机通过串口互联，但不知如何做到。  
1、我尝试通过virt-manager虚拟一个串口设备，将两个虚机的虚拟串口都绑定到主机的ttyS0（通过setserial -g查看，以UART不是unknown为依据，可以判断主机有两个物理串口ttyS0和ttyS4），但这显然想当然了，虚机A echo ttyS0并不能使得虚机B cat ttyS0读到内容。  
2、虚机A使用物理ttyS0，虚机B使用物理ttyS4，应该需要主机的两个串口物理连接。但我没有找到连接线，而且也不想用这种物理方法。  
3、虚机A串口虚机方式为output to file，实际上是将串口输出定向到主机对应的文件，但这却不能做到虚机互联。  
4、虚机A和虚机B都使用命名管道方式虚机串口，关联到主机的同一管道。这和方法1一样，有些想当然了，主机pipe.out同时只能获得一个虚机串口的输出，表明这种方式是适应虚机和主机之间联系的。  
5、...

**kdb**  
对比kgdb，只需要一台机器。  
不是一个source-level debugger，支持查看内存、寄存器等，甚至支持设置断点，主要用来做一些分析。  
据[elinux.org/KDB](http://elinux.org/KDB)，2.6.35内核才开始支持KDB，之前的通过补丁支持。  

**User Mode Linux([1](http://en.wikipedia.org/wiki/User-mode_Linux),[2](http://user-mode-linux.sourceforge.net/index.html))**  
让kernel像一个用户态进程一样跑起来，可以使用gdb直接调试。  
一些资源：  
1、[Installing User Mode Linux](http://web2.clarkson.edu/class/cs644/kernel/setup/uml/uml.html)  
2、[UML/User-mode Linux](http://lenky.info/2012/04/06/uml-user-mode-linux/)，[利用UML调试内核](http://lenky.info/2012/04/21/%E5%88%A9%E7%94%A8uml%E8%B0%83%E8%AF%95%E5%86%85%E6%A0%B8/)，[uml使用详细](http://lenky.info/2012/08/26/uml%E4%BD%BF%E7%94%A8%E8%AF%A6%E7%BB%86/)  
xen和UML可能是冲突的，因为参照资料中的步骤，make时会出现各种错误。不得其解，暂行搁置。  

参考  
- [Your kernel just oopsed - What do you do, hotshot?](http://www.mulix.org/lectures/kernel_oopsing/kernel_oopsing.pdf)  
- [搭建内核开发调试环境](http://adam8157.info/blog/2012/04/setup-kernel-developing-environment/)  
- [Linux kernel live debugging, how it's done and what tools are used?](http://stackoverflow.com/questions/4943857/linux-kernel-live-debugging-how-its-done-and-what-tools-are-used)  
- [Kernel Hacking Environment](http://unix.stackexchange.com/questions/9330/kernel-hacking-environment)  
- [使用KGDB调试内核 on QEMU(一步一步跟我学)](http://www.kgdb.info/kgdb/use_kgdb/using_kgdb_base_qemu/)  
- Linux Kernel Development, Chapter "Debugging".  
