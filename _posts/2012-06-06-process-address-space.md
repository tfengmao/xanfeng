---
title: Linux process address space
layout: post
tags: linux process memory address allocation virtual
category: linux
---
*本文是 ULK Ch9 的笔记.*

在"[Linux memory management](http://xanpeng.github.com/2012/05/31/linux-memory-management/)"中, 我了解了Linux内核如何管理物理内存的. 其中比较有意思的一点是, 内核会留出128MB的虚拟地址空间, 用来映射和管理物理高端内存.  
那么, 进程到底如何使用内存的呢? 这就是本文要了解的.

内核以直截了当的方式获得动态内存(*除固定地被持久占有的内存外的内存*), 主要有以下函数, 如果成功, 这些函数返回 page 结构指针或虚拟地址:  
1) __get_free_pages, alloc_pages() 从分区页框分配器中获得页框.  
2) kmem_cache_alloc(), kmalloc() 使用 slab 分配器为专用/通用对象分配块.  
3) vmalloc(), vmalloc_32() 获得一块非连续的内存区.

内核给内核分配内存, 给用户态进程分配内存, 采用的是截然相反的两种策略.  
1) 对内核: 内核请求的优先级是最高的, 所以不会推迟处理内核函数的请求; 相信自己, 假定内核函数是没有错误的, 因此不必处理各种编程错误.  
2) 对用户态进程: 进程对动态内存的请求是不紧急的, 所以尽量推迟; 用户进程是不被信任的, 内核必须随时准备捕获用户进程引起的寻址错误.

Linux 内核会推迟响应用户的动态内存请求, 用户进程请求动态内存, 它并没有直接得到页框, 而是**仅仅获得对一个新的虚拟地址空间的使用权**. 这个虚拟地址空间是**进程地址空间**的一部分, 称之为虚存区(*virtual memory area, **vma**, 命名可参考 ULK Ch9*).

---

#地址空间

我们已经很熟悉进程的地址空间了, 就是那个"4G, 3:1"的东西. 更确切地, 进程的地址空间由允许进程使用的所有虚拟地址组成.

##创建地址空间

进程的地址空间并非自动存在, 在创建一个新进程的时候, 内核调用 copy_mm() 函数, 这个函数通过建立新进程的所有页表和内存描述符(struct page)来创建进程的地址空间. 通常, 每个进程都有自己的地址空间, 但是**轻量级进程**可以共享同一地址空间(*设置 CLONE_VM 标志*), 因此内核创建轻量级进程比创建普通进程要快得多.  
copy_mm() 判断如果设置了 CLONE_VM, 就把父进程(current)地址空间赋给子进程; 如果没有设置 CLONE_VM, 就创建一个新的地址空间: 分配一个新的**内存描述符**, 把它的地址存放在新进程描述符的 tsk->mm 中, 并把 current->mm 的值赋值给 tsk->mm, 然后改变一些字段, 参考如下代码(*版本不同, 代码ms有较大变动*):

    tsk->mm = kmem_cache_alloc(mm_cachep, SLAB_KERNEL);
    memcpy(tsk->mm, current->vm, sizeof(*tsk->mm));
    atomic_set(&tsk->mm->mm_users, 1);
    atomic_set(&tsk->mm->mm_count, 1);
    init_rwsem(&tsk->mm->mmap_sem);
    tsk->mm->core_waiters = 0;
    tsk->mm->page_table_lock = SPIN_LOCK_UNLOCKED;
    tsk->mm->ioctx_list_lock = RW_LOCK_UNLOCKED;
    tsk->mm->ioctx_list = NULL;
    tsk->mm->default_kioctx = INIT_KIOCTX(tsk->mm->default_kioctx, *tsk->mm);
    tsk->mm->free_area_cache = (TASK_SIZE/3+0xfff)&0xfffff000;
    tsk->mm->pgd = pgd_alloc(tsk->mm);
    tsk->mm->def_flags = 0;

##删除地址空间

当进程结束时, 内核调用 exit_mm() 函数释放进程的地址空间.

---

#内存描述符

前面提到的内存描述符(memory descriptor)是一个特别的数据结构, 它包含与进程地址空间有关的全部信息, 这个结构的类型是 mm_struct, 进程描述符的 mm 字段就指向这个结构.

所有的内存描述符存放在一个双向链表中, 每个描述符在 struct list_head mmlist 字段存放链表相邻元素的地址. 链表第一个元素是 init_mm 的 mmlist 字段, init_mm 是初始化阶段**进程0**所使用的内存描述符.

