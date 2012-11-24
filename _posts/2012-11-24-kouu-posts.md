---
title: kouu的博文简介
layout: post
category: coding
tags: kouu
---
在了解很多话题时，都会搜索到[kouu](http://hi.baidu.com/new/_kouu)的博文，他的文章思路清晰，有birdview和关键细节，我受益很多。我阅读了他关键的旧文，这里简要记录，按时间排序。

**[从"read"看系统调用的耗时](http://hi.baidu.com/_kouu/item/dd6444d8386de1e4795daad9)**  
读同样大小为N的文件，一次读一个字节。速度：read < fread < mmap。  
原因：read很“老实”，一次读一个字节，触发N次系统调用；fread有内部缓冲，默认4K，触发N/4K次系统调用；mmap直接把文件映射进内存（read/fread也会利用disk cache，但仍然需要系统调用进入内核访问），支持像数组一样访问文件，只有一次mmap系统调用。  

**[由fastlock引发的...](http://hi.baidu.com/_kouu/item/4782dd8f19026d5d840fabd9)**  
*-暂无了解-*

**[linux内存管理浅析](http://hi.baidu.com/_kouu/item/4c73532902a05299b73263d0)**  
介绍了地址映射、虚拟地址管理、物理内存管理、内核空间管理、页面换入换出、用户空间内存管理等内容。  
作者制作了一张内存管理数据结构关系图。  

**[关于char**与const char**](http://hi.baidu.com/_kouu/item/8f78e742aa7fa332fa8960d0)**  
\*作为解引用符，关联性是right-to-left的。const char\*\*和char\*\*都是没有限定符的指针，指向的类型也不一样，前者是const char\*，后者是char\*。  

**[关于随机数](http://hi.baidu.com/_kouu/item/23dc511070208c25f6625cd0)**  
`rand()`得到的是伪随机数，因为种子和算法都是事先指定的。  
`/dev/random`得到真随机数，因为数据来源是外设中断，因为某些外部设备产生中断是随机的。  

**[关于C代码中的“逆向思维”](http://hi.baidu.com/_kouu/item/804195c274c3e262f7c95dd0)**  
循环从N->0，比从0->N要快（值得怀疑，因为有编译器优化），因为后者和0比较，CMP指令可以省掉。——计算机世界的微雕技术...  

**[浅尝异步IO](http://hi.baidu.com/_kouu/item/e70588f5c6eff2c4a835a2d9)**  
通过字符设备实现自己的异步io框架。因为进程的页表是继承自内核页表的，因此进程调入内核后，无需进行页表切换。  
内核2.6.2x开始提供了真正的异步IO——AIO。AIO会记录用户传入的buffer，以及用户进程的mm，然后在存取buffer之前，将页表切换为对应用户页表（use_mm函数），于是可以直接使用buffer，简化了问题，不过页表切换也影响了性能。  

**[神奇的vfork](http://hi.baidu.com/_kouu/item/93af230d0a22bc354ac4a3d9)**  
vfork使父子进程共享地址空间，因此对vfork的使用有限制：生成子进程后，父进程在vfork中被挂起，直到子进程有自己的内存空间（exec\*\*）或退出(\_exit)，并且在此之前，子进程不能从调用vfork的函数return。`man vfork`查看更多。  

**[linux异步信号handle浅析](http://hi.baidu.com/_kouu/item/479391211a84e3c9a5275ad0)**  
用户进程从内核态返回用户态，一般发生在三种情况下：系统调用（用户进程主动进入内核）完成、中断（用户进程被动进入内核）处理完成、被调度执行（用户进程从等待变成执行）。  
进程收到异步信号后，进程并不是立即被“中断”，而是先在`task_struct`中记录收到了某信号，然后等到进程将从内核态返回到用户态的时候，流程才被“中断”，handler才被调用。内核对此特别做了动作，因为本来是要返回到用户进程继续执行的。  
所以过程大致是这样的：  
用户态 -> 内核态 -> 准备返回用户态 -> 发现有信号需要处理 -> 修改内核堆栈使返回到handler -> 进入到用户态的handler执行 -> handler执行完毕 -> handler返回地址设置为内核公用的vsyscall页中对应的代码 -> 进入内核 -> 返回用户态最初的流程。  

**[linux线程同步浅析](http://hi.baidu.com/_kouu/item/2b9cac2385f64550c38d59d0)**  
直接使用信号来实现睡眠和唤醒是不靠谱的，可能会遇到“先唤醒、后睡眠”。  
可以通过pthread来做（`pthread_cond_wait`，`pthread_cond_signal`），pthread在条件变量内部会记录信号是否已经发生，如果`pthread_cond_signal`先于`pthread_cond_wait`，后者会放弃睡眠。  

**[剖析一个由sendfile引发的linux内核BUG](http://hi.baidu.com/_kouu/item/b74558542f6b9ca9acc857d0)**  
*-暂无了解-*

**[linux线程浅析](http://hi.baidu.com/_kouu/item/282b80a933ccc3a829ce9dd9)**  
linux上的线程就是基于轻量级进程，根本的数据结构都是`task_struct`。  
POSIX中的多线程标准是pthreads，Linux的实现是NPTL。

