---
title: 文件读写流程
layout: post
tags: linux file read write ext3 io
category: linux
---

以ext3为例，来看文件读写流程。  

###read path

用户程序调用glibc的read函数。  
read()系统调用的实现是fs/read_write.c中的SYSCALL_DEFINE3(read)。它调用vfs_read()完成主要功能。

vfs_read()检查file->f_op->read函数指针是否为空，如果不为空则执行这个函数指针指向的函数。如果为空执行do_sync_read。  
ext3_file_operations中就是将read指针定义为do_sync_read。

2.6内核增加了异步IO能力，运行用户空间发起IO操作，而不需要等待IO操作的完成。实现异步IO的file_operations方法，都使用kiocb控制块。  
do_sync_read调用filp->f_op->aio_read，ext3的实现函数是generic_file_aio_read。

generic_file_aio_read判断f_flags是否包含**O_DIRECT**，如果包含，则执行direct IO。  
如果不是O_DIRECT，则做的是基于page-cache的读取。调用的是函数do_generic_file_read.

do_generic_file_read检查**page-cache**中是否有目标页面，如果存在则读取并返回；如果不存在就从磁盘读取，并在page-cache中也留一份。  
从磁盘读取是由address_space->a_ops->readpage做的，它指向的函数是mpage_readpages。

mpage_readpages循环调用do_mpage_readpage，每次读取一个页面。这里可以看出，内核事实上是一个页面为单位从磁盘上读取数据的。  
随着层层调用，最后执行到mpage_bio_submit，该函数调用submit_bio，至此读流程进入通用块设备层(Generic Block Layer)。

通用块设备层的工作此处略过，留待后面分析。

**内核预读机制**  
大多数磁盘操作是顺序的，且普通文件在磁盘上的存储一般都是占用连续的扇区。预读（read-ahead）就是在数据真正被访问之前，从普通文件或块设备文件中读取多个连续的文件页面到内存中。多数情况下，内核的预读机制可以明显地提高磁盘性能。  
不过当进程的大多数访问是随机读时，预读是对系统有害的，因为浪费了内核cache空间。

预读算法：  
- 批量  
- 提前  
- 预测：核心任务。  
  Linux，FreeBSD等主流OS都遵循一个简单有效的原则，即把读模式分为**随机读**和**顺序读**两大类，并只对顺序读进行预读。

内核判断两次读访问是顺序的标准是：请求的第一个页面与上次访问的最后一个页面是相邻的。访问一个给定的文件，预读算法使用两个页面集-当前窗口（current window）和前进窗口（ahead window），算法的细节此处略过。

###write path

用户程序调用write，一般由glibc write代理调用内核的系统调用。  
write系统调用的实现位于fs/read_write.c的SYSCALL_DEFINE3(write)，它调用vfs_write。

vfs_write检查file->f_op->write是否为空，如果为空执行do_sync_write，不过ext3中这个函数指针指向的也是do_sync_write。

do_sync_write执行filep->f_op->aio_write，也就是generic_file_aio_write，它调用__generic_file_aio_write。

__generic_file_aio_write中判断是否为O_DIRECT（unlikely），不是则执行generic_file_buffered_write，它调用generic_perform_write。

generic_perform_write调用address_space->a_ops->write_begin和address_space->a_ops->write_end，ext3中分别指向ext3_write_begin和（ext3_ordered_write_end，ext3_writeback_write_end，ext3_journalled_write_end）之一。

**内核刷新脏页机制**  
generic_file_buffered_write执行完之后，会逐层返回直至系统调用结束。但此时要写的数据，只是拷贝到内核缓冲区，并将相应的页面标记为脏，并未真正写到磁盘上。  
在下面条件下把脏页写回磁盘：  
- page-cache太满或脏页数量非常大；  
- 脏页停留在内存中的时间过长；  
- 某个进程要求更改的块设备或文件数据被刷新，通常通过调用sync，fsync，fdatasync来实现。

**pdflush内核线程**  
它有两个作用：  
1、系统地扫描page-cache，以找到要刷新的脏页；  
2、保证所有的页不会长期处于dirty状态。  

系统中pdflush线程数量是动态变化的，太少时就创建新的，太多时就杀死部分。在系统空闲内存低于一个特定的阈值时，pdflush将脏页刷新回磁盘。
