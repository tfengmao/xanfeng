---
title: pthreads多线程库(二)
layout: post
tags: pthreads linux kernel schedule thread process
category: linux
---

一些感悟：简单的东西远没有你想象的那么简单，复杂的东西远没有你想象的那么复杂。    
对于前者：细想scanf，它是如何捕捉你敲击的键的，stdio是否有buffer。    
对于后者：Linux kernel是一个极其复杂的项目，它的每一个subsystem也很复杂，但假以时日+合适的资料，我们总是可以了解更多。  
Code is not important, principle behind code matters! ——by xanpeng

我一直想去深入了解多线程编程和parallel programming，近期将付诸努力于此。  

我在"[pthreads多线程库(一)](http://xanpeng.github.com/2012/03/28/linux-pthread/)"中介绍了pthreads 多线程库：pthreads是一个标准, Linux下对应的实现是NPTL，实现代码位于glibc中。  
pthreads API的说明和使用在很多书籍里面都有详细的介绍，比如"POSIX多线程程序设计"，*这本书当然不错，但貌似被鼎鼎大名的冰河大牛力荐，我就不是特别理解了。我是把它当成一本参考书，从中我不能获得缘由，一本参考书为什么能得到力荐了，我想要么是冰河大牛只需要其参考书价值，要么就是冰河大牛对缘由已了然于胸了吧——其实这段文字没啥严密逻辑了，就当是弱弱的吐槽吧。*  

上篇文章提出了很多问题，很多还没有答案，回顾：  
1、NPTL是否在用户态做schedule?  
NPTL应该不在用户态做schedule，除非它傻。  
2、假如NPTL不是在用户态做schedule，而是通过对应的kernel thread来做，那么其中的细节是什么?  
仍然未知。  
3、NPTL用户态线程和内核线程的对应关系是什么? "1-1"还是"m-n"? 是否可配?  
我使用的是[glibc 2.11](http://en.wikipedia.org/wiki/GNU_C_Library#Version_history)，它"Used in SLES 11, eglibc 2.11 used in Debian Squeeze"，该环境下是1-1的。至于是否可以m-n，以及为什么，仍然未知。   

更多问题：  
1、[POSIX threads](http://en.wikipedia.org/wiki/POSIX_Threads)定义了标准，这个标准有哪些主要方面？ NPTL是否完全满足？Linux Kernel是否完全支持(或很好地支持)？  
2、什么是进程组？线程组？  
3、pthreads(指的是 NPTL)的主要概念及其原理？  
4、用户创建一个线程时，从用户态到内核的操作流程？线程task_struct数据存储在那里？  
5、NPTL或者Linux kernel中谁的支持使得能够实现mutex？  
6、内核同步方法(“Linux 内核设计与实现”第8,9章)？  

西邮的[edisionte](http://edsionte.com/techblog/)的线程方面的文章：  
1、[线程的那些事儿](http://edsionte.com/techblog/archives/3223)  
2、[进程在Linux内核中的角色扮演](http://edsionte.com/techblog/archives/3254)  
3、[线程那些事儿(2)-实践](http://edsionte.com/techblog/archives/3272)  

[线程的那些事儿](http://edsionte.com/techblog/archives/3223)：  
> 1、进程是系统资源分配的基本单位，线程是程序独立运行的基本单位。线程有时候也被称作小型进程，首先，这是因为多个线程之间是可以共享资源的；其次，多个线程之间的切换所花费的代价远远比进程低。    
> 2、如果内核要对线程进行调度，那么线程必须像进程那样在内核中对应一个数据结构。进程在内核中有相应的进程描述符，即task_struct结构。事实上，从Linux内核的角度而言，并不存在线程这个概念。内核对线程并没有设立特别的数据结构，而是与进程一样使用task_struct结构进行描述。也就是说线程在内核中也是以一个进程而存在的，只不过它比较特殊，它和同类的进程共享某些资源，比如进程地址空间、进程的信号、打开的文件等。我们将这类特殊的进程称之为轻量级进程（Light Weight Process）。  
> 3、POSIX标准规定在一个多线程的应用程序中，所有线程都必须具有相同的PID。从线程在内核中的实现可得知，每个线程其实都有自己的pid。为此，Linux引入了**线程组**的概念。在一个多线程的程序中，所有线程形成一个线程组。每一个线程通常是由主线程创建的，主线程即为调用pthread_create()的线程。因此该线程组中所有线程的pid即为主线程的pid。  
> 4、在内核中还有一种特殊的线程，称之为**内核线程**（Kernel Thread）。由于在内核中进程和线程不做区分，因此也可以将其称为内核进程。毫无疑问，内核线程在内核中也是通过task_struct结构来表示的。内核线程永远都运行在内核态，而不同进程既可以运行在用户态也可以运行在内核态。  

[进程在Linux内核中的角色扮演](http://edsionte.com/techblog/archives/3254)：  
> 1、在Linux内核中，内核将进程、线程和内核线程一视同仁，即内核使用唯一的数据结构task_struct来分别表示他们；内核使用相同的调度算法对这三者进行调度；并且内核也使用同一个函数do_fork()来分别创建这三种执行线程（thread of execution）。  
> 2、进程描述符task_struct的多角色扮演...  
> 3、do_fork()的多角色扮演...  

![do_fork()的多角色扮演](http://edsionte.com/techblog/wordpress/wp-content/uploads/2011/09/proc_thread_colar.jpeg)

[线程那些事儿(2)-实践](http://edsionte.com/techblog/archives/3272)：  
这里做了和我同样的工作，就是写个test.c，里面起了多个threads，然后ps可以看出每个threads PID、PPID是相同的，当然LWP(轻量级进程的 PID)是不同的。  

[2012-05-25更新记录]   
10天前，我开始写这篇post，目的是彻底认清pthread库及其后面的机理，认清Linux内核如何支援。为此，我做了一些有趣的事情，到目前结果虽不完整，有待继续更新，但也能一舒我胸中块垒。  
-> 了解"[进程组和线程组](http://xanpeng.github.com/linux/2012/05/16/process-threads-group.html)"，  
-> 然后去思考`ps aux`后面的细节。  
-> 想到ps、pthread都是位于glibc库中，便想如何不通过阅读代码和文档，去查看调用路径，于是有了“[ltrace简析](http://xanpeng.github.com/linux/2012/05/18/ltrace.html)”。  
-> 在编译ltrace的时候，需要libelf的支持，于是去再次了解elf，产生这篇“[elf和elftoolchain](http://xanpeng.github.com/linux/2012/05/17/elf-libelf.html)”。  
-> 想从`ps aux`这条命令管中窥豹，于是去看了getpid，再次了解系统调用，玩玩乐乐几天之后，不再有兴趣去看ps，因为觉得剩下来的只是细节而已，而我现在最想知道的是**原理和主体机制**。但仍结出虽不完整的“[getpid浅析](http://xanpeng.github.com/linux/2012/05/19/ps-getpid.html)”。  
-> 仍想到ps、pthread都位于glibc（我不知道为何认定pthread位于glibc，因为显然`ldd ls`看出，存在一个单独的 libpthread.so)，延续对elf的思考，这次想了解符号(symbol)，并希望借此对很多事情有更深的理解，于是结出“[动态连接库和符号](http://xanpeng.github.com/2012/05/21/solib-elf-symbol/)”。  