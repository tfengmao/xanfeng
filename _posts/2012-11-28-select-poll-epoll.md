---
title: IO多路复用(select,poll,epoll)
layout: post
category: coding
tags: select pselect poll epoll level-triggerd edge-triggered
---

暂且不论IO多路复用（IO multiplexing），来看select、poll、epoll的使用场景和区别。  

###select

如果程序需要读某个fd，调用read()，这个函数行为默认是blocking的，如果fd还没有数据，进程会卡在read()处。（*这里不考虑O_NONBLOCK*）  
如果程序需要读写多个fd，怎么办？我们往往会发现，进程blocking在某个read(fdx)时，即使后续处理的fdy available，也暂得不到处理——这是浪费。  
这就是为什么需要`select`（`man 2 select_tut`）。  

select约定了三个fd集合：readfds，writefds，exceptfds。用户程序根据读写需要，在对应的集合里面设定需要监听的fd，然后通过select进入内核。  

内核的工作主要在`do_select()@fs/select.c`，其逻辑很简单：  
1、按序（fd从小到大）判断fd是否available，如是则retval++；  
2、如果retval==0，则通过poll_schedule_timeout(timeout+TASK_INTERRUPTIBLE)睡眠，timeout是用户程序传入的；  
3、如果retval>0，表示三个集合中至少有一个fd available，返回retval；  
4、如果有睡眠，醒后会再遍历一次，如果仍没有fd available，返回0；  
另外如果select遇到异常，比如分配不到内存、fd无效、被信号中断（参考[linux异步信号handle浅析](http://xanpeng.github.com/coding/2012/11/24/kouu-posts.html)），返回-1，并设置errno。  

如果select()返回正值，用户程序需要再遍历一次，操作available的fd。  
exceptfds是比较特别的，一般认为它被用来处理异常的fd，但实际上多用来监测socket上的out-of-band(OOB)数据（`man select_tut`）。  

来看看怎么使用select()。一般的示例都是socket相关的，这里看一个“**错误的**”select示例——monitor regular file（file-based fd）总是成功(available)，因为对于regular file：  
1、文件不同于流，文件内容在那里，总是ready的，监听它[没有意义](http://news-posts.aplawrence.com/662.html)。“一个文件已经/还没有ready for writing”的[说法是奇怪的](http://www.groupsrv.com/linux/about159067.html)，同理“ready for reading”一样奇怪。[内核也不好做这个语义上的控制](https://github.com/xanpeng/kernel-misc/blob/master/nerver-read-write-file-in-kernel.md)。  
2、另一方面，从内核代码可以看出，socket在数据available时，会设置POLLIN等标志——这是select的判断依据（通过关键字搜索判断）。<del>而对常规文件，是没有也不方便设置这样的标志的</del>（对于普通文件，还不知道select是怎么瞬间返回的）。  
{% highlight c %}
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>
#include <pthread.h>

void *gendata(void *arg)
{
    FILE *fp = (FILE *)arg;
    char *data = "tmpdata";
    int len = strlen(data);
    int i;
    for (i = 0; i < 10; ++i) {
        fprintf(stderr, "fwrite %d\n", fwrite(data, 1, len, fp));
        sleep(1);
    }
}

int main(void)
{
    fd_set rfds;
    struct timeval tv;
    int retval;
    int maxfd;

    FD_ZERO(&rfds);
    FD_SET(0, &rfds);
    maxfd = 0;
	
	/*
    // 开个线程写临时文件，select立即返回，来不及监听stdin
    FILE *fp = tmpfile();
    if (!fp) { perror("tmpfile()"); exit(EXIT_FAILURE); }
    int fpd = fileno(fp);

    pthread_t thread;
    int ret = pthread_create(&thread, NULL, gendata, (void*) fp);
    if (ret) { perror("pthread_create()"); exit(EXIT_FAILURE); }
    */

    // 不通过线程，监听一个没人操作的普通文件，也不去操作它，select仍然立即返回，来不及监听stdin
    FILE *fp = fopen("/home/xan/lab/select.file", "r");
    if (!fp) { perror("fopen()"); exit(EXIT_FAILURE); }
    int fpd = fileno(fp);

    FD_SET(fpd, &rfds);
    maxfd = fpd;
    fprintf(stderr, "maxfd: %d\n", maxfd);

    tv.tv_sec = 5;
    tv.tv_usec = 0;
    retval = select(maxfd+1, &rfds, NULL, NULL, &tv);
    if (retval == -1) perror("select()");
    else if (retval) {
        int i, bytes_read;
        char buf[64];
        for (i = 0; i <= maxfd; ++i) {
            if (FD_ISSET(i, &rfds)) {
                bytes_read = read(i, buf, sizeof(buf));
                if (bytes_read) printf("\nread from [%d]: %s\n", i, buf);
            }
        }
    }
    else printf("No data within five seconds.\n");

    exit(EXIT_SUCCESS);
}
{% endhighlight %}

###non-blocking regular file无意义

延续上一节的内容，继续讨论常规文件，以O_NONBLOCK的方式操作常规文件是没有意义的：  
1、[Can regular file reading benefited from nonblocking-IO?](http://stackoverflow.com/questions/5613354/can-regular-file-reading-benefited-from-nonblocking-io)  
2、[Non-blocking I/O with regular files](http://www.remlab.net/op/nonblock.shtml)  

O_NONBLOCK不是异步IO，blocking、non-blocking的效果是一样的。摘引第二个链接的内容：  
> Regular files are **always** readable and they are also **always** writeable. This is clearly stated in the relevant POSIX specifications. **I cannot stress this enough. Putting a regular file in non-blocking has ABSOLUTELY no effects** other than changing one bit in the file flags.  
>  
> Reading from a regular file might take a long time. For instance, if it is located on a busy disk, the I/O scheduler might take so much time that the user will notice the application is frozen.  
>   
> Nevertheless, non-blocking mode will not work. It simply will not work. Checking a file for readability or writeability always succeeds immediately. If the system needs time to perform the I/O operation, it will put the task in non-interruptible sleep from the read or write system call.

###pselect

pselect类似于select，二者的重要区别是对信号量的处理。但其中的细节十分微妙，不好理解。`man select_tut` “Combining Signal and Data Events”部分的解释也十分模糊，对理解有弊无利。  
先来解释这里的问题到底是什么。参考“[The new pselect() system call](http://lwn.net/Articles/176911/)”，在需要信号处理时，程序往往是这么写的：  
{% highlight c %}
static void handler(int sig) { /* do nothing */  }
    
int main(int argc, char *argv[])
{
    fd_set readfds;
    struct sigaction sa;
    int nfds, ready;

    sa.sa_handler = handler;     /* Establish signal handler */
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    sigaction(SIGINT, &sa, NULL);
	/* ... */    
    ready = select(nfds, &readfds, NULL, NULL, NULL);
	/* ... */
{% endhighlight %}

正常情况下，如果select调用中收到信号，比如SIGINT时，select会中断执行、返回-1并设置error为EINTR。  
但是这段代码存在一个“race condition”：如果SIGINT在select调用之前（更细致地，进入内核之前？）发生，那么其后执行的select()将**不能被中断**，假如没有设置timeout，则进程就会永远被block在select()处。  

你定然觉得奇怪：为什么这个情况下的select()不能被中断？！原因是：  
1、信号处理函数是在用户态定义的，但却是被内核触发调用的。有两处资料证明：1）“[linux异步信号handle浅析](http://xanpeng.github.com/coding/2012/11/24/kouu-posts.html)”；2）ULK3第十一章“传递信号”，  
> 我们在第四章“从中断和异常返回”一节中提到，内核在允许进程恢复用户态下的执行之前，检查进程TIF_SIGPENDING标志的值。每当内核处理完一个中断或异常时，就检查是否存在挂起信号...  

2、信号分常规信号(regular signal，1～31，SIGINT是2)和实时信号(real-time signal，32～64)。(在内核中)同种常规信号不排队，如果一个常规信号被发送多次，只有其中一次被发送到接收进程。相反实时信号是排队的。  

理解了这个竞态问题的关键后，来考虑怎么解决此竞态问题。“[The new pselect() system call](http://lwn.net/Articles/176911/)”提到一种work-around的方法，不过实现复杂。因此POSIX.1g才提出了增强版的select——pselect。使用pselect解决此问题的代码是这样的：  
{% highlight c %}
sigset_t emptyset, blockset;

sigemptyset(&blockset);         /* Block SIGINT */
sigaddset(&blockset, SIGINT);
sigprocmask(SIG_BLOCK, &blockset, NULL);

sa.sa_handler = handler;        /* Establish signal handler */
sa.sa_flags = 0;
sigemptyset(&sa.sa_mask);
sigaction(SIGINT, &sa, NULL);

/* Initialize nfds and readfds, and perhaps do other work here */
/* Unblock signal, then wait for signal or ready file descriptor */

sigemptyset(&emptyset);
ready = pselect(nfds, &readfds, NULL, NULL, NULL, &emptyset);
... 
{% endhighlight %}

pselect之所以能解决此竞态，是因为pselect进入内核之后，在do_pselect函数中，才取消对信号的阻止：  
{% highlight c %}
if (sigmask) {
	/* XXX: Don't preclude handling different sized sigset_t's.  */
	if (sigsetsize != sizeof(sigset_t))
		return -EINVAL;
	if (copy_from_user(&ksigmask, sigmask, sizeof(ksigmask)))
		return -EFAULT;

	sigdelsetmask(&ksigmask, sigmask(SIGKILL)|sigmask(SIGSTOP));
	sigprocmask(SIG_SETMASK, &ksigmask, &sigsaved);
}
{% endhighlight %}

另外关于手动触发信号，了解到这个信息：  
{% highlight text %}
# stty -a
speed 38400 baud; rows 42; columns 167; line = 0;
intr = ^C; quit = ^\; erase = ^?; kill = ^U; eof = ^D; eol = <undef>; eol2 = <undef>; swtch = <undef>; start = ^Q; stop = ^S; susp = ^Z; rprnt = ^R; werase = ^W;
lnext = ^V; flush = ^O; min = 1; time = 0;
-parenb -parodd cs8 -hupcl -cstopb cread -clocal -crtscts
-ignbrk -brkint -ignpar -parmrk -inpck -istrip -inlcr -igncr icrnl ixon -ixoff -iuclc -ixany -imaxbel -iutf8
opost -olcuc -ocrnl onlcr -onocr -onlret -ofill -ofdel nl0 cr0 tab0 bs0 vt0 ff0
isig icanon iexten echo echoe echok -echonl -noflsh -xcase -tostop -echoprt echoctl echoke
{% endhighlight %}

###poll

`int poll(struct pollfd *fds, nfds_t nfds, int timeout);`  
poll和select的逻辑是一致的，只不过poll是通过链表的方式组织fd的，比select的数组要灵活。  
看到有人还这么描述poll：“poll还有一个特点是"水平触发(level-triggered)"，如果报告了fd后，没有被处理，那么下次poll时会再次报告该fd。”——从代码可以看出的确如此，只不过我不确定这就是“水平触发”？  

ppoll和poll的关系，类似于pselect和select的关系。  

另外，poll、select面临同样的问题，便是进入内核时，需要把fd数组或链表拷进内核，在fd很多时会影响性能。fd很多时，它们的这种轮询方式也很耗时。  

###epoll


