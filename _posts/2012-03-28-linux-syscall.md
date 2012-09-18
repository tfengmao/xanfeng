---
title: Linux 系统调用
layout: post
tags: linux userland kernel system call
category: linux
---

本文 "严重抄袭" "[Linux内核之旅](http://www.kerneltravel.net/)"的文章"[系统调用](http://www.kerneltravel.net/journal/iv/syscall.htm)", 特此告知, 鸣谢!

下文中都用 **syscall** 表示系统调用.

#概述

syscall 是什么, 做什么?  
1. 用户空间和内核空间的接口.  
2. 为用户空间提供一种硬件的抽象接口.  
3. 目的: 保证系统稳定可靠, 避免应用程序肆意妄为, 惹出大麻烦.  

在 Linux 中, syscall 是用户空间访问内核的唯一手段, 除异常和陷入外, 它们是内核唯一的合法入口. 实际上, 设备文件和/proc之类的文件系统, 最终也还是通过 syscall 进行访问的.

Linux syscall 像大多数 Unix 系统一样, 作为C库的一部分提供, 如:

    应用程序printf() -> C库中的printf() -> C库中的write() -> write() syscall

内核只跟 syscall 打交道, 内核不关心库函数及应用程序如何使用 syscall, 但内核必须牢记 syscall 所有潜在的用途并保证它们有良好的通用性和灵活性.  
Unix 接口设计有一句通用的格言: "**提供机制而不是策略**", "需要提供什么功能"是机制问题, "怎样实现这些功能"这是策略问题. 程序开发中如果独立处理机制和策略, 开发软件就更容易.

**内核在执行 syscall 的时候处于进程上下文**, current 指针指向当前任务, 即引发 syscall 的那个进程.  
在进程上下文中, 内核可以休眠, 可以被抢占. 这两点都很重要, 休眠说明 syscall 可以使用内核提供的绝大部分功能.

#syscall details

