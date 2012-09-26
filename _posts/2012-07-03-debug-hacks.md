---
title: 内核开发调试技术
layout: post
tags: debug hacks kernel gdb kprobe crash assemble
category: programming
---

*本文是书籍《Debug Hacks中文版-深入调试的技术和工具》的笔记. 这是一本好书, 它介绍了应用程序和内核调试的多方面的技能, 虽然每个点不是非常深入, 但我事先有些许基础, 故而读起来比较畅快! 我给这本书打4.5分, 他是作者数十年经验的传承, 十分珍贵, 然而不足之处在于对于某些工具仅谈其使用, 而不谈其原理. 比如假使在讲述GDB时, 如指出GDB依赖于ptrace()系统调用, 那就完美了!*

*本书作者是日本人, 于是加强了我对日本人的好感, 延续了在我眼中勤俭务实的优点. 在IT方面, 除了老美之外, 数TW和日本同行为我尊敬. 我对TW ITer有些了解, 他们的谦谨好学, 在论坛开放的讨论风格为我敬佩, 相比之下, 国内的讨论多有傲慢和怒骂.*

言归正传. "Debug Hacks"和另外一本书"Binary Hacks"类似, 也采用recipes的方式写作, 同样的章节安排形式也见于"Effective C++"等. 碰巧"Binary Hacks"也是日本人写作的, 如其题目, 是介绍binary级别操作的一本书, 由此更可见日本人对基础的夯实态度.  

Debug Hacks分6章中列出了66个话题. 根据平时使用的频率, 我将它们分为**3类调试能力**:  
1. 常规能力.  
2. 升阶能力.  
3. 神级能力.  
前两种表示常用的技能, 而"神级"则表示少用, 难用, 应用面更具体的技能, 一般了解"神级"之前, 前两级应已掌握, 彼时已神.

我也不列出所有条目, 而是按照上面的分类大致地概括. 不过TOC本身就是难得的信息, 后续遇到疑问时完全值得再根据TOC参考本书. 本书每个话题的结尾还列出了参考文献, 也是极佳的资料.

---

调试的思路和心得是要自己摸索和总结的. 动手之前, 先弄清楚问题场景. 再考虑如何重现, 定位和解决. 这应是通用的基本思路了.  
定位问题时, 我们会做"合理的"猜测, 并做一些验证. 最给力的也被用的最多的就是printf了.

#保护现场

