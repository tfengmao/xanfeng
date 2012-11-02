---
title: 用户态-内核态通信方式
layout: post
category: linux
tags: kernel userland netlink procfs
---

值此小光棍节，夜深，我写此文。  
往事不敢追忆，而今奋起，期把握明日机会，追明日幸福。  
且愿我中意的女孩、中意我的女孩们此生幸福。  

###总纲——内核态与用户态通信的实现与分析（[链接](http://www.ibm.com/developerworks/cn/linux/l-netlink/index.html)）

多数的Linux内核态程序都需要和用户空间的进程交换数据，但是内核态无法对传统的Linux进程间同步和通信的方法提供足够的支持。下面总结并比较几种内核态与用户态进程通信的实现方法，并推荐使用netlink套接字实现**中断环境与用户态进程通信**。  

在一台运行Linux的计算机中，CPU在任何时候只会有如下四种状态：  
【1】在处理一个硬中断。  
【2】在处理一个软中断，如softirq、tasklet和bh。  
【3】运行于内核态，但有进程上下文，即与一个进程相关。  
【4】运行一个用户态进程。  
其中，【1】、【2】和【3】是运行于内核空间的，而【4】是在用户空间。其中除了【4】，其他状态只可以被在其之上的状态抢占。比如，软中断只可以被硬中断抢占。  

Linux内核模块是一段可以动态在内核装载和卸载的代码，装载进内核的代码便立即在内核中工作起来。  
Linux**内核代码的运行环境**有三种：用户上下文环境、硬中断环境和软中断环境。但三种环境的局限性分两种，因为软中断环境只是硬中断环境的延续。  
1、内核态环境——用户上下文：  
介绍：内核态代码的运行与一用户空间进程相关，如系统调用中代码的运行环境。  
局限：不可直接将本地变量传递给用户态的内存区，因为内核态和用户态的内存映射机制不同。  
2、内核态环境——硬中断和软中断环境：  
介绍：硬中断或软中断过程中代码的运行环境，如IP数据报的接收代码的运行环境，网络设备的驱动程序等。  
局限：不可直接向用户态内存区传递数据；代码在运行过程中不可阻塞。  

Linux传统的进程间通信有很多，如各类管道、消息队列、内存共享、信号量等等。但它们都无法介于内核态与用户态使用，原因：  
管道（不包括命名管道）：局限于父子进程间的通信。  
消息队列：在硬、软中断中无法无阻塞地接收数据。  
信号量：无法介于内核态和用户态使用。  
内存共享：需要信号量辅助，而信号量又无法使用。  
套接字：在硬、软中断中无法无阻塞地接收数据。  