[Linux内核之旅](http://www.kerneltravel.net/)的一篇文章"[系统调用](http://www.kerneltravel.net/journal/iv/syscall.htm)"也讲述了 syscall, 是不错的资料. 下文主要参考其中第4节 "系统调用的实现".

Linux syscall 利用了 x86 体系结构中的软件中断(*软件中断虽然叫中断, 但实际上属于异常, 更准确说是陷阱, 是CPU发出的中断, 而且是由编程者触发的一种特殊异常*), 软件中断和平时常说的中断(*硬件中断*)不同, 它是通过软件指令触发而并非外设引发的中断, 也就是说, 是编程人员开发出的一种异常, 具体的讲就是调用 `int $0x80` 汇编指令, 这条汇编指令将产生向量为128的编程异常.

之所以 syscall 需要借助异常(***注意区别于C++等中的异常概念***)来实现, 是因为当用户态的进程调用 syscall 时, CPU便被切换到内核态执行内核函数, 而进入内核(*高特权级别*)必须经过系统的门机制([I386的体系结构](http://www.kerneltravel.net/journal/ii/part1.htm)), 这里的异常实际上就是通过系统门陷入内核.

更详细地解释一下这个过程. `int $0x80` 指令的目的是产生一个编号为128的编程异常, 这个编程异常对应的是**中断描述符表IDT**中的第128项, 也就是对应的系统门描述符. 门描述符中含有一个预设的内核空间地址, 它指向了系统调用处理程序: system_call()

所有的系统调用都会统一转到这个地址, 但 Linux 有几百个系统调用, 都从这里进入内核后, 又该如何派发到它们到各自的服务程序去呢? 解决这个问题的方法非常简单: 首先 Linux 为每个系统调用都进行了编号(0-NR_syscall), 同时在内核中保存了一张**系统调用表**, 保存了系统调用编号和对应的服务例程, 因此在通过系统门陷入内核前, 需要把**系统调用号**一并传入内核, 在x86上, 这个传递动作是通过在执行 `int 0x80` 前把调用号装入eax寄存器实现的. 这样系统调用处理程序一旦运行, 就可以从eax中得到数据, 然后再去系统调用表中寻找相应服务例程了.

除了需要传递系统调用号以外, 许多 syscall 还需要传递一些**参数**到内核, 比如 `sys_write(unsigned int fd, const char * buf, size_t count)` 调用就需要传递文件描述符fd, 要写入的内容buf, 以及写入字节数count等几个内容到内核. 碰到这种情况, Linux 会有**6个寄存器**可被用来传递这些参数: eax(存放系统调用号), ebx, ecx, edx, esi 及 edi 来存放这些额外的参数. 具体做法是在system_call() 中使用 **SAVE_ALL** 宏把这些寄存器的值保存在内核态堆栈中.

当服务例程结束时, system_call() 从 eax 获得 syscall 的返回值, 并把这个返回值存放在曾保存用户态 eax 寄存器栈单元的那个位置上. 然后跳转到 `ret_from_sys_call()`, 终止系统调用处理程序的执行.  
当进程恢复它在用户态的执行前, `RESTORE_ALL` 宏会恢复用户进入内核前被保留到堆栈中的寄存器值. 其中 eax 返回时会带回 syscall 的返回码(负数说明调用错误, 0或正数说明正常完成).

#系统调用思考

##调用上下文分析

syscall 虽说是要进入内核执行, 但它并非一个纯粹意义上的内核例程. 首先它是代表用户进程的, 这点决定了虽然它会陷入内核执行, 但是**上下文**仍然是处于进程上下文中, 因此可以访问进程的许多信息(*比如current结构*), 而且可以被其他进程抢占(*在从系统调用返回时，由system_call函数判断是否该再调度*), 可以休眠, 还可接收信号等等. 

所有这些特点都涉及到了进程调度的问题, 此处不做深究, 但需明白在 syscall 完成后, 再回到或者说把控制权交回到发起调用的用户进程前, 内核会**有一次调度**. 如果发现有优先级别更高的进程或当前进程的时间片用完, 那么就会选择高优先级的进程或重新选择进程运行. 除了再调度需要考虑外, 再就是内核需要检查**是否有挂起的信号**, 如果发现当前进程有挂起的信号, 那么还需要先返回用户空间执行信号处理例程(*处于用户空间*), 然后再回到内核, 重新返回用户空间, 有些麻烦但这个反复过程是必须的.

##调用性能问题

syscall 需要从用户空间陷入内核空间, 处理完后, 又需要返回用户空间. 其中除了 syscall 服务例程的实际耗时外, 陷入/返回过程和 syscall 处理程序(查系统调用表, 存储/恢复用户现场)也需要花费一些时间, 这些时间加起来就是一个 syscall 的响应速度. syscall 不比别的用户程序, 它对性能要求很苛刻, 因为它需要陷入内核执行, 所以和其他内核程序一样要求代码简洁, 执行迅速. 幸好 Linux 具有令人难以置信的上下文切换速度, 使得其进出内核都被优化得简洁高效; 同时所有 Linux syscall 处理程序和每个 syscall 本身也都非常简洁.

绝大多数情况下, Linux syscall 的性能是可以接受的, 但是对于一些对性能要求非常高的应用来说, 它们虽然希望利用系统调用的服务, 但却希望加快响应速度, 避免陷入/返回和 syscall 处理程序带来的花销, 因此采用由内核直接调用 syscall 服务例程, 最好的例子就 HTTPD--它为了避免上述开销, 从内核调用 socket 等 syscall 服务例程.

#增加 syscall 

通常 syscall 靠C库支持, 添加自己的 syscall 之后, glibc恐怕并不提供支持.  
不过, Linux本身提供了一组宏, 用于直接访问 syscall, 它会设置好寄存器(存储系统调用号)并调用陷入指令. 这些宏是 **_syscalln**(), n的范围0～6, 代表需要传递给 syscall 的参数个数. 比如open() syscall 的定义：

    long open(const char *filename, int flags, int mode);

不靠库支持, 直接使用宏调用此 syscall 的形式为:

    #define _NR_OPEN 5
    _syscall3(long, open, const char*, filename, int, flags, int, mode);

增加自己新建的 syscall, 需要做以下工作:  
1. 编写 syscall 对应的代码. 这里需要遵循一定的规则.  
2. 注册 syscall. 在 syscall table 的最后加入一个表项.  
3. syscall 必须被编译进内核映像(不能编译成模块). 

如何添加一个新的 syscall 的细节, 看这份文档: "[tldp: Implementing a System Call on Linux 2.6 for i386](http://tldp.org/HOWTO/html_single/Implement-Sys-Call-Linux-2.6-i386/)".

#more
  
Linux内核设计与实现  
[Linux systemcall reference](http://syscalls.kernelgrok.com/)  
[Implementing a System Call on Linux 2.6 for i386](http://tldp.org/HOWTO/html_single/Implement-Sys-Call-Linux-2.6-i386/)   
