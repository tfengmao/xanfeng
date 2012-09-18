---
title: Linux kernel symbols
layout: post
tags: linux kernel symbol module
category: linux
---

本文由两个问题引起:  
1. 变更登入的 kernel 之后, modprobe 某些模块时, 出现"disagrees about version of symbol module_layout"错误.  
2. 在原有模块代码中, 新增全局变量之后, modprobe 时提示错误"Unknown symbol...".  

*在 modprobe 失败后, 可以用 `dmesg` 或 `vim /var/log/messages` 查看失败信息.*

设模块名为 mymod, 初步猜测:  
1. mymod 使用到内核函数, 如 printk, 但是 mymod 找不到对应的代码.  
2. mymod 在编译的时候, 引入某种和内核相关的 version 信息, 安装时该信息和当前内核不匹配, 于是"disagrees about version..."  
3. 依照"[.so, elf & symbols](http://xanpeng.github.com/2012/05/21/solib-elf-symbol/)"的经验, 内核在某处放置有所有 symbols. "Unknown symbol" 就是指在"某处"找不到该 symbol.  

---

#kernel symbols

编译并安装新内核之后, 在/boot目录下面会有文件 "System.map-版本号", 这里面放置了全局的 symbols, 包含函数和变量. 通过 `file` 命令可以看出这是一个 ASCII text 文件.  
以 printk() 和 jiffies 为例, 查看对应的 entry:

    # cat System.map-2.6.32.x | grep -w printk
    ffffffff80377195 T printk
    
    // 实际上 printk 相关的 entry 有很多
    # cat System.map-2.6.32.x | grep printk
    ffffffff80377195 T printk
    ffffffff804cd2d0 r __ksymtab_printk_timed_ratelimit
    ffffffff804cd2e0 r __ksymtab_printk_ratelimit
    ffffffff804cd380 r __ksymtab_vprintk
    ffffffff804cd390 r __ksymtab_printk
    
    # cat System.map-2.6.32.x | grep -w jiffies 
    ffffffff8063b900 A jiffies

但是, 在其中却没有发现变量 errno, 只看到 journal_errno, 这是为什么?

    # cat System.map-2.6.32.x | grep errno
    ffffffff80171fb0 T journal_errno
    ffffffff804d1250 r __ksymtab_journal_errno
    ffffffff804df520 r __kcrctab_journal_errno
    ffffffff804ec15f r __kstrtab_journal_errno

称 System.map 为 kernel symbol table(kst). kst 包含 kernel image 所有的 symbols, 在 kernel space 中是全局可见的.  
所以, 我们自己编写的测试模块 mymod 也可以使用其中的函数, 如 printk.  

---

#EXPORT_SYMBOL & EXPORT_SYMBOL_GPL

在 modprobe mymod 的时候, mymod 中定义的全局变量是否会被加入到 kst 中?  
答案是不会. 模块中定义的变量的能见度局限于模块内部, 除非作者显式地 export.

export 采用两个宏帮忙: EXPORT_SYMBOL 和 EXPORT_SYMBOL_GPL. 二者的区别是, 使用_GPL导出的 symbols 仅能被 GPL licensed 的程序使用. 汗...

