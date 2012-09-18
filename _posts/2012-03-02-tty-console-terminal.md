---
title: tty, pts, console & terminal
layout: post
tags: tty console terminal pts linux
category: linux
---

你们都用 [PuTTY](http://en.wikipedia.org/wiki/PuTTY) 连接一台公共服务器, 你输入 **w**/**who** 命令, 得到:  
    $ w  
    USER     TTY        LOGIN@   IDLE   JCPU   PCPU WHAT  
    root     pts/0     13:38    0.00s  0.04s  0.00s w  
    root     pts/1     Wed19   42:12m  0.03s  0.03s -bash  
    $ who  
    root     pts/0        Mar  2 13:38 (xxx.xxx.xxx.xxx)  
    root     pts/1        Feb 29 19:53 (xxx.xxx.xxx.xxx)  

这里的 pts/X 是什么意思?  
甚至, putty 是如何连接 Server 的?  
更甚至, [screen 程序](http://en.wikipedia.org/wiki/GNU_Screen)又是怎么一回事?

了解更多之后, 我们发现 tty[[1]](http://en.wikipedia.org/wiki/Tty_\(Unix\))[[2]](http://en.wikipedia.org/wiki/Teleprinter), 这又是什么概念?  
执行下面的指令:  
    linux-hvsm:~ # ps a | grep tty  
    6637 tty1     Ss+    0:00 /sbin/mingetty --noclear tty1  
    6638 tty2     Ss+    0:00 /sbin/mingetty tty2  
    6639 tty3     Ss+    0:00 /sbin/mingetty tty3  
    6640 tty4     Ss+    0:00 /sbin/mingetty tty4  
    6641 tty5     Ss+    0:00 /sbin/mingetty tty5  
    6642 tty6     Ss+    0:00 /sbin/mingetty tty6  

这里的 /sbin/[mingetty](http://linux.die.net/man/8/mingetty), ttyX 又是什么?

tty, pts, terminal, console 这些相关的概念是不是让人无比纠结?  
幸好 unix.stackexchange 上有人讨论了这些概念: [讨论1](http://unix.stackexchange.com/questions/33155/why-there-are-six-getty-processes-running-on-my-desktop), [讨论2](http://unix.stackexchange.com/questions/4126/what-is-the-exact-difference-between-a-terminal-a-shell-a-tty-and-a-con).  
本文参考这些讨论, 做出利己理解的总结.

#答案
In unix terminology, short answer:  
> terminal = tty = text input/output environment  
> console = physical terminal  
> shell = command line interpreter  

In unix terminology, long answer:  

* a **tty** is a particular kind of device file which implements a number of additional commands (ioctls) beyond read and write.
    * Some ttys are provided by the kernel on behalf of a hardware device, 如输入设备(键盘), 输出设备(字符模式屏幕).
    * Other ttys, sometimes called **pseudo-ttys**, are provided (through a thin kernel layer) by programs called **[terminal emulators](http://en.wikipedia.org/wiki/Terminal_emulator)**, 如 [Xterm](http://en.wikipedia.org/wiki/Xterm), [Screen](http://en.wikipedia.org/wiki/Gnu_screen), [Secure Shell](http://en.wikipedia.org/wiki/Secure_shell)等.
* **terminal**, in its most common meaning, is synonymous with tty. 传统意义上, terminal 一般被认为是和计算机交互的一种方式, 即包含键盘和显示器的操作界面.
* shell: 本文不说明, 我能接受 "shell 就是 OS 的前台, 是用户和 OS 交互的界面" 这样的理解.
* console: is generally a terminal in the *physical sense* that is by some definition the primary terminal *directly connected* to a machine. The console *appears to* the operating system as a (kernel-implemented) tty. On some systems, such as Linux and FreeBSD, the console appears as several ttys (special key combinations switch between these ttys) - 这里就是前面提到的多个 ttyX 的问题了.

有人问了一个相关的问题: ["Why is a virtual terminal “virtual”, and what/why/where is the “real” terminal?"](http://askubuntu.com/questions/14284/why-is-a-virtual-terminal-virtual-and-what-why-where-is-the-real-terminal), 可以从中了解更多.

#回到我自己的问题
我在 Windows OS 下使用 [putty](http://en.wikipedia.org/wiki/PuTTY) 连接到 Linux server.  
==>  
我使用 putty 软件模拟的终端和 Server 交互. 我执行 w/who, 发现有很多 pts/X, 这说明有多个终端正连接到 server. file /dev/pts/X 可以看出, 这是字符设备文件. 我同时打开两个 putty 窗口连接同一 server, 再 w/who 一下, 根据 ip 地址发现我使用了两个 pts/X, 这说明我建立了两个连接, 就好像我用两套"键盘+显示器"操作同一主机一样.

我的笔记本运行的是 Ubuntu 11.10 desktop environment, 其中有个 [Terminal 程序](https://help.ubuntu.com/community/UsingTheTerminal), 我执行该程序, 发现这是一个 command-line 的操作界面.  
这个程序是 "real" terminal, 而不是像 putty 连接那样只是 emulated terminal?  
遗憾的是, 这个程序仍不是 "real" terminal, 不过它和 putty 还是有巨大差别的, putty 穿越网络来访问主机, 而这个程序乃是 "原住民". 我想 "real terminal" 应该是[这个讨论](http://askubuntu.com/questions/14284/why-is-a-virtual-terminal-virtual-and-what-why-where-is-the-real-terminal)中图示的东西吧:

![alt "real" terminal](http://upload.wikimedia.org/wikipedia/commons/thumb/7/76/ASR-33_1.jpg/220px-ASR-33_1.jpg)

那么前面提到的 ttyX, mingetty 进程又是什么呢?  
Chris Down 回答了这个问题了[[link]](http://unix.stackexchange.com/a/33156/12224), 他说每个 getty 进程对应的是一个 [virtual console](https://en.wikipedia.org/wiki/Virtual_console),  
> Usually in Linux (see Linux console), the **first six virtual consoles** provide a text terminal with a login prompt to a Unix shell. The graphical X Window System starts in the seventh virtual console. In Linux, the user switches between them with the key combination Alt plus a function key – for example Alt+F1 to access the virtual console number 1. Alt+Left arrow changes to the previous virtual console and Alt+Right arrow to the next virtual console.  

> *The need* for virtual consoles has lessened now that most applications work in the graphical framework of the X Window System, where each program has a window and **the text mode programs can be run in terminal-emulator windows**(这有利于理解 desktop env 下的 Terminal 程序). If several sessions of the X Window System are required to run in parallel, such as in the case of fast user switching or when debugging X programs on a separate X server, each X session usually *runs in a separate virtual console*.  

前 6 个 virtual console 用于提供 text terminal, 第 7 个开始就是 graphical terminal 了. 但是这么多 virtual console 有什么用? 或许 linux 就要默认提供这么多, 如此那么我可以从 VC1 切换到 VC4 去么, 就像 Ubuntu desktop 里[切换工作区](http://www.addictivetips.com/ubuntu-linux-tips/quickly-switch-workspaces-in-ubuntu-unity-with-indicator-workspaces/)一样.  
[这个讨论](http://superuser.com/questions/311256/sending-change-virtual-console-command-in-putty)说, 无法通过 ssh 切换 VC, 不过可以试试 Screen.  

[chvt](http://linux.die.net/man/1/chvt) 命令可用来代替这些组合键, 还有[更多命令](http://en.wikipedia.org/wiki/Virtual_console#Interface).  
    # openvt -v -f -c 2 -- tail -f /var/log/messages
    openvt: using VT /dev/tty2
    # ps aux | grep tty
    root      6637  0.0  0.0   4344   744 tty1     Ss+  Feb29   0:00 /sbin/mingetty --noclear tty1
    root      6639  0.0  0.0   4344   716 tty3     Ss+  Feb29   0:00 /sbin/mingetty tty3
    root      6640  0.0  0.0   4344   716 tty4     Ss+  Feb29   0:00 /sbin/mingetty tty4
    root      6641  0.0  0.0   4344   716 tty5     Ss+  Feb29   0:00 /sbin/mingetty tty5
    root      6642  0.0  0.0   4344   716 tty6     Ss+  Feb29   0:00 /sbin/mingetty tty6
    root     15218  0.0  0.0   8400   732 tty2     Ss+  16:26   0:00 tail -f /var/log/messages
    root     15228  0.0  0.0   4344   768 pts/0    R+   16:27   0:00 grep tty   
    # fgconsole
    2

发现可以通过 openvt, chvt, deallocvt 这一系列命令操作 ttyX, 至此对 ttyX 至少有了一个直观的认识了. 通过 [**fgconsole**](http://stackoverflow.com/questions/3034567/how-do-i-find-the-current-virtual-terminal) 命令我知道当前位于 tty2. 但在上面的操作中, 我不能得到 tail messages 的效果, 有点郁闷.  

了解了这些之后, 理解 session 的概念就更加得心应手了.[这里](http://hi.baidu.com/_kouu/blog/item/fd187c30623877b75edf0ed2.html)有篇文章对 linux session 做了细致的描述, 值得一看.
