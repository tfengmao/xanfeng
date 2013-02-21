---
title: printk控制
layout: post
tags: log level printk dmesg
category: linux
---

[Debugging by Printing](http://www.makelinux.net/ldd3/chp-4-sect-2)

printk 用来打印内核日志, 它的用法举例:

    printk(KERN_DEBUG "Here I am: %s:%i\n", _ _FILE_ _, _ _LINE_ _);
    printk(KERN_CRIT "I'm trashed; giving up on %p\n", ptr);

有 8 个日志级别(loglevel, priority), 定义于 <linux/kernel.h> 中,

    #define	KERN_EMERG	"<0>"	/* system is unusable			*/
    #define	KERN_ALERT	"<1>"	/* action must be taken immediately	*/
    #define	KERN_CRIT	"<2>"	/* critical conditions			*/
    #define	KERN_ERR	"<3>"	/* error conditions			*/
    #define	KERN_WARNING	"<4>"	/* warning conditions			*/
    #define	KERN_NOTICE	"<5>"	/* normal but significant condition	*/
    #define	KERN_INFO	"<6>"	/* informational			*/
    #define	KERN_DEBUG	"<7>"	/* debug-level messages			*/

没有指定级别的 printk 使用 DEFAULT_MESSAGE_LOGLEVEL,

    @ kernel/prink.c
    /* printk's without a loglevel use this.. */
    #define DEFAULT_MESSAGE_LOGLEVEL 4 /* KERN_WARNING */

根据设定的级别, printk 可能将日志打印到 console, serial port, parallel printer.  

如果 loglevel < console_level, 日志一次一行地被发到 console (由 trailing newline 触发). console_level 初始化为 DEFAULT_CONSOLE_LOGLEVEL, 可以被 sys_syslog 系统调用修改.

    @ kernel/prink.c
    /* We show everything that is MORE important than this.. */
    #define MINIMUM_CONSOLE_LOGLEVEL 1 /* Minimum loglevel we let people use */
    #define DEFAULT_CONSOLE_LOGLEVEL 7 /* anything MORE serious than KERN_DEBUG */    

如果 klogd 和 syslogd 都在运行, kernel messages 写到 /var/log/messages 的末尾, 不受 console_level 控制.

如果 klogd 不在运行, 日志不会发送到 userland, 除非用户主动读取 /proc/kmsg (一般通过 dmesg). 

使用 klogd 时需要注意, 它不记录连续相等的日志, 只记录第一个不同的行和重复次数.

**修改级别**

我们一般通过读写 /proc/sys/kernel/printk 控制 printk 级别, 这个文件包含 4 个值: current loglevel, 不指定级别时的 default level, the minimum allowed loglevel, the boot-time default loglevel.

    # cat /proc/sys/kernel/printk
    3       4       1       7

写入一个数值, 修改的是 current loglevel,

    # echo 8 > /proc/sys/kernel/printk

更多：  
1、使用 ioctl(TIOCLINUX) 选择接收日志的 virtual terminal.
2、how messages get logged：  
printk将消息写入大小为 __LOG_BUF_LEN(4KB~1MB) 的 ring buffer, 写入后 wakeup 等待消息的进程:  
+ 在 syslog 系统调用中睡眠的进程 - syslog 可以选择将 logdata 留在那里给其他进程.  
+ 正在读取 /proc/kmsg 的进程 - consume data from the log buffer.  

如果 klogd 进程在运行, 它获取 kernel messages 然后分发到 syslogd, syslogd 通过查看 /etc/syslog.conf 确定如何处理消息.  
如果 klogd 不在执行, 消息驻留在 ring buffer 中, 直到有人读取或者 buffer overflow.

*turning the messages on and off*  
*rate limiting*  
*printing device numbers*  
