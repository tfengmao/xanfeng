---
title: linux 虚拟内存, 地址空间布局, page cache, ...
layout: post
tags: linux kernel memory cache address page buffer process
category: linux
---

*本文的内容是零散的, 有待进一步完善. 我在"[linux memory management](http://xanpeng.github.com/2012/05/31/linux-memory-management/)"中对物理内存管理做了 bird-view 级别的描述, 可以与本文相互映照.*

我们平时编写程序, 执行程序的时候, 总是会接触到"内存(memory)", "buffer", "cache" 这样的概念, 多少会在英文术语和中文翻译中迷失. 本文尝试建立对这些概念和其原理的初步认识.

#问题
考虑 Linux 操作系统, 假设我们执行程序 /bin/more, 它读取某个大文件, 并"一页一页地"输出到控制台(stdout). 对此, 我有这些疑问:

* more 程序是否有自己的*应用程序缓存(application buffer, user buffer)*, 
* more 将文件从硬盘读入到内存的什么地方? 
* **page cache** 是什么, 在这个过程中有什么用? 
* 内存管理都是内核来做的? 
* more 的输出是直接从某内核内存区域输出到 stdout 的吗?
* more 程序中是否会数据从内核 copy 到 userland? 有 **system buffer** 吗?

#Gustavo Duarte 的博文
要解决上面的问题, 不是一件容易的事情, 阅读书籍+源码一定可以做到, 然而定需要花费大量时间. 幸好在 google "page cache" 的时候, 我找到了 **[Gustavo Duarte 的博客](http://duartes.org/gustavo/blog/)**, 他的每一篇文章都是图文并茂, 真正做到了"深入浅出"的讲解, 强烈推荐阅读!  
和本文关注点"内存"相关的一系列文章如下, 本文"严重"参考了这个系列:

1. [Getting Physical With Memory](http://duartes.org/gustavo/blog/post/getting-physical-with-memory)
2. [Anatomy of a Program in Memory](http://duartes.org/gustavo/blog/post/anatomy-of-a-program-in-memory)
3. [How The Kernel Manages Your Memory](http://duartes.org/gustavo/blog/post/how-the-kernel-manages-your-memory)
4. [Page Cache, the Affair Between Memory and Files](http://duartes.org/gustavo/blog/post/page-cache-the-affair-between-memory-and-files)

  
#Anatomy of a Program in Memory
OS不让你直面物理内存, 它提供了 virtual memory 层. **virtual address** 和物理内存通过 **page table**(由 kernel 管理) 相联系. 每一个进程都有自己独立的 **virtual memory space**, 在32位系统中, 大小为 4G. 在 linux 和 windows 系统中, 内核进程和用户进程所占的 virtual memory 比例分别是 1:3 和 2:2. (windows 经过配置也可以为 1:3.)

![alt user/kernel memory split](http://static.duartes.org/img/blogPosts/kernelUserMemorySplit.png)

上图并不是说 kernel 要使用 1G 的物理内存, 而只是指 kernel 只有 1G 这一部分的 address space 可供映射物理内存.

Linux 中一个进程在虚拟内存中的经典布局为:

![alt segment layout in a Linux process](http://static.duartes.org/img/blogPosts/linuxFlexibleAddressSpaceLayout.png)

上图中 random stack offset 和 random mmap offset 等是随机值, 是为了防止恶意程序的, 因为如果 stack 的位置是固定的, 则恶意程序能够通过计算访问 stack, 库函数等.

略介绍该布局图各个部分的含义:

* stack: 堆栈区, 一般程序员们都知道. Linux 中 "ulimit -s" 可以查看和设置堆栈最大值(RLIMIT_STACK), 当程序使用堆栈超过 RLIMIT_STACK 时, 会报 **stack overflow** 错误.
* memory mapping segment: 在这里 kernel 将硬盘文件的内容直接映射到内存, 任何程序都可以通过 mmap() 系统调用做这样的映射工作. Memory mapping 用在 file IO 中非常方便和高效, 因而被用于装载动态库. 用户也可以创建 **匿名内存映射(anonymous memory mapping)**, 这样的映射没有对应的文件, 可以被用于映射程序数据. 在 Linux 中, 如果你通过 malloc() 请求一大块内存, C 库将创建一个匿名内存映射, 而不是使用 heap. 这里存在一个阈值 MMAP_THRESHOLD, 一般为 128 kb, 可以通过 [mallopt()](http://www.kernel.org/doc/man-pages/online/pages/man3/undocumented.3.html) 调整.
* heap: 提供 runtime 的内存分配, C 中可以通过 malloc() 家族申请 heap 内存, C++ 可以通过 new 关键字申请 heap 内存. heap 内存不足的时候, 可以通过 brk() 系统调用扩张.
* BSS segment: 用来存程序未初始化的 static 变量的. 这块内存是 anonymous 的.
* data segment: 存储初始化过的 static 变量和全局变量. 这块内存不是 anonymous 的, 它映射到程序二进制镜像的对应部分. 同时, 它是 private memory mapping 的, 意味着其变更不反应到硬盘文件.
* text segment: 只读区域, 映射程序的二进制镜像文件.

#How The Kernel Manages Your Memory
了解了进程的虚拟地址空间内存布局之后, 我们来看看操作系统如何来管理内存.

Linux 中进程是 task_struct 结构的实例, 这个结构包含 struct mm_struct *mm 指针, 它管理上文提及的内存布局.

![alt mm_struct in task_struct](http://static.duartes.org/img/blogPosts/mm_struct.png)

mm 结构中还包含链表指针 struct vm_area_struct *mmap, 这个指针对应的内存区域应该就是 memory mapping segment, 这个链表的每一项表示一个 **virtual memory area(VMA)**.

![alt mmap in mm_struct](http://static.duartes.org/img/blogPosts/memoryDescriptorAndMemoryAreas.png)

vm_area_struct 结构中包含指示读写权限的标志变量, 包含指示被映射文件路径的 vm_file. *mmap 链表由红黑树(mm_rb)管理, 以支持内核快速的查询.  
cat /proc/pid_of_process/maps 则仅简单遍历该链表.
    # vim test.cc
    C^z to put it to background.
    # ps aux | grep vim
    root     18969  0.3  0.0  18288  3484 pts/3    T    19:19   0:00 vim test.cc
    root     19114  0.0  0.0   4340   760 pts/3    S+   19:20   0:00 grep vim
    # cat /proc/18969/maps
    00400000-005b8000 r-xp 00000000 08:01 213434                             /bin/vim-normal            // 共 3 行
    007ca000-00915000 rw-p 00000000 00:00 0                                  [heap]
    7f533930a000-7f533930e000 r-xp 00000000 08:01 106723                     /lib64/libattr.so.1.1.0    // 共 4 行
    7f533950f000-7f5339511000 r-xp 00000000 08:01 106781                     /lib64/libdl-2.11.1.so     // 共 4 行
    7f5339713000-7f5339867000 r-xp 00000000 08:01 106798                     /lib64/libc-2.11.1.so      // 共 4 行
    7f5339a6c000-7f5339a71000 rw-p 00000000 00:00 0 
    7f5339a71000-7f5339a78000 r-xp 00000000 08:01 106716                     /lib64/libacl.so.1.1.0     // 共 4 行
    7f5339c79000-7f5339cb7000 r-xp 00000000 08:01 106775                     /lib64/libncurses.so.5.6   // 共 4 行
    7f5339ec1000-7f5339f16000 r-xp 00000000 08:01 106790                     /lib64/libm-2.11.1.so      // 共 4 行
    7f533a117000-7f533a136000 r-xp 00000000 08:01 106824                     /lib64/ld-2.11.1.so
    7f533a2a9000-7f533a2de000 r--s 00000000 08:08 385827                     /var/run/nscd/passwd
    7f533a2de000-7f533a31d000 r--p 00000000 08:09 5154103                    /usr/lib/locale/en_US.utf8/LC_CTYPE
    7f533a31d000-7f533a322000 rw-p 00000000 00:00 0 
    7f533a32d000-7f533a334000 r--s 00000000 08:09 32866478                   /usr/lib64/gconv/gconv-modules.cache
    7f533a334000-7f533a335000 rw-p 00000000 00:00 0 
    7f533a335000-7f533a336000 r--p 0001e000 08:01 106824                     /lib64/ld-2.11.1.so
    7f533a336000-7f533a337000 rw-p 0001f000 08:01 106824                     /lib64/ld-2.11.1.so
    7f533a337000-7f533a338000 rw-p 00000000 00:00 0 
    7fffba730000-7fffba745000 rw-p 00000000 00:00 0                          [stack]
    7fffba7ff000-7fffba800000 r-xp 00000000 00:00 0                          [vdso]
    ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]

##memory pages
现在迎来另一个概念 "memory page".  
4GB 的虚拟地址空间被划分为 pages, 32 位的 x86 处理器支持 4KB, 2MB 和 4MB 的页大小. 一般 Linux 和 Windows 对于用户部分的虚拟地址空间都使用 4KB 的页大小, 则 3GB 的用户空间对应的页为:  
![alt 3GB virtual user space](http://static.duartes.org/img/blogPosts/pagedVirtualSpace.png)

正如前文所说, 虚拟内存和物理内存之间通过 **page tables** 联系, 每个进程都有自己的 page tables, 在 mm_struct 中由 pgd_t *pgd 指针指示. page table 的每一项 page table entry(PTE) 为:  
![alt page table entry](http://static.duartes.org/img/blogPosts/x86PageTableEntry4KB.png)

物理地址空间被 kernel 划分为 **page frames**, 这个概念对于 kernel 来说非常重要, 因为**page frame is the unit of physical memory management**. 32 位的 Linux 和 Windows 都使用 4KB 的 page frame. 这里是一个 2GB RAM 的 page frame 划分:  
![alt page frames for 2GB physical memory](http://static.duartes.org/img/blogPosts/physicalAddressSpace.png)

Linux 中 page frame 由 struct page 描述, 其中包含很多标志变量, 用以跟踪整个物理内存的状态. 物理内存的管理由 **buddy memory allocatioin** 技术代劳. 一个 page frame 是否是 free 的, 要看它能否经由 buddy system 被分配出来.
一个被分配的 page frame 可以是匿名的, 用以保存程序数据; 也可以用在 **page cache** 中, 保存在文件或块设备中的数据.  
从虚拟内存到物理内存的映射, 可以由下面的简图展示:  
![virtual memory to physical memory](http://static.duartes.org/img/blogPosts/heapMapped.png)

#Page Cache, the Affair Between Memory and Files
OS 处理文件的时候面对两大难题: 1) 访问硬盘很慢; 2) the need to load file contents in physical memory once and share the contents among programs.  
page cache 机制用以解决这两个问题. 假设有一个程序 render 需要打开文件 scene.data, 并一次读取 512 bytes. kernel 处理流程图示:  
![alt render program to read file](http://static.duartes.org/img/blogPosts/readFromPageCache.png)

读取了 12KB 数据之后, render 的 heap 和对应的 page frames 图示:  
![alt heap and page frames of render](http://static.duartes.org/img/blogPosts/nonMappedFileRead.png)

可以看到, 上图 page frames 中有几个 page cache 是重复的. 这是因为: **all *regular* file IO happens through the page cache*, 在 x86 linux 中, kernel 将文件看成连续的 4KB 的块集合, 在一个普通的文件读取中, kernel 必须将 page cache 的内容拷贝到 user buffer, 这带来了物理内存浪费. 不过可以通过 **memory-mapped files** 解决这个问题:  
![alt memory-mapped files](http://static.duartes.org/img/blogPosts/mappedFileRead.png)

当使用 file mapping 时, kernel 将程序的 virtual pages 直接映射到 page cache. file mapping 可以是 private 或 shared 的.

#解答问题
回到本文开始的问题:

* more 程序是否有自己的*应用程序缓存(application buffer, user buffer)*
    * more 程序位于 utils-linux 包中, 阅读其源代码, 发现它主要调用 C stdio 的 getc() 和 putchar(), 而 C 的标准 IO 库是有自己的缓存的, 我相信该缓存是属于我们这里说的 "user buffer", 或 "application buffer", 或"应用程序缓存".
* more 将文件从硬盘读入到内存的什么地方? 
    * 根据上面 page cache 部分的描述, 文件**应该**是被读入到 page cache, 并从 page cache 拷贝到 user buffer.
* **page cache** 是什么, 在这个过程中有什么用? 
    * 此时该问题不言自明. 某些 page frames 用作 anonymous 内存, 某些 page frames 用作 page cache.
* 内存管理都是内核来做的? 
    * buddy memory allocation system. 应属于运行于内核态的代码.
* more 的输出是直接从某内核内存区域输出到 stdout 的吗?
    * 通过阅读代码, 应该不是这样的, 因为使用的是 getc() 和 putchar().
* more 程序中是否会数据从内核 copy 到 userland? 有 **system buffer** 吗?
    * 应该是有的, page cache 到 user buffer. 此时还不知道 system buffer 是何物, *怀疑是否真有这个概念*.

[book]: http://tldp.org/LDP/sag/sag.pdf "The Linux System Administrator's Guide, Memory Management"
