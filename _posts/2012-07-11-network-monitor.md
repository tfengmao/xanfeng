---
title: linux网络监测
layout: post
category: linux
tags: linux network monitor ifconfig tools
---

*这是一篇记了一半，觉得没料的文章，本想删掉，但还是先放着吧。*

[Linux Network Statistics Tools / Commands](http://www.cyberciti.biz/faq/network-statistics-tools-rhel-centos-debian-linux/): nstat, ss, netstat, ifconfig, sar.

nstat, ss, netstat, ifconfig, ip, sar, iftop, atop

###网桥

关于网桥的IP地址: [http://wangcong.org/blog/archives/1657](http://wangcong.org/blog/archives/1657).  
理解linux网桥: [http://my.unix-center.net/~lishuai860113/?p=209](http://my.unix-center.net/~lishuai860113/?p=209).  
`man brctl`:

> An  ethernet bridge is a device commonly used to connect different networks of ethernets together, so that these ethernets will appear as one ethernet to the participants.
> 
> Each of the ethernets being connected corresponds to one physical interface in the bridge. These individual ethernets are  bundled into one bigger ('logical') ethernet, this bigger ethernet corresponds to the bridge network interface.

现在的理解是, linux中的网桥类似于一个交换机, 不同的网卡可以作为其中的一个端口. 可以用网桥来隔离下层网络结构, 比如通过网桥构建一个业务网络, 这个网络是相对固定的, 其下层的网络变动可以不影响现有的业务网络.  
下面这幅图片, 能帮助理解虚拟环境下网桥的作用.

![](/images/network-bridge.png)

###ifconfig

ifconfig主要被用来做网卡信息的查询, 以及up/down等简单操作. `man ifconfig`可以看出, 它还可以在系统用来配置网卡, 不过在新版本内核中, 已经由`ip`替代. 不过配置工作不是当下的关注点.

ifconfig的输出解释:  

{% highlight text %}
# ifconfig | more
br0     Link encap:Ethernet  HWaddr E0:xx:xx:xx:xx:xx
          inet addr:xxx.xxx.xxx.xxx  Bcast:xxx.xxx.255.255  Mask:255.255.0.0
          inet6 addr: fe80::e224:7fff:fe94:5592/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:21994145 errors:0 dropped:0 overruns:0 frame:0
          TX packets:174861 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1488447785 (1419.4 Mb)  TX bytes:265144488 (252.8 Mb)

eth0   Link encap:Ethernet  HWaddr E0:xx:xx:xx:xx:xx
          inet6 addr: fe80::e224:7fff:fe94:5592/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:22231622 errors:0 dropped:0 overruns:0 frame:0
          TX packets:313606 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:1813934287 (1729.9 Mb)  TX bytes:272636988 (260.0 Mb)
          Memory:f9360000-f9380000
{% endhighlight %}

br0: 设备名; link encap: 基本描述; HWaddr: 硬件地址.  
inet addr: 网络ip地址; Bcast: 广播地址; Mask: 子网掩码.  
UP: 网卡已经启用.  
BROADCAST: 支持广播.  
RUNNING: 网卡正在运行.  
MULTICAST: 支持多播.  
MTU: 最大传输单元.  
Metric: 度量值, 用于估算路由成本.  

RX packets: 接收时, 正确的数据包数.  
errors: 接收时, 错误的数据包数.  
dropped: 接收时, 丢弃的数据包数.  
overruns: 接收时, 由于过速丢弃的数据包数.  
frame: 接收时, 由于frame错误而丢弃的数据包数.  

TX packets: 发送时, 正确的数据包数.  
errors: 发送时, 错误的数据包数.  
dropped: 发送时, 丢弃的数据包数.  
overruns: 发送时, 由于过速丢弃的数据包数.  
frame: 发送时, 由于carrier错误而丢弃的数据包数. 

collisions: 冲突信息包的数目.  
txqueuelen: 发送队列的大小, 此处是1000MB.  

RX bytes: 接收的数据量.  
TX bytes: 发送的数据量.  
