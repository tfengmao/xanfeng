---
title: 对象存储(object-based storage)
layout: post
category: linux
tags: osd scsi driver
---

###概述

云计算领域开始较多地涉及对象存储（object-based storage）。  
假设一个云存储系统，对外提供对象存储能力。一般地除了提供POSIX文件系统接口，还会提供块设备访问形式——对象存储设备（object-based storage device，osd）。  
貌似没有特别的设备，专门针对对象存储。貌似情况是：同样的设备，用不同的协议去访问，比如SCSI协议。  
osd也需要一个协议，这个协议和SCSI处于同一级别。  

这个协议就称之为OSD协议，它是一个T10 SCSI command set。跟SCSI类似，分成osd-initiator、osd-target两部分。  
关于这个协议，有这些资料：1）Documentation/scsi/osd.txt；2）http://www.t10.org/ftp/t10/drafts/osd2/；3）[open-osd: OSD Initiator library for 2.6.29](http://lwn.net/Articles/313613/)；4）https://github.com/bharrosh/open-osd。  
首先要去了解T10的osd2协议，但是其资料是非公开的。我想是因为要保护协议制定者，使得他们掌握实现主动权。  

内核只提供了initiator的实现（drivers/scsi/osd/），对应于模块libosd：  
{% highlight text %}
# modinfo libosd
filename:       /lib/modules/2.6.37.1-1.2-desktop/kernel/drivers/scsi/osd/libosd.ko
license:        GPL
description:    open-osd initiator library libosd.ko
author:         Boaz Harrosh <bharrosh@panasas.com>
srcversion:     A959D366A4386A4DDC0D22C
depends:        
vermagic:       2.6.37.1-1.2-desktop SMP preempt mod_unload modversions 
{% endhighlight %}

可以认为，libosd只是提供了机制，就跟它的名字（lib开头）所暗示的一样。  
如果要能侦测和使用对象存储块设备（osd），需要相应的驱动。内核提供了一个驱动模块osdblk（[osdblk: a Linux block device for OSD objects](http://lwn.net/Articles/334536/)）：  
{% highlight text %}
# modinfo osdblk
filename:       /lib/modules/2.6.37.1-1.2-desktop/kernel/drivers/block/osdblk.ko
license:        GPL
description:    block device inside an OSD object osdblk.ko
author:         Jeff Garzik <jeff@garzik.org>
srcversion:     EC60760A3AA88B74481FDF7
depends:        libosd,osd
vermagic:       2.6.37.1-1.2-desktop SMP preempt mod_unload modversions

# modinfo osd
filename:       /lib/modules/2.6.37.1-1.2-desktop/kernel/drivers/scsi/osd/osd.ko
alias:          scsi:t-0x11*
alias:          char-major-260-*
license:        GPL
description:    open-osd Upper-Layer-Driver osd.ko
author:         Boaz Harrosh <bharrosh@panasas.com>
srcversion:     92C24460C09FE914D3A2E7C
depends:        libosd
vermagic:       2.6.37.1-1.2-desktop SMP preempt mod_unload modversions 
{% endhighlight %}

osdblk貌似是驱动本地物理设备的。另外内核还提供了一个对象存储文件系统[exofs](http://www.ibm.com/developerworks/cn/linux/l-nilfs-exofs/)：  
{% highlight text %}
# modinfo exofs
filename:       /lib/modules/2.6.37.1-1.2-desktop/kernel/fs/exofs/exofs.ko
license:        GPL
description:    exofs
author:         Avishay Traeger <avishay@gmail.com>
srcversion:     601FDBF0D2D33AEBC38AEB8
depends:        osd,libosd
vermagic:       2.6.37.1-1.2-desktop SMP preempt mod_unload modversions
{% endhighlight %}

还不知道如何使用，不然可以看看object-based storage到底是什么。  
但osdblk、exofs都不是本文的目的。如文首描述，本文目的是：如果有一个分布式对象存储系统，用户要以块设备的形式使用它，该怎么做。  
现在我们大概知道：1）这个分布式对象存储系统，要依照T10 osd协议，实现OSD target；2）客户端（打算使用osd块设备）要模拟osdblk的做法，实现一个osd initiator。osdblk不行，因为它应该是针对本地设备的。  

###细节和示例

先说设备和驱动（device & driver），ULK3、LDD3都细致描述了这个话题，比如[LDD3 Ch14 The Linux Device Model](http://lwn.net/images/pdf/LDD3/ch14.pdf) “Putting It All Together”部分举例说明了整个流程。  
简单说来，Linux设备模型有三部分：bus、device和driver，从/sys可以看到其层级关系。这个关系是：先有bus，bus关联drivers和devices，当加入device时，bus会遍历drivers去match device。  
一个device可以match到多个driver，也可能没有match到任何driver。代码参见driver\_add->bus\_probe\_device()。  
看起来，**device是可以没有对应的driver的**。比如一个虚拟bus下，没有任何driver，加入device时，定然match不到任何driver，也照样可以工作。  

