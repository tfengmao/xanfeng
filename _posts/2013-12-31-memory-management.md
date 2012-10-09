---
title: 【置顶】内存管理相关文章
layout: post
category: linux
tags: memory kernel
---

[Linux内存管理](http://xanpeng.github.com/linux/2012/05/31/linux-memory-management.html)  
[Linux cached memory](http://xanpeng.github.com/linux/2012/05/04/linux-cached-memory.html)  
[linux内存管理的一些问题](http://xanpeng.github.com/linux/2012/03/10/mm-questions.html)  
[由malloc引发的思考](http://xanpeng.github.com/linux/2012/03/10/deep-into-malloc.html)  
[物理内存管理](http://xanpeng.github.com/linux/2012/03/09/physical-mm.html)  
[虚拟内存，地址空间，page cache](http://xanpeng.github.com/linux/2012/03/01/buffer-cache.html)  

---

看过Mel Gorman的书之后，近期的思考：  
Linux内存管理包含很多内容，很容易让人迷失，不过注意到“巨起于细”和“事分先后”之后，理解便会容易很多。
  
系统启动时，**BIOS**会检测内存，并探测到内存的关键数据。  

系统需要一种方式来描述内存，内核使用了很多数据结构，建立了很多概念，用来表达内存，如Nodes、Zones、Pages。Linux运用这些描述工具，实际上可以表达复杂的内存物理架构，不过我们平时接触的都是简单的结构——比如，就一个内存条。  

Linux使用分页的方式来使用内存，用到**页表**，以32位地址为例，地址这样被解释：10+10+12，最后12位表示页内偏移，所以每页大小是4KB；前面两个10位表示两级，级越多，表示范围就越广，后面的10+12可以表示1024×4KB=4MB大小的内存范围，所以第一个10位结构的每一项都能指向4MB的空间。如果是10+5+5+12，则每项可指向512*512*4KB=256MB的内存空间。  
不过现有的内核代码是否适合这么“扩级”，还不知道。  
常见的是3级页表结构，每一级在内核中都有对应的名称，我在点点上略微记录了下。  

内核空间和用户空间都要以页表的方式来使用内存。  
所以系统启动时，要正式使用内存，首先要建立页表。  
这分两步：第一步当然是**bootstrap**，建立一个临时的页表，只有两个表项，覆盖1～9MB这8MB的内存，足以应付要加载的vmlinuz(2+MB大小)，这里0～1MB是有特殊用途的。  
第二步是建立真正的页表。然后就支持正常的操作了。  

到底有几个页表？用户态和内核态的页表是否相同？页表存在哪里？  
这些问题其实都不重要，重要的就是Linux并不是直接对物理内存寻址的，而是利用页表机制(分页机制)。  
那么有多少种不同的映射方式，就有多少个页表了。  

在32位系统中，我们常说的“用户空间：内核空间=3：1”，需要注意的是，这说的是虚拟地址，不是说实际物理内存。  
内核肯定是掌控整个物理内存的，不管你是512M、896M、2G、4G还是16G。  
在32位地址空间里面，内核占用3～4G区域，也就是0xC0000000以上的地址都属于内核地址空间。  
**内核地址空间**的布局，搜索“Kernel Address Space”可以看到示意图，在Gorman的书中是Fig 4.1。  

内核地址空间，地址映射的方式——页表的映射机制是简单的，虚拟地址-PAGE_OFFSET(0xC0000000)=物理地址。  
这对应有一份(一个，或一套...)页表。  
这么说来，如果物理内存<896MB，内核空间就直接管了。如果>1G，内核不能直接管了，因为管不到，内核地址空间就只能表示896MB的范围(后面128MB有特殊用途)，就涉及**高端内存**管理了。  

而对于每一个**进程**，它们都有自己的页表，其映射方式当然没有内核那么简单了。  
具体地，每个进程的页表完全是不同的吗？是否有复用的地方？是否只有用到某个地址之后，才在页表中增加一个条目，还是实现完全分配好？  
就不知道了。  

内存分配和释放等工作实际上是内核完成的，但是又说内核限于内核地址空间，那么对于一个0～3G范围的地址，内核如何处理？因为它是不能直接对应到物理地址的。  
实际上，内核地址空间的896M～1G这个范围上，有两个重要的区域：**kmap** address space、**Fixed** Virtual address space。  
kmap区域用来做临时映射，用来管高端内存，要申请1G之外的物理内存，内核要访问它，要整一个内核空间的地址，因此在这个kmap区域做一个临时的映射。  
这里会有一个页表。  

而Fixed Virtual address space区域，是一个预留区，用来做非连续内存分配的。  
非连续内存分配指虚拟连续，而物理不连续的分配。  
对应的接口是**vmalloc()**。  

我对896MB～1G中的这两个区域，还只是模糊的理解，大概是这个样子的。  
kmap区域很小的样子。  

从上面可以看出，要多少页表了。  

---

上面是内存管理的一个方面，是基础。  
在这些概念构建的世界中，程序如何分配物理内存？  
这就涉及两个部分：**buddy** allocator、**slab** allocator。  
buddy系统分配大块内存，最小是以页（4KB）为单位分配的。  
slab系统分配小内存，而且slab系统的分配是对象式的，用slab接口构建一个cache，专门放这种结构，构建另一个cache，专门放另外一种结构。  
slab的目的就是快，主要方式就是cache起来。  
这一块又涉及硬件cache：L1/L2 cache之类的，常说的**slab着色**的目的是让不同的object cache用不同的硬件cache line，以获得更好的性能，不要随随便便就被从硬件cache中刷出之类的。  

这两套概念都有自己的API。  
slab是向谁申请物理页的？向buddy allocator！所以，buddy是基本，slab是buddy之上的机制，用来消除内部碎片的。  

另外页框回收，swap，shared memory virtual filesystem等，就是另外一方面的话题了，暂时不说。

---

内核编程时，经常用到的申请内存的接口，就是slab系统的接口了。  
用户态编程时，使用的glibc库，用到的内存接口是**ptmalloc2**的，它在buddy系统之上构建了自己的一层，实现特别的特性，当然最终是通过buddy系统的接口去申请内存的了。  

---

一些资料：  
http://www.csn.ul.ie/~mel, <Understanding the Linux Virtual Memory Manager>, Mel Gorman  
http://linux-mm.org/HighMemory  
http://linux-mm.org/VirtualMemory  
http://stackoverflow.com/questions/5272408/does-linux-use-self-map-for-page-directory-and-page-tables  
http://www.stanford.edu/~stinson/paper_notes/fundamental/mem_mgmt/ia32_pts.txt  
http://stackoverflow.com/questions/10880555/how-does-the-system-choose-the-right-page-table  
http://www.thehackademy.net/madchat/ebooks/Mem_virtuelle/linux-mm/vmoutline.html  

有新的认识之后再更新本文。