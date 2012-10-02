---
title: 直接设备访问
layout: post
category: linux
tags: access device driver vfs
---

通过文件系统访问设备是常见的方式，是否可以不使用文件系统，而直接访问设备。  
答案是肯定的。就好像你实现一个文件系统，最终你是需要面对设备的，需要直接访问设备的。  
那么用户态可以直接访问设备吗？  
也是可以的，因为设备也是一个文件，直接访问设备文件即可，用的api是read、write等。  
`dd if=/dev/zero of=/dev/sdx bs=1M count=10`，dd命令就是直接访问设备，千万慎用这条命令，它会搞垮sdx上原有的数据，使用`dd if=/dev/zero of=./data_10m bs=1M count=10`、`losetup /dev/loop0 ./data_10M`和`dd if=/dev/zero of=/dev/loop0 bs=1k count=10`代替。  
1. 看dd的源代码(在coreutils包里面)可知，写数据时用的是write接口。  
2. strace dd也可以看出来，使用read从/dev/zero里面读，使用write写入/dev/sdx。  

不过有一点值得思虑：这么做绕过VFS了吗？  
应该是没有的。因为write到内核就是fs/read_write.c里面的SYS_DEFINE3(write...)，执行的vfs_write。  
不过仍然是直接设备访问，拿dd loop的例子来说，首先走vfs_write，然后下到udev（mount看到/dev不是ext3的）的write方法，再下到drivers/block/loop.c的write方法(loop设备实际上没定义write方法，说明最后走的是loop对应的文件的write方法)。  
可以验证这个流程：修改内核，在loop.c对应位置加上dump_stack()；或者使用systemtap。  
*然而非常遗憾的是，二者都需要我重编内核。systemtap需要开启DEBUG等选项。编译内核时才发现我的x200已经不够用了。*

直接设备访问时，读取的块放在那里，写入的块如何组织，怎么考虑对齐...这些都是更深层的问题。不知道哪里有资料介绍这方面的内容呢？
