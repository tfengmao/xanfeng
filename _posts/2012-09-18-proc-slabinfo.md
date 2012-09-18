---
title: /proc/slabinfo是怎样生成的
layout: post
category: linux
tags: proc slab
---

“[篡权的ss](http://m114.org/system/usurped-power-ss.html)”写得很好，是关于iproute2工具包里的ss(socket statistics)命令的。阅读源码，发现至少`ss -s`是通过解析/proc/slabinfo获得数据的。  
稍加分析，可知为什么`ss -s`能够通过/proc/slabinfo得到数据：  
1、proc是一个虚拟文件系统，系统启动时创建，默认包含有一些文件。procfs提供了一些便利的接口，比如create_proc_entry，可用来往proc中增加新的文件或目录。  
2、proc里面很多文件都使用了seq_file。  
3、slabinfo必然利用了procfs的接口，并且使用seq_file。在mm/slab.c中可以找到对应代码。  
4、slab申请的常用接口是kmem_cache_create，这个函数里应该记录每一次申请，`cat /proc/slabinfo`时才可查，发现每次申请都是链在cache_chain里的。  

于是，便知道/proc/slabinfo如何得来。举个例子，ss查slabinfo里的5种数据：  
{% highlight c %}
static const char *slabstat_ids[] =
{
	"sock",
	"tcp_bind_bucket",
	"tcp_tw_bucket",
	"tcp_open_request",
	"skbuff_head_cache",
};
{% endhighlight %}  

比如tcp_bind_bucket就是通过kmem_cache_create创建的：  
{% highlight text %}
# find net/ -type f -name "*.c" | xargs grep -n tcp_bind_bucket
net/ipv4/tcp.c:2893:            kmem_cache_create("tcp_bind_bucket",
{% endhighlight %}  

