---
title: dmesg和内核ring buffer
layout: post
tags: dmesg kernel buffer trace
category: linux
---

dmesg为我们多用，`man dmesg`告知dmesg用来显示和管理kernel ring buffer，那么后者为何物，以及dmesg显示何类信息，是本文待阐述的内容。

documentation/trace/ring-buffer-design.txt包含了详细的设计方案*(看来documentation下的内容应是后续查找tutor的首选)*，
其中细节不是目前所需，但看起来却是是设计**无锁(lockless)**日志系统的绝佳参考资料。

ring buffer（rb）和kernel ring buffer（krb）是不同的东西，前者泛指一类buffer设计方案，后者特指Linux内核使用的ring buffer。

rb有两种模式：  
1、producer/consumer模式：如果生产的太多，没有来得及消费，“仓库”占满了，那么就暂停生产--这种模式可能会丢失最近的事件记录。  
2、overwrite模式：生产者填满“仓库”的时候，它仍然继续生产，并覆盖最旧的事件记录--这种模式可能会丢失最旧的事件记录。

krb位于内核(对应于/proc/kmsg)，又称之为Kernel Log Buffer，包含和内核、内核模块相关的消息，如装载新模块的消息、注册新的文件系统的消息等。这些消息被划分为不同的级别：  
- LOG_EMERG system is unusable  
- LOG_ALERT action must be taken immediately  
- LOG_CRIT critical conditions  
- LOG_ERR error conditions  
- LOG_WARNING warning conditions  
- LOG_NOTICE normal, but significant, condition  
- LOG_INFO informational message  
- LOG_DEBUG debug-level message  

kernel日志(如printk的数据)视情况发往不同的地方：  
- 如果klogd和syslogd都在运行，kernel messages被写到/var/log/messages的末尾(或者syslogd配置给定的地方)，此时和级别无关，所有级别日志信息都写入。  
- 如果klogd没有运行，消息不会发往用户空间(不会写messages文件?)，除非显式读取/proc/kmsg(一般通过kmsg)。

读取krb的首选方案是dmesg，dmesg其实是通过系统调用syslog去读取的。

参考资料中的“Kernel logging: APIs and implementation”包含了一些图片，直观展示了krb及其上的操作。

dmesg的输出和/var/log/messages的内容有什么区别呢？它们有父子集的关系吗？  
我在unix.stackexchange上[问了这个问题](http://unix.stackexchange.com/questions/35851/whats-the-difference-of-dmesg-output-and-var-log-messages)，但目前还没得到满意的答案。

参考  
- [Kernel ring buffer and dmesg](http://www.web-manual.net/linux-3/the-kernel-ring-buffer-and-dmesg/)  
- [How to read ring buffer within linux kernel space?](http://stackoverflow.com/questions/9533708/how-to-read-ring-buffer-within-linux-kernel-space)  
- [Debugging by Printing](http://www.makelinux.net/ldd3/chp-4-sect-2)  
- [**Kernel logging: APIs and implementation**](http://www.ibm.com/developerworks/linux/library/l-kernel-logging-apis/index.html)  
- [Difference between /var/log/messages, /var/log/syslog, and /var/log/kern.log?](http://askubuntu.com/questions/26237/difference-between-var-log-messages-var-log-syslog-and-var-log-kern-log)  
