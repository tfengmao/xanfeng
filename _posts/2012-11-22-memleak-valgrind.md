---
title: 内存泄漏和Valgrind
layout: post
category: coding
tags: memory leak valgrind
---

###内存泄漏

内存泄漏（memory leak）：该释放的内存没有被释放。一般指在用户态程序中，malloc内存之后，没有对应的free。  

这里我关注的是：我malloc了内存，我就是不去free，进程退出之后这些没有被free的内存，还能被其他进程使用吗？  
答案是**可以**，进程退出时，操作系统会负责回收进程分配的资源，包括内存、打开文件等。操作系统能这么做，是因为在分配资源时，资源是和进程关联的。  
我写了一个小程序，申请64M内存，然后通过脚本循环执行1000次。——毫无压力。  
另外，“[What REALLY happens when you don't free after malloc?](http://stackoverflow.com/questions/654754/what-really-happens-when-you-dont-free-after-malloc)”也讨论了这个话题。  

当然，如果是daemon进程，自然不能放过每一个需要free的内存，否则24*7地执行下去，很快内存会被耗尽。  
对其他短期执行的进程，认真的去做“free”，则更多地是一个好习惯而已了。  

###Valgrind

[valgrind](http://valgrind.org/)因其内存检测能力出名，但实际上，valgrind还[包含很多其他工具](http://valgrind.org/docs/manual/manual-intro.html)，主要分两方面：check，profiling（毫无经验的样子）。  

一般用valgrind，用的是其默认的`--tool=memcheck`。valgrind内存检查为什么能做得如此优秀，大概因为它会[虚拟CPU](http://valgrind.org/docs/manual/manual-core.html#manual-core.whatdoes)，让目标进程跑在其上。  
既然是虚拟CPU，联系其他方面，就会有一些有意思的问题：怎么[处理多线程](http://valgrind.org/docs/manual/manual-core.html#manual-core.pthreads)，[信号处理](http://valgrind.org/docs/manual/manual-core.html#manual-core.signals)，执行valgrind的同时如何使用[gdb调试](http://valgrind.org/docs/manual/manual-core-adv.html#manual-core-adv.gdbserver)，...  

为了更好地执行valgrind，可以选择使用这些选项编译目标程序：-g使valgrind能输出确定的代码行号；-O0取消代码优化，使得valgrind展示的信息更贴近源码。  

valgrind也不是100%准确，但在[99%的时候](http://valgrind.org/docs/manual/quick-start.html#quick-start.caveats)都是对的。  
所以`valgrind --tool=memcheck --leak-check=yes ls -l`的时候，输出一堆警告和错误，也不要怀疑valgrind是不是出错了。——因为大牛在写`ls -l`的时候，或许有些小内存他就是不想“多此一举”地去释放了。  

###Linux ate my ram！

`free -m`时，往往会发现，4G的ram，free显示只有几百M了。不用紧张，这并不表示Linux很吃内存，看这篇文章，“[Linux ate my ram!](http://www.linuxatemyram.com/)”:  
> Don't Panic! Your ram is fine!

这是因为内核把内存“借”给disk cache用了，当其他程序需要更多内存是，disk cache的内存是可以释放的。  
这篇文章的[作者设计了实验](http://www.linuxatemyram.com/play.html)证明这一点。他写了一个程序，不停地malloc(1M)，并立即memset以防止编译器优化，发现仍然可以申请到3600+M的内存，然后被OOM killer杀掉。这时`free -m`发现free的内存多了3600+M了，因为disk cache已经“被迫”刷回磁盘。  
可以通过修改`/proc/sys/vm/drop_caches`值强迫disk cache刷回磁盘。  