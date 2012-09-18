---
title: Linux 块 I/O 层和 Buffer
layout: post
tags: linux block io buffer
category: linux
---
*本文是《Linux内核设计与实现》第13章“块I/O层”的笔记。*

块设备最小的可寻址单元是**扇区**（sector），通常大小为512字节，块设备可以一次传输多个扇区。  
内核执行的所有磁盘操作都是按照**块**（block）来进行的，块是文件系统的一种抽象，对块大小的要求是：  
1、大小是扇区的倍数；  
2、内核要求块大小是2的倍数；  
3、**不能超过一个页的长度**；  
我的前面[一片文章](http://xanpeng.github.com/2012/02/24/sector-block/)比较过扇区和块这些概念。

当一个块被调入内存时，它要存储在一个**缓冲区**中。每个缓冲区与一个块对应，它相当于磁盘块在内存中的表示。由于块大小不能超过一个页面，所以一个页面可以容纳一个或者多个内存中的块。由于内核在处理数据时需要一些相关的控制信息（如块属于哪个设备，对应于哪个缓冲区等），所以**每一个缓冲区有一个对应的描述符**，用buffer_head结构表示，被称为缓冲区头，它包含了内核操作缓冲区所需要的全部信息。

    struct buffer_head {
        unsigned long         b_state;        // 缓冲区状态标志
        atomic_t              b_count;        // 缓冲区使用计数
        struct buffer_head    *b_this_page;   // 页面中的缓冲区
        struct page           *b_page;        // **存储缓冲区的页面**
        sector_t              b_blocknr;      // 逻辑块号
        u32                   b_size;         // 块大小（字节）
        char                  *b_data;        // 页面中的缓冲区
        struct block_device   *b_dev;         // 块设备
        bh_end_io_t           *b_end_io;      // IO完成方法
        void                  *b_private;     // 完成方法的数据
        struct list_head      b_assoc_buffers;// 相关的映射链表
    }

目前内核中IO操作的基本容器由**bio**结构体表示，它代表了正在现场的（活动的）以片段（segment）链表形式组织的块IO操作，一个segment是一小块连续的内存缓冲区。这样就不需要保证单个缓冲区一定要连续，即使一个缓冲区分散在内存的多个位置上，bio结构体也能对内核保证IO操作的执行，像这样的向量IO就是所谓的**聚散IO**.  

每一个块IO请求都通过一个bio结构体表示，每个请求包含一个或者多个块，这些块存储在bio_vec结构体数组中，这些结构体描述了每个片段在物理页中的实际位置，并且像向量一样被组织在一起。

块设备将它们挂起的块IO请求保存在**请求队列**中，由request_queue结构体表示，队列中的请求由request结构体表示。

**IO调度程序**  
内核既不会简单地按请求接受次序，也不会立即将其提交给磁盘，相反，它会在提交前，先执行名为**合并与排序**的预操作，这种预操作可以极大地提高系统的整体性能，在内核中负责提交IO请求的子系统被称为**IO调度程序**。  

Linux中使用的IO调度算法有：  
1、Linus电梯。  
2、deadline IO调度算法。  
3、Anticipatory（预测）IO调度算法。   
4、noop（空操作）IO调度算法。  
5、CFQ（完全公正）IO调度算法。