**内核态环境——用户上下文：**  
运行在用户上下文环境中的代码是可以阻塞的，这样，便**可以使用消息队列和UNIX域套接字***（how？）*来实现内核态与用户态的通信。但这些方法的数据传输效率较低，Linux内核提供copy_from_user()/copy_to_user()函数来实现内核态与用户态数据的拷贝，但这两个函数会引发阻塞，所以不能用在硬、软中断中。  
一般将这两个特殊拷贝函数用在类似于系统调用一类的函数中，此类函数在使用中往往"穿梭"于内核态与用户态：  
![](http://www.ibm.com/developerworks/cn/linux/l-netlink/images/image001.gif)  

**内核态环境——硬中断和软中断环境：**  
【1】一般进程间通信的方法  
比起用户上下文环境，硬中断和软中断环境与用户态进程无丝毫关系，而且运行过程不能阻塞。我们无法直接使用传统的进程间通信的方法实现这样的内核代码和用户态通信。  
但硬、软中断中也有一套同步机制——自旋锁（spinlock），可以通过自旋锁来实现中断环境与中断环境、中断环境与内核线程的同步，而内核线程是运行在有进程上下文环境中的，这样便可以在内核线程中使用套接字或消息队列来取得用户空间的数据，然后再将数据通过临界区传递给中断过程*（强烈怀疑这个过程）*。如图所示：  
![](http://www.ibm.com/developerworks/cn/linux/l-netlink/images/image002.gif)  

因为中断过程不可能无休止地等待用户态进程发送数据，所以要通过一个内核线程来接收用户空间的数据，再通过临界区传给中断过程。中断过程向用户空间的数据发送必须是无阻塞的。这样的通信模型并不令人满意，因为内核线程是和其他用户态进程竞争CPU接收数据的，效率很低，这样中断过程便**不能实时地接收**来自用户空间的数据。  

【2】netlink socket  
在Linux 2.4版以后版本的内核中，几乎全部的中断过程与用户态进程的通信都是使用netlink socket实现的。netlink的最大特点是**支持中断过程**。  
netlink的通信依据是一个对应于进程的标识，一般定为该进程的ID。当通信的一端处于中断过程时，该标识为0。当使用netlink进行通信，通信的双方都是用户态进程时，使用方法类似于消息队列。但如果通信双方有一端是中断过程，使用方法则不同。它在内核空间接收用户空间数据时不再需要用户自行启动一个内核线程，而是通过另一个软中断调用用户事先指定的接收函数。如图所示：  
![](http://www.ibm.com/developerworks/cn/linux/l-netlink/images/image003.gif)  

这里使用了软中断而不是内核线程来接收数据，这样就可以保证**数据接收的实时性**。

###实例——内核态-用户态通信方式（[链接](http://people.ee.ethz.ch/~arkeller/linux/kernel_user_space_howto.html)）

【1】procfs、sysfs、debugfs  
这些fs都是可选的，/lib/modules/`uname -r`/build/.config告诉你你的内核是如何配置的。  
特点：  
> In order to exchange data between user space and kernel space the Linux kernel provides a couple of **RAM based** file systems. These interfaces are, themselves, **based on files**. Usually **a file represents a single value**, but it may also represent a set of values. The user space can access these values by means of the **standard read(2) and write(2) functions**. For most file systems the read and write function results in **a callback function in the Linux kernel** which has access to the corresponding value.

procfs有两套API供使用：procfs API（数据大小<PAGE_SIZE）和seq_file API（支持数据大小>PAGE_SIZE）。这两种方式我都有文提及。  
sysfs数据大小也是小于PAGE_SIZE。  
这些不同的fs分别有不同的用途，比如debugfs是为了debugging目的诞生的。

【2】configfs  
这个略微特别一些，和其它类似fs的重要区别是：在configfs中，所有的文件都可以通过mkdir在用户态创建。  
我也有文描述过configfs。  

【3】sysfs中的模块参数API  
内核模块可以处理传入参数，插入模块时可以接受参数传入，运行时也可以接受参数传入。  
module_param宏会为参数创建一个文件/sys/modules/module_name/name，创建时会指定权限，如果这个文件的访问权限设定为0，那么这个文件不会被创建，因此对应参数也无法在运行时被访问。  
运行[kernel-mm/mmap-brk-file-anonymous.md](https://github.com/xanpeng/kernel-mm/blob/master/mmap-brk-file-anonymous.md)中的例子printvma，可以看到模块参数的传入，但是在/sys/modules/printvma/下没有文件pid_mem。修改module_param的权限参数后，重新insmod，发现会有一个文件：  
{% highlight text %}
# grep module_param printvma.c
module_param(pid_mem, int, 0);
# insmod ./printvma.ko pid_mem=15834
# dmesg
[147435.631681] Got the process id to look up as 15884
[147435.631717] fish[15884]
[147435.631718] This mm_struct has 74 vmas.
...
# ls /sys/module/printvma/
holders/  initstate  notes/  refcnt  sections/  srcversion  supported

# grep module_param printvma.c
module_param(pid_mem, int, S_IRUSR | S_IRGRP | S_IROTH);
# rmmod printvma && make && insmod ./printvma.ko pid_mem=15834
# ls /sys/module/printvma/
holders/  initstate  notes/  parameters/  refcnt  sections/  srcversion  supported
# cat /sys/module/printvma/parameters/pid_mem
15884
{% endhighlight %}

【4】sysctl  
被设计用来在运行时修改内核参数，位于/proc/sys/下面。可以被cat、echo和sysctl(8)命令访问。如果值是通过echo设定的，该值在内核重启后失效（用sysctl设定呢？）。为了永久有效，应该在/etc/sysctl.conf中修改。  
这些值在内核代码中是有对应记录的，参考`struct ctl_table`。  

【5】字符设备  
用的很多，用的很广泛。创建一个字符设备（比如通过mknod），为其编写驱动（char device driver），设定如何接收和发送数据。

【6】socket方案之netlink  
优点：  

- Simple to interact with kernel, as only a constant has to be added to the Linux kernel source code. **No risk to pollute** the kernel or to drive it in instability, since the socket can immediately be used.
- Netlink sockets are **asynchronous** as they provide queues, meaning they **do not disturb kernel scheduling**. This is in contrast to system calls which have to be executed immediately.
- Netlink sockets provide the possibility of multicast.
- Netlink sockets provide a truly **bidirectional** communication channel: A message transfer can be initiated by either the kernel or the user space application.
- They have less overhead (header and processing) compared to standard UDP sockets.

缺点：  

- Each entity using netlink sockets has to **define its own protocol type** (family) in the kernel header file include/linux/netlink.h, necessiating a kernel **re-compilation** before it can be used.
- The maximum number of netlink families is fixed to 32. If everyone registers its own protocol this number will be exhausted.

所以一般使用“Generic Netlink Family”。实现分用户态和内核态两端，在[kernel-communication](https://github.com/xanpeng/kernel-communication)下面有示例。gnkernel.c是内核模块代码，gnuser.c是用户态代码。其中gnuser.c没有使用libnl或者libnetlink库，而是使用低端API，这在实际编程中是不推荐的。  
运行效果如下：  
{% highlight text %}
# insmod ./gnkernel.ko
# dmesg
[105461.079341] INIT GENERIC NETLINK EXEMPLE MODULE

# ./gnuser
kernel says: hello world from kernel space
# dmesg
[105276.831232] received: hello world!

// 现在只修改gnuser代码，改变消息（当然可以实现为参数传入），内核模块不动。
# ./gnuser
kernel says: hello world from kernel space
# dmesg
[105517.107065] received: [from user] hello world!

# rmmod gnkernel
# ./gnuser
received error
error received NACK - leaving
{% endhighlight %}