通过代码来验证上述说法.

    // 代码摘引自 http://onebitbug.me/introducing-linux-kernel-symbols
    root@lab# cat mymod.c
    #include <linux/module.h>
    #include <linux/init.h>
    #include <linux/kernel.h>
    #include <linux/jiffies.h>
    
    MODULE_AUTHOR("Stephen Zhang");
    MODULE_LICENSE("GPL");
    MODULE_DESCRIPTION("Use exported symbols");
    
    void my_func(void);
    EXPORT_SYMBOL_GPL(my_func);    

    int test_val=300;
    void my_func()
    {
        printk("This is my_func()\n");
        printk("test_val = %d",test_val);
        printk("End of my_func()\n");
    }

    static int __init lkm_init(void)
    {
        printk(KERN_INFO "[%s] module loaded.\n", __this_module.name);
        printk("[%s] current jiffies: %lu.\n", __this_module.name, jiffies);
        return 0;
    }
    
    static void __exit lkm_exit(void)
    {
        printk(KERN_INFO "[%s] module unloaded.\n", __this_module.name);
    }
    
    module_init(lkm_init);
    module_exit(lkm_exit);
    
    root@lab# cat Makefile
    obj-m += mymod.o
    KDIR=/home/xan/linux/linux-2.6.32.36-0.5 
    all:
    	$(MAKE) -C $(KDIR) SUBDIRS=$(PWD) modules
    clean:
    	rm -rf *.o *.ko *.mod.* .c* .t*

    root@lab# ls
    .mymod.ko.cmd     .mymod.o.cmd   Makefile      Module.symvers  mymod.c   mymod.mod.c  mymod.o
    .mymod.mod.o.cmd  .tmp_versions  Makefile.xen  modules.order   mymod.ko  mymod.mod.o

    root@lab# insmod ./mymod.ko
    root@lab# cat /proc/kallsyms | grep my_func
    ffffffffa05de100 r __ksymtab_my_func	[mymod]
    ffffffffa05de118 r __kstrtab_my_func	[mymod]
    ffffffffa05de110 r __kcrctab_my_func	[mymod]
    ffffffffa05de000 t my_func	[mymod]

insmod mymod 之后, 想从 System.map 中看到 my_func 是不可能的. 因为 System.map 是编译安装内核时产生/更新的.  
所以, 我们要看的是内存中的 kst. 在不同的系统下, 内存中的 kst 位于不同的位置, 如 /proc/kallsyms 或 /proc/ksyms.  
注释掉示例代码中的 EXPORT_SYMBOL_GPL, /proc/kallsyms 中不再有"完整的" my_func 信息了, 只有下面的内容:
 
    root@lab# cat /proc/kallsyms | grep my_func
    ffffffffa05e4000 t my_func	[mymod]
    root@lab# 

---

#version disagreement

回顾文首提出的两个问题, "disagrees about version of symbol module_layout" 和 "Unknown symbol...", 到这里我们发现, 我们已经可以解释"Unknown symbol", 还不能解释"disagrees about version".

"Unknown symbol"的原因很简单, 就是 Module-A 定义了 symbol foo, Module-B 去使用 symbol foo. 显然默认情况下, foo是局限于 Module-A 的, 此时只要 export foo 即可.

现在重点关注 version disagreement 问题.  

实际上, 这个问题的浅层原因是简单的.  
在我们的文件系统里面, 我们一般只维护一份内核源码. 我们一般修改模块代码, 然后重编这个模块, 再 modprobe 这个模块, 以验证改动是否正确.  
但是, 我们一般不只一个 kernel image 可供登入, 登录时 grub 的内核选取界面经常有多个 entry. 如果我们进入了 kernel-B, 我们维护的内核源码仍在那里, 但是在 kernel-A 下被编译的. 此时如果我们修改并编译模块, 再 modprobe 的时候就会出现 version disagreement 的错误.

在上面的场景下, 我们可以通过登入 kernel-A 去绕开问题. 但是设想这个场景: 你的工作是为某公司做 module dev, 你的开发环境与产品环境必然是不同的, 比如你使用的内核是2.6.32的, 而产品环境是2.6.12的, 这时怎么办呢?  
这是你就很难绕开这个问题了, 保证开发环境和产品环境是完全一致的 ---- 我觉得这是一个愚蠢的想法.  

更有甚者, 如果要开发一个模块, 能够适合多个发行版本, 点解?

搜索了很久, 未能解决这个问题. 难道只有保证所使用的 kernel 版本一致才行? 甚至保证所使用的编译器也要一致?

##versioning

"[Loadable Modules & the Linux 2.6 Kernel](http://www.drdobbs.com/open-source/184406112?pgno=1)" 的 "Module Versioning" 部分讲到:

