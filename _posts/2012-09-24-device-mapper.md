---
title: 逻辑卷管理
layout: post
category: linux
tags: device map lvm
---

###device mapper

*参考：[Linux 内核中的 Device Mapper 机制](http://www.ibm.com/developerworks/cn/linux/l-devmapper/index.html)*

device mapper(下文称为dm)是Linux 2.6内核中支持逻辑卷管理的**通用设备映射机制**。它的构架是这样的：  
![](/images/dm_arch.png)

可以想到，多了这一层抽象，设备管理变得更灵活。

dm在内核中是作为一个块设备驱动被注册的，它有三个重要的对象概念：mapped device、映射表、target device。  
有两点很特别：  
- 上层操作的是mapped device。  
- target device并不一定是真正的物理设备，它可以是另一个mapped device。这样可以构造出多层结构。  

有几处不明白：  
- 映射表(mapping table)是存在内存中，还是设备里？  
- 上层IO请求是bio表示的，如果映射方式是mirror(代表一份数据存到多处?)，bio是否需要复制一份？  

**dmsetup**  
dm通过ioctl的方式向外提供接口，用户通过用户空间的device mapper库，向dm的字符设备发送ioctl命令，完成向内的通信。  
dm在用户空间主要有device mapper库和dmsetup工具。下面看看dmsetup的使用。

你可以将几块物理设备组合成一个大的逻辑设备，可以将一个物理设备划分为几个小的逻辑设备，下面就看看这方面的两个例子。利用dm提供的机制，我们可以构造更复杂的映射方式，这里略过。  

组合：`./dmjoin.sh /dev/sdm /dev/sdn`  
{% highlight bash %}
#!/bin/sh
size1=`blockdev --getsize $1`
size2=`blockdev --getsize $2`
echo "0 $size1 linear $1 0
$size1 $size2 linear $2 0" | dmsetup create joined
# 注：create后面的joined只是一个名字而已
{% endhighlight %}

分裂：`./dmsplit.sh /dev/sdm`  
{% highlight bash %}
#!/bin/sh
size=`blockdev --getsize $1`
halfsize=$((size/2))
echo "0 $halfsize linear $1 0" | dmsetup create split0
echo "0 $halfsize linear $1 $halfsize" | dmsetup create split1
{% endhighlight %}

对等：`./dmidentity.sh`  
{% highlight bash %}
#!/bin/sh
echo "0 `blockdev --getsize /dev/sdm` linear /dev/sdm 0" | dmsetup create identity
{% endhighlight %}

注1：dm遵循Linux内核设计准则，只提供机制，所以即使用户对同一设备做各种会相互干扰的操作，dmsetup也不会报错。  
注2：上面数字的单位都是sector。  

查询和删除(`man dmsetup` for more)：  
{% highlight text %}
# dmsetup ls
joined  (253, 1)
split1  (253, 4)
identity        (253, 0)
split0  (253, 3)

# dmsetup info
Name:              joined
State:             ACTIVE
Read Ahead:        1024
Tables present:    LIVE
Open count:        0
Event number:      0
Major, minor:      253, 1
Number of targets: 2

Name:              split1
State:             ACTIVE
Read Ahead:        1024
Tables present:    LIVE
Open count:        0
Event number:      0
Major, minor:      253, 4
Number of targets: 1

...

# dmsetup status
joined: 0 251658240 linear
joined: 251658240 377487360 linear
split1: 0 188743680 linear
identity: 0 377487360 linear
split0: 0 188743680 linear

# dmsetup table
joined: 0 251658240 linear 8:144 0
joined: 251658240 377487360 linear 8:192 0
split1: 0 188743680 linear 8:192 188743680
identity: 0 377487360 linear 8:192 0
split0: 0 188743680 linear 8:192 0

# dmsetup remove /dev/mapper/joined

{% endhighlight %}
