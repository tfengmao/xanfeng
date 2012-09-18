---
title: 浅尝kdump+crash
layout: post
category: linux
tags: kdump crash kernel dump memory
---

kdump是一种内核转储机制，而[crash](http://people.redhat.com/anderson/)则是分析转储文件的工具。下文会分析kdump如何实现转储，但crash如何解析转储文件，并从中获取很多有用信息，还不得而知。

*后来发现，本文部分解答了我在“[Kernel Crash Logging and Core Dump](http://xanpeng.github.com/linux/2012/04/09/kernel-crash-logging.html)”中的疑惑。*

###kdump

先列出kdump的资料：  
1、王聪的“[kdump: introduction and overview](http://wangcong.org/blog/wp-content/uploads/2010/12/rh-kdump-2.pdf)”。优点是权威，缺点是简略。  
2、某网友的“[linux的崩溃转储-kdump的艺术](http://blog.csdn.net/dog250/article/details/5303644)”。优点是讲述最根本、我最关心的原理，缺点是行文随意，不便阅读。  
3、内核documentation，讲述部分配置细节。

本文不讲配置使用kdump的详细过程，只是描述原理性的内容，使用细节使用时可查。主要以第2份资料和内部资料（不便公开）为据，并以实验为验证。

kdump的大致流程：  
1、经过配置，正在运行的系统额外加载一个内核，称之为capture kernel或crash kernel，而对应的正在运行的内核称为primary kernel。这通过在menu.lst中增加crashkernl=x@y，意为将crash kernel加载在y开始的x Mb内存中。  
2、当系统panic时，kdump调用kexec快速启动crash kernel，将panic内核的内存镜像保存在/proc/vmcore下。其中crash kernel是一个单独的内核，被/etc/sysconfig/kdump中的KDUMP_KERNELVER配置。

疑问：  
1、能不能由panic kernel自己做转储？  
A：primary kernel已经panic，其内部状态和硬件状态是不确定的，此时转储的映像是不可靠的。  
2、为什么crash kernel能做转储？  
A：panic时，系统切换到crash kernel，它能访问整个内存空间。  
3、为crash kernel预留多少内存？  
A：一般是64MB/128MB，内核本身不占多少空间（2M左右），但运行时的内核需占据较多内存。而且，根据第1份资料，x@y的配置不一定总成功，原因不明。

###crash

这个工具用来分析内核转储文件vmcore（`crash /path/to/vmlinux vmcore`），也可用gdb，不过[crash](http://people.redhat.com/anderson/crash_whitepaper/)更强大。它能看到panic时，  
1、当时的进程（ps）  
2、堆栈信息（bt，bt+pid）  
3、崩溃时的日志（dmesg）  
4、进程打开的文件（files，files+pid）  
5、挂载的FS（mount）  
6、网络（net）

`man crash`可以看到更多命令。不过我好奇的是，crash是如何分析vmcore的，做一些猜想：  
1、二者必然遵循格式上的约定，crash从特定位置读取特定信息，这是必然的。  
2、另一个关键问题便是建立地址和symbol的连系，一个配置了CONFIG_DEBUG_INFO的vmlinux是必需的。相信我的博文“[Linux kernel symbols](http://xanpeng.github.com/linux/2012/05/29/linux-kernel-symbols.html)”能解释如何建立连系，只是一个查表（“symbol-地址”对应表）的过程。但实际代码如何，尚不得知，不再考究，即便心有惴惴。
