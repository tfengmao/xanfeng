---
title: 块设备驱动
layout: post
category: linux
tags: driver block disk hardware kernel
---

*本文是ldd3 ch16的笔记。*  
块设备驱动：block driver。*在表达时，一不小心我们就混淆了块设备和块设备驱动，为了表述清楚，**下文用bdriver表示块设备驱动**。*  

我觉得做文件系统、做存储等，都需要理解块设备。内存快，块设备慢，bdriver做的如何直接影响系统性能。  
“[The Linux I/O Stack Diagram](http://www.thomas-krenn.com/en/oss/linux-io-stack-diagram/linux-io-stack-diagram_v0.1.pdf)”显示，VFS下是文件系统，文件系统不管通过page cache，还是dio，都需要经过块设备层(**block I/O layer**)。  
块设备层针对“慢”的块设备做了非常多的逻辑，实际上是提供了很多功能和性能方面的机制，比如请求队列、I/O scheduler、各个功能抽象接口。  

硬件厂商生产出一款块设备，准备使用在Linux系统中，厂商需要提供bdriver。  
这个bdriver不是胡乱写的，需要遵循块设备层制定的标准。  
*因此，理解bdriver，实际上是为了理解块设备层。*  
bdriver提供的功能——或者说如何使用块设备——大致有：注册、取消注册、打开和关闭等block_device_operations.  

但最重要的是响应上层读写的**request**方法。  
每一个bdirver都有一个请求队列**request queue**，request queue是一个极其复杂的数据结构，因为它要做的事情太多太杂：它提供插件机制供选择不同的I/O scheduler、包含各种描述queue的参数(容纳请求的数目，块设备sector大小等)、启动暂停queue等控制函数，最重要的，便是**操纵request**的能力，如给queue中的request重新排序，使得可以用更少的硬件访问次数搞定。

bdriver的普通功能不消细说，无非是那样：内核中有某个list，指向系统中所有的块设备，所以bdriver通过注册，挂在这个list上面，并且初始化一些变量和函数指针。如此这般。  
比较特别的有，块设备操作函数中media_changed()相关的和可插拔硬件相关。  

bdriver的非普通功能一时我也不能细说，request如何放到request queue里面，request queue有哪些操作，有哪些和性能调整有关的操作，...，不得而知。  
创建request queue时，关联request方法。  

---

记录一些特别的：  
- major number可以在注册时传进去，也可以不指定，不指定就由内核分配。  
- minor number表示块设备的各个分区。一般注册时传一个16，表示最多支持16个分区。  
- 调用块设备open方法时，相当于开启硬件，比如让磁盘转起来。  
- 内核不仅仅支持512字节大小的sector，blk_queue_hardsect_size告诉块设备层硬件sector大小。  
- 用户调用request()，request()返回的时候，不保证真正完成了硬件请求。  
- request()运行在atomic context中。  
- request()可以运行在用户进程中(内核部分)，也可以纯粹在内核中运行。request()不感知运行在二者谁之下。  
- request queue中的request，可以不是用来访问块设备的，用block_fs_request()判别。  
- 有一种特别的barrier request，用来保证request时序。但是这需要块设备支持，因为很多块设备缓存write request，如果数据还在硬件缓存中时遇到宕机，仍然会有不一致。遵从barrier request的块设备通过blk_queue_ordered()告诉内核。  
- request()结束后，通过某种机制告诉上层(块设备层，即block I/O layer，即block subsystem)。  
