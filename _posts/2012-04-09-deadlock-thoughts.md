---
title: 一些 deadlock 相关的想法
layout: post
tags: deadlock lock thoughts
category: coding
---

缘起：[a conversation on DLM lock levels used in X](http://blog.coly.li/?p=81) 中 mark 提到
> if you think about it – being forced to drop the lock and re-acquire would eliminate the possibility of deadlock, at the expense of performance

不是很解，来理理。

加锁(lock)的目的是控制对资源(res)的“有序”访问。res 要么是有限的，不能人手一份，如大量内存；要么某些 res 就必须拿来共享，如全局变量。

假设对于 resA，保证同一时刻只能有一个节点持有 resA 的锁。那么如果 N1 加锁成功，N2 就不能加锁，N2 需要等待 N1 释放锁。如果同时有 N2、N3、N4 等着 N1 释放，此时让 N2、N3、N4 排队，比如以 FIFO 的顺序去加锁。  
这种场景下，貌似永远不会发生死锁。

假设有 resA、resB，同样地，对于每个资源，保证同一时刻只能有一个节点持有它的锁。  
如果 resA 和 resB 之间木有任何关系，那么和上面一样，貌似永远不会发生死锁。  
如果 resA 和 resB 之间有关系，设 N1 持有 resA，并等待着去持有 resB，同时 N2 持有 resB，并等待着去持有 resA。于是悲剧了，貌似出现了经典的死锁场景。

此时似乎可以理解 Mark 说的话了，他描述的应是对于单个 res 来说的。如果有多个相互依赖的 res，对单个 res 的限制貌似完全无用，该怎么死还是会怎么死。需要 coder 去仔细地控制 res 之间的依赖了。

不过，为什么是“at the expense of performance”，我就不解了。难道还有其他方式可以有更好的 performance，只不过会带来 deadlock 的风险？

上面说的都是自己的想象，尚没有理论支持...