**core dump**  
对于用户进程, 可以设置[内核转储, 即core dump](http://en.wikipedia.org/wiki/Core_dump)(*将发生问题时进程的内存信息记录下来*), 有了core文件, 就可以马上调试, 查看出错时堆栈的backtrace等. 大多数Linux发行版默认关闭了core dump功能, 可以通过`ulimit -c unlimited`(*表示不限制core文件大小*)开启, 当错误发生时, 就会自动在当前目录下生成core文件. 当然core文件也可以被配置到专有目录下生成, 通过[修改/etc/sysctl.conf](http://stackoverflow.com/questions/2065912/core-dumped-but-core-file-is-not-in-current-directory)可达此目的.

core dump还有高级用法, 分别是 (a)开启整个系统的core dump 和(b)利用coredump_filter掩码排除进程的共享内存dump, 以防止core文件过大. 这两个用法不细述.

**通过网络获取内核信息**  
kernel故障时, 故障信息可能在重启后消息, 本地也极可能不能登录, 此时如何获取到kernel info? 答案是可以通过网络, 比如通过netconsole可以让printk信息通过UDP发送到远程主机. 比如通过[串口](http://xanpeng.github.com/2012/04/06/linux-serial-port/)获取kernel crash info. 有多个程序可利用串口收集crash info, 比如[ipmitool](http://xanpeng.github.com/2012/04/06/linux-serial-port/). 细节不表.

**IPMI watchdog timer和NMI watchdog**  
我对IPMI watchdog timer没有了解, 只是觉得这二者应是同一方面的, 便列在一起.  
但对于NMI watchdog, 我却刚好用过. NMI意为Non Maskable Interrupt, 是不可屏蔽中断的意思, 主要用来检测系统死机, 通常定时器每秒钟产生一定次数的时钟中断(可配置), 但是如果在禁止中断的情况下陷入死循环或者死锁, 定时器处理就无法执行. 但这种情况下, NMI仍可工作, CPU仍能执行NMI处理程序, NMI处理程序就能通过监视定时器中断是否被执行, 如果超过一定时间没有执行, 就认为发生了死机.  

近期我刚好发现了一个内核死锁, 碰巧事先设置了kbox(??), kbox就是使用了NMI, 死锁时NMI汇报了kernel trace, 根据trace信息, 我很快定位到了问题, 原来是某文件系统在某次spinlock时没有关中断.  
因而这个事件之后, 我深刻认识到了运行一个kbox这样的侦测器的重要性.

**kdump**(*升阶能力*)  
用以获取内核崩溃转储. kdump是Linux-2.6.13之后合并进入主线的内核崩溃转储功能, 是非常有用的一个工具, 对它我现在十分陌生, 准备单开一篇讲述.

**SysRq**(*神级能力*)  
系统不能响应时, 可以尝试SysRq键, 因为SysRq利用了中断, 不过如果出错代码禁止了中断(*比如spin_lock_irqsave()*), 那就没辙了, 只能使用NMI watchdog. SysRq的使用参考[这篇文档](http://www.thegeekstuff.com/2008/12/safe-reboot-of-linux-using-magic-sysrq-key/), 感觉平时用的非常少.

**内核独有的汇编指令ud2, sti, cli等**(*仙级能力*)  
我是凡人来的嘛.

---

#分析

我们想尽办法保护程序出错的现场, 尽量收集更多的数据, 其目的就是为分析工作提供更多更全的资料.  
分析和解决问题是调试的目的, 这绝不是一个轻松的工作, 它需要你有广泛的知识面, 并且至少能够理解每个知识点的关键. 你要是使用编程语言的高手, 是算法达人, 懂得内核 -- 总之, 这是一个颇有挑战和成就感的事情!  
同时, 它需要经验, 需要敏感度. 但这两点的前提就是知识面和理解深度.  

以上是关键和前提. 本部分要讲的则偏重于工具的使用, 下面开始.

**GDB**  
GNU Debugger, Linux开发者必须熟悉的一个工具. 它支持断点, 条件断点, 单步调试, backtrace等多种功能. GDB本质上是依赖于ptrace()系统调用和INT 3软中断的, 我在"[gdb 原理](http://xanpeng.github.com/2012/05/06/gdb/)"和"[how debuggers work](http://xanpeng.github.com/2012/06/30/how-debuggers-work/)"都已说到. 不过GDB功能强大, 远没有使用ptrace()的玩具程序那么简单. 我会单开一篇详细整理GDB的使用. 不过要先提一点: 一定要敢想, 你能想到的, 只要不太飘渺, GDB多已实现了.

**GDB backtrace**  
单列于此, 只为了强调它是多么的重要. 细节会在GDB相关的POST中说明.

**MALLOC_CHECK_**  
glibc中有个方便的调试标志MALLOC_CHECK_, 可以利用环境变量进行调试, 探测内存相关库函数(malloc, free等)的错误使用引发的bug, 如内存双重释放等. `man malloc` 可以看到这个调试标志的更多信息. 使用方法是 `env MALLOC_CHECK_=1 /path/to/prog`.

**Valgrind**(*升阶能力*)  
说到内存方面的问题, 当然离不开Valgrind. 久闻其名, 但还没有使用过. 可能会单开一篇去细述.

**内核调试**  
这不是一个简单的话题. 这方面我的经验非常有限, 我不知道有什么便利的方法. 目前我的做法是根据kernel panic时的串口日志, **查看源码+printk**搞定, 当然是对于简单的问题, 复杂的问题需要升阶的能力和经验. 比如下面使用的crash工具.  
但我现在略犹豫是否要深入内核, 所以暂缓继续了解这块. -- *但实际上, 深入下去也无非如彼...没有什么超难的东西, 只是细节而已, 资料+问询可解决. 可以看我的吐槽文: "[用Doxygen+Graphviz生成函数调用图](http://xanpeng.github.com/2012/06/14/doxygen-graphviz/)"*

**内核调试之kprobes**  
上面说到"查看源码+printk"是良方, 但printk往往需要被多次调整, 从而引起多次重编译, 很不方便. [kprobes](http://lwn.net/Articles/132196/)把printk的工作放到内核模块, 提供了很多便利.  
细节在我有使用经验时会单开细述, 此处略谈其原理. kprobes支持在目标点前后"插入"指令, 获得目标点的地址或symbols和我的博文"[Linux kernel symbols](http://xanpeng.github.com/2012/05/29/linux-kernel-symbols/)"相关(*畅快, 理解无阻碍啊, 虽不保证细节绝对正确*), 而"插入"指令我觉得也不是真正地修改可执行文件, 而是通过中断使得CPU选择执行kprobes代码吧, 这个纯属猜测了. 或者Linux本身就考虑了这种需求, 在现有实现中已有类似于pre-handler, post-handler之类的钩子吧?

**内核调试之systemtap**  
systemtap是kprobes创建的工具, 于是其他就暂不用细说了.

**内核调试之[crash](http://people.redhat.com/anderson/)**(*升阶能力*)  
`whatis crash` - "Analyze Linux crash dump data or a live system".  
如上所述, 我暂不花很多时间了解这个话题, 略过.

**/proc/meminfo, /proc/pid/mem**(*升级能力*)  
可以从这两个地方快速获取进程的内存内容.



