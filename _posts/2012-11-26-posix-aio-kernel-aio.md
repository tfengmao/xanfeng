---
title: Linux异步IO初探
layout: post
category: coding
tags: glibc aio libaio
---

kouu的“[linux异步IO浅析](http://hi.baidu.com/_kouu/item/2b3cfecd49c17d10515058d9)”讲得很清楚，在Linux下异步IO(Asynchronous IO)有两套机制：1）POSIX aio，一般用的是glibc的实现，`man aio_read`；2）内核native aio，包含在libaio库中，`man io_submit`。  

进一步之前，你或许想问：什么是同步/异步IO，什么又是blocking/non-blocking IO？  
IBM dw上的文章“[使用异步 I/O 大大提高应用程序的性能](http://www.ibm.com/developerworks/cn/linux/l-async/)”对此讲得很清楚：  
![](/images/io-model.gif)  
另外，“[IO - 同步，异步，阻塞，非阻塞 （亡羊补牢篇）](http://blog.csdn.net/historyasamirror/article/details/5778378)”也不错。  

还是回到kouu的文章。Linux的IO流程比较复杂，分直接IO(Direct IO)和非直接IO两种。非直接IO指会利用Page Cache，直接IO指不经过Page Cache，用户程序直接面对存储设备。  
在使用Page Cache的时候，用户读写文件，实际上是从Page Cache读、写到Page Cache就完事了，真正的存储设备访问动作是由Page Cache代理的。——所以，非直接IO模式下，不管用户程序是同步/异步IO，用户程序不能“大展手脚”。  
因此看起来，异步IO时，更适合同时使用直接IO。  

另一方面，glibc aio是通过线程模拟实现异步的；而内核支持的libaio则是真正利用了**CPU和存储设备可并行工作**的特性。  

且止于此，列出一些有用的资料：  
1、kouu的“[linux异步IO浅析](http://hi.baidu.com/_kouu/item/2b3cfecd49c17d10515058d9)”最值推荐，对两种机制的实现原理都做出阐述。  
2、[异步IO的演化记录](http://www.cnblogs.com/raymondshiquan/articles/2659202.html)  
3、淘宝褚霸的[Linux下异步IO(libaio)的使用以及性能](http://blog.yufeng.info/archives/741)，通过实验证明libaio使得设备能力被完全利用。  
4、[linux AIO（异步IO）那点事儿](http://cnodejs.org/topic/4f16442ccae1f4aa270010a7)  

另附glibc aio的示例：  
{% highlight c %}
#include <aio.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main()
{
    int fd, ret;
    struct aiocb my_aiocb;

#define BUFF_SIZE (1024 * 1024 * 20)
    fd = open("test.file", O_RDONLY);
    if (fd < 0) perror("open");

    bzero((char *)&my_aiocb, sizeof(struct aiocb));
    my_aiocb.aio_buf = malloc(BUFF_SIZE + 1);
    if (!my_aiocb.aio_buf) perror("malloc");

    my_aiocb.aio_fildes = fd;
    my_aiocb.aio_nbytes = BUFF_SIZE;
    my_aiocb.aio_offset = 0;

    ret = aio_read(&my_aiocb);
    if (ret < 0) perror("aio_read");

    while (aio_error(&my_aiocb) == EINPROGRESS);

    if ((ret = aio_return(&my_aiocb)) <= 0)
        printf("read failed\n");

    return 0;
}
{% endhighlight %}