mm_users 字段存放共享 mm_struct 结构的轻量级进程的个数. mm_count 字段是内存描述符的主使用计数器, 每当递减时, 内核都要检查它是否变为0, 如果是就要解除这个内存描述符. 即使有多个轻量级进程, 计算 mm_count 时仍只算作一个单位.

mm_alloc() 函数用来获取一个新的内存描述符, 它调用的是 kmem_cache_alloc() 来初始化新的内存描述符, 这说明这些描述符是被保存在 slab 分配器高速缓存中的. mm_count, mm_users 的初始值都是 1. 

##内核线程的内存描述符

内核线程仅运行在内核态, 它们永不会访问低于 TASK_SIZE(等于 PAGE_OFFSET, 通常是 0xc0000000)的地址. 而且, 与普通进程相反, 内核线程不用 vma, 因此, 内存描述符的很多字段对内核线程是没有意义的.

因为大于 TASK_SIZE 的虚拟地址的相应页表项都应该总是相同的, 因此一个内核线程的页表字段到底是何值根本就没什么关系, 但为了无用的 TLB 和高速缓存刷新, 内核线程使用一组最近运行的普通线程的页表. 结果, 每个 task_struct 包含了两种内存描述符指针: mm 和 active_mm. mm 指向进程所拥有的内存描述符, active_mm 指向运行时所使用的内存描述符. 普通进程这两个字段存放相同的指针, 内核指针不拥有任何内存描述符, 它们的 mm 总是 NULL. 当内核线程运行时, 它的 active_mm 字段被初始化为前一个运行进程的 active_mm.

---

#虚存区 Virtual Memory Area

Linux 通过 vm_area_struct 描述 VMA, vm_start 表示区间的起始地址, 也就是第一个虚拟地址, vm_end 表示区间的结尾地址, vm_mm 指向拥有这个区间的进程的 mm_struct 内存描述符.  
进程所拥有的所有 vma 是通过一个简单的链表链接在一起的, 出现在链表中的 vma 是按内存地址的升序排列的. vm_next 字段指向链表的下一个元素, 内核通过进程的 mm_struct.mmap 字段来查找 vma, mmap 字段指向链表中的第一个 vma.  
mm_struct.map_count 记录进程所拥有的 vma 数目, 默认情况下, 一个进程可以最多拥有 65536 个不同的 vma, sysadmin 可以通过写 /proc/sys/vm/max_map_count 来修改这个限定值.

进程所拥有的所有 vma 从来**不会重叠**, 并且内核尽力把新分配的 vma 与紧邻的现有 vma 进行**合并**. 当一个新的 vma 加入到进程的地址空间时, 内核检查一个已经存在的 vma 是否可以扩大, 如果不能, 就创建一个新的 vma. 如果从进程的地址空间删除一个 vma, 内核就要调整受影响的 vma 大小.

![](http://i.imgur.com/ttWLk.png)

##查找 vma

内核频繁执行的一个操作就是查找包含指定虚拟地址的 vma, 由于链表是经过排序的, 能方便于查找. 但是其算法复杂度是 O(n), 如果进程拥有的 vma 非常多, 如 malloc() 的专用调试器那样的进程会有成百上千的 vma, 这种情况下, vma 链表的管理会变得非常低效, 与内存相关的系统调用的性能就会降低到令人无法忍受的程度.

因此, Linux 2.6把内存描述符存放在红黑树(red-black tree)中, 红黑树的要素:  
1) 每个节点通常有两个孩子, 左孩子和右孩子.  
2) 对每个节点 N, N 的左子树上所有元素都排在 N 之前, 右子树上所有元素都排在 N 之后.  
3) 每个节点必须是黑或者红.  
4) 树根必须为黑.  
5) 红节点的孩子必须为黑.  
6) 从一个节点到后代叶子节点的每个路径都包含相同数量的黑节点. 当统计黑节点个数时, 空指针也算黑节点.  
 
这些规则确保具有n个节点的任何红黑树的高度最多为 2*log(n+1). 在红黑树中搜索一个元素因此变得非常高效, 算法复杂度为 O(logN). 在红黑树中插入和删除一个元素也是高效的, 因为算法能快速地遍历到插入和删除点. 

因此, 为了存放进程所有的 vma, Linux 既使用了链表, 也使用了红黑树. 这两种数据结构都包含指向同一 vma 的指针, 前面说到 mm_struct.mmap 指向 vma 链表的头部, 而红黑树的根是由 mm_struct.mm_rb 指向的.

##vma 的处理

