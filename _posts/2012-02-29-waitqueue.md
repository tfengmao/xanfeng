---
title: 等待队列waitqueue
layout: post
tags: linux waitqueue schedule kernel
category: linux
---

##什么是waitqueue

> Wait queues are used to enable processes to wait for a particular event to occur without the need for constant **polling**. Processes sleep during wait time and are woken up automatically by the kernel when the event takes place. 

##数据结构

{% highlight c %}
struct __wait_queue_head {
    spinlock_t lock;
    struct list_head task_list;
};
typedef struct __wait_queue_head wait_queue_head_t;

typedef struct __wait_queue wait_queue_t;
typedef int (*wait_queue_func_t)(wait_queue_t *wait, unsigned mode, int flags, void *key);
struct __wait_queue {
    unsigned int flags; // WQ_FLAG_EXCLUSIVE: be woken up exclusively.
#define WQ_FLAG_EXCLUSIVE	0x01
    void *private;
    wait_queue_func_t func;
    struct list_head task_list;
};
{% endhighlight %}

可以通过add_wait_queue将一个wait_queue_t加到wait_queue_head_t中（实际是链表操作而已）。  
wait_queue_t.private关联进程，我们一般使用时，关联的多是当前进程current。  
使用完之后，需要通过某种方式将wait_queue从wait_queue_head_t中移出。  

##使用和示例

使用waitqueue:  
1、调用wait_event之类的函数将当前进程加入waitqueue，使其睡眠，并将控制权交给scheduler。  
2、在其他地方调用wake_up之类的函数唤醒在waitqueue中睡眠的进程。需注意一点：  
> When process are put to sleep using **wait_event**, you must always ensure that there is a corresponding **wake_up** call at another point in the kernel.

{% highlight c %}
#include <linux/wait.h>
#include <linux/module.h>
#include <linux/init.h>
#include <linux/sched.h>
#include <linux/time.h>

static DECLARE_WAIT_QUEUE_HEAD(exampleq);

static void wait_func(void)
{
    if (! wait_event_timeout(exampleq, 0, msecs_to_jiffies(10000)))
    {
        printk("exampleq: woken up by timeout or wake_up()\n");
    }
}

static int __init waitq_test_init(void)
{
    printk("before wait_func()\n");
    wait_func();

    printk("after wait_func() and before wake_up()\n");
    wake_up(&exampleq);

    printk("after wake_up()\n");
    return 0;
}

static void __exit waitq_test_exit(void)
{
}

MODULE_LICENSE("GPL");
module_init(waitq_test_init);
module_exit(waitq_test_exit);

/*******
before wait_func()
exampleq: woken up by timeout or wake_up()
after wait_func() and before wake_up()
after wake_up()
*******/
{% endhighlight %}

##FAQ

Q：waitqueue中的进程被唤醒后，是否会被移出waitqueue？  
A：使用DEFINE_WAIT定义的waitqueue entry，其wake_func_queue_t为autoremove_wake_function, 该函数wakeup进程之后，会将进程从队列中移除。  
{% highlight c %}
int autoremove_wake_function(wait_queue_t *wait, unsigned mode, int sync, void *key)
{
    int ret = default_wake_function(wait, mode, sync, key);

    if (ret)
        list_del_init(&wait->task_list);
    return ret;
}
{% endhighlight %}

##参考资料
1、professional linux kernel architecture, 14.4 "wait queues and completions".  