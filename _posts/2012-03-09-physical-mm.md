---
title: linux 物理内存管理
layout: post
tags: linux physical memory
category: linux
---

*我在"[Linux memory management](http://xanpeng.github.com/2012/05/31/linux-memory-management/)"对本话题做了更好的理解, 不过本文的"slab分配器"部分值得一看.*

#数据结构
linux 支持**非一致存储结构(NUMA)**(对应于一致存储结构 UMA), 它引入"存储节点"的概念, 把访问时间相同的存储空间叫做一个存储节点. linux 把物理内存划分为三个层次来管理: 存储节点(Node), 管理区(Zone)和页面(Page).

存储节点的数据结构为 pg_data_t, 每一个 NUMA 节点都有一个 pg_data_t 负责描述这个节点的内存布局.

存储管理区的数据结构为 zone, 每一个 pg_data_t 包含多个 zone, 一般是三个: ZONE_DMA(<16M), ZONE_NORMAL(16M~896M), ZONE_HIGHMEM(>896M).  
1) ZONE_DMA: 专门用于 DMA.  
2) ZONE_HIGHMEM: **高端内存**. 在32位系统中, 地址空间是4G, 其中内核规定3~4G的范围是内核空间，0~3G是用户空间. 内核的地址映射是写死的, 就是指这3~4G的对应的页表是写死的, 它映射到了物理地址的0~1G上(实际上没有映射1G, 只映射了896M. 剩下的空间留下来映射大于1G的物理地址, 而这一部分显然不是写死的). 大于896M的物理地址是没有写死的页表来对应的, 内核不能直接访问它们(必须要建立映射), 称它们为高端内存(当然, 如果机器内存不足896M, 就不存在高端内存. 如果是64位机器, 也不存在高端内存, 因为地址空间很大很大, 属于内核的空间也不止1G了).  
3) ZONE_NORMAL: 不属于 ZONE_DMA 或 ZONE_HIGHMEM 的内存区域.  

pg_data_t 还包含一个 zonelist, 代表了分配策略, 即内存分配时的 zone 优先级. 一种内存分配往往不是只能在一个 zone 里进行分配的，比如分配一个页给内核使用时, 最优先是从 NORMAL 里面分配, 不行的话就分配 DMA 里面的好了(HIGH 就不行, 因为还没建立映射), 这就是一种分配策略.

pg_data_t 还维护了一个<code>struct page *node_mem_map</code>. 存储介质中的每一个物理页面都有一个对应的 <code>struct page</code>, node_mem_map 记录这些页面的起始位置.

#内存的分配和回收
在内存初始化完成以后, 内存中就常驻有内核映象(内核代码和数据). 以后, 随着用户程序的执行和结束, 就需要不断地分配和释放物理页面. 内核应为分配一组连续的页面建立一种稳定高效的分配策略, 并且必须解决一个重要的内存管理问题--外碎片问题. linux采用著名的伙伴(Buddy)系统算法来管理物理内存.  
但是请注意, 在 linux 中, CPU 不能按物理地址来访问存储空间, 而必须使用虚拟地址. 因此对于内存页面的管理, 通常是先在虚存空间中分配一个虚存区间, 然后才根据需要为此区间分配相应的物理页面并建立起映射. 也就是说, 虚存区间的分配在前, 而物理页面的分配在后.

##伙伴算法的原理
linux 的伙伴算法把所有的空闲页面分为10个块组, 每组中块的大小是2的幂次方个页面, 例如第0组中块的大小都为2<sup>0</sup>(1个页面), 第1组中块的大小为都为2<sup>1</sup>(2个页面), 第9组中块的大小都为2<sup>9</sup>(512个页面). 也就是说, 每一组中块的大小是相同的, 且这同样大小的块形成一个链表.

假设要求分配的块其大小为128个页面(由多个页面组成的块我们就叫做页面块). 该算法先在块大小为128个页面的链表中查找, 看是否有这样一个空闲块. 如果有, 就直接分配; 如果没有, 该算法会查找下一个更大的块, 具体地说, 就是在块大小为256个页面的链表中查找一个空闲块. 如果存在这样的空闲块, 内核就把这256个页面分为两等份, 一份分配出去, 另一份插入到块大小为128个页面的链表中. 如果在块大小为256个页面的链表中也没有找到空闲页块, 就继续找更大的块, 即512个页面的块. 如果存在这样的块, 内核就从512个页面的块中分出128个页面满足请求, 然后从384个页面中取出256个页面插入到块大小为256个页面的链表中. 然后把剩余的128个页面插入到块大小为128个页面的链表中. 如果512个页面的链表中还没有空闲块, 该算法就放弃分配, 并发出出错信号.

以上过程的逆过程就是块的释放过程, 这也是该算法名字的来由. 满足以下条件的两个块称为伙伴:  
1) 两个块的大小相同  
2) 两个块的物理地址连续  
伙伴算法把满足以上条件的两个块合并为一个块, 该算法是迭代算法, 如果合并后的块还可以跟相邻的块进行合并, 那么该算法就继续合并.

##slab 分配机制
采用伙伴算法分配内存时, 每次至少分配一个页面. 但当请求分配的内存大小为几十个字节或几百个字节时应该如何处理? 如何在一个页面中分配小的内存区, 小内存区的分配所产生的内碎片又如何解决? slab 分配机制解决这个问题.

![alt slab 分配器的主要结构](https://www.ibm.com/developerworks/cn/linux/l-linux-slab-allocator/figure1.gif)

创建 slab 缓存结构(静态方式):
	struct kmem_cache * my_cachep;

使用kmem_cache_create内核函数, 通常在内核初始化或者首次加载模块时执行:
	struct kmem_cache *
	kmem_cache_create( const char *name, size_t size, size_t align,
			unsigned long flags;
			void (*ctor)(void*, struct kmem_cache *, unsigned long),
			void (*dtor)(void*, struct kmem_cache *, unsigned long));

使用kmem_cache_destroy销毁缓存, 通常在模块卸载时执行:
	void kmem_cache_destroy( struct kmem_cache *cachep );

使用kmem_cache_alloc从一个命名的缓存中分配一个对象:
	void kmem_cache_alloc( struct kmem_cache *cachep, gfp_t flags );

使用kmem_cache_zalloc执行类似于kmem_cache_alloc的工作, 不过还执行额外的memset操作.

使用kmem_cache_free将一个对象释放回slab:
	void kmem_cache_free( struct kmem_cache *cachep, void *objp );

内核中最常用的内存管理函数是 kmalloc 和 kfree 函数, 它们也使用了类似于前面定义的函数的 slab 缓存, kmalloc 没有为要从中分配对象的某个 slab 缓存命名, 而是循环遍历可用缓存来查找可以满足大小限制的缓存. 找到之后, 就(使用 __kmem_cache_alloc)分配一个对象. 
	void *kmalloc( size_t size, int flags );
	void kfree( const void *objp );

#参考资料
1. <深入分析 linux 内核源码>
2. [linux 内存管理浅析](http://hi.baidu.com/_kouu/blog/item/f72e707ffa8478310cd7da28.html)
3. linux 源码 2.6.32
4. [Linux slab 分配器剖析](https://www.ibm.com/developerworks/cn/linux/l-linux-slab-allocator/)
