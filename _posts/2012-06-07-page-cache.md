---
title: Linux page cache
layout: post
tags: memory page cache file block disk
category: linux
---

*本文是 ULK3 Ch15 页高速缓存的笔记.*  
"page cache"的中译是"页高速缓存", 下文都使用"page cache".

page cache是Linux内核所使用的主要磁盘高速缓存. 绝大多数情况下, 内核在读写磁盘时都使用page cache. 只有在O_DIRECT标志被置位时才会出现例外, 此时进程的I/O数据传送绕过page cache, 而使用了进程用户态地址空间的缓冲区.  
page cache中的每个页所包含的数据肯定属于某个文件, 这个文件(*更确切地说是文件的inode*)就称为页的owner.

page cache由"一页一页" 数据组成, 即数据单元是struct page, 这个结构就是"[Linux memory management](http://xanpeng.github.com/2012/05/31/linux-memory-management/)"和"[linux 虚拟内存, 地址空间布局, page cache, ...](http://xanpeng.github.com/2012/03/01/buffer-cache/)"中提到的"页", 它用来描述物理页框. 这是很好理解的, 所有操作都是在"内存+CPU"上进行的(*所有的?*).  
page cache中的页可能是如下类型:  
- 含有普通文件数据的页.  
- 含有目录的页.  
- 含有直接从块设备文件(跳过FS层)读出的数据的页.  
- 含有用户进程数据的页, 但页中的数据已经被交换到磁盘.  
- 属于特殊文件系统文件的页, 如共享内存的进程间通信所使用的特殊文件系统shm.  

如何识别page cache中的页: 一个页中包含的磁盘块在物理上不一定是相邻的, 因此不能用设备号和块号来标识, 而是要通过页的owner和 owner数据中的索引(通常是一个 inode 和在相应文件中的偏移量)来识别.

内核设计者实现page cache主要为了满足两种需要:  
- 快速定位含有给定相关数据的特定页, 即提供高速的搜索操作.  
- 记录在读写页中数据时应当如何处理page cache中的每个页, 如从普通文件, 块设备文件或交换区读一个数据页必须用不同的实现方式.

---

#address_space

page cache的核心数据结构是address_space对象, 它嵌入在inode结构中.  
如果page cache中的页的owner是一个文件, 那么inode.i_data=address_space对象, 而inode.i_mapping总是指向inode的数据页所有者的address_space对象, 所以此时i_mapping指向i_data.  
如果页中数据来自块设备文件, 那么address_space对象放在与块设备特殊文件系统bdev中对应文件的"主"索引节点中, 块设备描述符的bd_inode字段引用这个索引节点.

address_space.a_ops指向address_space_operations, 其中定义了处理页的各种方法, 最重要的方法是readpage,writepage,prepare_write,commit_write.

---

#基树

Linux支持大到TB级别的文件, 访问大文件时, page cache中可能充满太多的文件页, 以至于顺序扫描需要消耗大量时间. 为了实现page cache的高效查找, Linux 2.6使用了大量搜索树, 每个address_space对象对应一颗搜索树--称之为基树(radix tree), 树根由address_space.page_tree指向. 基树的每个节点可以有多到64个指针指向其他节点或页描述符, 叶子节点存放页描述符.

树本身结构没有什么可说的, 只要学过数据结构和算法, 对此很容易理解.  

这里需要特别提出有意思的一点: 还记得"[Linux memory management](http://xanpeng.github.com/2012/05/31/linux-memory-management/)"提到的分页系统吧, 一般地它将虚拟地址分成3部分解析, 最高10位表示全局目录, 用来确定页表, 接着中间的10位是用来表示一个页表中的偏移量, 最后12位是页内的偏移量. 基树使用类似的方法, 如果基树深度为1, 则用页索引(*页首的虚拟地址?*)的低6位表达slots数组的下标; 如果基树深度为2, 则将页索引的低12位分成两个6位的字段, 高位字段用以表示第一层节点的下标, 低位字段用以表示第二层节点的下标; 依此类推.

基树还针对某些特定查询作出优化. 如为了快速搜索脏页, 基树在每个中间节点包含一个针对每个孩子节点的脏标记, 只要有一个孩子节点设置了脏标记, 那么这个节点就设置脏标记. 这样去为搜索提供剪枝条件.

---

#page cache的处理函数

查找页: find_get_page(), find_get_pages().  
增加页: add_to_page_cache().  
删除页: remove_from_page_cache().  
更新页: read_cache_page().  

---

#将块放入page cache

当内核发现指定块的缓冲区所在的页不在page cache中时, 就分配一个新的块设备缓冲区页, 并调用函数grow_buffers()将它添加到page cache中.

当内核试图获得更多的空闲内存时, 就释放块设备缓冲区页.

当内核需要读/写一个单独的物理设备块时(如一个超级块), 必须检查所请求的块缓冲区是否已经在page cache中. 于是涉及如何在page cache中搜索指定的块缓冲区.

---

#把脏页写入磁盘

进程修改数据后, 相应的页就被标记为脏页, 即把它的PG_dirty标志置位.  

##pdflush

Unix系统允许把脏缓冲区写入块设备的操作延迟执行, 以提升性能. Linux 2.6使用一组通用内核线程**pdflush**系统地扫描page cache搜索脏页并写入磁盘.

pdflush线程的产生和消亡规则:  
- pdflush线程数目要求: [2, 8].  
- 如果最近的1s期间没有空闲的pdflush, 就应该创建新的pdflush.  
- 如果最近一次pdflush变为空闲的时间超过了1s, 就应该删除一个pdflush.  

#sync(), fsync()和fdatasync()
