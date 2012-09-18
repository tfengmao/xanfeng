---
title: linux 工作队列(workqueue)
layout: post
tags: linux workqueue kernel
category: linux
---

在处理内核相关工作中, 我们经常看到工作队列(workqueue)的身影. 本文描述何为 linux workqueue.  
本文基于 2.6.32 的内核, 此时的工作队列还不是 [cmwq](http://www.mjmwired.net/kernel/Documentation/workqueue.txt).

#为什么使用 workqueue

在内核代码中, 经常希望延缓部分工作到将来某个时间执行, 这样做的原因很多, 比如

* 在持有锁时做大量(或者说费时的)工作不合适.
* 希望将工作聚集以获取批处理的性能.
* 调用了一个可能导致睡眠的函数使得在此时执行新调度非常不合适.
* ...

内核中提供了许多机制来提供延迟执行, 使用最多则是 workqueue.

* 如中断的下半部处理可延迟中断上下文中的部分工作;
* 定时器可指定延迟一定时间后执行某工作; 
* workqueue 则允许在进程上下文环境下延迟执行;
* 内核中曾短暂出现过的慢工作机制 (slow work mechanism); 
* 异步函数调用(asynchronous function calls);
* 各种私有实现的线程池;
* ...

#术语

* workqueue: 所有工作项(需要被执行的工作)被排列于该队列.
* worker thread: 是一个用于执行 workqueue 中各个工作项的内核线程, 当 workqueue 中没有工作项时, 该线程将变为 idle 状态.
* single threaded(ST): worker thread 的表现形式之一, 在系统范围内, 只有一个 worker thread 为 workqueue 服务.
* multi threaded(MT): worker thread 的表现形式之一, 在多 CPU 系统上每个 CPU 上都有一个 worker thread 为 workqueue 服务.

#使用步骤

1. 创建 workqueue(如果使用内核默认的 workqueue, 此步骤略过).
2. 创建工作项 work_struct.
3. 向 workqueue 提交工作项. 


#工作项  

数据结构(略有调整):
    struct work_struct {
        atomic_long_t data;
        struct list_head entry;
        work_func_t func;
        struct lockdep_map lockdep_map;
    };
    struct delayed_work {
        struct work_struct work;
        struct timer_list timer;
    };
静态地创建工作项:
    DECLARE_WORK(n, f)
    DECLARE_DELAYED_WORK(n, f)
动态地创建工作项:
    INIT_WORK(struct work_struct work, work_func_t func); 
    PREPARE_WORK(struct work_struct work, work_func_t func); 
    INIT_DELAYED_WORK(struct delayed_work work, work_func_t func); 
    PREPARE_DELAYED_WORK(struct delayed_work work, work_func_t func); 

#内核默认的 workqueue
查阅源码可知, 内核默认的全局 workqueue 应为:
    // 定义
    static struct workqueue_struct *keventd_wq __read_mostly;
    
    // 初始化
    ...
    keventd_wq = create_workqueue("events"); // MT worker thread 模式.
    ...

    // 确认对应的内核线程数目, 应等于 CPU 核数
    $ lscpu
    Architecture:          x86_64
    CPU(s):                3
    Thread(s) per core:    1
    $ ps aux | grep "events"
    root         9  0.0  0.0      0     0 ?        S    Feb09   1:15 [events/0]
    root        10  0.0  0.0      0     0 ?        S    Feb09   0:59 [events/1]
    root        11  0.0  0.0      0     0 ?        S    Feb09   0:59 [events/2]
工作项加入 keventd_wq 由下面两个函数之一完成:
    int schedule_work(struct work_struct *work)
    {
	    return queue_work(keventd_wq, work);
    }
    int schedule_delayed_work(struct delayed_work *dwork, unsigned long delay)
    {
        return queue_delayed_work(keventd_wq, dwork, delay);
    }

#用户自定义的 workqueue
创建 workqueue:
    create_singlethread_workqueue(name) // 仅对应一个内核线程
    create_workqueue(name) // 对应多个内核线程, 同上文.
向 workqueue 中提交工作项:
    int queue_work(workqueue_t *queue, work_t *work); 
    int queue_delayed_work(workqueue_t *queue, work_t *work, unsigned long delay); 
取消 workqueue 中挂起的工作项以及释放 workqueue 此处略过.

#工作队列的优点

* 使用简单.
* 执行在进程上下文, 从而可以睡眠, 被调度和抢占.
* 在多核环境下使用也非常友好.

#example
<script src="https://gist.github.com/1938890.js"> </script>
Makefile:
    obj-m := wq.o
    default:
        make -C /usr/src/linux-2.6.32.12-0.7 M=`pwd` modules


#本文主要参考

* [IBM developworks Linux 的并发可管理工作队列机制探讨](http://www.ibm.com/developerworks/cn/linux/l-cn-cncrrc-mngd-wkq/index.html?ca=drs-)
