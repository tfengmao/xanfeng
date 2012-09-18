---
title: linux 等待队列(wait queue)
layout: post
tags: linux waitqueue schedule kernel
category: linux
---

#什么是 wait queue
> Wait queues are used to enable processes to wait for a particular event to occur without the need for constant **polling**. Processes sleep during wait time and are woken up automatically by the kernel when the event takes place. 

#数据结构
**wait queue 数据结构**  
    struct __wait_queue_head {
        spinlock_t lock;
        struct list_head task_list;
    };
    typedef struct __wait_queue_head wait_queue_head_t;

**wait queue entry 数据结构**  
    typedef struct __wait_queue wait_queue_t;
    typedef int (*wait_queue_func_t)(wait_queue_t *wait, unsigned mode, int flags, void *key);
    struct __wait_queue {
        unsigned int flags; // WQ_FLAG_EXCLUSIVE: be woken up exclusively.
    #define WQ_FLAG_EXCLUSIVE	0x01
        void *private;
        wait_queue_func_t func;
        struct list_head task_list;
    };

#函数
使用 wait queue:

1. 调用 wait_event 之类的函数将当前进程加入 wait queue, 使其睡眠, 并将控制权交给调度器.
2. 在其他地方调用 wake_up 之类的函数唤醒在 wait queue 中睡眠的进程. 需注意一点:
> When process are put to sleep using **wait_event**, you must always ensure that there is a corresponding **wake_up** call at another point in the kernel.

初始化 wait queue 和 wait queue entry:
    #define init_waitqueue_head(q)
    void init_waitqueue_entry(wait_queue_t *q, struct task_struct *p);
    #define DEFINE_WAIT(name) DEFINE_WAIT_FUNC(name, autoremove_wake_function)

**wait_event 家族**  
    #define wait_event(wq, condition)
    void add_wait_queue(wait_queue_head_t *q, wait_queue_t *wait);
    void prepare_to_wait(wait_queue_head_t *q, wait_queue_t *wait, int state);
    void prepare_to_wait_exclusive(wait_queue_head_t *q, wait_queue_t *wait, int state);
    #define wait_event_interruptible(wq, condition) // 睡眠的进程可以被信号唤醒
    #define wait_event_timeout(wq, condition, timeout) // 条件满足, 或超时的时候, 进程被唤醒
    #define wait_event_interruptible_timeout(wq, condition, timeout) // 条件满足/超时/收到信号的时候进程被唤醒

**wake_up 家族**  
    #define wake_up(x)			__wake_up(x, TASK_NORMAL, 1, NULL)
    #define wake_up_nr(x, nr)		__wake_up(x, TASK_NORMAL, nr, NULL)
    #define wake_up_all(x)			__wake_up(x, TASK_NORMAL, 0, NULL)
    #define wake_up_locked(x)		__wake_up_locked((x), TASK_NORMAL)
    #define wake_up_interruptible(x)	__wake_up(x, TASK_INTERRUPTIBLE, 1, NULL)
    #define wake_up_interruptible_nr(x, nr)	__wake_up(x, TASK_INTERRUPTIBLE, nr, NULL)
    #define wake_up_interruptible_all(x)	__wake_up(x, TASK_INTERRUPTIBLE, 0, NULL)
    #define wake_up_interruptible_sync(x)	__wake_up_sync((x), TASK_INTERRUPTIBLE, 1)

#sample code
注意: 该示例代码是**"恶意"**的, 它让你卡在 insmod.
<script src="https://gist.github.com/1940201.js"> </script>

#FAQ
Q: wait queue 中的进程被唤醒后, 是否会被移出 wait queue?  
A: 使用 DEFINE_WAIT 定义的 wait queue entry, 其 wake_func_queue_t 为 autoremove_wake_function, 该函数 wakeup 进程之后, 会将进程从队列中移除.
    int autoremove_wake_function(wait_queue_t *wait, unsigned mode, int sync, void *key)
    {
        int ret = default_wake_function(wait, mode, sync, key);

        if (ret)
            list_del_init(&wait->task_list);
        return ret;
    }

#参考资料
* professional linux kernel architecture, 14.4 "wait queues and completions".

