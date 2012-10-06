---
title: linux内存管理的一些问题
layout: post
tags: linux memory management questions cache highmem zone physical virtual
category: linux
---

*我在"[linux memory management](http://xanpeng.github.com/2012/05/31/linux-memory-management/)"中对物理内存管理做了 bird-view 级别的描述, 可以帮助理解本文的问题.*

在学习linux内存管理的时候,阅读书籍,网络资料和源代码,短期内仍难免彷徨.这里提出一些问题和尝试的解答,以期有助于理解.

Q0:内核虚拟内存(>0xC0000000)和用户虚拟内存是否会mapping到物理内存的不同部分?

A0:<del>不是</del>. 这个问题问的没什么逻辑(-_-)...应该是物理内存mapping到虚拟地址空间去.  
1G的内核虚拟内存是不是只mapping到物理内存0-1G的空间? <del>实际情况**可能**如此</del>, 不是的, 有一部分用来映射高端内存.  
这块物理空间属于ZONE_NORMAL,这个zone的物理地址和虚拟地址之间是**线性映射(linearly mapped)**的<sup>[1][],[2][]</sup>.如果物理内存>896M,那么存在ZONE_HIGHMEM,这个zone的物理-虚拟地址映射方式不是线性的.  
分两种情况:  
1)物理内存<896M,此时用户虚拟内存和内核虚拟内存不得不映射到同一个区域(ZONE_NORMAL)了.  
2)物理内存>896M,此时内核虚拟内存是映射到物理内存的ZONE_NORMAL区域的,用户内存则ZONE_NORMAL和ZONE_HIGHMEM都会使用到<sup>[1][]</sup>(设想物理内存仅为896+4M大小怎么办).

更多资料:[Why Linux Kernel ZONE_NORMAL is limited to 896 MB?](http://stackoverflow.com/questions/8252785/why-linux-kernel-zone-normal-is-limited-to-896-mb)  

[1]: http://stackoverflow.com/a/5845375/264035 "ZONE_NORMAL association with kernel/user-pages"
[2]: http://stackoverflow.com/a/6148462/264035 "zone_NORMAL and ZONE_HIGHMEM on 32 and 64 bit kernels"

Q1:我们知道虚拟内存和物理内存,物理内存被分成了不同的**zone**,其中>896M的是ZONE_HIGHMEM,称之为**高端内存**.我们一般考虑32bit机器**4G**的虚拟地址空间.那么,如果物理内存只有**512M**,虚拟地址空间和物理内存之间是如何映射的?

A1:linux需要至少4M的物理内存支持<sup>[tldp](http://tldp.org/FAQ/Linux-FAQ/linux-distributions.html#how-much-memory-does-linux-need)</sup>,这种情况下估计需要大量的swap操作,而且说明可工作最小的linux kernel大小是小于4M的.  
在机器拥有512M内存时,我想首先内核镜像(代码和数据)是长期处于内存的,这一部分内存是固定的,不涉及虚拟地址和物理地址的转换.  
每一个进程都有自己的虚拟地址空间,32bit机器能表达的地址空间大小是4G,大家熟悉的**3:1**划分的意思是,用户进程能够使用的虚拟地址空间是3G大小(0x0-0xBFFFFFFF),内核进程能使用的虚拟地址空间是剩下的1G,或者说用户虚拟内存是0x0-0xBFFFFFFF,内核虚拟内存是0xC0000000-0xFFFFFFFF.  
但并不是每一个单元的虚拟内存都要对应(mapping)一个单元的物理内存,所以32bit的机器中物理内存不需要是4G.  
在只有512M内存的时候,内核虚拟内存的映射还是老样子,线性映射于ZONE_NORMAL区.用户虚拟内存也必须使用ZONE_NORMAL的内存了,它的映射方式仍然是通过使用page table.这说明:*同一个物理内存页面可以同时被两种不同的映射方式映射,并和两个不同的虚拟页面关联*.

更多资料:[How does the linux kernel manage less than 1GB physical memory?](http://stackoverflow.com/questions/4528568/how-does-the-linux-kernel-manage-less-than-1gb-physical-memory)

Q2:内核如何使用内存?

A2:内核镜像(代码和数据)是占据着固定的物理内存的,这块略过.除此之外运行于内核空间的程序也有动态静态申请内存的需要,我们都熟知用户态程序申请内存使用的malloc函数,对于内核程序,这个函数显然不再适用. <del>内核要求运行速度和可靠,linux内核内存管理使用了**slab**分配机制,它对应的API有kmem_cache_create, kmalloc等.</del> 使用 slab 的场景是小内存分配, 对于本文题, 答案应是内核使用伙伴系统算法分配内存.

更多资料:[Linux slab 分配器剖析](https://www.ibm.com/developerworks/cn/linux/l-linux-slab-allocator/)

Q3:用户态程序如何使用内存?如果说内核内存空间仅能映射到1G物理内存空间,那么用户申请的内存需要从>1G的物理内存分配时,内核岂不是做不到"代劳"?

A3:用户态程序申请内存的方式为我们熟知,就是malloc.当malloc的内存需要从>1G的物理内存获取时,仍然是由内核代劳的,**估计**内核去判断哪些物理内存页面是free的,然后抓取并设置malloc相关的记录,使之包含抓取的free物理页面的地址.这个过程中,内核代码仍然是执行于内核地址空间的,它*仍然访问不到>1G的空间,也仍然不需要访问>1G的空间,因为物理内存的状态都是在<1G的内核空间里面维护的*. --**居然被我猜对了! 我的猜测是觉得这样比较合理, 看来其实很多"深奥"的知识, 是和日常生活的知识同理的.**
