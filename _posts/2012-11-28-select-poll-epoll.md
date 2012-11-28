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

来看看怎么使用select()。一般的示例都是socket相关的，这里看一个“**错误的**”select示例——monitor regular file（file-based fd），出于你意料地，这是没有效果的，因为对于regular file：  
1、文件不同于流，文件内容在那里，总是ready的，监听它[没有意义](http://news-posts.aplawrence.com/662.html)。“一个文件已经/还没有ready for writing”的[说法是奇怪的](http://www.groupsrv.com/linux/about159067.html)，同理“ready for reading”一样奇怪。[内核也不好做这个语义上的控制](https://github.com/xanpeng/kernel-misc/blob/master/nerver-read-write-file-in-kernel.md)。  
2、另一方面，从内核代码可以看出，socket在数据available时，会设置POLLIN等标志——这是select的判断依据。而对常规文件，是没有也不方便设置这样的标志的。  
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
    // 开个线程写临时文件，没有效果
    FILE *fp = tmpfile();
    if (!fp) { perror("tmpfile()"); exit(EXIT_FAILURE); }
    int fpd = fileno(fp);

    pthread_t thread;
    int ret = pthread_create(&thread, NULL, gendata, (void*) fp);
    if (ret) { perror("pthread_create()"); exit(EXIT_FAILURE); }
    */

    // 即使不是临时文件，开个终端写文件，也没有效果
	// 但stdin是有效果的
    FILE *fp = fopen("/home/xan/lab/select.file", "r");
    if (!fp) { perror("fopen()"); exit(EXIT_FAILURE); }
    int fpd = fileno(fp);

    FD_SET(fpd, &rfds);
    maxfd = fpd;
    fprintf(stderr, "maxfd: %d\n", maxfd);

    tv.tv_sec = 5;
    tv.tv_usec = 0;
    retval = select(1, &rfds, NULL, NULL, &tv);
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

