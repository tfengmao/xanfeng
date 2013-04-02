---
title: 内核工作队列(workqueue)
layout: post
tags: linux workqueue kernel
category: linux
---

本文基于2.6.32的内核，此时的工作队列还不是[cmwq](http://www.mjmwired.net/kernel/Documentation/workqueue.txt)。  

**为什么使用workqueue**  
在内核代码中，经常希望延缓部分工作到将来某个时间执行，这样做的原因很多，比如：  
1、在持有锁时做大量(或者说费时的)工作不合适。  
2、希望将工作聚集以获取批处理的性能。  
3、调用了一个可能导致睡眠的函数使得在此时执行新调度非常不合适。  
4、工作队列的优点：(1)使用简单；(2)执行在进程上下文，从而可以睡眠被调度和抢占；(3)在多核环境下使用也非常友好。  

内核中提供了许多机制来提供延迟执行，使用最多则是workqueue：  
1、如中断的下半部处理可延迟中断上下文中的部分工作；  
2、定时器可指定延迟一定时间后执行某工作；  
3、workqueue 则允许在进程上下文环境下延迟执行；  
4、内核中曾短暂出现过的慢工作机制(slow work mechanism)；  
5、异步函数调用(asynchronous function calls)；  
6、各种私有实现的线程池；  

**术语**  
workqueue：所有工作项（需要被执行的工作）被排列于该队列。  
worker thread：是一个用于执行workqueue中各个工作项的内核线程，当workqueue中没有工作项时，该线程将变为idle状态。  
single threaded(ST)：worker thread的表现形式之一，在系统范围内，只有一个worker thread为workqueue服务。  
multi threaded(MT)：worker thread的表现形式之一，在多CPU系统上每个CPU上都有一个worker thread为workqueue服务。  

**使用步骤**  
1、创建workqueue（如果使用内核默认的workqueue，此步骤略过）；  
2、创建工作项work_struct；  
3、向workqueue提交工作项；  

**工作项**  

{% highlight c %}
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
{% endhighlight %}

1、静态地创建工作项：  
DECLARE_WORK(n, f)  
DECLARE_DELAYED_WORK(n, f)  
2、动态地创建工作项：  
INIT_WORK(struct work_struct work, work_func_t func);   
PREPARE_WORK(struct work_struct work, work_func_t func);   
INIT_DELAYED_WORK(struct delayed_work work, work_func_t func);   
PREPARE_DELAYED_WORK(struct delayed_work work, work_func_t func);   

**内核默认的workqueue**  

{% highlight text %}
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

// 工作项加入 keventd_wq 由下面两个函数之一完成:
int schedule_work(struct work_struct *work)
{
	return queue_work(keventd_wq, work);
}
int schedule_delayed_work(struct delayed_work *dwork, unsigned long delay)
{
	return queue_delayed_work(keventd_wq, dwork, delay);
}
{% endhighlight %}

**用户自定义的workqueue**  
1、创建workqueue：  
create_singlethread_workqueue(name) // 仅对应一个内核线程  
create_workqueue(name) // 对应多个内核线程, 同上文.  
2、向workqueue中提交工作项：  
int queue_work(workqueue_t *queue, work_t *work);   
int queue_delayed_work(workqueue_t *queue, work_t *work, unsigned long delay);   

**example**  
<script src="https://gist.github.com/1938890.js"> </script>

**资料**   
[IBM developworks Linux 的并发可管理工作队列机制探讨](http://www.ibm.com/developerworks/cn/linux/l-cn-cncrrc-mngd-wkq/index.html?ca=drs-)
