---
title: Linux Process Group & Threads Group
layout: post
tags: process thread group linux session
category: linux
---

不知为何, 网上讨论 Process Group 和 Threads Group 的都很少.  
[5-25] 经过更多的搜寻, 仍旧没有找到单独完整地讲述 Process Group 和 Threads Group 的. 我不由得想, kernel 是否仅仅是在 task_struct 数据结构上支持这么两个概念而已, 并没有针对于 process/threads group 改变原有的机理, 如进程调度? 因此, 此处暂停这块的了解.

# process group

在"Linux System Programming" 5.6 "Sessions and Process Groups" 中有提到 Process Group:  

+ process group is a collection of one or more processes generally associated with each other for the purposes of *job control*.  
+ primary attribute of a process group is that *signals* may be sent to all processes in the group.  
+ each process is a member of a process group.  
+ process group is identified by a process group ID(*pgid*).  
+ process group has a *leader*.  
+ pgid = leader's pid.  
+ leader 终止时, 如果进程组里仍有进程, 则进程组依然存在. -- 此时, pgid 如何变化? leader 如何变化?  

提到的 sessions 部分:

+ 当用户登入机器时, *login process* 创建一个新的 session, 包含一个进程: user's login shell.  
+ login shell 成为 *session leader*.  
+ *session ID* = session leader's pid.  
+ A session = A collection of 1+ *process groups*.  
+ session arranges a logged-in user's activities, 关联处理用户 terminal IO 的终端(tty device).  
+ process groups in a session are divided into 1 (one single) *forground* process group, and 0+ *background* process groups.  
+ when user exits a terminal, a *SIGQUIT* is sent to all processes in the foreground process group.  
+ on a given system, there are *many* sessions: one for each user login session, and others for processes not tied to user login sessions (如 daemons).  

其实有点跑题了, 不过我一向是把跑题当成是函数调用的...所以继续调用 sessions.

上面说到, 退出 terminal 时, 仅 foreground process group 被发 SIGQUIT, 那么如果我把任务放到后台跑(&), 岂不就可以安然退出 terminal 呢? 这么简单? 和 screen, tmux 有和区别?  
经过实际操作, 后台进程确实不会在退出 terminal 时被 QUIT 的.

既然说到 foreground 和 background, 我们当然有熟悉的 fg/bg, `man fg` 可以知道 "fg [job_id]", 这里的 job_id 不是 pid, 可以通过 `jobs` 查看当前 session 下的 jobs, 似乎如果你强关 terminal 之后, jobs 信息是被清空的, 即使还是有 background processes 在跑.

说到这里, 可以猜想, fg/bg 可能只是设置进程的 sessionid 字段吧?

说到这里, 不妨列出 task_struct 的一些字段(2.6.32.36):  

    pid_t pid;  
    pid_t tgid;   
    struct task_struct *real_parent; /* real parent process */  
    struct task_struct *parent; /* recipient of SIGCHLD, wait4() reports */  
    struct task_struct *group_leader;	/* threadgroup leader */  
    
    /*  
     * ptraced is the list of tasks this task is using ptrace on.  
     * This includes both natural children and PTRACE_ATTACH targets.  
     * p->ptrace_entry is p's link on the p->parent->ptraced list.  
     */  
    struct list_head ptraced;  
    struct list_head ptrace_entry;  

看到这里的 ptraced, 是否觉得熟悉而又多了些感悟呢? 看这里: "[gdb 原理](http://xanpeng.github.com/2012/05/06/gdb/)".

    struct list_head thread_group;  
    struct thread_struct thread;  
    unsigned int sessionid;  


# thread group

在"[Linux processes](http://xanpeng.github.com/2012/06/10/process/)"讨论了线程组, 实际上这是一个简单的概念而已.