有以下一些处理 vma 的函数:  
1) find_vma(): 查找给定地址的最临近区.  
2) find_vma_intersection(): 查找一个与给定的地址区间相重叠的 vma.  
3) get_unmapped_area(): 查找一个空闲的地址区间. 搜查进程的地址空间以找到一个可以使用的虚拟地址区间.  
4) insert_vm_struct(): 向内存描述符链表中插入一个 vma.  
5) do_mmap(): 分配一个新的虚拟地址区间.  
6) do_munmap(): 释放虚拟地址区间.

###do_mmap()

do_mmap() 为当前进程创建并初始化一个新的 vma. 不过, 分配成功后, 可以把这个新的 vma 和已有的进行合并.  
这个函数的大致工作:  
Step1. 先检查分配请求是否合法.  
Step2. 调用 get_unmapped_area() 获得新的虚拟地址区间.  
Step3. 调用 find_vma_prepare() 确定新区间之前的 vma 的位置, 以及红黑树中新区间的位置. 检查是否存在重叠.  
Step4. 检查插入新的 vma 是否引起进程地址空间的大小超过阈值, 如果超过则返回 -NOMEM.  
Step5. 检查 flags, 如没有设置 MAP_NORESERVE, 且新 vma 中含有私有可写页, 且无足够的空闲页框, 返回 -ENOMEM.  
Step6. 检查是否与前一个 vma 的权限一致等, 以决定是否合并.  
Step7. 调用 kmem_cache_alloc() 为新的 vma 非陪一个 vm_area_struct 结构, 并初始化.  
Step8. 判断是否可以共享, 并作相应处理.  
Step9. 调用 vma_link() 把新 vma 插入链表和红黑树.  
Step10. 更新 mm_struct.total_vm.  
Step11. 检查是否设置 VM_LOCKED, 是则调用 make_pages_present() 连续分配 vma 的所有**页**(*注意区分于物理**页框***), 并把它们锁在 RAM 中.  
Step12. 返回新 vma 的地址, 结束.  

###do_munmap()

do_munmap() 的参数是进程内存描述符的地址 mm, 地址区间的起始地址 start 和它的长度 len, 函数的作用是从当前进程的地址空间中删除一个 vma. 需要注意要删除的区间并不总是对应一个 vma, 它可能是一个 vma 的一部分, 也可能横跨多个 vma.  

大致流程:  
Step1. 参数检查. 确定要删除的虚拟地址区间之后的第一个 vma mpnt 的位置 (mpnt.end > start).  
Step2. 如果没有 mpnt, 也没有和要删除的虚拟地址空间重叠的 vma, 就什么也不做, 因为传入的区间上没有任何 vma.  
Step3. 如果 start > mpnt.start, 那么显然 mpnt 要被切分成两块, 用 split_vma() 函数.  
Step4. 如果 mpnt.start < start+len < mpnt.end, 同上, 也要切分.  
Step5. 调用 detach_vmas_to_be_unmapped() 从进程的地址空间中删除传入的区间.  
Step6. 调用 unmap_region() 清除区间对应的页表并释放页框.  

---

#缺页异常处理程序

do_page_fault() 函数是 80x86 上的缺页中断服务程序, 其总体方案如图.

![](http://i.imgur.com/owCA7.png)

实际中, 情况更为复杂, 缺页处理程序必须处理多种细分的特殊情况, 处理程序的详细流程如图.

![](http://i.imgur.com/gPee7.png)

do_page_fault() 读取引起缺页的虚拟地址, 如果地址不属于进程的地址空间, 则表示不正常, 分内核和用户态两种情况去继续处理.   
如果是地址空间内的地址, 则去判断权限等各种*乱七八糟*的情况, 然后继续处理.  

##请求调页

"请求调页"指的是一种动态内存分配技术, 它把页框的分配推迟到不能再推迟为止.  
请求调页背后的动机是: 进程开始运行的时候并不访问其地址空间中的全部地址, 事实上有一部分地址也许永远不被进程使用. 此外, 程序的局部性原理保证了程序执行的每个阶段, 真正引用的进程页只有一小部分, 因此临时用不着的页所在的页框可以由其他进程来使用.

---

#总结

实际上, 写完本文, 我并不清楚进程是如何分配到物理内存的 -_-!.  
我之前的文章"[linux 虚拟内存, 地址空间布局, page cache, ...](http://xanpeng.github.com/2012/03/01/buffer-cache/)"和本文相关度很高, 从中可以看出, 用户进程的虚址空间被划分为多个逻辑部分: stack, heap, data, text, bss等, 这些概念都是建立在 vma 之上的.
