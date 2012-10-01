---
title: Block I/O Layer
layout: post
category: linux
tags: block io
---

块设备是最慢的，用的又很多，它是木桶的那块短板，它的性能直接影响系统性能。  
所以针对块设备有一个block subsystem，也就是block I/O layer，块设备层。  
《Linux Kernel Development》3rd edition ch14介绍了块设备层。块设备层提供了request、request_queue、bio、bio_vec等机制，块设备驱动利用这些机制。  

更重要的，块设备层提供了4种I/O调度器，对request做merging、sorting，减少硬件寻道时间，提升性能。  
1. Linus电梯：做merge和sort，同时每个request有个定时器，如果被搁置太久，则优先被处理(处理逻辑优先级步骤2)。  
2. deadline：Linus电梯存在starvation，deadline为解决此问题而生。每个request有个定时器，这个定时保证相比Linus电梯是绝对的。  
3. anticipatory：deadline会引起这样的情况，正在写一块地方，来了一个读请求，于是立马调度到读请求处理，处理完之后又回到原来写的地方，刚好开始又来了一个读请求...(这么看起来，deadline是读优先于写的，具体忘记了...)...anticipatory的想法很简单，就是处理完这样的读之后，空等一会，再继续处理。这样防止紧接着的读请求。  
4. CFQ，completely fair queue：为每个进程维护一个request queue，queue内照样是merge和sort，queue之间是round-robin。可见cfq很适合桌面环境。  
5. Noop：不做啥事，适合flash memory之类的。  

block i/o layer大致就是这样，详情细节会有些出入，再议。  
结合其余的subsystems，会有更多的故事可说。比如buffer_head，比如当我直接访问设备时，一定要经过block i/o layer吗？(应是要的，块设备驱动都用到request等机制了...)一定要经过VFS层吗？不经过的话，会是怎样？

---

一些有意思的点：  
- the kernel (as with hardware and the sector) needs the block to be a power of two. The kernel also requires that a block be no larger than the page size.  
- The purpose of a buffer head is to describe this mapping between the on-disk block and the physical in-memory buffer. Acting as a descriptor of this buffer-to-block mapping is the data structure’s only role in the kernel.  
- Each bio_vec is treated as a vector of the form <page, offset, len>.  
- The bi_idx field, a more important usage, however, is to allow the splitting of bio structures.With this feature, drivers implementing a Redundant Array of Inexpensive Disks (RAID). All the RAID driver needs to do is copy the bio structure and update the bi_idx field to point to where the individual drive should start its operation. 这条有意思，bi_idx可以用来实现RAID的跨设备卷(?)，在不同的设备中，只要复制bio、设置bi_idx值即可。  

