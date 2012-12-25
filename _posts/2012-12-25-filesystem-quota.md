---
title: 文件系统配额(quota)
layout: post
category: linux
tags: filesystem quota disk
---

文件系统配额（filesystem quota），我的理解是从文件系统的角度去控制对磁盘的访问，我觉得磁盘配额说的就是文件系统配额。  
quota很好理解，就是对于某FS，设置你能用多少空间，他又能用多少空间。当然实际上quota可以针对user或group，控制的资源有：你可以打开多少文件，你可以创建多少文件等。  

当然Windows也有quota，比如选择D盘的属性，就有一个“配额”tab，默认是关闭的。  
如常，这里说的是Linux filesystem quota。  

###使用方法

参考：“[磁碟配額(Quota)與進階檔案系統管理](http://linux.vbird.org/linux_basic/0420quota.php)”和“[Linux File System Quotas](http://www.yolinux.com/TUTORIALS/LinuxTutorialQuotas.html)”

Linux FS quota并不像ls、cat等命令一样“天生支持”，需要：  
1）系统支持，编译时要开启相关选项（`cat .config | grep -i quota`）  
2）安装linuxquota包。  
3）在目标设备挂载路径根下`touch aquota.user`(对应user quota)，`touch aquota.group`(对应group quota)。  
4）编辑/etc/fstab，设置usrquota, grpquota选项。  
5）`mount -o remount ...`  

quota相关有很多命令，简介：  
quotacheck - scan a filesystem for disk usage, create, check and repair quota files  
quota - display disk usage and limits  
repquota - summarize quotas for a filesystem  

quotactl - manipulate disk quotas  
setquota - set disk quotas  
edquota - edit user quotas  
quotaon, quotaoff - turn filesystem quotas on and off  

warnquota - send mail to users over quota  
quota_nld - quota netlink message daemon  
convertquota - convert quota from old file format to new one  
quotastats - Program to query quota statistics  

常用的是第1，2部分，quotacheck用来扫描当前状态，应该是记录在aquota.\*里面。`edquota`用来编辑配额值。  

###内核支持

前面说到，quota需要内核支持，我想是因为：  
1）quota并非posix定义的接口，因此不是放之四海而皆准，所以要明确选择开启。  
2）quota数据是在内核中维护的，因为用户态不安全，不过同第一点，所需的内核数据结构和逻辑是需要通过模块显式加载的，不是存在于通用结构中的。  

因此内核的支持之一是创建数据结构，之二是和用户通信。通信通过netlink来做，用来反应用户态的quotacheck、edquota等操作。这部分可参考Documentation/filesystems/quota.txt.  

###cephfs

cephfs应该是没有quota支持的，其[FAQ](http://ceph.com/w/index.php?title=FAQ&redirect=no)和[maillist](http://thread.gmane.org/gmane.comp.file-systems.ceph.devel/361)都有提到。而且ceph mount并不接受usrquota选项，实验和代码阅读都能证明这一点。  
不过cephfs依赖的是其他fs，如btrfs、ext4、xfs。这些fs是支持quota的，不知是否可以通过设置底层fs quota，达到控制cephfs quota的目的。——我想应该是不可以。  