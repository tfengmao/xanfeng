---
title: 内核线程
layout: post
tags: linux kernel thread
category: linux
---

###example<sup>[kt]</sup>

<script src="https://gist.github.com/1946789.js"> </script>

上面的示例代码在执行后，通过`ps | grep`可以看到如下结果：  
    # insmod ./kthread.ko
    # ps aux | grep kthread         <-- 上面示例中创建线程的名字为"kthread"
    root         2  0.0  0.0      0     0 ?        S    Feb29   0:00 [kthreadd]
    root      1735  0.0  0.0      0     0 ?        D    10:49   0:00 [kthread]
    # rmmod kthread
    # ps aux | grep kthread
    root         2  0.0  0.0      0     0 ?        S    Feb29   0:00 [kthreadd]

**kthreadd**是一个，kernel thread daemon，是所有其他kernel thread的父亲<sup>[so](http://stackoverflow.com/a/4167208/264035)</sup>.  
<del>*这个[讨论][sx]也认为kernel thread和kernel process差别不大，可以基于此继续讨论该问题。*</del>

[kt]: http://kerneltrap.org/node/20903 "example code"
[sx]: http://unix.stackexchange.com/questions/31595/are-kernel-threads-really-kernel-processes "Are kernel threads really kernel processes?"
