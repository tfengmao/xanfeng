---
title: 了解 Linux 内核
layout: post
tags: linux kernel config compile 
category: linux
---

看过 Greg Kroah-Hartman 的 <[Linux Kernel In a Nutshell](http://www.kroah.com/lkn/)> 之后, 发现网络资料的严重弱势, 大牛就是大牛, 将主要细节娓娓道来, 就像讲一个故事. 现在看网络资料, 相比之下它们就是小儿科, 将普普通通的每一个步骤都如临大敌地看待, 让初学者误解, 畏惧, 看不透. 所以:  
**要以权威的书籍等资料作为参考**.

@p4 - 源码放置位置  
The kernel source code should also **nerver** be placed in the */usr/src/linux* directory, as that is the location of the kernel that the system libraries were built against, not your custom kernel. Do **not** do any kernel development under the */usr/src/* directory tree at all, but only in a local usr directory where nothing bad can happen to the system.

@p5 - 编译工具  
Tools to build the kernel  
+ compiler: `gcc --version`  
+ linker: `ld -v`  
+ make: `make --version`  

@p6 - 模块工具  
Tools to use the kernel  
+ util-linux: `fdformat --version`  
+ module-init-tools: `depmod -V`  

@p8 - FS 工具  
filesystem-specific tools  
+ ext2/ext3/ext4: `tune2fs`  
+ Quotas: `quota -V`  

@p10 - 其他工具  
other tools  
+ udev: `udevinfo -V`  
+ process tools: `ps --version`  

@p17 - 配置  
If you have just expanded the kernel source code, there will be **no.config** file, so it needs to be created. It can be created **from scratch**, created **by basing it on the “default configuration”**, taken **from a running kernel version**, or taken **from a distribution kernel release**.   
+ Configure from scratch: `make config`, 2000+ 选项  
+ Default Configuration Options: `make defconfig`, 每个内核版本都会有一个 "default" kernel configuration.

有了 .config 之后, 就可以 `make menuconfig` 做一些修改.

@p26 - `make -j`  
It is best to give a number to the -j option that corresponds to **twice** the number of processors in the system. `make -j` will create a new thread for every subdirectory in the kernel tree, which can easily cause your machine to become unresponsive and take a much longer time to complete the build.   

Build a portion of the kernel: `make drivers/usb/serial`.

@p30 - 安装  
Almost all distributions come with a script called *installkernel* that can be used by the kernel build system to automatically install a built kernel into the proper location and modify the bootloader so that nothing extra needs to be done by the developer.  

`make modules_install` will install all the modules that you have built and place them in the proper location in the filesystemfor the new kernel to properly find.   

`make install` install the main kernel image, will kick off the following process:  
1. The kernel build systemwill verify that the kernel has been successfully built properly.  
2. The build systemwill install the static kernel portion into the/boot directory and name this executable file based on the kernel version of the built kernel.  
3. Any needed initial ramdisk images will be automatically created, using the modules that have just been installed during the *modules_install* phase.  
4. The bootloader programwill be properly notified that a new kernel is present,and it will be added to the appropriate menu so the user can select it the next time the machine is booted.  
5. After this is finished, the kernel is successfully installed, and you can safely reboot and try out your new kernel image. Note that this installation does **not overwrite **any older kernel images.

@p31 - 安装 Installing by hand.

@p35 - upgrading a kernel

@p45 - /proc/config.gz  
Almost all distributions provide the kernel configuration files as part of the distribution kernel package. Read the distribution-specific documentation for how to find these configurations.  
Most distribu-tion kernels are built to include the configuration within the /proc filesystem.

@p46 - **Find which module is needed**.

@p63-p84 - kernel configuration recipes  
介绍了 config 中每个 option 的含义.

@p87 - Ch9 Kernel boot command-line parameter reference.

@p117 - Ch10 Kernel build command-line reference.

@p122 - Ch11 Kernel configuration option reference.  

---

#编译内核

编译安装Linux内核是每个内核开发者并经的一步了，他们起步的时候都会做这个动作吧，网上也有很多很好的资料，如：  
- [How to compile Linux kernel - Tutorial](http://www.dedoimedo.com/computers/linux-kernel-compilation.html)  
- [What is the Linux Kernel and What Does It Do?](http://www.howtogeek.com/howto/31632/what-is-the-linux-kernel-and-what-does-it-do/)  
这篇文章介绍了什么是内核，mocrokernel和Monolithic内核的区别；介绍了Linux Kernel重要的几个文件：vmlinuz，config，initrd，System.map。  
- [How to compile the Linux kernel](http://tuxradar.com/content/how-compile-linux-kernel)  
这篇文章的特色是，它会告诉你你可以对其他什么文章有兴趣，于是你可以了解更多信息.

王聪大牛的"[深入理解Linux内核构建系统（一）](http://wangcong.org/blog/archives/240)"把Makefile中的target简单直白地描述了.其中我比较感兴趣的有`make tags`和`make cscope`,不由的让人想[ctags](http://ctags.sourceforge.net/)和[cscope](http://cscope.sourceforge.net/)和这两个target有什么关系.

1、先准备源代码。

2、make menuconfig    #配置what，who and when will be included in the kernel image。  
配置文件名为“.config”，一般地下载的全新源码中不会包含配置文件。可以使用自己的配置文件，ubuntu/redhat下位于/boot，suse位于/proc/config.gz.  

其他可选操作：  
- make oldconfig	  
- make mrproper	#clear all pre-compiled binaries and remove .config file.  
- make clean    #just clean all pre-compiled binaries. 

3、复杂、严肃的配置过程

4、make all 或者 make && make modules_install && make install

5、修改 /boot/grub/menu.lst

---

#编译模块

内核source tree中的documentation/kbuild/modules.txt说明了如何编译安装内核模块。本文仅摘要关键。

===1、介绍  
kbuild可以拿来编译within-tree(内部模块代码)和out-of-tree(external，外部模块代码)的内核模块代码。内部模块一般是在内核编译安装的过程中，被`make modules`编译，编译外部模块时，需要的**前提条件**是：提供有预先编译好的完整内核以及源代码(关键是其中的各种.o文件？)。

在修改、调试已有内部模块代码时，也可以当成外部模块来处理。

===2、如何编译**外部模块**  
2.1、编译外部模块  
编译(build)

    # make -C <path-to-kernel> M=\`pwd\`
    
    For the running kernel use:(意思是“编译的模块是打算提供给正在运行的内核”？)
    # make -C /lib/modules/\`uname -r\`/build M=\`pwd\`
    For the above command to succeed, the kernel must have been built with modules enabled.

安装(install)

    # make -C <path-to-kernel> M=\`pwd\` modules_install

2.2、target介绍(设$KDIR是内核源码的顶层目录)：  
- make -C $KDIR M=\`pwd\`  
编译模块代码，代码路径是当前目录，编译的所有输出文件也都放在当前目录。  
不会去修改内核源码。  
- make -C $KDIR M=\`pwd\` modules  
同上，编译模块代码。 
- make -C $KDIR M=\`pwd\` modules_install  
安装外部模块，安装路径默认为/lib/modules/<kernel-version>/**extra**，可以通过INSTALL_MOD_PATH修改。  
- make -C $KDIR M=\`pwd\` clean  
删除编译模块时生成的所有文件，不会修改kernel代码。  
- make -C $KDIR M=\`pwd\` help  

2.3、可用选项

2.4、**prepare kernel tree** for module build   
modules_prepare用来提供外部模块编译所需要的信息。不过modules_prepare不会编译Module.symvers，即使设置了CONFIG_MODVERSIONS，因此需要完整地编译一次内核，以支持module versioning。

2.5、编译单个文件  
示例（module foo.ko，包含bar.o, baz.o）
  
    make -C $KDIR M=\`pwd\` bar.lst 
    make -C $KDIR M=\`pwd\` bar.o
	make -C $KDIR M=\`pwd\` foo.ko
	make -C $KDIR M=\`pwd\` /

===3、example commands

===4、creating a kbuild file for an external module

===5、include files

===6、module installation

===7、Module versioning & Module.symvers  
Module versioning is enabled by the CONFIG_MODVERSIONS tag，用来做简单的ABI consistency check。

Module.symvers contains a list of all exported symbols from a kernel build.  
7.1、Symbols from the kernel (vmlinux + modules)  
7.2、Symbols and external modules  
7.3、Symbols from another external module

更多资料：[An LKM Must Match The Base Kernel](http://tldp.org/HOWTO/Module-HOWTO/basekerncompat.html)

===8、tips & tricks

---

#Q&A#

问题：根据某个kernel源码包编译安装（make && make modules）模块时，产生错误，dmesg查看得到错误信息“xxxx：no symbol version for module_layout”（xxxx是某模块名）。  

答案：

---



[ss]: http://www.slideshare.net/geekaholic/kernel-configuration "slideshare: configure the kernel"
 