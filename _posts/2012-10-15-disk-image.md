---
title: 磁盘镜像
layout: post
category: linux
tags: disk image virtual
---

磁盘镜像，是我对“disk image”的翻译，很可能词不达意，但我想使所有文章都是中文标题...  
对这个话题，我感兴趣的实际是：VHD、QCOW2等面向虚拟机镜像文件的disk image format**有什么区别，各有什么优劣**。  

[wiki](http://en.wikipedia.org/wiki/Virtual_disk_image#Virtualization)对这个话题有详细的解释，disk image除了在虚拟化环境下的含义之外，还有通用的含义，比如/dev/sda1上是ext3，我可以在/dev/sda2上——可能也是ext3文件系统——的某个文件diskimage_sda1，把整个/dev/sda1 sector-by-sector地复制进来。  
那个文件diskimage_sda1就称之为disk image，这时候，这个文件包含了一个文件系统——Amazing！  

根据wiki的定义：

> A disk image is a single file **or storage device** containing the complete contents and structure representing a data storage medium or device...

需要指出，disk image不一定是一个文件，它也可以是一个设备。  
文件和设备形式的image，可以由dd完成（“of=文件”比“of=设备”要快很多）：  

> dd if=/dev/sdj1 of=fsimage  
> dd if=/dev/sdj1 of=/dev/sdj2 bs=64M  

如果源设备有fs，则dd之后的设备可以直接被mount，dd之后的文件也可以通过loop设备mount：  

> losetup /dev/loop0 fsimage  
> mount /dev/loop0 /mnt/dir  
>   
> mount /dev/sdj2 /mnt/dir  

不错，Linux是有built-in的virtual drive功能的，就是loop device driver了，它在背后如何实现，再说了。  
而Windows是没有built-in的irtual drive功能的，所以假如你下了一个iso格式的片，你还需要安装UltraISO之类的软件去打开。不过貌似[Windows 8要built-in支持ISO、VHD格式文件](http://blogs.msdn.com/b/b8/archive/2011/08/30/accessing-data-in-iso-and-vhd-files.aspx)，这样不用下软件就可以看iso里面的片了。  

----------

看起来，虚拟化环境下的VHD格式文件，其打开和使用方式，不过是类似于loop device driver做的动作而已了。  
当然，我们都知道，事实不复杂，也不会是这里说的那么简单...  

**Raw vs. VHD vs. QCOW2 vs. VMDK**  
来看虚拟化场景下的各种disk image format。  

貌似为了防止各种格式乱套，出现了一个标准“[Open Virtualization Format](http://en.wikipedia.org/wiki/Open_Virtualization_Format)”。  

Raw格式是指“没有格式”...“原来怎样，我就怎样”，dd出来的就是Raw格式。  
出现其它各种格式，就是因为Raw“太低端”，不能支持高级特性，比如加密、快照、链接克隆、瘦分配等。  

[VHD](http://en.wikipedia.org/wiki/VHD_\(file_format\))格式现在属于Microsoft所有，但其format specification是公开的。  
它有3种子格式：  
1、Fixed格式。建立时就固定大小了。  
2、Dynamic格式。建立时是某个大小，比如2G，使用过程中可以动态增长，比如增长到128G。  
3、Differencing格式。基于Dynamic格式，存在某个父镜像，它记录对父镜像的改动。  

[QCOW2](http://en.wikipedia.org/wiki/Qcow#qcow2)是诞生自QEMU的格式，是完全open的。它的特性从Wiki link里面可以找到，Google也能方便搜得。  

[VMDK](http://en.wikipedia.org/wiki/VMDK)属于VMware，先后已有3+个版本，低版本的format specification貌似是公开的。  

它们之间有什么差异，各有什么优劣？我想每种格式的侧重点不同，对不同的功能的支持程度不同吧。  