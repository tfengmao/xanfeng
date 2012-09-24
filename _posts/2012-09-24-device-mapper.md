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

###lvm2

参考：[The Linux Logical Volume Manager](http://www.redhat.com/magazine/009jul05/features/lvm2/)、[LVM2 Resource Page](http://sources.redhat.com/lvm2/)、[LVM 2 FAQ](http://tldp.org/HOWTO/LVM-HOWTO/lvm2faq.html#AEN298)、[LVM新手指南-lvm使用入门](http://forum.ubuntu.org.cn/viewtopic.php?f=54&t=254335)。  

lvm2：Linux Logical Volume Manager，version 2。相比lvm1，它有更优的内部设计、更多的功能（volume mirroring and clustering）。  
lvm是用户态技术，内核里面没有对应的东西。lvm2需要三个东西才能跑起来：内核的device mapper机制、用户态的libdevmapper、用户态的lvm2工具集。  

lvm利用device mapper，做了更细致的逻辑卷管理。它涉及下面的概念：  
一个physical disk被划分为1～n个physical volumes(**PV**)，组合PVs构建出logical volume groups(**VGs**)，在VG上划分出logical volume(**LV**)。  
PV由固定大小的physical extents(**PE**)组成，LV也是由固定大小的logical extents(**LE**)组成，PE和LE经常被设置为相同大小值，一般取值4MB。  
LV的LEs和PEs建立映射关系。映射有多种，比如linear mapping、stripped mapping。不同方式有不同用处，如stripped mapping可用来提供更大的磁盘带宽。  

操作示例：[The Linux Logical Volume Manager](http://www.redhat.com/magazine/009jul05/features/lvm2/)。  
我不知道lvm是否能直接操作raw disk，RedHat的示例表明是可以的。我操作的示例是fdisk /dev/sdm划分出来的/dev/sdm1～4.  
{% highlight text %}
# pvcreate /dev/sdm1 /dev/sdm2 /dev/sdm3 /dev/sdm4
  No physical volume label read from /dev/sdm1
  Physical volume "/dev/sdm1" successfully created
...

# pvdisplay
  "/dev/sdm1" is a new physical volume of "38.30 GB"
  --- NEW Physical volume ---
  PV Name               /dev/sdm1
  VG Name
  PV Size               38.30 GB
  Allocatable           NO
  PE Size (KByte)       0
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               sCesx0-aPRV-j8h2-bkkS-08mC-dgyo-v6tzdC
...

# vgcreate vg_one /dev/sdm1 /dev/sdm2 /dev/sdm3
  Volume group "vg_one" successfully created

# vgdisplay		--> 注意PE size、Total PE
  --- Volume group ---
  VG Name               vg_one
  System ID
  Format                lvm2
  Metadata Areas        3
  ...
  VG Size               114.90 GB
  PE Size               4.00 MB
  Total PE              29415
  Alloc PE / Size       0 / 0
  Free  PE / Size       29415 / 114.90 GB
  VG UUID               MlHThr-4KdY-eFof-5D4d-rKkL-yjhw-gNQnT4

# vgextend vg_one /dev/sdm4
  Volume group "vg_one" successfully extended

# vgdisplay
...(可看出变化)

# lvcreate -n lvg_one --size 180G vg_one
  Insufficient free extents (46078) in volume group vg_one: 46080 required	--> 大小检查

# lvcreate -n lvg_one --size 179G vg_one
  Logical volume "lvg_one" created

# lvremove /dev/mapper/vg_one-lvg_one
Do you really want to remove active logical volume "lvg_one"? [y/n]: y
  Logical volume "lvg_one" successfully removed

# lvcreate -n lv_one -l 10000 vg_one
  Logical volume "lv_one" created

# lvcreate -n lv_two -l 10000 vg_one
  Logical volume "lv_two" created

# lvdisplay
...
# lvextend -l +10000 /dev/mapper/vg_one-lv_one
  Extending logical volume lv_one to 78.12 GB
  Logical volume lv_one successfully resized
# lvdisplay
...

# ll /dev/mapper/
total 0
lrwxrwxrwx 1 root root     16 Sep 21 21:32 control -> ../device-mapper
brw-r----- 1 root disk 253, 0 Sep 24 17:03 vg_one-lv_one
brw-r----- 1 root disk 253, 1 Sep 24 17:03 vg_one-lv_two

# ll /dev/disk/by-id		(从输出可以看出使用device mapper的痕迹)
dm-name-vg_one-lv_one -> ../../dm-0
dm-name-vg_one-lv_two -> ../../dm-1
dm-uuid-LVM-MlHThr4KdYeFof5D4drKkLyjhwgNQnT4MFsq9JAUxPQmhL3ivdma9iHY2lCPl4FH -> ../../dm-1
dm-uuid-LVM-MlHThr4KdYeFof5D4drKkLyjhwgNQnT4TFchVOmdpMLUq9iKV1RHuYi2chTVqerT -> ../../dm-0
lvm2-pvuuid-QK3862-6xnn-jD0a-A4aS-DEt0-FDgQ-F5bpQ0 -> /dev/sdm4
lvm2-pvuuid-WdBGXz-svr8-hgXB-eGdn-QAUs-l8IN-uAFv8s -> /dev/sdm2
lvm2-pvuuid-puPDyK-l1rE-JaoZ-63YM-zUur-caO7-XwUHrZ -> /dev/sdm3
lvm2-pvuuid-sCesx0-aPRV-j8h2-bkkS-08mC-dgyo-v6tzdC -> /dev/sdm1
scsi-xxx -> ../../sdm
scsi-xxx-part1 -> ../../sdm1
...
{% endhighlight %}  
注：lvdisplay并没有看到LE Size。

示意图片(均来自RedHat)：  
![](/images/lvm_internal.png)  
![](/images/lvm_extent.png)  
![](/images/lvm_mapping.png)  
