---
title: big kernel lock
layout: post
tags: lock kernel linux
category: linux
---

[What's New in Linux 2.6.39: Ding Dong, the Big Kernel Lock is Dead](https://www.linux.com/learn/tutorials/447301-whats-new-in-linux-2639-ding-dong-the-big-kernel-lock-is-dead)

> Linus Torvalds has [released the 2.6.39](https://lkml.org/lkml/2011/5/19/16) kernel. This release brings new features, new drivers, and one big accomplishment: Ridding the Linux kernel of the Big Kernel Lock.
> 
> The Big Kernel Lock was almost removed in the 2.6.37 kernel. That is, the kernel could be built without it — but some of the code was still there.
> 
> With the 2.6.39 kernel, the BKL is [finally gone](https://lwn.net/Articles/424677/) with a patch from Arnd Bergmann. This has been a long-running saga, and [LWN](https://lwn.net/Articles/384855/) has some good coverage of what the BKL is (or was) and the [effort to get rid of i](https://lwn.net/Articles/424657/)t. Why the effort to get rid of it? The short answer is that the BKL was behind some performance issues and latencies that you really don't want.

[Big kernel lock semantics](http://halobates.de/blog/p/85)

> Again some historical background: the original Unix and Linux kernels were written **for single processor systems**. They employed a very simple model to avoid races accessing data structures in the kernel. Each process running in kernel code owns the CPU until it yields explicitly (I ignore interrupts here, which complicate the model again, but are not substantial to the discussion). That’s a classical cooperative multi-tasking or [coroutine](http://en.wikipedia.org/wiki/Coroutine) model. It worked very well, after all both Unix and Linux were very successful.
> Good literature on the topic is the older [Unix Systems for Modern Architectures](http://www.linuxjournal.com/article/34) book by Curt Schimmel, which provides a survey of the various locking schemes to convert a single processor kernel into a SMP system.

more details: [BigKernelLock](http://kernelnewbies.org/BigKernelLock)
> llseek  
> tty layer  
> block layer  
> file locking  
> superblock operations  