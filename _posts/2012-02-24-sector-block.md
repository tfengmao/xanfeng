---
title: 扇区(sector), 物理/逻辑块(block)
layout: post
tags: sector block mount
category: linux
---

涉及文件系统,操作系统,硬盘的时候,我们会接触到扇区(sector),物理块(physical block),逻辑块(logical block)的概念.它们究竟是什么意思?

**扇区(sector)**  
磁盘的最小寻址单元,一般大小为512bytes,但在2009年,西部数据提出使用4096bytes的扇区.[[1]](http://en.wikipedia.org/wiki/Cylinder-head-sector#cite_ref-2)[[2]](http://en.wikipedia.org/wiki/Disk_sector)
sector在磁盘上对应于每一个"圆环"的一段.

**物理块(physical block)**  
实际上,我并没有在网络上看到很多的"物理块"的说法,这里暂且这么说.  
观点1:物理块由一个或多个sector组成([sector and block explained](http://help.filemaker.com/app/answers/detail/a_id/3815/~/drive-blocks-vs.-sectors---explained)).  
观点2:(physical) block==sector ([How Do I Find The Hardware Block Read Size for My Hard Drive?](http://superuser.com/questions/121252/how-do-i-find-the-hardware-block-read-size-for-my-hard-drive))  
观点2的一个佐证:  
	xan@xmachine:~$ cat /sys/block/sda/queue/physical_block_size
	**512**
	xan@xmachine:~$

**逻辑块(logical block)**  
这个好理解了,逻辑块是文件系统层面的概念,它表示FS对底层存储的处理方式.  
获取filesyste block size的方法:  
	xan@xmachine:~$ sudo dumpe2fs /dev/sda1 | grep -i 'Block size'
	dumpe2fs 1.41.14 (22-Dec-2010)
	Block size:               4096
	xan@xmachine:~$ 
在mount文件系统的时候可以指定该值,man mount得到:  
> bs=value
> Give blocksize. Allowed values are 512, 1024, 2048, 4096.