> The 2.6 module loader implements strict version checking, relying on "version magic" strings ("vermagics"), which are included both in the kernel and in each module at build time. A vermagic, which could look like "2.6.5-1.358 686 REGPARM 4KSTACKS gcc-3.3," contains critical information (for example, an extended kernel version identifier, the target architecture, compilation options, and compiler version) and guarantees compatibility between the kernel and a module. The module loader compares the module's and kernel's vermagics character-for-character, and refuses to load the module if differences are detected.

意思是, 编译模块时, 编译程序会将使用的 kernel 版本, 所使用的编译器等信息写到模块里面, 在装载模块时, 会 character-by-character 地将这个信息和当前 kernel 比较. 如果不匹配, 就拒绝.  
`man modprobe`看到, modprobe 提供了 --force-vermagic, --force-modversion, 以及包含二者的 --force(-f) 选项, 使用这些选项, 可以绕过 version 检查. 但存在风险: "*Naturally, these checks are there for your protection, so using this option is dangerous*", **我在 `modprobe -f` 时就发生了 kernel crash**(串口日志表明, 是 module_trace_bprintk_format_notify 处的 null pointer 异常).

但仍解决不了疑问, 还带来了新问题: 什么时候可以安全地使用 modprobe -f?

##what does the book say

实际上, 关键在于你的模块使用的函数必须要能在当前的 kernel image 里面找到.  
"[Linux Loadable Kernel Module HOWTO](http://tldp.org/HOWTO/Module-HOWTO/index.html)"的"[6.2. An LKM Must Match The Base Kernel](http://tldp.org/HOWTO/Module-HOWTO/basekerncompat.html#AEN507)" 给出了设计 vermagic/.modinfo 的初衷, 就是为了应对场景: kernel 版本变迁, 引起某些 API 的变迁, 从而使旧模块在新的内核中行为出错.  
文章还提出, 有时, 开发者知道模块使用的 API 基本不会变动, 此时就没有必要为每一个内核去编译对应的模块了. 所以, insmod 提供了 "-f" 参数.  
文章还提到了 "CONFIG_MODVERSIONS", 但是我不愿去细究了.

##version disagreement 总结

1. 发生 version disagreement 的原因, 一般是在 kernel-A 编译, 去 kernel-B 安装. 实际上, 使用 `modprobe` 时, 它会选择正确的目录, 所以发生错误的话, 往往是因为我们手工替换了对应目录下的 .ko 文件.  
2. 如果你确保尽管 kernel-A, kernel-B 不同, 但也大同小异, 至少模块使用的 API 无变化, 那么可以使用 "--force" 选项, 主动取消 version 检查.  
3. 如果不能确保 API 一致, 还要使用 "--force", 那么就要承担 kernel crash 的风险. 实际上, 我遇到的 kernel crash 的情况就是因为在当前的 System.map 中根本找不到某 API.  
4. insmod 获得的信息优于 modprobe. 不解释+不深究!  

##try to hack!

假设 mymod.ko 在 kernel-A 下被编译, 当前处于 kernel-B, 那么是否可以:  
1. insmod -f ./mymod.ko 使安装成功?  
2. modprobe -f mymod 使安装成功?  
3. cp vmlinuz-kernelA vmlinuz-kernelB, cp System.map-kernelA System.map-kernelB, "蛮横" 地使 modprobe mymod 成功?

非常遗憾, 这3个尝试都失败了!  
\#3失败可以理解, 因为装载模块时, 它很可能不是从 kernel image 中去读取函数的.  
\#1和#2失败就让人费解了.  

我接着做了一个小测试, 我仅仅修改 .config 中的 kernel version 子串, 不做其他任何调整, 然后编译安装得到新内核 kernel-NEW. 进入kernel-NEW, 我 insmod ./mymod.ko, 直接成功了, 没有报错.

sigh...肯定有更多的细节为我不知...而这个细节并没有作为较重要的主题呈现在书籍中...我不去看了, 已花大量时间...挂念于心中及此处吧...

---

#reference

[Introducing Linux Kernel Symbols](http://onebitbug.me/introducing-linux-kernel-symbols)  
[Kernel Symbol Table and exporting symbol](http://www.learninglinuxkernel.com/Linux_Kernel_Module_Programming_03.html)  
