---
title: Linux process communication
layout: post
tags: process communication
category: linux
---

*本文是ULK Ch19 process communication的笔记. 本文实际上主要讨论了管道和命名管道.*

Unix系统提供的进程间通信的基本机制:  
- 管道和FIFO(命名管道).  
- 信号量.  
- 消息.  
- 共享内存区.  
- 套接字socket.  

---

#管道(pipe)

管道是进程之间的一个单向数据流: 一个进程写入管道的所有数据都由内核定向到另一个进程, 另一个进程由此就可以从管道中读取数据.  
Unix的命令shell中, 使用"|"操作符来创建管道, 如`ls | more`, 这个我们很熟悉.

管道被看作是打开的文件, 但在已安装的文件系统中没有相应的映像. 可以使用pipe()系统调用来创建一个新管道, 这个系统调用返回一对文件描述符(fd), 然后进程提供fork()把这两个描述符传递给它的子进程, 由此与子进程共享管道. 进程可以在read()系统调用中使用第一个fd从管道中读取数据, 同样也可以在write()系统调用中使用第二个fd向管道中写入数据.

POSIX只定义了半双工的管道, 每个进程在使用一个fd之前仍得把另外一个fd关闭. 有些Unix系统实现了全双工管道, 允许两个fd既可以被写入也可以被读取, 而Linux采用另外一种方式: 每个管道的fd仍然都是单向的, 但是在使用一个fd之前不必把另外一个fd关闭.

举例说明"ls | more", 它实际上进行的操作:  
1. 调用pipe(), 假设pipe()返回fd 3(管道的读通道)和4(管道的写通道).  
2. 调用两次fork().
3. 两次调用close()用来释放fd 3, 4.  

第一个子进程必须执行ls程序, 它执行的操作:  
1. 调用dup2(4, 1)把fd 4拷贝到fd 1, 从而fd 1代表管道的写通道.  
2. 两次调用close()用来释放fd 3, 4.  
3. 调用execve()执行ls程序, 此时ls的输出写到fd 1中, 由于已经重定向, 实际上就是写入管道中.

第二个子进程必须执行more程序, 它执行的操作:  
1. 调用dup2(3, 0), 使得fd 0代表fd 3.  
2. 两次调用close(), 释放fd 3, 4.  
3. 调用execve()执行more程序, 此时more的输入源是管道.

示例程序(*参考文章"[Linux进程间通信之管道](http://my.unix-center.net/~Simon_fu/?p=777)"*):

<script src="https://gist.github.com/2921587.js"> </script>

示例代码是创建一个管道, 创建一个子进程, 然后让子进程写管道, 主进程读管道, 从而得到输出"Hello World". `strace -o testpipe.log ./test_pipe`之后, `grep pipe testpipe.log`可以看出pipe创建了两个fd, 这个跟上面的描述一致:

    execve("./test_pipe", ["./test_pipe"], [/* 59 vars */]) = 0
    pipe([3, 4])                            = 0

但是同样`strace -o check_ls_pipe.log ls | more`, `grep pipe check_ls_pipe.log`并没有看出pipe()的调用, 这说明Linux 2.6.32.26对于`ls | more`这样的操作要么有内部细节的变动, 要么相比于ULK的说法, 已经变更了使用pipe的方式了. 这里需要提起注意.

根据ULK3, 管道被创建时, 内核都创建了一个索引节点和两个文件对象, 一个用于读, 另一个用于写. 当索引节点指的是管道时, i_pipe字段指向pipe_inode_info结构.  
另外每个管道都还有自己的管道缓冲区(pipe_buffer), Linux 2.6.10之前, 每个管道是一个pipe_buffer, 而2.6.11之后, 管道和FIFO都可以使用16个pipe_buffer.

根据ULK3, 管道是作为一组VFS对象来实现的, 没有对应的磁盘映像. 在Linux 2.6中, 这些VFS对象被组织为pipefs特殊文件系统, 以加速处理. 但pipefs在系统目录树中没有mount point, 因此用户根本看不到它(*的确如此...但怎么确定pipefs的存在呢?*), 因为有了pipefs, 内核就可以实现**命名管道**(named pipe, FIFO), 而命名管道的出现弥补了管道的不足: 只能在亲子进程间共享.

##命名管道

在Linux 2.6中, 命名管道, 或者FIFO(first in, first out)和管道几乎是相同的: 在文件系统中不拥有磁盘块, 打开的FIFO总是与一个内核缓冲区关联, 缓冲区中临时存放两个或多个进程之间交换的数据, 使用相同的pipe_inode_info结构.

FIFO和管道的不同点:  
- FIFO索引节点出现在系统目录树上而不是pipefs特殊文件系统中. `mkfifo xtfifo`可以看出在当前目录下创建了管道文件.

    prw-r--r-- 1 root root 0 Jun 13 18:44 xtfifo

- FIFO是一种双向通信管道, 也就是说可以用读/写模式打开一个FIFO.

示例程序(*参考文章"[Linux进程间通信之命名管道](http://my.unix-center.net/~Simon_fu/?p=777)"*):  
服务器端的代码:

<script src="https://gist.github.com/2921784.js"> </script>

客户端的代码:

<script src="https://gist.github.com/2921787.js"> </script>

执行"./fifo_server", 将会等待fifo_client发来的消息, 启动两个client, 得到如下结果:

    /* fifo_client 0 的输入 */
    # ./fifo_client
    %: hello server, i am client 0
    %:

    /* fifo_client 1 的输入 */
    # ./fifo_client
    %: hello server, i am client 1
    %:

    # ./fifo_server
    command from 26735: hello server, i am client 0
    command from 27049: hello server, i am client 1

看完这个例子, 你可能会问: 这和使用普通文件有什么区别?  
的确, 看起来很像, 除了mkfifo()的调用, 其他调用跟使用普通文件一模一样. 但还是有本质区别的, 因为fifo不涉及磁盘操作.

---

#System V IPC

IPC是进程间通信(Interprocess Communication)的缩写, 通常指用户态进程执行下列操作的一组机制:  
- 通过信号量与其他进程同步.  
- 向其他进程发送消息或者从其他进程接收消息.  
- 和其他进程共享一段内存区.

IPC共享内存还是比较有意思的, 它貌似需要特别的数据结构支持, 参考下图, 此处不展开.

*待插入图片19-3*