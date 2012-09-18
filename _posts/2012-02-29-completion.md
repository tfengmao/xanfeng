---
title: linux 完成量(completion)
layout: post
tags: linux waitqueue kernel completion
category: linux
---

#什么是完成量(completions)
完成量在 [wait queue](http://xanpeng.github.com/2012/02/29/waitqueue/) 基础上实现, 是类似于信号量一样用来同步对 critical section 访问的一种机制.

#数据结构
    struct completion {
        unsigned int done;
        wait_queue_head_t wait;
    };

#函数
    wait_for_completion(struct completion *x);
    wait_for_completion_timeout(struct completion *x, unsigned long timeout);
    wait_for_completion_interruptible(struct completion *x);
    wait_for_completion_interruptible_timeout(struct completion *x, unsigned long timeout);

    complete(struct completion *x);
    complete_all(struct completion *x);
    complete_and_exit(struct completion *comp, long code);

#完成量(completion) vs 信号量(semaphore)
> Q: In the linux kernel, semaphores are used to provide mutual exclusion for critical sections of data and Completion variables are used to synchronize between 2 threads waiting on an event. **Why** not use semaphores for such a synchronization? Is there any advantage of using a completion variable over a semaphore?  
> A: There are two reasons you might want to use a completion instead of a semaphore.

>* First, multiple threads can wait for a completion, and they can all be released with one call to complete_all(). It's more complex to have a semaphore wake up an unknown number of threads.
>* Second, if the waiting thread is going to deallocate the synchronization object, there is a race condition if you're using semaphores. That is, the waiter might get woken up and deallocate the object before the waking thread is done with up(). This race doesn't exist for completions. (See Lasse's post.)

>([more](http://stackoverflow.com/a/4765574)...)

#参考资料
* professional linux kernel architecture, 14.4 "wait queues and completions".
* [Difference between completion variables and semaphores](http://stackoverflow.com/questions/4764945/difference-between-completion-variables-and-semaphores)

