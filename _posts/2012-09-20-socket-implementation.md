---
title: socket的实现
layout: post
category: linux
tags: socket kernel
---

一个典型的socket C/S流程是这样的：
![](/images/socket_flow.png)  

这些接口在内核中是如何实现的？《追踪Linux TCP/IP代码运行》带你看代码，可以看出内核网络“大厦”也不过是一砖一瓦建起来的。  

socket接口是这样的：int socket(int domain, int type, int protocol)。  
domain是指AF_UNIX、AF_INET、AF_INET6之类的，指定protocol family；type指SOCK_STREAM、SOCK_DGRAM、SOCK_RAW之类的；protocol就是对应domain而言，从protocol family中选择一个protocol，一般一个family里就一个protocol，所以一般取值0。

socket要抽象底层TCP、IP协议栈的功能，维护了很多数据结构，维护了很多主用于加速的哈希表，使用了大量函数指针(人们都称之为“钩子”)。我把这些结构的定义和注释放到kernel-net库里面了，主要是socket、sock、sk_buff、inet_sock等。

**int socket(int domain, int type, int protocol)**  
socket()经过glibc和系统调用下到内核，内核的入口是sys_socket，也就是SYSCALL_DEFINE3(socket, ...)。  
socket()的主要工作：（1）创建一个socket，对应地，就是创建socket结构变量、sock结构变量、sk_buff变量等一串变量。（2）在sockfs中创建一个文件，并将文件和sock关联起来(file->private=sock)，这样就可以通过文件API操作socket。  

这两部分主要的函数调用路径如下：  
sys_socket -> sock_create(family, type, protocol, &sock) -> __sock_create() -> sock_alloc() -> inet_create()  
	-> sk_alloc()  
	-> sock_init_data()  
sys_socket-> sock_map_fd()  
	-> sock_alloc_fd()  
	-> sock_attach_fd()  
	-> fd_install()  

从这个调用路径可见端倪，细节不表。这里更重要的是如何处理不同的domain(protocol family)，不同的type，不同的协议。  
1、不同的family  
内核维护一个数组net_families，系统启动时就构建其中内容，如其中之一的inet_init()，内容是inet_family_ops、inet6_family_ops、unix_family_ops之类的。比如用户传入AF_INET，AF_INET的值是2，net_families[2]返回的就是inet协议族需要的信息了。  
2、不同的type  
family定了之后，type范围也定了，比如AF_INET包含3个type：SOCK_STREAM(TCP)、SOCK_DGRAM(UDP)、SOCK_RAW(IP)。这些当然也是预先写死的，也是在inet_init()里做的。  
3、不同的protocol  
这个只是设置sk->sk_protocol就可以了。  

总结一下：先定family，操作一番；再定type，接着操作一番。

**int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen)**  
bind是服务端调用的，目的是设置刚创建的服务端socket的地址和端口。可以看到这里是根据fd来关联的，这是sock_create()关联sock和文件带来的好处。  

还是以inet为例，sock_create会初始化sock->ops，type不同，ops也不同，有三种：inet_stream_ops、inet_dgram_ops、inet_sockraw_ops。  
以TCP为例，调用的是inet_bind()。

**int listen(int sockfd, int backlog)**  
listen的作用是建立连接请求队列，backlog是待设的请求数，这个数值不是设多大就给多大，而是存在某个maxvalue，最后取值为min(backlog, maxvalue)。

**int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen)**  
accept等待client调用connect()建立连接，有同步和异步两种方式。accept会从接收队列中获取client请求，并为每一个客户连接建立一个client_socket，返回值便是其fd。  

---

上面bind、accept等处都用到get_port()函数，获取一个可用的端口号。这个函数十分特别，先定一个范围，再从中随机取一个值，看是否被占用，没有就OK，被占用则重取。成功之后，加到某个哈希表中。

kernel-net中放了一个最简单的socket用户程序例子。