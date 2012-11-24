---
title: kouu的博文简介
layout: post
category: coding
tags: kouu
---
在了解很多话题时，都会搜索到[kouu](http://hi.baidu.com/new/_kouu)的博文，他的文章思路清晰，有birdview和关键细节，我受益很多。我阅读了他关键的旧文，这里简要记录，按时间排序。

1、[从"read"看系统调用的耗时](http://hi.baidu.com/_kouu/item/dd6444d8386de1e4795daad9)  
读同样大小为N的文件，一次读一个字节。速度：read < fread < mmap。  
原因：read很“老实”，一次读一个字节，触发N次系统调用；fread有内部缓冲，默认4K，触发N/4K次系统调用；mmap直接把文件映射进内存（read/fread也会利用disk cache，但仍然需要系统调用进入内核访问），支持像数组一样访问文件，只有一次mmap系统调用。  

2、[由fastlock引发的...](http://hi.baidu.com/_kouu/item/4782dd8f19026d5d840fabd9)  
*-暂无了解-*

3、[linux内存管理浅析](http://hi.baidu.com/_kouu/item/4c73532902a05299b73263d0)  
介绍了地址映射、虚拟地址管理、物理内存管理、内核空间管理、页面换入换出、用户空间内存管理等内容。  
作者制作了一张内存管理数据结构关系图。  

4、[关于char\*\*与const char\*\*](http://hi.baidu.com/_kouu/item/8f78e742aa7fa332fa8960d0)  
\*作为解引用符，关联性是right-to-left的。const char\*\*和char\*\*都是没有限定符的指针，指向的类型也不一样，前者是const char\*，后者是char\*。  

5、[关于随机数](http://hi.baidu.com/_kouu/item/23dc511070208c25f6625cd0)  
`rand()`得到的是伪随机数，因为种子和算法都是事先指定的。  
`/dev/random`得到真随机数，因为数据来源是外设中断，因为某些外部设备产生中断是随机的。  

6、[关于C代码中的“逆向思维”](http://hi.baidu.com/_kouu/item/804195c274c3e262f7c95dd0)  
循环从N->0，比从0->N要快（值得怀疑，因为有编译器优化），因为后者和0比较，CMP指令可以省掉。——计算机世界的微雕技术...  

7、[浅尝异步IO](http://hi.baidu.com/_kouu/item/e70588f5c6eff2c4a835a2d9)  
通过字符设备实现自己的异步io框架。因为进程的页表是继承自内核页表的，因此进程调入内核后，无需进行页表切换。  
内核2.6.2x开始提供了真正的异步IO——AIO。AIO会记录用户传入的buffer，以及用户进程的mm，然后在存取buffer之前，将页表切换为对应用户页表（use_mm函数），于是可以直接使用buffer，简化了问题，不过页表切换也影响了性能。  

8、[神奇的vfork](http://hi.baidu.com/_kouu/item/93af230d0a22bc354ac4a3d9)  
vfork使父子进程共享地址空间，因此对vfork的使用有限制：生成子进程后，父进程在vfork中被挂起，直到子进程有自己的内存空间（exec\*\*）或退出(\_exit)，并且在此之前，子进程不能从调用vfork的函数return。`man vfork`查看更多。  

9、[linux异步信号handle浅析](http://hi.baidu.com/_kouu/item/479391211a84e3c9a5275ad0)  
用户进程从内核态返回用户态，一般发生在三种情况下：系统调用（用户进程主动进入内核）完成、中断（用户进程被动进入内核）处理完成、被调度执行（用户进程从等待变成执行）。  
进程收到异步信号后，进程并不是立即被“中断”，而是先在`task_struct`中记录收到了某信号，然后等到进程将从内核态返回到用户态的时候，流程才被“中断”，handler才被调用。内核对此特别做了动作，因为本来是要返回到用户进程继续执行的。  
所以过程大致是这样的：  
用户态 -> 内核态 -> 准备返回用户态 -> 发现有信号需要处理 -> 修改内核堆栈使返回到handler -> 进入到用户态的handler执行 -> handler执行完毕 -> handler返回地址设置为内核公用的vsyscall页中对应的代码 -> 进入内核 -> 返回用户态最初的流程。  

10、[linux线程同步浅析](http://hi.baidu.com/_kouu/item/2b9cac2385f64550c38d59d0)  
直接使用信号来实现睡眠和唤醒是不靠谱的，可能会遇到“先唤醒、后睡眠”。  
可以通过pthread来做（`pthread_cond_wait`，`pthread_cond_signal`），pthread在条件变量内部会记录信号是否已经发生，如果`pthread_cond_signal`先于`pthread_cond_wait`，后者会放弃睡眠。  

11、[剖析一个由sendfile引发的linux内核BUG](http://hi.baidu.com/_kouu/item/b74558542f6b9ca9acc857d0)  
*-暂无了解-*

12、[linux线程浅析](http://hi.baidu.com/_kouu/item/282b80a933ccc3a829ce9dd9)  
linux上的线程就是基于轻量级进程，根本的数据结构都是`task_struct`。  
POSIX中的多线程标准是pthreads，Linux的实现是NPTL。

13、[linux虚拟文件系统浅析](http://hi.baidu.com/_kouu/item/6bfca5cc5d9778d4964452d0)  
比较特别的是描述了文件系统挂载。  
dentry是由slab分配的，用完时并不是直接被释放，而是被放到一个LRU链表中缓存起来，便于后续使用。  

14、[linux文件读写浅析](http://hi.baidu.com/_kouu/item/4e9db87580328244ef1e53d0)  
介绍了disk cache、通用块设备层、IO调度器和设备驱动等。  

15、[linux页面回收浅析](http://hi.baidu.com/_kouu/item/3590d5f2f9d48cb431c199d9)  
内存映射中文件映射的页面是磁盘高速缓存页面的子集。这篇对内存映射描述得不错。  
页面回收分主动和被动。被动回收由内核的页框回收算法(PFRA)搞定。  
磁盘高速缓存的页面（包括文件映射的）都是可以回收的，当然脏页要先写回；匿名映射页面不可回收，因为用户进程正在使用。  
要想回收匿名映射的页面，只好先把页面上的数据转储到磁盘，这就是页面交换（swap），页面可以被交换到磁盘上的交换文件或者交换分区上（下面统称为交换文件）。  
PFRA最后的必杀技是OOM killer。  

16、[linux时钟浅析](http://hi.baidu.com/_kouu/item/c3a81c36745225c11b9696d9)  
17、[linux进程状态浅析](http://hi.baidu.com/_kouu/item/7111e61acd04a9f487ad4ed0)  
18、[linux进程调度浅析](http://hi.baidu.com/_kouu/item/38c81042455c97d2c1a592d9)  

19、[linux网络报文接收发送浅析](http://hi.baidu.com/_kouu/item/6cf8c62998da170a42634ad0)  
20、[linux网桥浅析](http://hi.baidu.com/_kouu/item/25787d38efec56637c034bd0)  

21、[由mmap引发的SIGBUS](http://hi.baidu.com/_kouu/item/99690a0eae8568036c9048d0)  
22、[由mmap引发的SIGBUS（续）](http://hi.baidu.com/_kouu/item/1a8552f58edef710d7ff8cd9)  
mmap之后，如果文件被修改得很厉害，比如截断，会导致进程收到SIGBUS信号，然后崩溃。`man mmap`有相关说明。  
可以参考可执行文件映射，使用`MAP_DENYWRITE`选项，在外部修改文件时提示“text busy”。  

23、[虚拟机浅析](http://hi.baidu.com/_kouu/item/4966ea143b1982f89c778ad9)  
> 看了几天的虚拟机，感觉里面的水还是非常之深，并且还找不到一本合适的引人深入的书。路漫漫其修远兮……  

24、[linux文件系统实现浅析](http://hi.baidu.com/_kouu/item/c94edefb7a1e88773d198bd9)  
描述了inode、dentry等概念的含义。dentry部分配图不错。 

25、[浅谈linux定时器模型](http://hi.baidu.com/_kouu/item/9256659429340bf0291647d0)  
“周期性睡眠”和“按需睡眠”，内核中一般的定时器应该是用“周期性睡眠”的思路实现的，对高精度的定时器需求，内核提供了hrtimer，采用“按需睡眠”的思路实现。  

26、[记一个链接库导出函数被覆盖的问题](http://hi.baidu.com/_kouu/item/977755130f1f44fd756a84d9)  

27、[在多线程程序里面fork](http://hi.baidu.com/_kouu/item/358716f4c5f0cd0ec6dc45d0)  
