---
title: how debuggers work
layout: post
tags: debugger ptrace interrupt
category: programming
---

我在0506总结了"[gdb 原理](http://xanpeng.github.com/2012/05/06/gdb/)", 提到了gdb本质上是使用ptrace系统调用. 今天看到国外同行写了一个"how debuggers work"系列, 理解深刻, 表达简明, 6星推荐! 从中可以看出对于debugger来说, ptrace和INT 3(*trap to debugger*)软中断居功至伟.

以下是链接:  
[part1 - basics](http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/).  
[part2 - breakpoints](http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints/).  
[part3 - debugging information](http://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information/).

